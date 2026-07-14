<a id="principios-programacion-segura"></a>

# 🧩 1. Principios de programación segura: validación y gestión de errores

Tu proyecto, tal y como está ahora, no comprueba quién hace las peticiones ni les pone ningún límite. Antes de tocar autenticación (eso empieza el apartado que viene), este apartado sienta las bases: qué significa que una aplicación sea "segura" y cómo se traduce eso en decisiones concretas de código, sin necesitar todavía ni un login.

---

## 🔒 Qué significa que una aplicación sea "segura"

La seguridad de una aplicación se apoya, clásicamente, en tres pilares:

| Pilar | En una aplicación web |
|---|---|
| **Confidencialidad** | Que nadie lea lo que no debe (datos de otro usuario, contraseñas...). |
| **Integridad** | Que nadie modifique lo que no debe (cambiar el precio de un producto ajeno, alterar un pedido). |
| **Disponibilidad** | Que el servicio siga funcionando (no se caiga por un ataque o un uso malintencionado). |

---

## 🎯 El modelo mental del atacante

Toda API tiene una **superficie de ataque**: todo endpoint expuesto, todo dato de entrada que acepta, todo mensaje de error que devuelve — cualquier punto por el que algo externo puede interactuar con tu sistema es, potencialmente, un punto de ataque.

!!! warning "La regla de oro"
    **Nunca te fíes de lo que llega de fuera.** Todo dato de entrada es hostil hasta que se valida — no porque asumas mala fe de cada usuario, sino porque no controlas qué envía realmente el cliente: puede ser un formulario mal rellenado, un bug del cliente, o alguien probando deliberadamente a romper tu API.

---

## 🛡️ Principios generales de programación segura

- **Validar en la frontera**: comprobar los datos de entrada en el punto donde entran al sistema, antes de que lleguen a la lógica de negocio — no confiar en que "ya vendrán bien" desde el cliente.
- **Mínimo privilegio / mínima exposición**: por defecto, todo cerrado; se abre explícitamente solo lo necesario — no al revés.
- **No filtrar información interna en los errores**: un mensaje de error no debería revelar detalles de implementación (nombres de tablas, rutas del servidor, versiones de librerías) que ayuden a un atacante.
- **No guardar secretos en el código**: contraseñas, claves, tokens — nunca escritos directamente en una clase Java; viven en configuración externa.
- **Defensa en profundidad**: varias capas de protección, no una sola barrera — si una falla, las siguientes siguen ahí.

Estos principios no son abstractos: detrás de cada uno hay un ataque real que previenen. Un vistazo rápido, sin profundizar todavía (irás viendo cada uno con más detalle):

- **Inyección**: ya la viste de primera mano en Acceso a Datos, con JDBC puro y `Statement` — código que trata datos externos como si fueran código.
- **Datos malformados**: entradas que rompen la lógica de la aplicación si no se comprueban antes de usarlas.
- **Fuerza bruta sobre logins**: probar contraseñas una tras otra hasta acertar.
- **Robo de credenciales en tránsito**: interceptar datos sensibles mientras viajan por la red.

---

## 🛠️ Dos prácticas concretas

La seguridad no empieza en el login — empieza mucho antes: en no fiarte nunca de la entrada del usuario y en no filtrar información en los errores. Siguiendo con la API de la librería que ya conoces:

### Validación de entrada con Bean Validation

El `create(...)` del controller usa `@Valid` sobre su DTO de entrada, y `LibroCreateDTO` lleva las anotaciones de `jakarta.validation`:

```java
public record LibroCreateDTO(
        @NotBlank(message = "El título no puede estar vacío")
        @Size(max = 150)
        String titulo,

        @NotNull
        @PositiveOrZero(message = "El precio no puede ser negativo")
        BigDecimal precio,
        // ...
) {}
```

`@Valid`, en la firma del método del controller, activa la comprobación de estas anotaciones **antes** de que el cuerpo del método se ejecute — si algo no cumple, ni siquiera llega a tu lógica de negocio. Esto es "validar en la frontera" hecho código: el dato hostil se rechaza en la puerta, no dentro de la casa.

### Gestión centralizada de errores

Sin nada más, cuando una validación falla, Spring devuelve una respuesta por defecto — funcional, pero poco cuidada y potencialmente reveladora. La solución es centralizar la gestión de errores en un único punto:

```java
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
                LocalDateTime.now().toString(), 400, "Error de validación",
                "La petición contiene campos inválidos", request.getRequestURI(), fields
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ErrorResponse> handleResponseStatusException(
            ResponseStatusException ex, HttpServletRequest request) {
        // convierte cualquier ResponseStatusException (los 404 que ya conoces) en el mismo formato
    }
}
```

`@RestControllerAdvice` marca esta clase como un punto **transversal**: se aplica a todos los controllers de la aplicación, no a uno concreto. `@ExceptionHandler(TipoDeExcepcion.class)` declara qué método atrapa qué tipo de excepción. El resultado: sea cual sea el error (una validación fallida, un `ResponseStatusException` de "no encontrado"...), la respuesta siempre tiene la misma forma — un `ErrorResponse` con `timestamp`, `status`, `error`, `message` y `path`.

!!! danger "Lo que este handler evita filtrar"
    Sin él, una excepción no controlada puede acabar devolviendo al cliente una traza de pila completa: nombres de clases internas, de tablas, incluso versiones de librerías. Eso es información gratis para un atacante sobre cómo está construido tu sistema por dentro. `GlobalExceptionHandler` garantiza que, pase lo que pase, el cliente solo ve el mensaje controlado que tú decides — nunca los detalles internos.

Esto conecta con algo que ya conocías: los `404` de `ResponseStatusException` que viste en el Tema 1 pasan también por este mismo handler — es la misma política de errores, ahora aplicada de forma uniforme a toda la aplicación.

---

## 🗺️ Lo que viene

El resto de este tema va a materializar, uno a uno, los principios que has visto hoy: mínima exposición se convierte en rutas cerradas por defecto ("Roles, rutas protegidas y tests de seguridad"), no guardar secretos en el código se convierte en el `jwt.secret` como propiedad externa ("Autenticación con JWT"), no almacenar contraseñas en claro se convierte en BCrypt ("Usuarios persistidos y BCrypt"). Hoy has visto los principios; en los próximos apartados les pones nombre y apellido en Spring Security.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Los tres pilares clásicos de la seguridad: **confidencialidad**, **integridad**, **disponibilidad**.
    - La **superficie de ataque** es todo punto por el que algo externo interactúa con tu sistema; la regla de oro es no fiarse nunca de los datos de entrada.
    - Principios clave: validar en la frontera, mínima exposición, no filtrar información en los errores, no guardar secretos en el código, defensa en profundidad.
    - `@Valid` + anotaciones de `jakarta.validation` en los DTOs validan la entrada antes de que llegue a la lógica de negocio.
    - `@RestControllerAdvice` + `@ExceptionHandler` centralizan la conversión de excepciones en respuestas HTTP coherentes, sin filtrar detalles internos.
