<a id="roles-y-rutas-protegidas"></a>

# 🧩 5. Roles, rutas protegidas y tests de seguridad

Completas la política de acceso ruta a ruta, y aprendes a probar automáticamente que esa política hace exactamente lo que dice.

---

## 🗺️ La política de autorización, leída como tabla

El bloque `authorizeHttpRequests` de tu `SecurityConfig.java` define **toda** la política de acceso de la aplicación. Léelo como una tabla, no como código:

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/libros`, `/api/v1/editoriales` | GET | Cualquiera |
| `/api/v1/libros/*/resenas` | POST | `USER` o `ADMIN` |
| `/api/v1/libros`, `/api/v1/editoriales` | POST/PUT/DELETE | Solo `ADMIN` |
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Cualquier otra ruta no listada | — | **Nadie** |

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
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

## 🚦 401 vs. 403, con criterio

Dos códigos que se confunden con frecuencia, pero responden a preguntas distintas:

| Código | Significa | Cuándo aparece |
|---|---|---|
| `401 Unauthorized` | No sabes quién eres (no autenticado) | Falta el token, o es inválido/caducado. |
| `403 Forbidden` | Sabemos quién eres, pero no puedes hacer esto (no autorizado) | Token válido, pero rol insuficiente para esa regla concreta. |

Con la tabla de arriba: un `POST /api/v1/libros` sin token da `401`; el mismo `POST` con el token de un usuario `USER` (no `ADMIN`) da `403`.

<!-- NOTA PARA DESARROLLO FUTURO: Este 403 tiene el mismo problema de formato que ya se resolvió para el 401 en "Seguridad básica: usuarios en memoria y HTTP Basic" (Actividad 2.2, AuthenticationEntryPoint a medida que reutiliza ErrorResponse). El 403 lo genera Spring Security por su cuenta (vía AccessDeniedHandler por defecto), fuera de GlobalExceptionHandler, por la misma razón: es un filtro/pieza de seguridad, no pasa por el DispatcherServlet ni por @RestControllerAdvice. Cuando se desarrolle este apartado a fondo, añadir una explicación teórica de AccessDeniedHandler (mismo patrón que AuthenticationEntryPoint, pero para peticiones YA autenticadas sin el rol necesario) y su implementación correspondiente en la actividad emparejada, registrándolo en exceptionHandling(...).accessDeniedHandler(...) del SecurityConfig. Borrar este comentario cuando se implemente. -->

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
    - **401** = no sabemos quién eres; **403** = sabemos quién eres, pero no tienes permiso.
    - Los tests de seguridad hacen un login real dentro del test y reutilizan el token obtenido en las peticiones siguientes.
    - La tabla de política de rutas es, en sí misma, esa documentación.
