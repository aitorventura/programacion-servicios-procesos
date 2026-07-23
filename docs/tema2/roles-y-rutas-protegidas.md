<a id="roles-y-rutas-protegidas"></a>

# 🧩 5. Roles, rutas protegidas y tests de seguridad

Completas la política de acceso ruta a ruta, cierras las últimas grietas de formato que quedaban abiertas desde apartados anteriores (`403` y `404` sin el formato de tu `ErrorResponse`), y aprendes a probar automáticamente que esa política hace exactamente lo que dice.

---

## 🗺️ La política de autorización, leída como tabla

El bloque `authorizeHttpRequests` de tu `SecurityConfig.java` define **toda** la política de acceso de la aplicación. Léelo como una tabla, no como código:

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/register`, `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/libros`, `/api/v1/editoriales` | GET | Cualquiera |
| `/api/v1/libros/*/resenas` | POST | `USER` o `ADMIN` |
| `/api/v1/libros`, `/api/v1/editoriales` | POST/PUT/DELETE | Solo `ADMIN` |
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Cualquier otra ruta no listada | — | **Nadie** |

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/register", "/api/v1/auth/login").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/libros", "/api/v1/libros/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/editoriales", "/api/v1/editoriales/**").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/libros/*/resenas").hasAnyRole("USER", "ADMIN")
        .requestMatchers(HttpMethod.POST, "/api/v1/libros").hasRole("ADMIN")
        .requestMatchers(HttpMethod.PUT, "/api/v1/libros/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/v1/libros/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.POST, "/api/v1/editoriales").hasRole("ADMIN")
        .requestMatchers("/error").permitAll()
        .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
        .anyRequest().denyAll()
)
```

La línea más importante de todo el bloque es la última: **`anyRequest().denyAll()`**. Cualquier ruta que no aparezca explícitamente en las reglas anteriores queda **cerrada por defecto** — es el principio de mínima exposición del primer apartado del tema, llevado hasta el final: nada se abre "por accidente" simplemente por existir.

!!! warning "Cada ruta nueva necesita su propia regla, o queda bloqueada"
    Si has ido añadiendo rutas propias durante el curso (el `PUT`/`DELETE` del Tema 1, el ranking de Acceso a Datos...) y no tienen una regla explícita en este bloque, `denyAll()` las bloqueará — aunque el endpoint en sí funcione perfectamente. Este es un error típico y real: "he probado mi endpoint nuevo y me da 403/401 sin motivo aparente" casi siempre significa "se me ha olvidado añadir su regla aquí".

---

## 🚦 401 vs. 403 vs. 404, con criterio

Tres códigos que se confunden con frecuencia, pero responden a preguntas distintas:

| Código | Significa | Cuándo aparece |
|---|---|---|
| `401 Unauthorized` | No sabes quién eres (no autenticado) | Falta el token, o es inválido/caducado. |
| `403 Forbidden` | Sabemos quién eres, pero no puedes hacer esto (no autorizado) | Token válido, pero rol insuficiente para esa regla concreta. |
| `404 Not Found` | La ruta pedida no corresponde a ningún endpoint real | La petición pasa tu política de seguridad (coincide con una regla que la permite), pero no hay ningún controller que la atienda. |

Con la tabla de arriba: un `POST /api/v1/libros` sin token da `401`; el mismo `POST` con el token de un usuario `USER` (no `ADMIN`) da `403`. El `404` es distinto a los otros dos: no depende de quién eres, sino de si ese endpoint existe de verdad.

### 🐛 Cuando el 404 no es un 404: `NoResourceFoundException`

Pruébalo tú mismo: con un token válido, pide una ruta que no existe pero que sí coincide con una regla que la permite —por ejemplo, `GET /api/v1/libros/esto-no-existe` (coincide con `/api/v1/libros/**`, que es pública). Deberías ver un `404`... pero lo que sale es un `500 Error interno`, generado por tu propio `GlobalExceptionHandler`.

La causa: cuando ninguna ruta de tu aplicación coincide con la petición, Spring lanza `NoResourceFoundException` — y como también es una `Exception`, el Handler 5 que construiste en el primer apartado del tema (`@ExceptionHandler(Exception.class)`, la "red de seguridad final") la atrapa antes de que Spring pueda darle su tratamiento por defecto, que sería un `404` correcto. Es exactamente el mismo problema que ese handler evitaba —una excepción sin capturar, filtrando un código erróneo— aplicado a un caso que nunca se había probado: una ruta que sencillamente no existe.

La solución es la misma de siempre: un handler más específico, que Spring prioriza automáticamente sobre el genérico, sin que tengas que tocar el orden de nada:

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

Añádelo a tu `GlobalExceptionHandler` de siempre, junto a los que ya tenías. Spring elige automáticamente el handler más específico para cada excepción —no el primero que encuentra ni el último—, así que este nuevo handler se antepone al genérico solo para `NoResourceFoundException`, sin afectar a ningún otro caso ya construido.

---

## 🩹 Cerrando la última grieta: `AccessDeniedHandler`

Queda una grieta más, del mismo tipo que ya cerraste con `AuthenticationEntryPoint` en la seguridad básica, pero para el otro lado de la moneda: el `403` de la tabla de arriba tampoco pasa por tu `GlobalExceptionHandler` —lo genera un filtro de seguridad, igual que el `401`—, así que sale con el formato por defecto de Spring, no con tu `ErrorResponse`.

Spring Security tiene, otra vez, su propia pieza para este trabajo: un **`AccessDeniedHandler`**, la contraparte exacta de `AuthenticationEntryPoint` —mismo patrón, mismo momento del ciclo de la petición, pero para "sí sé quién eres, no puedes hacer esto" en vez de "no sé quién eres":

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

Se registra en el mismo `.exceptionHandling(...)` donde ya tenías el `AuthenticationEntryPoint` — los dos conviven en el mismo bloque, cada uno resolviendo su propio caso:

```java
.exceptionHandling(exceptions -> exceptions
        .authenticationEntryPoint(customAuthenticationEntryPoint)
        .accessDeniedHandler(customAccessDeniedHandler)
)
```

Con esto, los tres códigos de la tabla de arriba —`401`, `403` y `404`— tienen ya el mismo formato `ErrorResponse` que el resto de tu API, sin excepciones sueltas por ningún sitio.

---

## 🧪 Tests de seguridad con MockMvc

Ya sabes construir tests MockMvc desde el Tema 1. Probar seguridad añade un matiz: necesitas un token real para las peticiones autenticadas. El patrón habitual es hacer un **login real** dentro del propio test, y reutilizar el token obtenido:

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
void crearLibro_DebeDevolver403_CuandoElRolNoEsSuficiente() throws Exception {
    String userToken = login("user", "user123");

    mockMvc.perform(post("/api/v1/libros")
                    .header("Authorization", "Bearer " + userToken)
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{}"))
            .andExpect(status().isForbidden());
}

@Test
void authMe_DebeDevolver401_CuandoNoHayToken() throws Exception {
    mockMvc.perform(get("/api/v1/auth/me"))
            .andExpect(status().isUnauthorized());
}
```

El método `login(...)` hace una petición real de login dentro del test, extrae el `accessToken` de la respuesta, y ese token se reutiliza en las peticiones siguientes con `.header("Authorization", "Bearer " + token)` — es exactamente el mismo flujo manual que ya practicaste con `curl`, pero automatizado y repetible.

---

## 📝 Documentación como buena práctica

Depurar **y documentar** van de la mano. La propia tabla de política de rutas que has visto al principio de este apartado ya es esa documentación — un documento vivo que cualquiera (tú dentro de seis meses, un compañero de equipo) puede consultar para saber exactamente qué puede hacer cada rol, sin tener que leer el código Java línea a línea. Mantener esa tabla al día, en un fichero aparte de tu propio proyecto (por ejemplo `docs/seguridad.md`), es exactamente el tipo de documentación que se espera de ti.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - La política de autorización se lee mejor como **tabla** (ruta × verbo × quién puede) que como código suelto.
    - `anyRequest().denyAll()` cierra por defecto cualquier ruta sin regla explícita — cada ruta nueva necesita su propia regla o queda bloqueada.
    - **401** = no sabemos quién eres; **403** = sabemos quién eres, pero no tienes permiso; **404** = la ruta no existe, y eso no depende de quién eres.
    - `NoResourceFoundException` (ruta inexistente) caía en el Handler 5 genérico del primer apartado del tema, dando `500` en vez de `404` — un handler específico lo arregla, sin tocar nada más: Spring prioriza el más concreto automáticamente.
    - `AccessDeniedHandler` es la contraparte de `AuthenticationEntryPoint` para el `403`: mismo patrón, registrado en el mismo `.exceptionHandling(...)`, para que autenticación y autorización tengan ambas el formato `ErrorResponse`.
    - Los tests de seguridad hacen un login real dentro del test y reutilizan el token obtenido en las peticiones siguientes.
    - La tabla de política de rutas es, en sí misma, esa documentación.
