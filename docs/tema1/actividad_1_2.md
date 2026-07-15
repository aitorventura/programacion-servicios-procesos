# 🧪 Actividad 1.2: Swagger/OpenAPI — documenta y prueba tu API

!!! warning "Descarga la plantilla"
    📄 [Plantilla 1.2 — Swagger/OpenAPI: documenta y prueba tu API](plantillas/Actividad_1_2_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — con parte a tu cargo"
    Tu propio GameVault ya tiene el CRUD completo (`GET`/`POST`/`PUT`/`DELETE`) de `Videojuego` y de `Estudio` (Acceso a Datos, Actividad 1.2). Aquí lo documentas con OpenAPI/Swagger y lo usas como cliente para probar, de verdad, las ocho operaciones — sin `curl`. Las cuatro de `Videojuego` van guiadas; las cuatro de `Estudio` las repites tú solo, con el mismo patrón.

## Qué vas a practicar

- Añadir la documentación automática OpenAPI/Swagger a tu propio GameVault.
- Leer y entender qué parte de Swagger UI viene de dónde en tu código.
- Documentar con `@ApiResponses` qué códigos puede devolver de verdad un endpoint, más allá del `200` genérico.
- Probar `GET`, `POST`, `PUT` y `DELETE` desde el navegador, sobre `Videojuego` y sobre `Estudio`, y observar sus códigos de estado.

---

## Requisitos previos

Tu CRUD completo de `Videojuego` y de `Estudio` funcionando (Tema 1, Actividad 1.2 de Acceso a Datos): las cuatro operaciones de cada uno, con sus DTOs y su validación ya puestos.

---

## Paso 1 — Añadir springdoc a tu proyecto

Esto es imprescindible antes de seguir: sin esta dependencia no existe Swagger UI ni el resto de la actividad. En tu `pom.xml`, añade:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.3</version>
</dependency>
```

Crea, en tu paquete `config`, una clase `OpenApiConfig`:

```java
package com.tunombre.gamevault.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI gamevaultOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Mi GameVault")
                        .version("v1")
                        .description("API de mi propio GameVault, curso 2026/2027."));
    }
}
```

`@Bean` le dice a Spring que este método construye un objeto que debe gestionar como bean — en este caso, la configuración que springdoc va a usar para generar la documentación. No tienes que llamarlo tú en ningún sitio: Spring lo detecta y lo usa automáticamente.

De paso, añade esto a tu `application-dev.yaml`: sirve la interfaz en `/documentacion` en vez de la ruta por defecto (`/swagger-ui.html`), y ordena los endpoints alfabéticamente por verbo en vez de por el orden de declaración:

```yaml
springdoc:
  swagger-ui:
    path: /documentacion
    operations-sorter: method
```

Reinicia tu aplicación y entra en `http://localhost:8080/documentacion` antes de seguir — si no aparece nada, revisa que la dependencia se ha descargado (Maven) y que la aplicación ha reiniciado de verdad con el cambio.

!!! tip "Dos matices de estas dos propiedades"
    `/documentacion` es solo el punto de entrada — Swagger UI redirige siempre a `/swagger-ui/index.html`, esa es la página real donde vive la interfaz; no hace falta que la URL final se quede en `/documentacion`. Y `operations-sorter: method` no da el orden CRUD `GET → POST → PUT → DELETE` — ordena alfabéticamente por el nombre del verbo, así que verás `DELETE` primero (`D` va antes que `G` en el alfabeto): `DELETE, GET, POST, PUT`.

**Captura**: la pantalla completa de Swagger UI tras entrar en `/documentacion`, con la URL final visible en la barra de direcciones y los endpoints de `videojuego-controller` en orden `DELETE, GET, GET, POST, PUT`.

---

## Paso 2 — Sembrar algunos datos de prueba

Antes de probar nada, conviene tener algo real que leer — un `GET` contra una tabla vacía funciona, pero no enseña gran cosa. Inserta un par de filas de cada tabla, con la herramienta que prefieras (terminal `psql`, pgAdmin, o cualquier otro cliente gráfico que ya uses) — esta vez fijando tú el `id` en vez de dejar que lo genere la base de datos, para que se vea sin ninguna duda qué `estudio_id` apunta a qué estudio:

```sql
INSERT INTO estudio (id, nombre, pais) VALUES
    (101, 'Supergiant Games', 'Estados Unidos'),
    (102, 'FromSoftware', 'Japón');

INSERT INTO videojuego (id, titulo, precio, fecha_lanzamiento, estudio_id) VALUES
    (101, 'Hades', 24.99, '2020-09-17', 101),
    (102, 'Elden Ring', 59.99, '2022-02-25', 102);
```

Usar `id` como `101`/`102` (en vez de `1`/`2`) es a propósito: evita chocar con filas que ya tuvieras de actividades anteriores, generadas automáticamente con `id` bajos. Comprueba con `SELECT * FROM estudio;` y `SELECT * FROM videojuego;` que las cuatro filas se han creado, y fíjate en que `videojuego.estudio_id` coincide exactamente con el `id` del estudio correspondiente — esa es la relación que vas a usar en los pasos siguientes.

---

## Paso 3 — Recorrer Swagger UI

**Localiza y anota**:

1. El bloque `videojuego-controller` (o el nombre que le corresponda) — despliega uno de sus endpoints, por ejemplo `GET /api/v1/videojuegos/{id}`. Compara lo que ves en pantalla (parámetros, respuestas posibles) con la anotación `@GetMapping("/{id}")` y el `@PathVariable Long id` de tu propio código: ¿de dónde sale cada dato que ves en Swagger UI?
2. La sección de **Schemas** (normalmente al final de la página) — busca `VideojuegoCreateDTO` o `VideojuegoResponseDTO`. Compáralo con la clase Java real: ¿coinciden los campos?

**Pregunta**: si añadieras un campo nuevo a `VideojuegoCreateDTO` en tu código y reiniciaras la aplicación, ¿tendrías que actualizar algo a mano en Swagger UI para que aparezca? Razona tu respuesta con lo visto en la teoría sobre cómo se genera el contrato OpenAPI.

**Captura**: la sección de Schemas con `VideojuegoCreateDTO` o `VideojuegoResponseDTO` desplegado.

---

## Bloque A — Las cuatro operaciones de `Videojuego` (guiado)

Antes de probar nada, documenta con `@ApiResponses` los cuatro métodos de `VideojuegoController` — igual que en el ejemplo de la teoría, cada uno con los códigos que de verdad puede devolver:

```java
@Operation(summary = "Obtener un videojuego por su id")
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Videojuego encontrado"),
        @ApiResponse(responseCode = "404", description = "No existe ningún videojuego con ese id")
})
@GetMapping("/{id}")

@Operation(summary = "Listar todos los videojuegos")
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Lista de videojuegos (puede estar vacía)")
})
@GetMapping

@Operation(summary = "Crear un videojuego nuevo")
@ApiResponses({
        @ApiResponse(responseCode = "201", description = "Videojuego creado correctamente"),
        @ApiResponse(responseCode = "400", description = "El cuerpo de la petición no supera las validaciones"),
        @ApiResponse(responseCode = "404", description = "El estudio indicado no existe")
})
@PostMapping

@Operation(summary = "Actualizar un videojuego existente")
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Videojuego actualizado correctamente"),
        @ApiResponse(responseCode = "400", description = "El cuerpo de la petición no supera las validaciones"),
        @ApiResponse(responseCode = "404", description = "No existe ningún videojuego con ese id, o el estudio indicado no existe")
})
@PutMapping("/{id}")

@Operation(summary = "Borrar un videojuego")
@ApiResponses({
        @ApiResponse(responseCode = "204", description = "Videojuego borrado correctamente"),
        @ApiResponse(responseCode = "404", description = "No existe ningún videojuego con ese id")
})
@DeleteMapping("/{id}")
```

Reinicia y despliega cada endpoint en Swagger UI: fíjate en que la sección "Responses" de cada uno lista ahora sus códigos reales, no solo el `200` genérico de antes.

**Captura**: la sección "Responses" de `POST /api/v1/videojuegos` desplegada (sin ejecutar nada todavía), mostrando los tres códigos documentados (`201`/`400`/`404`) con su descripción.

Con Swagger UI ya abierto sobre el bloque `videojuego-controller`, prueba las cuatro operaciones en orden. Para cada una, además de lo que se pide, haz una **captura de Swagger UI mostrando la petición enviada y la respuesta recibida** (cuerpo y código de estado visibles).

### `GET` — leer

Despliega `GET /api/v1/videojuegos`, pulsa **Try it out** → **Execute**, sin rellenar nada. Comprueba el `200` y que la lista incluye los videojuegos que ya tenías. Repite con `GET /api/v1/videojuegos/{id}` sobre uno concreto.

**Captura**: la respuesta de `GET /api/v1/videojuegos/{id}`.

### `POST` — crear

Despliega `POST /api/v1/videojuegos`, pulsa **Try it out** y sustituye el cuerpo de ejemplo por uno propio:

```json
{
  "titulo": "Celeste",
  "precio": 19.99,
  "fechaLanzamiento": "2018-01-25",
  "estudioId": 101
}
```

Pulsa **Execute**. Anota el código de estado devuelto y el cuerpo de la respuesta — debería ser un `201 Created` con el videojuego recién creado, incluyendo el `id` que le ha asignado la base de datos.

**Captura**: la petición y la respuesta del `POST`.

### `PUT` — actualizar

Prueba `PUT /api/v1/videojuegos/{id}` sobre el `id` que te ha devuelto el `POST` anterior, cambiando por ejemplo el precio. **Predicción**: ¿qué código de estado esperas ver esta vez, y por qué debería ser distinto del `POST`? Escribe tu respuesta antes de pulsar **Execute**.

**Captura**: la petición y la respuesta del `PUT`.

### `DELETE` — borrar

Prueba `DELETE /api/v1/videojuegos/{id}` sobre ese mismo videojuego, y comprueba el código de estado. A continuación, un `GET /api/v1/videojuegos/{id}` sobre ese mismo `id`: comprueba qué código obtienes ahora, y relaciónalo con lo que aprendiste sobre el `404` en la Actividad 1.1.

**Captura**: las dos respuestas (`DELETE` y el `GET` posterior) con el código de estado visible.

---

## Bloque B — Las cuatro operaciones de `Estudio` (sin guía)

Repite exactamente el mismo patrón del Bloque A — `GET`, `POST`, `PUT`, `DELETE` — esta vez sobre `estudio-controller`, con datos de un estudio real (por ejemplo, uno nuevo con `nombre` y `pais`). Sin más indicaciones que estas:

- Las mismas cuatro operaciones, en el mismo orden.
- Los mismos códigos de estado esperados que en `Videojuego` — recuerda la tabla del apartado de teoría si tienes dudas.
- `@ApiResponses` sobre los cuatro métodos de `EstudioController` (no solo `create()`), con los códigos que de verdad puede devolver cada uno. Aquí no hay ningún estudio relacionado que pueda no existir — piensa qué códigos tienen sentido en cada método y cuáles no aplican, comparado con `Videojuego`.
- La misma comprobación final: borra el estudio que has creado y verifica con un `GET` posterior que ya no existe.

**Captura**: la sección "Responses" de `POST /api/v1/estudios` desplegada, mostrando los códigos que has documentado tú — igual que hiciste en el Bloque A.

**Captura**: una por cada una de las cuatro operaciones (petición + respuesta visibles), igual que en el Bloque A.

---

## Pregunta final

En la Actividad 1.1 usaste `curl`; hoy has usado Swagger UI, sobre dos controllers distintos. Ambos han hablado con exactamente la misma API, sin que hayas tenido que tocar ni una línea del servidor para adaptarte a uno u otro cliente. ¿Qué ventaja concreta del uso de protocolos estándar demuestra esto? Da un ejemplo de lo que tendrías que hacer si tu API usara un protocolo propio, no estándar, y quisieras que un cliente nuevo pudiera hablar con ella.

---

## ✅ Cierre

Tu GameVault ya tiene documentación automática y un cliente interactivo para probar, sin `curl`, las ocho operaciones de `Videojuego` y `Estudio`. En la próxima actividad (1.3) sigues con los tests automatizados (MockMvc) — la forma de probar estos mismos endpoints sin depender de que abras el navegador cada vez.
