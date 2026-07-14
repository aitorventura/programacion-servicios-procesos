# 🧪 Actividad 2.1: Validación y `GlobalExceptionHandler`

!!! info "Práctica guiada"
    Vas a construir en tu GameVault la gestión centralizada de errores, viendo primero por qué la respuesta por defecto de Spring no es suficiente — aunque la validación de entrada ya la tenías, hoy la haces más útil de cara al cliente.

## Qué vas a practicar

- Personalizar los mensajes de la validación con Bean Validation que ya tienes en un DTO de entrada.
- Ver la diferencia entre la respuesta de error por defecto de Spring y una gestionada.
- Construir un `@RestControllerAdvice` centralizado.

---

## Requisitos previos

Tu CRUD de `Videojuego` y de `Estudio` funcionando, con validación y `@Valid` (Acceso a Datos, Tema 1).

---

## Paso 1 — Mensajes de error personalizados en `VideojuegoCreateDTO`

Tu DTO ya tiene validación — la construiste en Acceso a Datos (Tema 1), con `@Valid` ya puesto en el controller. Pero cada anotación usa el mensaje por defecto de Bean Validation, genérico y en inglés (`"must not be blank"`, por ejemplo). Añade un mensaje propio, descriptivo, con el atributo `message`:

```java
public record VideojuegoCreateDTO(
        @NotBlank(message = "El título no puede estar vacío")
        @Size(max = 150, message = "El título no puede superar los 150 caracteres")
        String titulo,

        @NotNull(message = "El precio es obligatorio")
        @PositiveOrZero(message = "El precio no puede ser negativo")
        BigDecimal precio,

        @NotNull(message = "La fecha de lanzamiento es obligatoria")
        @PastOrPresent(message = "La fecha de lanzamiento no puede ser futura")
        LocalDate fechaLanzamiento,

        @NotNull(message = "El ID del estudio es obligatorio")
        @Positive(message = "El ID del estudio debe ser positivo")
        Long estudioId
) {}
```

Este mensaje es justo lo que vas a recuperar dentro de poco, en el `GlobalExceptionHandler` del Paso 3, con `error.getDefaultMessage()` — sin un mensaje propio, ese campo sería igual de genérico y poco útil.

---

## Paso 2 — Ver la respuesta por defecto

Provoca un error de validación real:

```bash
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"","precio":-5,"fechaLanzamiento":null,"estudioId":-1}'
```

**Observa** la respuesta completa (con `-i` ves las cabeceras). Deberías obtener un `400`, pero con un cuerpo genérico de Spring.

**Preguntas**: ¿el mensaje de error te dice exactamente qué campo ha fallado y por qué? ¿Tiene un formato coherente con el resto de errores de tu API (por ejemplo, el `404` de "videojuego no encontrado")?

---

## Paso 3 — `GlobalExceptionHandler`, guiado al completo

Crea el paquete `exception` con `ErrorResponse`:

```java
package com.tunombre.gamevault.exception;

import java.util.Map;

public record ErrorResponse(
        String timestamp,
        int status,
        String error,
        String message,
        String path,
        Map<String, String> fields
) {
    public ErrorResponse(String timestamp, int status, String error, String message, String path) {
        this(timestamp, status, error, message, path, Map.of());
    }
}
```

Y `GlobalExceptionHandler`:

```java
package com.tunombre.gamevault.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.server.ResponseStatusException;

import java.time.LocalDateTime;
import java.util.LinkedHashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> fields = new LinkedHashMap<>();
        for (FieldError error : ex.getBindingResult().getFieldErrors()) {
            fields.putIfAbsent(error.getField(), error.getDefaultMessage());
        }

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
}
```

`@RestControllerAdvice` hace que esta clase intercepte excepciones de **todos** tus controllers, no solo de uno. Cada `@ExceptionHandler` decide qué tipo de excepción atrapa y cómo la convierte en un `ErrorResponse` uniforme.

---

## Paso 4 — Repetir la petición y comparar

Repite exactamente la misma petición del Paso 2:

```bash
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"","precio":-5,"fechaLanzamiento":null,"estudioId":-1}'
```

**Compara** ambas respuestas (la del Paso 2 y esta) campo a campo. **Responde**: ¿qué información nueva aparece ahora que antes no estaba (pista: el campo `fields`)? ¿Qué información que antes podía filtrarse (si hubiera sido una excepción no controlada, no una de validación) ya no puede llegar al cliente?

---

## Mini-reto — mensajes personalizados en `EstudioCreateDTO`

Repite el patrón del Paso 1, esta vez sobre `EstudioCreateDTO` (paquete `catalogo.dto`): ya tiene `@NotBlank` en `nombre` y en `pais` (Acceso a Datos, Tema 1), pero sin mensaje propio. Añádeselo a los dos. Provoca el error con una petición POST inválida a `/api/v1/estudios` y comprueba que `GlobalExceptionHandler` también lo captura, con tus mensajes nuevos en el campo `fields` — sin tener que tocar ni una línea de la clase que acabas de construir.

---

## Pregunta final

¿Por qué conviene que **todas** las excepciones de la aplicación pasen por un único punto (`GlobalExceptionHandler`) antes de convertirse en respuesta HTTP, en vez de gestionar cada error donde ocurre? Nombra un dato interno concreto de tu propio proyecto (una clase, una tabla, una ruta de fichero) que este handler evita que se filtre a quien esté probando tu API desde fuera.

---

## ✅ Cierre

Tu API ya no confía ciegamente en lo que le llega, y sus errores tienen un formato coherente que no filtra detalles internos. En la próxima actividad empiezas con Spring Security: usuarios en memoria y HTTP Basic — la primera barrera de acceso real.
