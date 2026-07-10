<a id="principios-programacion-segura"></a>

# 🧩 1. Principios de programación segura: validación y gestión de errores

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Principios de programación segura: validación y gestión
de errores" del Tema 2 (RA5 - Programación segura) del módulo Programación de Servicios
y Procesos (0490), semana real 7 del calendario. Sigue las convenciones de estilo del
README.md del repo.

Criterio de evaluación de RA5 que cubre este apartado (curriculum.md):
- a) Principios y prácticas de programación segura.

ESTRUCTURA OBLIGATORIA — teoría primero, proyecto después. El alumnado no ha visto
nunca seguridad informática aplicada al desarrollo.

PARTE 1 — Teoría general, desde cero:
- Qué significa que una aplicación sea "segura": los tres pilares clásicos
  (confidencialidad, integridad, disponibilidad) explicados con ejemplos cotidianos de
  una aplicación web (que nadie lea lo que no debe, que nadie modifique lo que no debe,
  que el servicio siga funcionando).
- El modelo mental del atacante: qué es la superficie de ataque de una API (todo
  endpoint expuesto, todo dato de entrada, todo mensaje de error) y la regla de oro:
  NUNCA te fíes de lo que llega de fuera — todo dato de entrada es hostil hasta que se
  valide.
- Los principios generales de programación segura, cada uno definido en 2-3 frases con
  un ejemplo genérico: validar en la frontera, mínimo privilegio / mínima exposición
  (cerrado por defecto), no filtrar información interna en los errores, no guardar
  secretos en el código, defensa en profundidad (varias capas, no una sola barrera).
- Un vistazo a los ataques más comunes contra una API para que los principios no queden
  abstractos (una frase cada uno): inyección (ya la demostraron en AD con JDBC y
  Statement), datos malformados que rompen la lógica, fuerza bruta sobre logins, robo de
  credenciales en tránsito.

PARTE 2 — Aterrizaje en GameVault: la seguridad no empieza en el login — empieza en no
fiarse nunca de la entrada del usuario y en no filtrar información en los errores. Dos
prácticas concretas sobre el proyecto real: validación de entrada con Bean Validation y
gestión centralizada de errores. Apóyate en:
- Validación de entrada: la anotación `@Valid` que ya usan VideojuegoController.java y
  VideojuegoReviewController.java sobre sus DTOs de entrada, y las anotaciones de
  jakarta.validation en los propios DTOs (ábrelos y usa las que realmente tengan:
  revisa VideojuegoCreateDTO.java y ReviewRequestDTO.java) — explica el principio
  "valida en la frontera": ningún dato del exterior entra sin pasar el filtro.
- Gestión centralizada de errores: com/aleroig/gamevault/exception/
  GlobalExceptionHandler.java y ErrorResponse.java — explica `@RestControllerAdvice` /
  `@ExceptionHandler` (un único punto donde las excepciones se convierten en respuestas
  HTTP coherentes) y el principio de seguridad detrás: no devolver al cliente trazas de
  pila ni mensajes internos (¿qué aprendería un atacante de un stacktrace con nombres de
  tablas y versiones de librerías?).
- Conecta con lo que el alumnado ya ha visto: los 404 de ResponseStatusException del
  Tema 1 y el ErrorResponse uniforme son parte de la misma política de errores.
- Añade una panorámica breve de otros principios de programación segura aplicados al
  proyecto, cada uno en 2-3 frases y señalando dónde se materializará en las próximas
  semanas: mínima exposición (rutas cerradas por defecto — semana 11), no guardar
  secretos en el código (el jwt.secret como propiedad externa — semana 10), no almacenar
  contraseñas en claro (BCrypt — semana 9), y la inyección SQL ya demostrada en AD con
  JDBC puro (referencia cruzada).

No entres todavía en Spring Security ni autenticación: eso empieza en el siguiente
apartado, seguridad-basica-http-basic.md.
```
