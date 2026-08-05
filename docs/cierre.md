# 🏁 Cierre del módulo

## Lo que has construido

A lo largo de Programación de Servicios y Procesos, GameVault ha ido ganando una capa de comportamiento distinta en cada tema, siempre sobre el mismo proyecto:

| Tema | Qué has añadido |
|---|---|
| **Tema 0** | Repaso de POO, colecciones y programación funcional en Java — la base sobre la que se apoya todo lo demás. |
| **Tema 1** | La API REST completa: lectura y escritura sobre `Videojuego` y `Estudio`, documentada con OpenAPI/Swagger, probada con MockMvc, y medida bajo peticiones simultáneas con Actuator. |
| **Tema 2** | Seguridad de extremo a extremo: validación y `GlobalExceptionHandler`, HTTP Basic, usuarios persistidos con BCrypt, login con JWT, y roles sobre rutas protegidas. |
| **Tema 3** | Tareas en segundo plano con hilos reales: RabbitMQ y el registro de actividad, eventos internos y listeners `@Async` para el warm-up de caché, y un `TaskExecutor` propio con prioridades. |
| **Tema 4** | Comunicación en red: sockets clásicos cliente/servidor, WebSocket con STOMP para el canal en vivo, y su seguridad en el handshake. |

Un mismo proyecto, una API que ha pasado de leerse a mano con `curl` a estar asegurada, monitorizada, con tareas asíncronas y comunicación en tiempo real.

---

## Huecos reales que quedan (y por qué)

Tu GameVault, tal como queda hoy, no está completo — y no es un descuido. El objetivo del módulo no era que repitieras la misma tarea (un test más, un endpoint más, una regla de seguridad más) sobre cada rincón del proyecto hasta agotarlo: era que entendieras cada patrón una vez, lo aplicaras un par de veces con criterio propio, y supieras reconocerlo la próxima vez que haga falta. Eso deja huecos a propósito — estos son reales, verificados sobre tu propio código, no hipotéticos:

- **Tests que no se actualizaron cuando llegó la seguridad**: `VideojuegoControllerTest` y `EstudioControllerTest` (Actividad 1.3) usan `@AutoConfigureMockMvc(addFilters = false)`, que desactiva por completo los filtros de Spring Security. Se escribieron antes de que el proyecto tuviera ninguna seguridad, y nunca se han vuelto a tocar para añadir un caso `401` (sin token) o `403` (sin rol `ADMIN`) — a pesar de que `create`, `update` y `delete` exigen rol `ADMIN` desde la Actividad 2.5. El único sitio del proyecto que sí prueba un `403` real es `VideojuegoApiIntegrationTest` (Acceso a Datos, Actividad 2.3), y solo para `update` de `Videojuego`: ni `EstudioController`, ni `create`/`delete` de `Videojuego`, ni ningún endpoint de `Review` tienen un test que compruebe quién puede hacer qué. `AuthController` y `ActividadController` no tienen ningún test propio.
- **`docs/seguridad.md`** (la tabla de políticas de seguridad que empezaste en la Actividad 2.5): a día de hoy no incluye las rutas de `Review` (Acceso a Datos, Tema 3) — sí has añadido ya, en la Actividad 4.3, la fila de `/ws-actividad`. Un documento de seguridad que no se actualiza al mismo ritmo que el código deja de ser de fiar, y aquí tienes la prueba de que sí se puede mantener al día.
- **Casos de error sin cubrir**: por ejemplo, no hay ningún test que compruebe qué ocurre si el `JwtHandshakeInterceptor` recibe un token caducado en vez de ausente, un matiz distinto al que sí has probado.

Ninguno de estos huecos es un error que tengas que arreglar para aprobar. Si quieres, son el ejercicio que sigue: coge uno, ciérralo tú mismo aplicando exactamente lo que ya sabes, y comprueba si todavía necesitas que alguien te lo explique paso a paso o si ya te basta con el criterio que has construido durante el curso.

---

## Reflexión final

Has trabajado sobre un proyecto real, no sobre ejercicios sueltos y desconectados entre sí — con las ventajas de eso (el contexto se acumula, las decisiones de un tema condicionan al siguiente) y también con sus inconvenientes (nada queda perfecto ni completo, exactamente como en cualquier proyecto real fuera de clase). Programación de Servicios y Procesos termina aquí, pero GameVault sigue siendo tuyo: nada te impide, si te interesa, seguir cerrando huecos como los de arriba por tu cuenta.
