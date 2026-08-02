# 🧪 Actividad 2.5: Roles y rutas protegidas

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.5 — Roles y rutas protegidas](plantillas/Actividad_2_5_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada"
    Hoy completas la política de autorización de tu GameVault, con las rutas propias que has ido añadiendo durante el curso incluidas, y la depuras y documentas hasta dejarla cerrada.

## Qué vas a practicar

- Completar una política de rutas cerrada por defecto.
- Depurar un caso real de ruta bloqueada por olvido.
- Cerrar un `500` mal formado en una ruta inexistente, con un handler específico.
- Cerrar un segundo `500` mal formado, esta vez por un id con el tipo equivocado, con otro handler específico.
- Implementar un `AccessDeniedHandler` a medida, para que el `403` tenga el mismo formato que el resto de tu API.

---

## Requisitos previos

Tu login con JWT de la Actividad 2.4, el CRUD completo de `Videojuego` y de `Estudio` (Acceso a Datos, Tema 1), y cualquier ruta propia que hayas ido añadiendo por tu cuenta: filtros, consultas nativas sobre JSONB, el descuento por estudio, el ajuste de precio, el ranking...

---

## Paso 1 — Construir la tabla de política, y luego el código

Esta es la tabla objetivo — incluye tanto las rutas del catálogo base como las que **tú** has ido añadiendo por tu cuenta durante el curso. Agrupada por controller, para que sea más fácil de leer y de contrastar con tu propio código:

**`AuthController`**

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/register`, `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/auth/me` | GET | Cualquiera **autenticado** |

**`VideojuegoController`**

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/videojuegos` (y toda subruta `GET`: filtros, búsquedas, consultas nativas...) | GET | Cualquiera |
| `/api/v1/videojuegos` | POST | Solo `ADMIN` |
| `/api/v1/videojuegos/{id}` | PUT, DELETE | Solo `ADMIN` |
| `/api/v1/videojuegos/descuento/{estudioId}` | POST | Solo `ADMIN` |

**`EstudioController`** — *esta la decides tú entera, basándote en la lógica que ya has aplicado en `VideojuegoController`*

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/estudios` (y toda subruta `GET`: `/{id}`, `ranking`, `ranking-con-posicion`...) | GET | ¿? |
| `/api/v1/estudios` | POST | ¿? |
| `/api/v1/estudios/{id}` | PUT, DELETE | ¿? |
| `/api/v1/estudios/{id}/ajustar-precio` | POST | ¿? |

**Infraestructura y documentación**

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/actuator/health` | GET | ¿? *(decide tú — piensa en quién necesita comprobar que el servicio está vivo)* |
| `/documentacion`, `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Todo lo demás | — | Nadie |

!!! danger "Revisa tus propios controllers antes de dar la tabla por completa"
    Esta tabla recoge las rutas del catálogo base y de `/auth`, pero **tu proyecto puede tener más** — cualquier endpoint que hayas ido añadiendo por tu cuenta (un buscador, un top de novedades, una acción sobre un recurso concreto...) necesita su propia fila, o `denyAll()` lo bloqueará sin avisar. Antes de escribir el código, repasa `VideojuegoController` y `EstudioController` método a método y comprueba que cada uno tiene un sitio en esta tabla.

    No todos los métodos necesitan una fila **propia**: los filtros, búsquedas y consultas nativas (JSONB incluido) son `GET` bajo la misma ruta base, así que ya quedan cubiertos por `.requestMatchers(HttpMethod.GET, ".../**").permitAll()` — no hace falta una regla por cada uno. Presta especial atención, en cambio, a los `POST` sobre una acción concreta (como `descuento` o `ajustar-precio` arriba): esos sí necesitan su propia regla, porque no coinciden con la de la ruta base.

**Antes de escribir código**, decide y anota las cuatro filas de `EstudioController`. `Estudio` ya tiene el CRUD completo desde Acceso a Datos, con la misma forma que `Videojuego` — así que la pregunta no es "¿qué hace cada endpoint?" (eso ya lo sabes), sino "¿tiene sentido tratarlo igual que su equivalente en `Videojuego`, o hay una razón real para tratarlo distinto?". No hay una respuesta fija: la lógica de `Videojuego` es tu punto de partida (mantener el mismo criterio en toda la app, salvo que tengas un motivo concreto para lo contrario, es en sí mismo una buena decisión de diseño), pero la decisión y la justificación son tuyas. Justifica cada fila en una frase — con especial atención a `PUT`/`DELETE`, donde de verdad puede haber más de una respuesta razonable.

Ahora completa tu `SecurityConfig` regla a regla, siguiendo la tabla ya decidida:

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/register", "/api/v1/auth/login").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/v1/auth/me").authenticated()
        .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos/descuento/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
        .requestMatchers(HttpMethod.PUT, "/api/v1/videojuegos/*").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/v1/videojuegos/*").hasRole("ADMIN")
        // añade aquí las reglas de EstudioController (GET, POST, PUT, DELETE, ajustar-precio),
        // siguiendo el mismo patrón que las de VideojuegoController de arriba — la decisión es tuya
        .requestMatchers("/error").permitAll()
        .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/documentacion").permitAll()
        .anyRequest().denyAll()
)
```

Con la configuración recién escrita, documéntala antes de seguir — mejor ahora, con las decisiones frescas, que al final de la actividad. Crea `docs/seguridad.md` en tu propio proyecto. Aquí tienes el documento ya hecho para `AuthController`, `VideojuegoController` e infraestructura — completa tú la tabla entera de `EstudioController` (marcada con `???`) con lo que acabas de decidir y justificar. La fila de `/actuator/health` todavía queda como `???`: la resuelves en el Paso 3, y entonces vuelves aquí a rellenarla.

```markdown
# Política de seguridad — GameVault

## AuthController

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/auth/register`, `/api/v1/auth/login` | POST | Cualquiera |
| `/api/v1/auth/me` | GET | Cualquiera autenticado |

## VideojuegoController

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/videojuegos` (y toda subruta GET: filtros, búsquedas, consultas nativas...) | GET | Cualquiera |
| `/api/v1/videojuegos` | POST | Solo ADMIN |
| `/api/v1/videojuegos/{id}` | PUT, DELETE | Solo ADMIN |
| `/api/v1/videojuegos/descuento/{estudioId}` | POST | Solo ADMIN |

## EstudioController

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/api/v1/estudios` (y toda subruta GET: /{id}, ranking, ranking-con-posicion...) | GET | ??? |
| `/api/v1/estudios` | POST | ??? |
| `/api/v1/estudios/{id}` | PUT, DELETE | ??? |
| `/api/v1/estudios/{id}/ajustar-precio` | POST | ??? |

## Infraestructura y documentación

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/actuator/health` | GET | ??? |
| `/documentacion`, `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Todo lo demás | — | Nadie |
```

**Captura**: el fichero, con `EstudioController` ya rellena (la fila de `/actuator/health` la completas en el Paso 3).

---

## Paso 2 — Verificar `denyAll()`

Prueba una ruta que, a propósito, no está en tu lista (por ejemplo, si tuvieras un `GET /api/v1/algo-inventado`, o cualquier verbo sobre una ruta real que no hayas cubierto explícitamente):

```bash
curl -i http://localhost:8080/api/v1/algo-inventado
```

**Comprueba**: que la respuesta es un `401` —no un `404` ni un `403`—. Spring Security evalúa `denyAll()` y rechaza la petición; como el `curl` no lleva ningún token válido, `ExceptionTranslationFilter` llama a tu `AuthenticationEntryPoint`. La misma petición con un JWT válido terminaría en `403`.

**Captura**: esta respuesta, con el `401`.

**Pregunta**: ¿por qué "cerrado por defecto" es la opción segura, comparado con "abierto por defecto, cerrado explícitamente donde haga falta"?

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

**Captura**: la respuesta ya funcionando, con la regla añadida.

Vuelve ahora a `docs/seguridad.md` (Paso 1) y rellena la fila de `/actuator/health` con lo que acabas de decidir — el documento se actualiza en el mismo momento en que cambia el código, no al final.

---

## Paso 4 — Una ruta que no existe: por qué da `500`

Necesitas un token de `admin` válido. Si ya no tienes a mano el de la Actividad 2.4 (Paso 5) —lo normal, si retomas esta actividad otro día—, pide uno nuevo:

```bash
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

```bash
ADMIN_TOKEN="pega-aqui-el-token-de-admin"
```

Con ese token, prueba una ruta que no existe pero que sí coincide con una regla que la permite — **ojo**: tiene que ser una ruta con dos segmentos tras la base, como en el ejemplo; con uno solo (`/api/v1/videojuegos/esto-no-existe`) coincidiría con `GET /api/v1/videojuegos/{id}`, y el error sería otro completamente distinto (una conversión de tipo fallida al intentar convertir el texto a `Long`, no una ruta inexistente):

```bash
curl -i http://localhost:8080/api/v1/videojuegos/999/no-existe \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

**Comprueba**: en vez de un `404`, ves un `500 Error interno` — el mismo formato que generaría cualquier bug real de tu código, aunque aquí el único "problema" es que la ruta no existe.

**Captura**: esta respuesta, con el `500`.

Añade un séptimo handler a tu `GlobalExceptionHandler` (Actividad 2.1, con el sexto añadido en la Actividad 2.4 para `AuthenticationException`), siguiendo exactamente el mismo patrón que los seis que ya tienes:

```java
// Handler 7 — una ruta que no coincide con ningún endpoint real
@ExceptionHandler(NoResourceFoundException.class)
public ResponseEntity<ErrorResponse> handleNoResourceFoundException(
        NoResourceFoundException ex, HttpServletRequest request) {
    // tu turno: construye el ErrorResponse (404, "Not Found", el mensaje que prefieras)
    // y devuélvelo con ResponseEntity.status(...), igual que en los otros seis handlers
}
```

Si te atascas con la firma exacta o el import de `NoResourceFoundException`, la teoría de este apartado trae el handler completo para consultar. Repite la misma petición y comprueba que ahora da `404`, con el formato de tu `ErrorResponse`.

**Captura**: la misma petición, ya corregida.

---

## Paso 5 — Un id que no es un id

Con el mismo token de admin, prueba ahora una ruta de un solo segmento, con algo que no sea un número:

```bash
curl -i http://localhost:8080/api/v1/videojuegos/esto-no-existe \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

**Comprueba**: en vez de un `400` (el id no tiene el formato correcto, un problema de la petición), ves otra vez un `500 Error interno` — por un motivo distinto al del Paso 4: esta ruta sí coincide con `GET /api/v1/videojuegos/{id}`, pero Spring no puede convertir el texto que has puesto al `Long` que espera el parámetro.

**Captura**: esta respuesta, con el `500`.

Añade un octavo handler, con el mismo patrón de siempre:

```java
// Handler 8 — un parámetro de ruta que no se puede convertir al tipo esperado
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<ErrorResponse> handleMethodArgumentTypeMismatchException(
        MethodArgumentTypeMismatchException ex, HttpServletRequest request) {
    // tu turno: construye el ErrorResponse (400, "Parámetro inválido", un mensaje que use
    // ex.getValue() y ex.getName() para decir exactamente qué ha fallado) y devuélvelo
}
```

Si necesitas repasar el cuerpo exacto o el import, la teoría de este apartado lo tiene completo. Repite la misma petición y comprueba que ahora da `400`, con el formato de tu `ErrorResponse` y un mensaje que dice exactamente qué valor no es válido.

**Captura**: la misma petición, ya corregida.

---

## Paso 6 — `AccessDeniedHandler` a medida, para que el `403` tenga formato

Crea la clase, junto a tu `ErrorResponseAuthenticationEntryPoint` de la Actividad 2.2 — es el mismo patrón exacto, con dos cambios: implementa `AccessDeniedHandler` en vez de `AuthenticationEntryPoint` (con su único método, `handle(request, response, accessDeniedException)`, en vez de `commence(...)`), y el código es `403` (`"No autorizado"`) en vez de `401`:

```java
@Component
@RequiredArgsConstructor
public class ErrorResponseAccessDeniedHandler implements AccessDeniedHandler {

    private final JsonMapper jsonMapper;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {
        // tu turno: construye el ErrorResponse (403, "No autorizado") y escríbelo en la respuesta,
        // igual que ya has hecho en ErrorResponseAuthenticationEntryPoint
        // (no olvides response.setCharacterEncoding("UTF-8") antes de escribir, para los mensajes con tildes)
    }
}
```

Si necesitas repasar el cuerpo exacto, tu propia clase de la Actividad 2.2 (o la teoría de este apartado) lo tiene completo. Añade el parámetro a tu `securityFilterChain` (Actividad 2.4, Paso 4) y regístrala junto al `AuthenticationEntryPoint` que ya tenías, en el mismo `.exceptionHandling(...)` —no toques nada más del método, solo estas dos líneas—:

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

## Cierre del tema

1. Repasa `docs/seguridad.md` (lo has ido completando desde el Paso 1): ¿tiene ya todas las filas rellenas, `EstudioController` y `/actuator/health` incluidos? ¿Falta alguna ruta propia que hayas añadido y que todavía no esté documentada?
2. Escribe un repaso propio (4-5 frases) de la evolución completa de este tema: validación y errores → HTTP Basic provisional → usuarios BCrypt en PostgreSQL → JWT → roles y política completa. ¿Qué parte te parece, mirando atrás, la decisión de diseño más importante de todo el recorrido, y por qué?
