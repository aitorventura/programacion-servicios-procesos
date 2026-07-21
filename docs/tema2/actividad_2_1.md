# 🧪 Actividad 2.1: Validación y `GlobalExceptionHandler`

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.1 — Validación y GlobalExceptionHandler](plantillas/Actividad_2_1_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada"
    Vas a construir en tu GameVault la gestión centralizada de errores. Primero provocas, a propósito, los cinco tipos de error que puede devolver tu API tal como está hoy — sin gestionar nada. Después construyes un `GlobalExceptionHandler` completo. Al final, repites exactamente los mismos cinco errores para comprobar cómo cambia cada respuesta.

## Qué vas a practicar

- Personalizar los mensajes de la validación con Bean Validation que ya tienes en tus DTOs de entrada.
- Provocar, a propósito y desde Swagger UI, cada tipo de error que ya conoces de la teoría.
- Construir un `@RestControllerAdvice` centralizado, con un handler por cada tipo de error.
- Comparar la respuesta de cada error antes y después de gestionarlo.

---

## Requisitos previos

Tu CRUD de `Videojuego` y de `Estudio` funcionando, con validación y `@Valid` (Acceso a Datos, Tema 1).

---

## Paso 1 — Mensajes de error personalizados en `VideojuegoCreateDTO` y `EstudioCreateDTO`

Tu DTO ya tiene validación — la has construido en Acceso a Datos (Tema 1), con `@Valid` ya puesto en el controller. Pero cada anotación usa el mensaje por defecto de Bean Validation, genérico y en inglés (`"must not be blank"`, por ejemplo). Añade tú mismo un mensaje propio a cada anotación, con el atributo `message`:

```java
public record VideojuegoCreateDTO(
        @NotBlank(message = "...")
        @Size(max = 150, message = "...")
        String titulo,

        @NotNull(message = "...")
        @PositiveOrZero(message = "...")
        BigDecimal precio,

        @NotNull(message = "...")
        @PastOrPresent(message = "...")
        LocalDate fechaLanzamiento,

        @NotNull(message = "...")
        @Positive(message = "...")
        Long estudioId
) {}
```

Escribe tú cada mensaje: algo claro, en español, que le diga a quien consuma la API exactamente qué campo ha fallado y por qué — sin copiar literalmente el nombre de la anotación.

Este mensaje es justo lo que vas a recuperar dentro de poco, en el `GlobalExceptionHandler` del Paso 3, con `FieldError::getDefaultMessage` — sin un mensaje propio, ese campo sería igual de genérico y poco útil.

Repite el mismo patrón sobre `EstudioCreateDTO` (paquete `catalogo.dto`): ya tiene `@NotBlank` en `nombre` y en `pais`, sin mensaje propio. Añádeselo también, con tus propios mensajes.

El resto de esta actividad se centra en `Videojuego` para comparar las respuestas paso a paso, pero todo lo que construyas en el `GlobalExceptionHandler` es transversal a toda la aplicación: en cuanto lo tengas, `Estudio` queda cubierto exactamente igual, sin tocar ni una línea de esa clase. Si en algún momento provocas un error de validación sobre `/api/v1/estudios`, verás el mismo formato de `ErrorResponse`, con tus mensajes de `EstudioCreateDTO` en el campo `fields`.

---

## Paso 2 — Provoca los cinco errores, antes de gestionar nada

Ahora mismo tu API no centraliza ningún error: cada tipo de fallo se muestra como Spring decida por defecto, sin ningún criterio común entre ellos. Antes de arreglarlo, provócalos todos desde Swagger UI y comprueba con tus propios ojos qué aspecto tienen.

### 1. Un DTO que no pasa la validación

En el formulario de `POST /videojuegos`, manda un cuerpo inválido:

```json
{"titulo":"","precio":-5,"fechaLanzamiento":null,"estudioId":-1}
```

**Captura**: la respuesta completa que muestra Swagger UI. Deberías obtener un `400`, pero con un cuerpo genérico de Spring.

### 2. Un recurso que no existe

En el formulario de `GET /videojuegos/{id}`, pide un `id` que sepas que no existe (por ejemplo, `999999`).

**Captura**: la respuesta. Ya es un `404` razonable —lo has construido en Acceso a Datos, Tema 1, con `ResponseStatusException`—, pero fíjate en su formato: ¿se parece en algo al que acabas de capturar en el paso anterior?

### 3. Un parámetro de consulta que Spring Data no entiende

En el formulario de `GET /videojuegos`, escribe `noExiste,asc` en el campo `sort` (un campo que no existe en la entidad).

**Captura**: la respuesta. Deberías obtener un `500 Internal Server Error` con una traza de pila completa.

### 4. Una restricción que solo existe en la base de datos

Este caso tiene una particularidad: si tu entidad `Estudio` tiene su relación con `Videojuego` marcada como `@OneToMany(mappedBy = "estudio", cascade = CascadeType.ALL, orphanRemoval = true)`, borrar un `Estudio` con `Videojuego`s asociados los borra también a ellos automáticamente — la clave foránea nunca llega a quejarse, porque JPA resuelve el problema por ti antes de que la base de datos tenga ocasión de hacerlo.

Para verla de verdad, comenta **temporalmente** el `cascade` y el `orphanRemoval` de esa anotación en `Estudio.java`, dejándola así:

```java
@OneToMany(mappedBy = "estudio")
private List<Videojuego> videojuegos = new ArrayList<>();
```

Elige un `Estudio` que tenga al menos un `Videojuego` asociado (por ejemplo, uno de los que has sembrado con `data.sql`) y bórralo desde el formulario de `DELETE /estudios/{id}`. Ahora sí, la petición llega hasta la base de datos, que rechaza el borrado por la clave foránea.

**Captura**: la respuesta.

### 5. Un error que no es ninguno de los anteriores

Este último caso es distinto de los cuatro anteriores: no tiene un disparador natural. Los otros cuatro ocurren porque son fallos **conocidos** (una validación, un recurso que no existe, un parámetro raro, una restricción de la base de datos); si este quinto tuviera también una forma reproducible de provocarlo desde fuera, dejaría de ser "lo no previsto" y sería un sexto caso conocido, con su propio handler específico. La única manera honesta de verlo es provocar tú mismo, a propósito y de forma temporal, un fallo genérico en tu código.

Añade una línea al principio de cualquier método de tu `VideojuegoController` o `VideojuegoService` —por ejemplo, `getById`—:

```java
throw new RuntimeException("Fallo simulado");
```

Prueba ese endpoint desde Swagger UI. Ninguno de tus cuatro handlers está pensado para una `RuntimeException` genérica, así que se escapa igual que se escaparía un bug real que no hubieras previsto.

**Captura**: la respuesta.

---

## Paso 3 — `GlobalExceptionHandler`, completo

Crea el paquete `exception` (`src/main/java/com/tunombre/gamevault/exception/`) con `ErrorResponse`:

```java
package com.tunombre.gamevault.exception;

import java.util.List;
import java.util.Map;

public record ErrorResponse(
        String timestamp,
        int status,
        String error,
        String message,
        String path,
        Map<String, List<String>> fields
) {
    public ErrorResponse(String timestamp, int status, String error, String message, String path) {
        this(timestamp, status, error, message, path, Map.of());
    }
}
```

Y, en el mismo paquete, `GlobalExceptionHandler` con los cinco handlers — uno por cada error que has provocado en el Paso 2. Cada uno está explicado en detalle en la teoría de este apartado ("Handler 1" a "Handler 5"); aquí tienes el código completo, listo para copiar dentro de tu propio proyecto:

```java
package com.tunombre.gamevault.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.data.core.PropertyReferenceException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.server.ResponseStatusException;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handler 1 — un DTO con @Valid que no cumple sus anotaciones
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, List<String>> fields = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.groupingBy(
                        FieldError::getField,
                        Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
                ));

        ErrorResponse response = new ErrorResponse(
                LocalDateTime.now().toString(),
                HttpStatus.BAD_REQUEST.value(),
                "Error de validación",
                "La petición contiene campos inválidos",
                request.getRequestURI(),
                fields
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    // Handler 2 — un error que ya lanzas tú explícitamente (orElseThrow, por ejemplo)
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ErrorResponse> handleResponseStatusException(
            ResponseStatusException ex, HttpServletRequest request) {

        int statusCode = ex.getStatusCode().value();
        HttpStatus status = HttpStatus.resolve(statusCode);
        String error = status != null ? status.getReasonPhrase() : "Error";
        String message = ex.getReason() != null ? ex.getReason() : error;

        ErrorResponse response = new ErrorResponse(
                LocalDateTime.now().toString(), statusCode, error, message, request.getRequestURI()
        );
        return ResponseEntity.status(ex.getStatusCode()).body(response);
    }

    // Handler 3 — un parámetro que Spring Data no sabe interpretar (por ejemplo, un sort inválido)
    @ExceptionHandler(PropertyReferenceException.class)
    public ResponseEntity<ErrorResponse> handlePropertyReferenceException(
            PropertyReferenceException ex, HttpServletRequest request) {

        ErrorResponse response = new ErrorResponse(
                LocalDateTime.now().toString(), 400, "Parámetro inválido",
                "Uno de los parámetros de la consulta no es válido", request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    // Handler 4 — una restricción que solo existe en la base de datos (una clave foránea, por ejemplo)
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrityViolationException(
            DataIntegrityViolationException ex, HttpServletRequest request) {

        ErrorResponse response = new ErrorResponse(
                LocalDateTime.now().toString(), 409, "Conflicto de datos",
                "La operación viola una restricción de la base de datos (una referencia que no existe, un registro del que aún dependen otros, un valor duplicado, etc.)",
                request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(response);
    }

    // Handler 5 — la red de seguridad final, para cualquier otra excepción no prevista
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(
            Exception ex, HttpServletRequest request) {

        ErrorResponse response = new ErrorResponse(
                LocalDateTime.now().toString(), 500, "Error interno",
                "Ha ocurrido un error inesperado", request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
```

`@RestControllerAdvice` hace que esta clase intercepte excepciones de **todos** tus controllers, no solo de uno. Cada `@ExceptionHandler` decide qué tipo de excepción atrapa y cómo la convierte en un `ErrorResponse` uniforme — en orden, del más específico (una validación concreta) al más genérico (cualquier excepción no prevista).

---

## Paso 4 — Repite los cinco errores y compara

Con el `GlobalExceptionHandler` ya en marcha, repite exactamente las mismas cinco peticiones del Paso 2, en el mismo orden, y compara cada respuesta nueva con la que capturaste entonces.

### 1. El DTO que no pasa la validación

Repite la misma petición inválida a `POST /videojuegos` desde Swagger UI.

**Captura**: la respuesta nueva, junto a la del Paso 2.

**Responde**: ¿qué información nueva aparece ahora que antes no estaba (pista: el campo `fields`)?

### 2. El recurso que no existe

Repite la misma petición a `GET /videojuegos/{id}` con el `id` inexistente.

**Captura**: la respuesta nueva, junto a la del Paso 2.

**Responde**: aunque el código de estado no cambia (sigue siendo un `404`), ¿qué ha cambiado en el formato de la respuesta?

### 3. El parámetro que Spring Data no entiende

Repite la misma petición a `GET /videojuegos` con `sort=noExiste,asc`.

**Captura**: la respuesta nueva — comprueba que ahora obtienes un `400` con tu formato de `ErrorResponse`, no la traza de pila de antes.

**Pregunta**: ¿por qué un `500` era la respuesta equivocada para este caso, aunque técnicamente la aplicación "haya fallado" al intentar ordenar por ese campo? Piensa en quién ha cometido el error — el cliente, al pedir un campo que no existe, o el servidor, al no saber gestionarlo.

### 4. La restricción de la base de datos

Con el `cascade`/`orphanRemoval` todavía comentado en `Estudio.java`, repite el mismo `DELETE /estudios/{id}` sobre un `Estudio` que tenga `Videojuego`s asociados.

**Captura**: la respuesta nueva — comprueba que ahora obtienes un `409` con tu formato de `ErrorResponse`.

**Pregunta**: ¿por qué `409 Conflict` encaja mejor aquí que `400 Bad Request`? Piensa en la diferencia entre "esta petición está mal construida" y "esta petición está bien construida, pero no encaja con el estado actual de tus datos".

Descomenta ahora `cascade = CascadeType.ALL, orphanRemoval = true` en `Estudio.java` — no debe quedarse desactivado. El handler que acabas de construir es la segunda barrera, no un sustituto de la primera.

### 5. El error que no era ninguno de los anteriores

Vuelve a añadir temporalmente la misma línea (`throw new RuntimeException("Fallo simulado");`) y repite la misma petición.

**Captura**: la respuesta nueva.

**Pregunta**: ¿por qué este último handler **no** debería usar `ex.getMessage()` como cuerpo de la respuesta, a pesar de ser mucho más cómodo para depurar mientras programas? Relaciónalo con el principio de "no filtrar información interna en los errores" de la teoría de este apartado.

Ahora sí, quita esa línea de tu código — no debe quedarse ahí. Era solo para provocar el error a propósito, y ya has hecho las dos capturas que necesitabas.

---

## Pregunta final

¿Por qué conviene que **todas** las excepciones de la aplicación pasen por un único punto (`GlobalExceptionHandler`) antes de convertirse en respuesta HTTP, en vez de gestionar cada error donde ocurre? Nombra un dato interno concreto de tu propio proyecto (una clase, una tabla, una ruta de fichero) que este handler evita que se filtre a quien esté probando tu API desde fuera.

---

## ✅ Cierre

!!! warning "Comprueba que no queda nada de las pruebas a propósito"
    Si por lo que sea todavía no lo has revertido: quita el `throw new RuntimeException("Fallo simulado");` que has añadido en el Paso 2 y el Paso 4 para el quinto error, y asegúrate de que `cascade = CascadeType.ALL, orphanRemoval = true` está de nuevo en la relación `@OneToMany` de `Estudio.java` (la del cuarto error). Ninguno de los dos debe quedarse así en tu código.

Tu API ya no confía ciegamente en lo que le llega, y sus errores tienen un formato coherente que no filtra detalles internos. En la próxima actividad empiezas con Spring Security: usuarios en memoria y HTTP Basic — la primera barrera de acceso real.
