<a id="roles-y-rutas-protegidas"></a>

# 🧩 5. Roles y rutas protegidas

Al terminar el apartado anterior, tu API ya distinguía quién eres (JWT) de qué puedes hacer (roles) — pero solo lo habías comprobado sobre una ruta suelta, `POST /videojuegos`, y ese `403` seguía saliendo con el formato por defecto de Spring, no con tu `ErrorResponse`. Hoy conviertes esa comprobación puntual en una política completa, ruta a ruta y cerrada por defecto, y cierras ese formato pendiente —y el de un `404` que en realidad sale disfrazado de `500`—.

---

## 🗺️ La política de autorización, leída como tabla

El bloque `authorizeHttpRequests` de tu `SecurityConfig.java` define **toda** la política de acceso de la aplicación. Léelo como una tabla, no como código:

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/register`, `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/auth/me` | GET | Cualquiera **autenticado** — cualquier rol vale, no hace falta `ADMIN` |
| `/api/v1/libros`, `/api/v1/editoriales` | GET | Cualquiera |
| `/api/v1/libros`, `/api/v1/editoriales` | POST/PUT/DELETE | Solo `ADMIN` |
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Cualquier otra ruta no listada | — | **Nadie** |

Es un ejemplo con el dominio de siempre (`libro`/`editorial`) — no un catálogo cerrado de casos. Cuando lleves esto a tu propio proyecto en la actividad, verás la tabla real, con todas tus rutas.

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/register", "/api/v1/auth/login").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/auth/me").authenticated()
        .requestMatchers(HttpMethod.GET, "/api/v1/libros", "/api/v1/libros/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/editoriales", "/api/v1/editoriales/**").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/libros").hasRole("ADMIN")
        .requestMatchers(HttpMethod.PUT, "/api/v1/libros/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/v1/libros/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.POST, "/api/v1/editoriales").hasRole("ADMIN")
        .requestMatchers(HttpMethod.PUT, "/api/v1/editoriales/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/v1/editoriales/*").hasRole("ADMIN")
        .requestMatchers("/error").permitAll()
        .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
        .anyRequest().denyAll()
)
```

`/error` es la ruta interna que usa el propio Spring Boot para construir la respuesta cuando una petición termina en un error que no ha llegado a capturar ninguno de tus `@ExceptionHandler` (un fallo a nivel de servlet, por ejemplo): Spring reenvía la petición, por dentro, a `/error`. Si esa ruta no es pública, ese reenvío también pasa por tu política de seguridad — y puedes acabar viendo un `401`/`403` de tu propia `denyAll()` en vez del error real que se suponía que ibas a ver. Por eso va siempre en `permitAll()`, junto a las rutas de Swagger.

La línea más importante de todo el bloque es la última: **`anyRequest().denyAll()`**. Cualquier ruta que no aparezca explícitamente en las reglas anteriores queda **cerrada por defecto** — es el principio de mínima exposición del primer apartado del tema, llevado hasta el final: nada se abre "por accidente" simplemente por existir.

!!! info "La tabla de arriba no es una ley universal — es una decisión de negocio"
    Que `POST`/`PUT`/`DELETE` sobre `/api/v1/libros` sean solo para `ADMIN` no es una regla fija de Spring Security ni una convención que debas copiar siempre: es la decisión correcta **para este dominio**, donde el catálogo lo gestiona el equipo del videoclub, no los usuarios. No hay una tabla de verbo → rol que valga para cualquier API. Piensa en una red social: ahí un `DELETE` sobre una publicación normalmente sí lo puede hacer un `USER`, siempre que sea el autor de esa publicación —algo que ni siquiera se resuelve con un rol, sino comprobando de quién es el recurso—. La pregunta que de verdad importa, ruta a ruta, no es "¿qué verbo es?", sino "¿qué tiene sentido para lo que hace este endpoint, en esta aplicación concreta?". Es exactamente el mismo razonamiento que vas a aplicar tú mismo en la actividad, al decidir toda la política de `EstudioController` — apoyándote en la que ya está decidida para `Videojuego`, pero sin copiarla a ciegas donde no tenga sentido.

!!! warning "Cada ruta nueva necesita su propia regla, o `denyAll()` la bloquea"
    Con `.anyRequest().authenticated()` como tenías hasta ahora, cualquier ruta sin regla propia (como `/api/v1/auth/me`) funcionaba igual, con solo estar autenticado. Al cambiar el catch-all a `denyAll()`, eso deja de ser cierto: lo que no tenga su línea explícita queda bloqueado, aunque el endpoint funcione perfectamente y mandes un token válido — "me da 403/401 sin motivo aparente" casi siempre significa "me he dejado esta ruta sin regla". Ojo también con los verbos de escritura sobre **subrutas de una acción concreta** (`POST /libros/5/aplicar-descuento`): no coinciden con la regla de la ruta base (`/api/v1/libros`) y necesitan la suya propia. En la actividad revisas tu propio proyecto ruta a ruta para no dejarte ninguna.

---

## 🚦 401 vs. 403 vs. 404, con criterio

Tres códigos que se confunden con frecuencia, pero responden a preguntas distintas:

| Código | Significa | Cuándo aparece |
|---|---|---|
| `401 Unauthorized` | No sabes quién eres (no autenticado) | Falta el token, o es inválido/caducado. |
| `403 Forbidden` | Sabemos quién eres, pero no puedes hacer esto (no autorizado) | Token válido, pero rol insuficiente para esa regla concreta. |
| `404 Not Found` | La ruta pedida no corresponde a ningún endpoint real | La petición pasa tu política de seguridad (coincide con una regla que la permite), pero no hay ningún controller que la atienda. |

Con la tabla de arriba: un `POST /api/v1/libros` sin token da `401`; el mismo `POST` con el token de un usuario `USER` (no `ADMIN`) da `403`. El `404` es distinto a los otros dos: no depende de quién eres, sino de si ese endpoint existe de verdad.

!!! note "`denyAll()` sin token también da `401`, no `403`"
    `denyAll()` rechaza siempre cualquier petición que llegue hasta esa regla. Después, Spring Security distingue quién ha realizado la petición: si no existe una autenticación válida, `ExceptionTranslationFilter` llama al `AuthenticationEntryPoint` y devuelve `401`; si la petición lleva un token válido, delega en el `AccessDeniedHandler` y devuelve `403`.

    Por tanto, la misma regla `denyAll()` puede terminar en uno u otro código: la diferencia no está en la regla, sino en si el solicitante está autenticado.

### 🐛 Cuando el 404 no es un 404: `NoResourceFoundException`

Pruébalo tú mismo: con un token válido, pide una ruta que no existe pero que sí coincide con una regla que la permite —por ejemplo, `GET /api/v1/libros/999/no-existe` (dos segmentos tras la base: no coincide con ningún endpoint real, pero sí cae dentro de `/api/v1/libros/**`, que es pública). Deberías ver un `404`... pero lo que sale es un `500 Error interno`, generado por tu propio `GlobalExceptionHandler`.

!!! warning "No cualquier ruta 'inexistente' sirve para este ejemplo"
    Si en vez de dos segmentos pruebas con uno solo —por ejemplo, `GET /api/v1/libros/esto-no-existe`—, esa ruta **sí coincide** con `GET /api/v1/libros/{id}`: Spring intenta convertir `"esto-no-existe"` al `Long id` del método, y lanza una excepción de conversión de tipo distinta (`MethodArgumentTypeMismatchException`), no `NoResourceFoundException`. Para provocar de verdad una ruta que no existe necesitas algo que no coincida con ningún patrón de tu controller, ni siquiera por accidente — de ahí los dos segmentos del ejemplo de arriba.

La causa: cuando ninguna ruta de tu aplicación coincide con la petición, Spring lanza `NoResourceFoundException` — y como también es una `Exception`, el Handler 5 que has construido en el primer apartado del tema (`@ExceptionHandler(Exception.class)`, la "red de seguridad final") la atrapa antes de que Spring pueda darle su tratamiento por defecto, que sería un `404` correcto. Es exactamente el mismo problema que ese handler evitaba —una excepción sin capturar, filtrando un código erróneo— aplicado a un caso que nunca se había probado: una ruta que sencillamente no existe.

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

### 🐛 Un id que no es un id: `MethodArgumentTypeMismatchException`

El aviso de arriba ya lo ha adelantado: hay otra forma de colarse un `500` donde debería haber un código más preciso. Pruébalo tú mismo: `GET /api/v1/libros/no-es-un-numero` (un solo segmento, coincide con `GET /api/v1/libros/{id}`). Deberías ver un `400` —el id no tiene el formato correcto, es un problema de la petición, no del servidor—, pero lo que sale es otra vez un `500 Error interno`.

La causa es la misma estructura de siempre, con un disparador distinto: Spring intenta convertir el segmento de la URL al tipo que espera el parámetro (`Long id`), no puede, y lanza `MethodArgumentTypeMismatchException`. Como tampoco tiene handler propio, cae en el `Handler 5` genérico — la excepción sin capturar, filtrando otra vez un código erróneo.

La solución, el mismo patrón exacto:

```java
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<ErrorResponse> handleMethodArgumentTypeMismatchException(
        MethodArgumentTypeMismatchException ex, HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            LocalDateTime.now().toString(), 400, "Parámetro inválido",
            "El valor '%s' no es válido para el parámetro '%s'".formatted(ex.getValue(), ex.getName()),
            request.getRequestURI()
    );
    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
}
```

`ex.getValue()` te da el valor que ha llegado y `ex.getName()` el nombre del parámetro — con eso, el mensaje dice exactamente qué ha fallado, sin que tengas que escribir uno genérico ni adivinarlo.

---

## 🩹 Cerrando la última grieta: `AccessDeniedHandler`

Queda una grieta más, del mismo tipo que ya has cerrado con `AuthenticationEntryPoint` en la seguridad básica, pero para el otro lado de la moneda: el `403` de la tabla de arriba tampoco pasa por tu `GlobalExceptionHandler` —lo genera un filtro de seguridad, igual que el `401`—, así que sale con el formato por defecto de Spring, no con tu `ErrorResponse`.

Spring Security tiene, otra vez, su propia pieza para este trabajo: un **`AccessDeniedHandler`**, la contraparte exacta de `AuthenticationEntryPoint` —mismo patrón, mismo momento del ciclo de la petición, pero para "sí sé quién eres, no puedes hacer esto" en vez de "no sé quién eres":

```java
@Component
@RequiredArgsConstructor
public class ErrorResponseAccessDeniedHandler implements AccessDeniedHandler {

    private final JsonMapper jsonMapper;

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
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(jsonMapper.writeValueAsString(error));
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

## 📝 Documentación como buena práctica

Depurar **y documentar** van de la mano. La propia tabla de política de rutas que has visto al principio de este apartado ya es esa documentación — un documento vivo que cualquiera (tú dentro de seis meses, un compañero de equipo) puede consultar para saber exactamente qué puede hacer cada rol, sin tener que leer el código Java línea a línea. Mantener esa tabla al día, en un fichero aparte de tu propio proyecto (por ejemplo `docs/seguridad.md`), es exactamente el tipo de documentación que se espera de ti.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - La política de autorización se lee mejor como **tabla** (ruta × verbo × quién puede) que como código suelto.
    - No existe una tabla verbo → rol universal: "solo `ADMIN` puede borrar" es correcto para este catálogo, pero no para cualquier API (en una red social, un `USER` sí puede borrar sus propias publicaciones). Cada ruta se decide según lo que hace y para quién existe, no por convención.
    - `anyRequest().denyAll()` cierra por defecto cualquier ruta sin regla explícita — cada ruta nueva necesita su propia regla o queda bloqueada, incluidas rutas que ya funcionaban con `authenticated()` (como `/auth/me`) y `POST` sobre subrutas de acciones concretas (`/libros/5/aplicar-descuento`), que no coinciden con la regla de la ruta exacta.
    - **401** = no sabemos quién eres; **403** = sabemos quién eres, pero no tienes permiso; **404** = la ruta no existe, y eso no depende de quién eres. Ojo: `denyAll()` sin ningún token también da `401`, no `403` — el `403` solo aparece cuando ya hay un token válido de por medio.
    - `NoResourceFoundException` (ruta inexistente) caía en el Handler 5 genérico del primer apartado del tema, dando `500` en vez de `404` — un handler específico lo arregla, sin tocar nada más: Spring prioriza el más concreto automáticamente. `MethodArgumentTypeMismatchException` (un id con el tipo equivocado, como `/libros/no-es-un-numero`) es el mismo problema con otro disparador — mismo patrón, `400` en vez de `500`.
    - `AccessDeniedHandler` es la contraparte de `AuthenticationEntryPoint` para el `403`: mismo patrón, registrado en el mismo `.exceptionHandling(...)`, para que autenticación y autorización tengan ambas el formato `ErrorResponse`.
    - La tabla de política de rutas es, en sí misma, esa documentación.
