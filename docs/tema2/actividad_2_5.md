# 🧪 Actividad 2.5: Roles y rutas protegidas

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.5 — Roles y rutas protegidas](plantillas/Actividad_2_5_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada"
    Hoy completas la política de autorización de tu GameVault, con las rutas propias que has ido añadiendo durante el curso incluidas, y la verificas con tests automatizados.

## Qué vas a practicar

- Completar una política de rutas cerrada por defecto.
- Depurar un caso real de ruta bloqueada por olvido.
- Cerrar un `500` mal formado en una ruta inexistente, con un handler específico.
- Cerrar un segundo `500` mal formado, esta vez por un id con el tipo equivocado, con otro handler específico.
- Implementar un `AccessDeniedHandler` a medida, para que el `403` tenga el mismo formato que el resto de tu API.
- Escribir tests de seguridad con login real.

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
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
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
| `/swagger-ui/**`, `/v3/api-docs/**`, `/error` | — | Cualquiera |
| Todo lo demás | — | Nadie |
```

**Captura**: el fichero, con `EstudioController` ya rellena (la fila de `/actuator/health` la completas en el Paso 3).

---

## Paso 2 — Verificar `denyAll()`

Prueba una ruta que, a propósito, no está en tu lista (por ejemplo, si tuvieras un `GET /api/v1/algo-inventado`, o cualquier verbo sobre una ruta real que no hayas cubierto explícitamente):

```bash
curl -i http://localhost:8080/api/v1/algo-inventado
```

**Comprueba**: que la respuesta es un `401` (no un `404` amable) — no un `403`, aunque sea `denyAll()`: como el `curl` no lleva ningún token, Spring Security ni siquiera llega a evaluar la regla, y responde con tu `AuthenticationEntryPoint` de siempre.

**Captura**: esta respuesta, con el `401`.

**Pregunta**: ¿por qué "cerrado por defecto" es la opción segura, comparado con "abierto por defecto, cerrado explícitamente donde haga falta"?

---

## Paso 3 — Depurar un caso real

!!! warning "Este fallo es intencionado — vas a diagnosticarlo tú"
    Con `denyAll()` activo, prueba ahora mismo: `GET /actuator/health` (el mismo endpoint que has activado en el Tema 1). Es muy probable que, si no lo has incluido explícitamente arriba, te dé un rechazo — aunque el endpoint en sí funcione perfectamente si lo pruebas de otra forma.

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
public class ErrorResponseAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {
        // tu turno: construye el ErrorResponse (403, "No autorizado") y escríbelo en la respuesta,
        // igual que ya has hecho en ErrorResponseAuthenticationEntryPoint
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

## Paso 7 — Test de seguridad, guiado al completo

Esto no puede ir dentro de `VideojuegoControllerTest`: esa clase lleva `@WebMvcTest` + `addFilters = false`, precisamente para que la seguridad no se ejecute. Aquí necesitas justo lo contrario, así que va en una clase nueva — por ejemplo `SeguridadEndpointsTest`, junto a tus `ControllerTest` de siempre.

!!! danger "`@ActiveProfiles(\"dev\")` es obligatorio aquí, o el test ni arranca"
    `VideojuegoControllerTest` nunca necesitó ningún perfil activo porque `@WebMvcTest` mockea el `service` y no toca ninguna base de datos. Esta clase sí levanta la aplicación entera con `@SpringBootTest`, así que necesita el datasource real de `application-dev.yml` —y ese fichero solo se aplica con el perfil `dev` activo—. Si lo lanzas desde el botón de tu IDE sin este `@ActiveProfiles`, no hay ningún perfil activo por defecto, y Spring Boot falla al arrancar con `Failed to configure a DataSource: 'url' attribute is not specified` — no es un bug tuyo, es que le falta decirle qué perfil usar.

```java
package com.tunombre.gamevault.seguridad;

import com.jayway.jsonpath.JsonPath;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("dev")
class SeguridadEndpointsTest {

    @Autowired
    private MockMvc mockMvc;

    @BeforeEach
    void asegurarUsuarioDePrueba() throws Exception {
        // registra "user" si todavía no existe (p. ej., en una base de datos recién creada) — se ignora
        // el resultado a propósito: da igual que sea un 201 o un 409 porque ya existía de antes
        mockMvc.perform(post("/api/v1/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {"username": "user", "password": "user123"}
                        """));
    }

    private String login(String username, String password) throws Exception {
        String response = mockMvc.perform(post("/api/v1/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"username": "%s", "password": "%s"}
                                """.formatted(username, password)))
                .andReturn().getResponse().getContentAsString();

        return JsonPath.read(response, "$.accessToken");
    }

    // crea un estudio propio y devuelve su id — para no depender de que data.sql
    // haya sembrado ninguno (en la base de datos de CI, recién creada, no hay ninguno)
    private long crearEstudio(String adminToken) throws Exception {
        String respuesta = mockMvc.perform(post("/api/v1/estudios")
                        .header("Authorization", "Bearer " + adminToken)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"nombre":"Estudio de prueba","pais":"España"}
                                """))
                .andReturn().getResponse().getContentAsString();

        return Long.parseLong(JsonPath.read(respuesta, "$.id").toString());
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
        long estudioId = crearEstudio(token);

        mockMvc.perform(post("/api/v1/videojuegos")
                        .header("Authorization", "Bearer " + token)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":%d}
                                """.formatted(estudioId)))
                .andExpect(status().isCreated());
    }

    @Test
    void rutaInexistente_DebeDevolver404() throws Exception {
        mockMvc.perform(get("/api/v1/videojuegos/999/no-existe"))
                .andExpect(status().isNotFound());
    }
}
```

Los cuatro tests cubren los tres casos de la tabla para una misma ruta —sin token (`401`), con rol insuficiente (`403`), con el rol correcto (éxito)— más el caso del Paso 4: una ruta que sencillamente no existe (`404`).

!!! tip "Por qué el `201` crea un estudio antes, en vez de usar `estudioId: 1`"
    El test del `201` no puede dar por hecho que existe un estudio con `id` 1: crear un videojuego necesita un estudio real al que asociarlo, y en la base de datos de CI —recién creada y vacía, sin `data.sql`— no hay ninguno. Si pusieras `estudioId: 1` a secas, en tu máquina pasaría (tienes estudios sembrados) pero en CI daría `404 Estudio no encontrado`. Es el mismo principio que `asegurarUsuarioDePrueba()`: **un test no debe depender de datos que quizá no estén ahí** — si necesita un estudio, se lo crea él. Fíjate en que el `401` y el `403` sí pueden seguir mandando `estudioId: 1`, porque su petición se rechaza en la capa de seguridad antes de llegar al `service` — nunca comprueban si ese estudio existe.

!!! danger "Si tienes CI (Tema 0), tienes que actualizarlo — o esto no se ejecuta nunca"
    Tu `ci.yml` filtra los tests con `-Dtest='*ControllerTest'`, así que `SeguridadEndpointsTest` **no se ejecuta** en GitHub Actions. Pero el nombre no es lo único que falla: esta clase, al llevar `@SpringBootTest`, arranca la aplicación entera —con una base de datos real detrás—, algo que tus `ControllerTest` nunca necesitaron porque mockean el `service`.

    Con `@ActiveProfiles("dev")` ya puesto en la clase (arriba) y un Postgres real disponible en CI, ya no hace falta filtrar por nombre en absoluto — puedes quitar el `-Dtest=...` y dejar que se ejecuten todos tus tests sin distinción. Ojo con `GamevaultApplicationTests` (la clase que ya traía tu proyecto desde el principio): es `@SpringBootTest` igual que esta, así que tiene el mismo problema si la lanzas sin ningún perfil activo — añádele también `@ActiveProfiles("dev")`.

    En tu `ci.yml` necesitas, como mínimo:

    - Un servicio de PostgreSQL en el workflow (bloque `services:`), igual que el de tu `docker-compose.yaml` pero accesible en `localhost` para el runner de GitHub Actions.
    - Las dos propiedades que solo viven en tu `application-dev-local.yml` — es justo el caso que anticipaba la teoría del apartado 3: un fichero en `.gitignore` no existe en CI, así que hay que inyectar sus valores por otra vía. Son la contraseña de `admin` (`gamevault.admin.password`) **y** el secreto de firma JWT (`gamevault.jwt.secret`) — sin este segundo, `SecurityConfig` ni siquiera arranca, porque necesita ese valor para construir el `JwtEncoder`/`JwtDecoder` (es el mismo error `Could not resolve placeholder 'gamevault.jwt.secret'` que verías si borraras ese fichero de tu propia máquina). **No las escribas en claro dentro de `ci.yml`** —ese fichero sí se sube al repositorio, así que sería exactamente el mismo problema que `.gitignore` evita— usa **dos GitHub Secrets**:

    1. En tu repositorio de GitHub, ve a **Settings** (la pestaña, no la de tu cuenta — la del repositorio).
    2. En el menú de la izquierda, **Secrets and variables → Actions**.
    3. Pestaña **Secrets** (no "Variables") → botón **New repository secret**.
    4. Crea `GAMEVAULT_ADMIN_PASSWORD` con el mismo valor que ya tienes en `application-dev-local.yml` (por ejemplo, `admin123`).
    5. Repite el proceso para `GAMEVAULT_JWT_SECRET`, con el valor de `gamevault.jwt.secret` de ese mismo fichero.
    6. A partir de ahora, `${{ secrets.GAMEVAULT_ADMIN_PASSWORD }}` y `${{ secrets.GAMEVAULT_JWT_SECRET }}` en tu `ci.yml` se resuelven a esos valores — sin que aparezcan en ningún fichero del repositorio, ni en los logs del workflow (GitHub los enmascara automáticamente).

    Por eso `asegurarUsuarioDePrueba()` registra a `user` si no existe: en una base de datos de CI, recién creada en cada ejecución, ese usuario no existe todavía —a diferencia de en tu máquina, donde ya lo has registrado a mano en la Actividad 2.3—, así que el test tiene que poder crearlo él solo.

    Así queda el `job` completo de tu `ci.yml` con los cambios ya aplicados — sustituye el tuyo por este:

    ```yaml
    jobs:
      test:
        runs-on: ubuntu-latest

        services:
          postgres:
            image: postgres:16-alpine
            env:
              POSTGRES_DB: gamevault_db
              POSTGRES_USER: gamevault_user
              POSTGRES_PASSWORD: password123
            ports:
              - 5432:5432
            options: >-
              --health-cmd pg_isready
              --health-interval 10s
              --health-timeout 5s
              --health-retries 5

        steps:
          - name: Descargar el código fuente
            uses: actions/checkout@v4

          - name: Instalar Java 21
            uses: actions/setup-java@v4
            with:
              java-version: '21'
              distribution: 'temurin'
              cache: 'maven'

          - name: Dar permiso de ejecución al wrapper
            run: chmod +x mvnw

          - name: Ejecutar los tests
            run: >-
              ./mvnw test -B
              -Dspring.datasource.url=jdbc:postgresql://localhost:5432/gamevault_db
              -Dgamevault.admin.password=${{ secrets.GAMEVAULT_ADMIN_PASSWORD }}
              -Dgamevault.jwt.secret=${{ secrets.GAMEVAULT_JWT_SECRET }}

          - name: Guardar el informe de tests
            if: always()
            uses: actions/upload-artifact@v4
            with:
              name: informe-tests
              path: target/surefire-reports/
    ```

!!! note "Esto, en realidad, habría que repetirlo para cada ruta protegida"
    Lo ideal en un proyecto real es escribir este mismo patrón para todas y cada una de las rutas protegidas, no solo para `crearVideojuego` — es la única forma de tener la certeza de que la política completa hace lo que dice. Aquí solo se pide para esta ruta y, en el Paso 8, para el `DELETE` de `Estudio`, para no alargar la actividad — pero el patrón ya lo tienes: aplícalo al resto de tu proyecto por tu cuenta si quieres una cobertura real.

---

## Paso 8 — Repite el patrón con `DELETE` de `Estudio`

Escribe los mismos tres tests (sin token, con rol insuficiente si aplica según tu decisión del Paso 1, con el rol correcto) para el `DELETE` de `Estudio`, en la misma clase `SeguridadEndpointsTest` del Paso 7. Los dos primeros son un hueco para que los rellenes tú, siguiendo el mismo patrón del Paso 7; el tercero es algo más largo —necesita crear un estudio de prueba antes de poder borrarlo—, así que ya lo tienes resuelto:

```java
@Test
void borrarEstudio_DebeDevolver401_SinToken() throws Exception {
    // tu turno: DELETE /api/v1/estudios/{id} sin token, comprueba que da 401
}

@Test
void borrarEstudio_DebeDevolver403_ConRolUser() throws Exception {
    // tu turno: login como "user", DELETE con ese token, comprueba que da 403
    // (o ajusta el rol según lo que hayas decidido en el Paso 1 para Estudio)
}

@Test
void borrarEstudio_DebeDevolver204_ConRolAdmin() throws Exception {
    String adminToken = login("admin", "admin123");

    // reutiliza el mismo helper del Paso 7: crea un estudio propio en vez de borrar
    // uno de data.sql (que en CI no existe, y en local podría tener videojuegos asociados)
    long estudioId = crearEstudio(adminToken);

    mockMvc.perform(delete("/api/v1/estudios/" + estudioId)
                    .header("Authorization", "Bearer " + adminToken))
            .andExpect(status().isNoContent());
}
```

**Captura**: la ejecución de todos tus tests de seguridad (Paso 7 y Paso 8), todos en verde.

---

## Cierre del tema

1. Repasa `docs/seguridad.md` (lo has ido completando desde el Paso 1): ¿tiene ya todas las filas rellenas, `EstudioController` y `/actuator/health` incluidos? ¿Falta alguna ruta propia que hayas añadido y que todavía no esté documentada?
2. Escribe un repaso propio (4-5 frases) de la evolución completa de este tema: validación y errores → HTTP Basic provisional → usuarios BCrypt en PostgreSQL → JWT → roles y política completa con tests. ¿Qué parte te parece, mirando atrás, la decisión de diseño más importante de todo el recorrido, y por qué?
3. Si tienes CI, haz `push` de tus cambios (`ci.yml` incluido) y comprueba, en la pestaña **Actions** de tu repositorio, que el workflow se ejecuta y termina en verde, con `SeguridadEndpointsTest` entre los tests pasados.

    **Captura**: el resultado en verde del workflow, con los tests pasados.
