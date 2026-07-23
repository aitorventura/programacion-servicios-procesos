# 🧪 Actividad 2.4: Login con JWT

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.4 — Login con JWT](plantillas/Actividad_2_4_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada"
    Hoy sustituyes HTTP Basic por JWT: login una vez, token en las peticiones siguientes. Es el mecanismo de autenticación final de tu GameVault.

## Qué vas a practicar

- Configurar el secreto JWT y los beans de codificación/decodificación.
- Generar un token con claims propios en el login.
- Cambiar la configuración de seguridad de HTTP Basic a JWT.
- Cerrar la grieta del `401` de login sin formato, con un sexto handler en tu `GlobalExceptionHandler`.
- Añadir un endpoint `GET /api/v1/auth/me` que exponga tu identidad actual a partir del token.
- Darle a Swagger un botón "Authorize" de verdad, con un `SecurityScheme`.

---

## Requisitos previos

Tus usuarios reales en PostgreSQL con BCrypt (Actividad 2.3).

---

## Paso 1 — El secreto y los beans JWT

Antes de nada, añade la dependencia que trae `JwtEncoder`, `JwtDecoder` y todo lo relacionado con JWT — no viene incluida en `spring-boot-starter-security`, es un starter aparte:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

Sin esta dependencia, ninguna de las clases de este apartado (`JwtEncoder`, `JwtDecoder`, `JwtAuthenticationConverter`...) existe en tu classpath, y el proyecto no compila. Actualiza el proyecto Maven tras añadirla (en tu IDE, o `mvn compile` desde terminal).

Ya tienes preparado, desde la Actividad 2.3, `application-dev-local.yml` (fuera de Git) y su plantilla `application-dev-local.yml.example`. Añade el secreto ahí, no en `application-dev.yml`:

```yaml
# application-dev-local.yml (NO se sube a Git)
gamevault:
  admin:
    password: admin123
  jwt:
    secret: un-secreto-largo-y-aleatorio-tuyo-para-desarrollo
```

```yaml
# application-dev-local.yml.example (SÍ se sube a Git — sin valores reales)
gamevault:
  admin:
    password: pon-aqui-tu-propia-contraseña
  jwt:
    secret: pon-aqui-tu-propio-secreto
```

En `application-dev.yml` (este sí se sube a Git) solo va lo que no es sensible:

```yaml
gamevault:
  jwt:
    expiration-minutes: 60
```

!!! warning "El secreto necesita al menos 32 caracteres"
    HS256 exige una clave de al menos 256 bits (32 caracteres) — con menos, Spring lanza una excepción al arrancar la aplicación. El del ejemplo de arriba ya cumple de sobra, pero si escribes el tuyo a mano, cuenta los caracteres. Una forma rápida de generar uno válido:
    ```bash
    openssl rand -base64 32
    ```
    Esto te da una cadena aleatoria de más de 32 caracteres, lista para pegar en `jwt.secret` — no hace falta que "signifique" nada, solo que sea larga e impredecible.

!!! warning "Nunca subas un secreto real a un repositorio"
    Un secreto de JWT filtrado permite falsificar tokens válidos para cualquier usuario — es un riesgo real, no una formalidad. Por eso vive en `application-dev-local.yml`, no en `application-dev.yml`: el mismo mecanismo que ya usaste para la contraseña del `admin` en la Actividad 2.3.


En tu `SecurityConfig`:

```java
@Value("${gamevault.jwt.secret}")
private String jwtSecret;

@Bean
public JwtEncoder jwtEncoder() {
    return new NimbusJwtEncoder(new ImmutableSecret<>(jwtSecret.getBytes(StandardCharsets.UTF_8)));
}

@Bean
public JwtDecoder jwtDecoder() {
    SecretKeySpec secretKey = new SecretKeySpec(jwtSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
    return NimbusJwtDecoder.withSecretKey(secretKey).macAlgorithm(MacAlgorithm.HS256).build();
}

@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
    return configuration.getAuthenticationManager();
}
```

`JwtEncoder` firma tokens nuevos; `JwtDecoder` verifica los que llegan en cada petición — ambos usan el mismo secreto compartido. El `AuthenticationManager` es el bean que de verdad comprueba usuario/contraseña contra tu `BdUserDetailsService`.

---

## Paso 2 — `JwtService`, guiado al completo

```java
package com.tunombre.gamevault.seguridad;

import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;

@Service
@RequiredArgsConstructor
public class JwtService {

    private final JwtEncoder jwtEncoder;

    @Value("${gamevault.jwt.expiration-minutes}")
    private long expirationMinutes;

    public String generarToken(Authentication authentication) {
        Instant ahora = Instant.now();
        List<String> roles = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .filter(a -> a.startsWith("ROLE_"))
                .map(a -> a.replace("ROLE_", ""))
                .toList();

        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("gamevault")
                .issuedAt(ahora)
                .expiresAt(ahora.plusSeconds(expirationMinutes * 60))
                .subject(authentication.getName())
                .claim("roles", roles)
                .build();

        JwsHeader jwsHeader = JwsHeader.with(MacAlgorithm.HS256).build();
        return jwtEncoder.encode(JwtEncoderParameters.from(jwsHeader, claims)).getTokenValue();
    }

    public long getExpiresInSeconds() {
        return expirationMinutes * 60;
    }
}
```

Cada claim tiene un porqué: `subject` es quién eres (lo que devolvió el login exitoso), `roles` viene de las autoridades ya calculadas por Spring Security a partir de tu `RolUsuario`, `issuedAt`/`expiresAt` fijan cuándo se emitió y cuándo caduca el token.

---

## Paso 3 — El endpoint de login

```java
package com.tunombre.gamevault.seguridad;

public record LoginRequestDTO(
        @NotBlank(message = "El nombre de usuario no puede estar vacío") String username,
        @NotBlank(message = "La contraseña no puede estar vacía") String password
) {}
public record LoginResponseDTO(String accessToken, String tokenType, long expiresInSeconds) {}
```

Tu `AuthController` ya existe desde la Actividad 2.3, con el endpoint `/register`. Añádele el login, con las dos dependencias nuevas:

```java
package com.tunombre.gamevault.seguridad;

@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequestDTO dto) {
        // el mismo método de la Actividad 2.3, sin cambios
    }

    @Operation(summary = "Autenticarse y obtener un token JWT")
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Login correcto, token generado"),
            @ApiResponse(responseCode = "400", description = "El cuerpo de la petición no supera las validaciones"),
            @ApiResponse(responseCode = "401", description = "Usuario o contraseña incorrectos")
    })
    @PostMapping("/login")
    public ResponseEntity<LoginResponseDTO> login(@Valid @RequestBody LoginRequestDTO dto) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(dto.username(), dto.password())
        );
        String token = jwtService.generarToken(authentication);
        return ResponseEntity.ok(new LoginResponseDTO(token, "Bearer", jwtService.getExpiresInSeconds()));
    }
}
```

**No olvides** abrir esta ruta en tu política de seguridad — es la única que debe ser pública sin excepción:

```java
.requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
```

(la regla de `/api/v1/auth/register` ya la tenías de la Actividad 2.3 — no hace falta tocarla).

Añade también un sexto handler a tu `GlobalExceptionHandler` (Actividad 2.1), junto a los otros cinco — es la primera vez en el proyecto que una excepción de autenticación se lanza dentro de un controller, así que hasta ahora no hacía falta:

```java
@ExceptionHandler(AuthenticationException.class)
public ResponseEntity<ErrorResponse> handleAuthenticationException(
        AuthenticationException ex, HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            LocalDateTime.now().toString(), 401, "No autenticado",
            "Usuario o contraseña incorrectos", request.getRequestURI()
    );
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(response);
}
```

Pruébalo con una contraseña incorrecta:

```bash
curl -i -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"contraseña-mala"}'
```

**Comprueba**: `401`, con el mismo formato `ErrorResponse` que el resto de tu API — no la traza de Spring por defecto.

**Captura**: esta respuesta.

---

## Paso 4 — El cambio de modo completo

Sustituye tu método `securityFilterChain` completo por este —incluye todo lo que ya tenías desde la Actividad 2.2 (rutas, `exceptionHandling` con tu `AuthenticationEntryPoint`, `csrf` desactivado) más lo nuevo de hoy; si tu proyecto tiene alguna regla más, propia tuya, consérvala igual—:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, AuthenticationEntryPoint authenticationEntryPoint) throws Exception {
    return http
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(HttpMethod.POST, "/api/v1/auth/register", "/api/v1/auth/login").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/estudios/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
                    .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/documentacion").permitAll()
                    .anyRequest().authenticated()
            )
            .exceptionHandling(exceptions -> exceptions
                    .authenticationEntryPoint(authenticationEntryPoint)
            )
            .csrf(AbstractHttpConfigurer::disable)
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())))
            .httpBasic(AbstractHttpConfigurer::disable)
            .build();
}

@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthoritiesClaimName("roles");
    authoritiesConverter.setAuthorityPrefix("ROLE_");

    JwtAuthenticationConverter authenticationConverter = new JwtAuthenticationConverter();
    authenticationConverter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
    return authenticationConverter;
}
```

`jwtAuthenticationConverter()` es la pieza que traduce el claim `"roles"` de tu token (una lista de strings como `["ADMIN"]`) en autoridades de Spring Security con el prefijo `ROLE_` — el mismo formato que ya usaban tus reglas `hasRole("ADMIN")` con HTTP Basic, así que no tienes que tocar esas reglas.

Fíjate en la regla nueva de `/v3/api-docs/**`, `/swagger-ui/**` y `/documentacion`: desde la Actividad 2.2, Swagger ha estado bloqueado detrás de autenticación, igual que el resto de la API. Sin esta regla, el navegador no podría ni cargar `/documentacion` —STATELESS no manda ningún token solo por navegar a una URL—, y el Paso 7 (el botón "Authorize") no sería alcanzable. La añades ahora porque es la primera vez que necesitas Swagger accesible sin login previo.

**Pregunta**: ¿qué pieza de tu configuración de la Actividad 2.2 desaparece exactamente en este paso, y qué la reemplaza?

---

## Paso 5 — Prueba del flujo completo

`GET /api/v1/videojuegos` es pública (`permitAll()`) — con o sin token, siempre da `200`, así que no sirve para comprobar que la autenticación funciona de verdad. Usa en su lugar `POST /api/v1/videojuegos`, que exige rol `ADMIN`, y compara tres situaciones: con el token correcto, sin ningún token, y con el token de un usuario sin ese rol.

!!! tip "¿Ya no existe el usuario `user` de la Actividad 2.3?"
    Si has reiniciado la base de datos entre medias, vuelve a crearlo antes de seguir:
    ```bash
    curl -i -X POST http://localhost:8080/api/v1/auth/register \
      -H "Content-Type: application/json" \
      -d '{"username":"user","password":"user123"}'
    ```

```bash
# 1. Login como admin
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# 2. Copia el "accessToken" de la respuesta y guárdalo
ADMIN_TOKEN="pega-aqui-el-token-de-admin"

# 3. POST protegido, con el token de admin
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Celeste","precio":19.99,"fechaLanzamiento":"2018-01-25","estudioId":1}'

# 4. La misma petición, sin token
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Celeste","precio":19.99,"fechaLanzamiento":"2018-01-25","estudioId":1}'

# 5. Login como user (el que registraste en la Actividad 2.3) y repite la petición con su token
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"user","password":"user123"}'

USER_TOKEN="pega-aqui-el-token-de-user"

curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Celeste","precio":19.99,"fechaLanzamiento":"2018-01-25","estudioId":1}'
```

**Comprueba**: `201` con el token de `admin`; `401` sin token (el servidor no sabe quién eres); `403` con el token de `user` (sabe quién eres, pero no tienes el rol que exige la ruta). Tres respuestas distintas para tres situaciones distintas, con la misma petición y solo las credenciales cambiando.

!!! note "El 403 no tiene, todavía, el formato de tu `ErrorResponse`"
    A diferencia del `401`, que ya formatea tu `AuthenticationEntryPoint`, este `403` sale con el formato por defecto de Spring Security — es la misma grieta que el `401` tenía antes de la Actividad 2.2, pero para autorización en vez de autenticación. Se cierra más adelante en el tema, con un `AccessDeniedHandler` a medida.

**Captura**: las tres respuestas (con token de `admin`, sin token, con token de `user`), una junto a la otra.

Decodifica el payload del token de `admin` (cópialo, pégalo en [jwt.io](https://jwt.io) o decodifica la segunda parte con `base64 -d`) y **anota** los claims que ves.

**Captura**: el payload ya decodificado, con los claims a la vista.

---

## Paso 6 — `GET /api/v1/auth/me`

Añade un endpoint sencillo que devuelva quién eres según tu token actual, documentado con `@Operation`/`@ApiResponses` igual que el resto de tu API. Define primero el DTO de respuesta, con el mismo patrón `record` que ya conoces:

```java
public record AuthMeResponse(String username, List<String> roles) {}
```

```java
@GetMapping("/me")
public ResponseEntity<AuthMeResponse> me() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

    List<String> roles = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .filter(a -> a.startsWith("ROLE_"))
            .map(a -> a.replace("ROLE_", ""))
            .toList();

    return ResponseEntity.ok(new AuthMeResponse(authentication.getName(), roles));
}
```

El `.filter()`/`.map()` para quitar el prefijo `ROLE_` es el mismo que ya usaste en `JwtService` (Paso 2) — hace falta repetirlo aquí para que los roles que ves en `/me` coincidan exactamente con los que decodificaste del JWT (que tampoco llevan el prefijo).

Pruébalo con el token de `admin` del Paso 5 y comprueba que la información coincide con lo que decodificaste a mano.

**Captura**: la respuesta de `GET /api/v1/auth/me`.

---

## Paso 7 — Un botón "Authorize" de verdad en Swagger

Añade el esquema de seguridad a tu `OpenApiConfig` (Actividad 1.2 de PSP):

```java
@Bean
public OpenAPI gamevaultOpenAPI() {
    final String esquema = "bearerAuth";
    return new OpenAPI()
            .info(new Info()
                    .title("Mi GameVault")
                    .version("v1")
                    .description("API de mi propio GameVault, curso 2026/2027."))
            .components(new Components()
                    .addSecuritySchemes(esquema, new SecurityScheme()
                            .type(SecurityScheme.Type.HTTP)
                            .scheme("bearer")
                            .bearerFormat("JWT")))
            .addSecurityItem(new SecurityRequirement().addList(esquema));
}
```

Reinicia, entra en `/documentacion` y pulsa el botón "Authorize" (ahora sí debería aparecer). Pega el token de `admin` del Paso 5 —sin escribir `Bearer ` delante, Swagger lo añade solo— y prueba algún endpoint protegido con "Try it out".

**Comprueba**: que ya no hace falta pegar la cabecera `Authorization` a mano en cada petición — Swagger la añade sola a partir de ahora.

**Captura**: el diálogo de "Authorize" con tu token ya pegado, y la respuesta de un "Try it out" sobre un endpoint protegido.

---

## Pregunta final

El payload de un JWT se puede decodificar sin conocer el secreto (lo acabas de hacer en jwt.io). ¿Por qué, entonces, el servidor confía en el token? Si alguien modificara a mano el claim `"roles"` de un token robado, cambiando `["USER"]` por `["ADMIN"]`, y volviera a mandarlo, ¿qué detectaría tu servidor?

---

## ✅ Cierre

Tu GameVault ya no reenvía credenciales en cada petición. En la próxima actividad completas la política de rutas y añades tests automatizados de seguridad.
