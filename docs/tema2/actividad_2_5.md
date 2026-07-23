# 🧪 Actividad 2.5: Roles y rutas protegidas

!!! info "Práctica guiada"
    Hoy completas la política de autorización de tu GameVault, con las rutas propias que has ido añadiendo durante el curso incluidas, y la verificas con tests automatizados.

## Qué vas a practicar

- Completar una política de rutas cerrada por defecto.
- Depurar un caso real de ruta bloqueada por olvido.
- Cerrar un `500` mal formado en una ruta inexistente, con un handler específico.
- Implementar un `AccessDeniedHandler` a medida, para que el `403` tenga el mismo formato que el resto de tu API.
- Escribir tests de seguridad con login real.

---

## Requisitos previos

Tu login con JWT de la Actividad 2.4, y las rutas que has ido construyendo durante el curso: CRUD de `Videojuego`, `PUT`/`DELETE` de `Estudio` (Tema 1), el ranking de estudios (si ya lo tienes de Acceso a Datos).

---

## Paso 1 — Construir la tabla de política, y luego el código

Esta es la tabla objetivo — incluye tanto las rutas del catálogo base como las que **tú** has ido añadiendo por tu cuenta durante el curso:

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/register`, `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/videojuegos`, `/api/v1/estudios` (y sus `/{id}`) | GET | Cualquiera |
| `/api/v1/videojuegos` | POST/PUT/DELETE | Solo `ADMIN` |
| `/api/v1/estudios` | POST | Solo `ADMIN` |
| `/api/v1/estudios/{id}` | PUT, DELETE | ¿? *(decide tú)* |
| `/api/v1/estudios/ranking` | GET | ¿? *(decide tú, si ya la tienes)* |
| `/actuator/health` | GET | ¿? *(decide tú — piensa en quién necesita comprobar que el servicio está vivo)* |
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Todo lo demás | — | Nadie |

!!! tip "Las reseñas todavía no existen en tu proyecto"
    Si más adelante añades el módulo de reseñas (Acceso a Datos, Tema 3), vas a tener que volver a esta tabla y a `SecurityConfig` para añadir `/api/v1/videojuegos/*/reviews` (`POST`, `USER` o `ADMIN`) — hoy no existe todavía, así que no lo incluyas.

**Antes de escribir código**, decide y anota: el `PUT`/`DELETE` de `Estudio` que construiste en el Tema 1, ¿debería tener el mismo nivel de protección que el resto de escrituras del catálogo (`ADMIN`)? Justifica tu respuesta en una frase.

Ahora completa tu `SecurityConfig` regla a regla, siguiendo la tabla ya decidida:

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/register", "/api/v1/auth/login").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/estudios/**").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
        .requestMatchers(HttpMethod.PUT, "/api/v1/videojuegos/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/v1/videojuegos/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.POST, "/api/v1/estudios").hasRole("ADMIN")
        // añade aquí, siguiendo el mismo patrón, la regla que decidiste para Estudio PUT/DELETE
        .requestMatchers("/error").permitAll()
        .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/documentacion").permitAll()
        .anyRequest().denyAll()
)
```

---

## Paso 2 — Verificar `denyAll()`

Prueba una ruta que, a propósito, no está en tu lista (por ejemplo, si tuvieras un `GET /api/v1/algo-inventado`, o cualquier verbo sobre una ruta real que no hayas cubierto explícitamente):

```bash
curl -i http://localhost:8080/api/v1/algo-inventado
```

**Comprueba**: que la respuesta es un rechazo (no un `404` amable, sino el resultado de `denyAll()`). **Pregunta**: ¿por qué "cerrado por defecto" es la opción segura, comparado con "abierto por defecto, cerrado explícitamente donde haga falta"?

---

## Paso 3 — Depurar un caso real

!!! warning "Este fallo es intencionado — vas a diagnosticarlo tú"
    Con `denyAll()` activo, prueba ahora mismo: `GET /actuator/health` (el mismo endpoint que activaste en el Tema 1). Es muy probable que, si no lo has incluido explícitamente arriba, te dé un rechazo — aunque el endpoint en sí funcione perfectamente si lo pruebas de otra forma.

Activa el log de Spring Security si quieres ver más detalle de la decisión:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

**Diagnostica**: ¿por qué esta ruta, que funcionaba bien antes de completar la política, ahora está bloqueada? Añade la regla que falta (¿debería ser pública, como el resto de lecturas del catálogo, o requerir autenticación?) y comprueba que vuelve a funcionar.

---

## Paso 4 — Una ruta que no existe: por qué da `500`

Con un token de `admin` válido (Actividad 2.4, Paso 5), prueba una ruta que no existe pero que sí coincide con una regla que la permite:

```bash
curl -i http://localhost:8080/api/v1/videojuegos/esto-no-existe \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

**Comprueba**: en vez de un `404`, ves un `500 Error interno` — el mismo formato que generaría cualquier bug real de tu código, aunque aquí el único "problema" es que la ruta no existe.

**Captura**: esta respuesta, con el `500`.

Añade el handler que falta a tu `GlobalExceptionHandler` (Actividad 2.1), junto a los cinco que ya tenías:

```java
@ExceptionHandler(NoResourceFoundException.class)
public ResponseEntity<ErrorResponse> handleNoResourceFoundException(
        NoResourceFoundException ex, HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            LocalDateTime.now().toString(), 404, "Not Found",
            "El recurso solicitado no existe", request.getRequestURI()
    );
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
}
```

Repite la misma petición y comprueba que ahora da `404`, con el formato de tu `ErrorResponse`.

**Captura**: la misma petición, ya corregida.

---

## Paso 5 — `AccessDeniedHandler` a medida, para que el `403` tenga formato

Crea la clase, junto a tu `ErrorResponseAuthenticationEntryPoint` de la Actividad 2.2:

```java
@Component
public class ErrorResponseAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {

        ErrorResponse error = new ErrorResponse(
                LocalDateTime.now().toString(), HttpStatus.FORBIDDEN.value(),
                "No autorizado", "No tienes permisos suficientes para acceder a este recurso",
                request.getRequestURI()
        );

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(error));
    }
}
```

Añade el parámetro a tu `securityFilterChain` (Actividad 2.4, Paso 4) y regístrala junto al `AuthenticationEntryPoint` que ya tenías, en el mismo `.exceptionHandling(...)` —no toques nada más del método, solo estas dos líneas—:

```java
public SecurityFilterChain securityFilterChain(HttpSecurity http,
        AuthenticationEntryPoint authenticationEntryPoint,
        AccessDeniedHandler accessDeniedHandler) throws Exception {
    return http
            // ...
            .exceptionHandling(exceptions -> exceptions
                    .authenticationEntryPoint(authenticationEntryPoint)
                    .accessDeniedHandler(accessDeniedHandler)
            )
            // ...
}
```

Prueba de nuevo el `POST /api/v1/videojuegos` con el token de `user` (Actividad 2.4, Paso 5) y comprueba que el `403` ya tiene el formato de tu `ErrorResponse`.

**Captura**: esta respuesta.

---

## Paso 6 — Test de seguridad, guiado al completo

```java
private String login(String username, String password) throws Exception {
    String response = mockMvc.perform(post("/api/v1/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {"username": "%s", "password": "%s"}
                            """.formatted(username, password)))
            .andReturn().getResponse().getContentAsString();

    return JsonPath.read(response, "$.accessToken");
}

@Test
void crearVideojuego_DebeDevolver401_SinToken() throws Exception {
    mockMvc.perform(post("/api/v1/videojuegos")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{}"))
            .andExpect(status().isUnauthorized());
}

@Test
void crearVideojuego_DebeDevolver403_ConRolUser() throws Exception {
    String token = login("user", "user123");

    mockMvc.perform(post("/api/v1/videojuegos")
                    .header("Authorization", "Bearer " + token)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}
                            """))
            .andExpect(status().isForbidden());
}

@Test
void crearVideojuego_DebeDevolver201_ConRolAdmin() throws Exception {
    String token = login("admin", "admin123");

    mockMvc.perform(post("/api/v1/videojuegos")
                    .header("Authorization", "Bearer " + token)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                            {"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}
                            """))
            .andExpect(status().isCreated());
}

@Test
void rutaInexistente_DebeDevolver404() throws Exception {
    mockMvc.perform(get("/api/v1/videojuegos/esto-no-existe"))
            .andExpect(status().isNotFound());
}
```

Los cuatro tests cubren los tres casos de la tabla para una misma ruta —sin token (`401`), con rol insuficiente (`403`), con el rol correcto (éxito)— más el caso del Paso 4: una ruta que sencillamente no existe (`404`).

---

## Paso 7 — Repite el patrón con `DELETE` de `Estudio`

Escribe los mismos tres tests (sin token, con rol insuficiente si aplica según tu decisión del Paso 1, con el rol correcto) para el `DELETE` de `Estudio` que construiste en el Tema 1. Solo se indica qué cubrir — la estructura es idéntica a la del Paso 6.

---

## Cierre del tema

1. Deja tu tabla de política del Paso 1 como documento final, actualizada con las decisiones reales que tomaste (incluida la de `Estudio`).
2. Escribe un repaso propio (4-5 frases) de la evolución completa de este tema: validación y errores → HTTP Basic provisional → usuarios BCrypt en PostgreSQL → JWT → roles y política completa con tests. ¿Qué parte te parece, mirando atrás, la decisión de diseño más importante de todo el recorrido, y por qué?

---

## ✅ Cierre

Este JWT que acabas de terminar es el que reutilizarás en Acceso a Datos para el `PUT` de reseñas con control de autoría — el mismo token, la misma identidad, en otro módulo del mismo proyecto.
