# 🧪 Actividad 2.4: Login con JWT

!!! info "Práctica guiada"
    Hoy sustituyes HTTP Basic por JWT: login una vez, token en las peticiones siguientes. Es el mecanismo de autenticación final de tu GameVault.

## Qué vas a practicar

- Configurar el secreto JWT y los beans de codificación/decodificación.
- Generar un token con claims propios en el login.
- Cambiar la configuración de seguridad de HTTP Basic a JWT.

---

## Requisitos previos

Tus usuarios reales en PostgreSQL con BCrypt (Actividad 2.3).

---

## Paso 1 — El secreto y los beans JWT

En `application-dev.yaml`:

```yaml
gamevault:
  jwt:
    secret: "un-secreto-de-desarrollo-que-no-subiras-a-produccion"
    expiration-minutes: 60
```

!!! warning "Nunca subas un secreto real a un repositorio"
    Este valor es válido solo para desarrollo local. En un proyecto real, el secreto de producción no viaja en ningún fichero versionado en Git — se inyecta como variable de entorno o desde un gestor de secretos.

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

`JwtEncoder` firma tokens nuevos; `JwtDecoder` verifica los que llegan en cada petición — ambos usan el mismo secreto compartido. El `AuthenticationManager` es el bean que de verdad comprueba usuario/contraseña contra tu `GamevaultUserDetailsService`.

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
public record LoginRequestDTO(@NotBlank String username, @NotBlank String password) {}
public record LoginResponseDTO(String accessToken, String tokenType, long expiresInSeconds) {}
```

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

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

---

## Paso 4 — El cambio de modo completo

Sustituye, en tu `SecurityConfig`, el bloque de HTTP Basic por JWT:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
                    .anyRequest().authenticated()
            )
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

**Pregunta**: ¿qué pieza de tu configuración de la Actividad 2.2 desaparece exactamente en este paso, y qué la reemplaza?

---

## Paso 5 — Prueba del flujo completo

```bash
# 1. Login
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# 2. Copia el "accessToken" de la respuesta y guárdalo
TOKEN="pega-aqui-el-token"

# 3. Petición autenticada
curl -i http://localhost:8080/api/v1/videojuegos -H "Authorization: Bearer $TOKEN"

# 4. La misma petición, sin token
curl -i http://localhost:8080/api/v1/videojuegos
```

**Comprueba**: `200` con el token, y compara con el resultado sin él sobre una ruta protegida (si tienes alguna que exija autenticación siempre, pruébala en vez de `GET /videojuegos`, que es pública).

Decodifica el payload de tu propio token (cópialo, pégalo en [jwt.io](https://jwt.io) o decodifica la segunda parte con `base64 -d`) y **anota** los claims que ves.

---

## Mini-reto — `GET /api/v1/auth/me`

Añade un endpoint sencillo que devuelva quién eres según tu token actual:

```java
@GetMapping("/me")
public ResponseEntity<Map<String, Object>> me() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    // construye y devuelve un mapa (o un DTO propio) con el username y los roles
}
```

Pruébalo con tu token del Paso 5 y comprueba que la información coincide con lo que decodificaste a mano.

---

## Pregunta final

El payload de un JWT se puede decodificar sin conocer el secreto (lo acabas de hacer en jwt.io). ¿Por qué, entonces, el servidor confía en el token? Si alguien modificara a mano el claim `"roles"` de un token robado, cambiando `["USER"]` por `["ADMIN"]`, y volviera a mandarlo, ¿qué detectaría tu servidor?

---

## ✅ Cierre

Tu GameVault ya no reenvía credenciales en cada petición. En la próxima actividad completas la política de rutas y añades tests automatizados de seguridad.
