# 🧪 Actividad 1.2: Swagger/OpenAPI — documenta y prueba tu API

!!! info "Práctica guiada"
    Esta semana, en Acceso a Datos, estás construyendo el CRUD completo de `Videojuego` en tu propio proyecto. Aquí vas a documentarlo con OpenAPI/Swagger y a usarlo como cliente para probar esos mismos endpoints de escritura — sin necesitar `curl` ni Postman.

## Qué vas a practicar

- Añadir la documentación automática OpenAPI/Swagger a tu propio GameVault.
- Leer y entender qué parte de Swagger UI viene de dónde en tu código.
- Probar `POST`, `PUT` y `DELETE` desde el navegador y observar sus códigos de estado.

---

## Requisitos previos

Tu propio GameVault con, al menos, el `GET` de `Videojuego` funcionando (Actividad 1.1 de AD). Si en Acceso a Datos ya tienes el `POST`/`PUT`/`DELETE` completos, mejor — pero puedes hacer los Pasos 1-2 de esta actividad aunque todavía no los tengas, y dejar los Pasos 3-4 para cuando los termines.

---

## Paso 1 — Añadir springdoc a tu proyecto

En tu `pom.xml`, añade:

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

Reinicia tu aplicación.

---

## Paso 2 — Recorrer Swagger UI

Abre `http://localhost:8080/swagger-ui.html` en el navegador.

**Localiza y anota**:

1. El bloque `videojuego-controller` (o el nombre que le corresponda) — despliega uno de sus endpoints, por ejemplo `GET /api/v1/videojuegos/{id}`. Compara lo que ves en pantalla (parámetros, respuestas posibles) con la anotación `@GetMapping("/{id}")` y el `@PathVariable Long id` de tu propio código: ¿de dónde sale cada dato que ves en Swagger UI?
2. La sección de **Schemas** (normalmente al final de la página) — busca `VideojuegoCreateDTO` o `VideojuegoResponseDTO`. Compáralo con la clase Java real: ¿coinciden los campos?

**Pregunta**: si añadieras un campo nuevo a `VideojuegoCreateDTO` en tu código y reiniciaras la aplicación, ¿tendrías que actualizar algo a mano en Swagger UI para que aparezca? Razona tu respuesta con lo visto en la teoría sobre cómo se genera el contrato OpenAPI.

---

## Paso 3 — Probar la escritura desde Swagger UI

!!! tip "Si todavía no tienes el POST/PUT completos en AD"
    Puedes hacer este paso más adelante, en cuanto termines esa parte en Acceso a Datos — no hace falta que lo hagas el mismo día que los Pasos 1-2.

En el bloque de `videojuego-controller`, despliega `POST /api/v1/videojuegos`, pulsa **Try it out** y sustituye el cuerpo de ejemplo por uno propio:

```json
{
  "titulo": "Celeste",
  "precio": 19.99,
  "fechaLanzamiento": "2018-01-25",
  "estudioId": 1
}
```

Pulsa **Execute**. Anota el código de estado devuelto y el cuerpo de la respuesta — debería ser un `201 Created` con el videojuego recién creado, incluyendo el `id` que le ha asignado la base de datos.

Ahora prueba un `PUT /api/v1/videojuegos/{id}` sobre ese mismo `id`, cambiando por ejemplo el precio. **Predicción**: ¿qué código de estado esperas ver esta vez, y por qué debería ser distinto del `POST`? Escribe tu respuesta antes de pulsar **Execute**.

---

## Mini-reto — repite el patrón con el DELETE

Sin más indicaciones que las del Paso 3, prueba desde Swagger UI:

1. `DELETE /api/v1/videojuegos/{id}` sobre el videojuego que has creado — comprueba el código de estado.
2. A continuación, un `GET /api/v1/videojuegos/{id}` sobre ese mismo `id` — comprueba qué código de estado obtienes ahora, y relaciónalo con lo que aprendiste sobre el `404` en la Actividad 1.1.

**Captura**: las dos respuestas de Swagger UI (DELETE y GET posterior) con el código de estado visible.

---

## Pregunta final

En la Actividad 1.1 usaste `curl`; hoy has usado Swagger UI. Ambos han hablado con exactamente la misma API, sin que hayas tenido que tocar ni una línea del servidor para adaptarte a uno u otro cliente. ¿Qué ventaja concreta del uso de protocolos estándar demuestra esto? Da un ejemplo de lo que tendrías que hacer si tu API usara un protocolo propio, no estándar, y quisieras que un cliente nuevo pudiera hablar con ella.

---

## ✅ Cierre

Tu GameVault ya tiene documentación automática y un cliente interactivo para probarlo sin `curl`. La semana que viene, en el Tema 1 sigues con los tests automatizados (MockMvc) — la forma de probar estos mismos endpoints sin depender de que abras el navegador cada vez.
