# 🧪 Actividad 2.2: Primera línea de defensa — HTTP Basic

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.2 — Primera línea de defensa: HTTP Basic](plantillas/Actividad_2_2_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — estado intermedio deliberado"
    Hoy añades Spring Security con usuarios en memoria y HTTP Basic. No es la versión final (eso llega en las Actividades 2.3 y 2.4) — es un paso intermedio para entender qué problema resuelve cada pieza.

## Qué vas a practicar

- Observar el efecto de Spring Security "sin configurar nada".
- Configurar usuarios en memoria y una primera política de rutas.
- Comprobar cómo viaja una credencial HTTP Basic, decodificándola tú mismo.
- Unificar el formato de un error que Spring Security genera por su cuenta, con un `AuthenticationEntryPoint` a medida.

---

## Requisitos previos

Tu GameVault con el CRUD de videojuegos y `GlobalExceptionHandler` (Actividad 2.1) funcionando.

---

## Paso 1 — Añadir Spring Security y ver el efecto inmediato

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Reinicia tu aplicación **sin añadir ninguna configuración propia todavía** y prueba:

```bash
curl -i http://localhost:8080/api/v1/videojuegos
```

**Predicción**: antes de ejecutar, ¿qué código de estado esperas? Escríbelo.

Deberías obtener un `401 Unauthorized` — incluso en un `GET` que hasta ahora era público.

**Captura**: la respuesta completa de este `curl -i`.

Comprueba también que Swagger UI (`/documentacion`) ha dejado de ser accesible sin autenticarte.

**Captura**: la pantalla de Swagger UI pidiendo credenciales (o el error al intentar cargarlo).

---

## Paso 2 — `SecurityConfig` inicial, con dos huecos por rellenar

Crea `SecurityConfig` en tu paquete `seguridad`. El usuario `user` y la ruta pública de `videojuegos` van dados, como referencia — el usuario `admin` y la ruta pública de `estudios` siguen exactamente el mismo patrón, y los completas tú:

```java
package com.tunombre.gamevault.seguridad;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withUsername("user")
                .password("{noop}user123")
                .roles("USER")
                .build();

        // TODO: crea aquí el usuario "admin", contraseña "admin123", rol "ADMIN" — mismo patrón que "user"

        return new InMemoryUserDetailsManager(user, admin);
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                        // TODO: añade aquí la misma regla, pero para los GET de estudio
                        .anyRequest().authenticated()
                )
                .csrf(AbstractHttpConfigurer::disable)
                .httpBasic(Customizer.withDefaults())
                .build();
    }
}
```

`{noop}` le dice a Spring Security que la contraseña no está cifrada (la contraseña se compara tal cual, sin BCrypt); `permitAll()` deja esa ruta abierta a peticiones sin autenticar; `anyRequest().authenticated()` exige estar autenticado para todo lo que no tenga ya una regla propia; `csrf(AbstractHttpConfigurer::disable)` desactiva la protección CSRF, ya explicada en la teoría; `httpBasic(Customizer.withDefaults())` activa HTTP Basic.

Reinicia y comprueba de nuevo el `GET` del Paso 1 — debería volver a funcionar sin credenciales, porque ahora está explícitamente marcado como público.

**Captura**: la respuesta de ese `GET`, ya en `200` otra vez sin mandar ninguna credencial.

---

## Paso 3 — Probar con curl y decodificar la cabecera

Sin credenciales, sobre una ruta protegida:

```bash
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" -d '{}'
```

**Comprueba**: `401`.

**Captura**: esta respuesta, sin credenciales, sobre una ruta protegida.

Con credenciales:

```bash
curl -i -u user:user123 http://localhost:8080/api/v1/videojuegos
```

**Comprueba**: `200`. Repite con `curl -v` (no `-i`) para ver la cabecera `Authorization` que `curl` ha generado por ti, y decodifícala a mano:

```bash
echo "dXNlcjp1c2VyMTIz" | base64 -d
```

(sustituye esa cadena por la que veas realmente en tu propia petición, tras `Basic `).

**Comprueba**: que el resultado es tu usuario y contraseña en texto legible — exactamente lo que has visto en la teoría.

**Captura**: la salida de `curl -v` con la cabecera `Authorization`, junto a la contraseña ya decodificada en tu terminal.

---

## Paso 4 — proteger la creación de videojuegos por rol

Repite el patrón de `requestMatchers(...)` ya mostrado en el Paso 2 para añadir una regla nueva: el `POST /api/v1/videojuegos` debe exigir el rol `ADMIN` específicamente (no basta con estar autenticado con cualquier usuario).

**Pista**: no uses `permitAll()` ni `authenticated()` esta vez — hace falta un método distinto, pensado para exigir un rol concreto: `hasRole("ADMIN")`. Tú construyes el `requestMatchers(...)` completo, con el verbo y la ruta correctos.

Prueba con los dos usuarios:

```bash
curl -i -u user:user123 -X POST http://localhost:8080/api/v1/videojuegos -H "Content-Type: application/json" -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
curl -i -u admin:admin123 -X POST http://localhost:8080/api/v1/videojuegos -H "Content-Type: application/json" -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: `403` con `user`, `201` con `admin`.

**Captura**: las dos respuestas, una junto a la otra.

**Fíjate**: el `403` tampoco tiene el formato de tu `ErrorResponse` — es la misma grieta que el `401`, y por el mismo motivo (lo genera Spring Security, no un `@ExceptionHandler`). En el Paso 5 la cierras para el `401`; la pieza equivalente para el `403` se llama `AccessDeniedHandler`, y la dejas pendiente para más adelante en el tema — no hace falta que la construyas todavía.

---

## Paso 5 — cerrando la grieta: un `AuthenticationEntryPoint` a medida

En la teoría has visto que el `401` de HTTP Basic no pasa por tu `GlobalExceptionHandler` — lo genera un filtro de seguridad, antes de que la petición llegue al `DispatcherServlet`, así que tu `@RestControllerAdvice` nunca llega a intervenir. Repite el `curl` del Paso 3 sin credenciales (el `POST` sobre una ruta protegida, no el `GET` del Paso 1 — ese ya es público desde el Paso 2) y compara el cuerpo de esa respuesta con el de cualquier error que ya gestiona tu `GlobalExceptionHandler` (por ejemplo, el `404` de un `id` inexistente): no se parecen en nada.

Un `@ExceptionHandler` no puede arreglar esto —nunca llega a ejecutarse—, pero Spring Security tiene su propia pieza para el mismo trabajo: un `AuthenticationEntryPoint`, que decide qué responder cuando una petición no autenticada llega a una ruta protegida. En tu paquete `seguridad`:

```java
package com.tunombre.gamevault.seguridad;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.tunombre.gamevault.exception.ErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.LocalDateTime;

@Component
public class ErrorResponseAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                          AuthenticationException authException) throws IOException {

        ErrorResponse error = new ErrorResponse(
                LocalDateTime.now().toString(), HttpStatus.UNAUTHORIZED.value(),
                "No autenticado", "Se requieren credenciales válidas para acceder a este recurso",
                request.getRequestURI()
        );

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(error));
    }
}
```

Reutiliza tu propio `ErrorResponse` de la Actividad 2.1 — el mismo formato de siempre, solo que construido a mano, porque aquí no hay ningún `@ExceptionHandler` que lo haga por ti.

Conéctalo en `securityFilterChain`, añadiendo el nuevo bean como parámetro (Spring lo inyecta solo, por tipo) y registrándolo en `exceptionHandling(...)`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, AuthenticationEntryPoint authenticationEntryPoint) throws Exception {
    return http
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/estudios/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
                    .anyRequest().authenticated()
            )
            .exceptionHandling(exceptions -> exceptions
                    .authenticationEntryPoint(authenticationEntryPoint)
            )
            .csrf(AbstractHttpConfigurer::disable)
            .httpBasic(Customizer.withDefaults())
            .build();
}
```

!!! tip "Por qué `.exceptionHandling(...)` y no `.httpBasic(basic -> basic.authenticationEntryPoint(...))`"
    Las dos formas funcionan hoy. Pero registrarlo dentro de `.httpBasic(...)` lo ataría a HTTP Basic — y en la Actividad 2.4 sustituyes `.httpBasic(...)` por JWT, con lo que esa línea desaparecería con él. `.exceptionHandling(...)` es un bloque aparte, independiente del mecanismo de autenticación activo: tu `ErrorResponseAuthenticationEntryPoint` sigue funcionando sin tocar una sola línea cuando cambies de Basic a JWT.

Repite el mismo `POST` sin credenciales del Paso 3 una vez más.

**Captura**: la respuesta del `401`, ya con el mismo formato que el resto de errores de tu API.

**Pregunta**: ¿por qué esta pieza tiene que implementarse como un `AuthenticationEntryPoint` de Spring Security, y no como un sexto `@ExceptionHandler` en tu `GlobalExceptionHandler`? Relaciónalo con el diagrama de la teoría que muestra dónde actúa el filtro respecto al `DispatcherServlet`.

---

## Pregunta final

Esta configuración tiene dos problemas serios para un proyecto real. Nómbralos con tus propias palabras, relacionando cada uno con algo concreto que hayas visto en esta actividad (uno tiene que ver con dónde viven `user`/`admin` en tu código; el otro con lo que has visto al decodificar la cabecera `Authorization`). ¿Qué actividad crees que resolverá cada uno, según lo que se ha adelantado en la teoría?

---

## ✅ Cierre

Tu API ya distingue quién puede hacer qué — pero con usuarios que se pierden al reiniciar y contraseñas casi en claro en cada petición. En la próxima actividad resuelves el primero: usuarios reales en PostgreSQL, con contraseñas protegidas por BCrypt.

---

## 🧭 Extra opcional — Swagger UI, tras el `AuthenticationEntryPoint`

!!! tip "No se entrega"
    Esta parte es solo para que lo veas con tus propios ojos. No hay captura ni pregunta que entregar — no forma parte de la plantilla.

Ahora que has completado el Paso 5, abre `/documentacion` en el navegador. Ya no te aparece ningún formulario de login: en su lugar ves directamente el JSON del `401` que devuelve tu `ErrorResponseAuthenticationEntryPoint`.

Es una consecuencia directa de lo que acabas de construir: tu clase no manda la cabecera `WWW-Authenticate` que hacía aparecer el cuadro de login nativo del navegador, así que el navegador ya no tiene ninguna pista para pedirte credenciales — ni aquí, ni en ninguna otra ruta protegida. De momento no hay forma cómoda de entrar a Swagger ni de autenticarte desde dentro; se resuelve más adelante en el tema, cuando Swagger tenga su propio botón "Authorize".
