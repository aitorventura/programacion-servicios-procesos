<a id="actividad-en-vivo-cierre"></a>

# 🧩 3. Actividad en vivo y seguridad del canal — cierre de RA3

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo. Este apartado cierra el RA3 y el módulo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Actividad en vivo y seguridad del canal — cierre de RA3"
del Tema 4 (RA3 - Programación de comunicaciones en red) del módulo Programación de
Servicios y Procesos (0490), semana real 19 del calendario — apartado que CIERRA el RA3
y el módulo 0490. Sigue las convenciones de estilo del README.md del repo.

Criterios de evaluación de RA3 que cubre este apartado (curriculum.md) — g) y h) ya
quedaron cubiertos de forma literal en la Actividad 4.1 con sockets reales; aquí se
amplían sobre el canal WebSocket como profundización, no como cobertura mínima:
- g) Aplicaciones que utilizan sockets para intercambiar información (ampliación sobre
  WebSocket/STOMP).
- h) Hilos para la comunicación simultánea de varios clientes (ampliación sobre
  WebSocket/STOMP).
- j) Depuración y documentación de las aplicaciones.

Contenido central: conectar el canal /ws-actividad (montado la semana pasada) con los
datos reales del proyecto, y entender qué implica en seguridad haber abierto ese canal.

Apóyate en el proyecto GameVault (com.aleroig.gamevault) como ejemplo real:
- El punto de emisión: com/aleroig/gamevault/actividad/ActividadService.java —
  su método registrar() ya recibe cada evento del catálogo (a través del consumer de
  RabbitMQ, como se analizó en el Tema 3). El plan: inyectar ahí SimpMessagingTemplate y,
  además de guardar en la base de datos, publicar el registro en /topic/actividad.
  Recorre el viaje completo de un dato con un diagrama — es el broche del curso porque
  atraviesa TODO lo construido: petición HTTP (hilo de Tomcat) → VideojuegoService →
  evento RabbitMQ → consumer (hilo del listener) → ActividadService.registrar() →
  guardado en PostgreSQL + push por WebSocket → navegadores suscritos.
- Hilos y clientes simultáneos (criterio h): ¿en qué hilo se ejecuta el push? (el del
  listener de RabbitMQ) ¿y quién gestiona los N clientes WebSocket conectados? (el
  contenedor, igual que Tomcat con HTTP) — conectar explícitamente con el "un hilo por
  cliente" programado a mano en la Actividad 4.1: mismo problema, solución gestionada
  por el framework.
- El aviso de seguridad del handshake (el calendario lo pide expresamente): la regla
  añadida en la Actividad 4.2 abre /ws-actividad, pero el GET /api/v1/actividad es solo
  ADMIN (SecurityConfig.java) — hay una incoherencia que discutir: ¿puede un anónimo
  suscribirse al topic y ver en vivo lo que por REST requiere rol ADMIN? Explica el
  problema (el handshake WebSocket no lleva el JWT por defecto; asegurar STOMP requiere
  configuración adicional) y las opciones (restringir el handshake, filtrar qué se
  emite, o asumirlo y documentarlo como decisión consciente para un canal de
  demostración). CAMBIO IMPORTANTE respecto a versiones anteriores de este prompt: no
  basta con "ver y documentar" el agujero sin remediarlo — cerrar el bloque de RA5
  (Programación Segura) dejando una vulnerabilidad real sin corregir manda el mensaje
  contrario al RA. Exige como entregable obligatorio (no opcional) la remediación
  MÍNIMA: exigir el JWT también en el handshake del WebSocket, restringiendo
  `registerStompEndpoints`/la regla de SecurityConfig para que el handshake a
  /ws-actividad requiera un token válido (por ejemplo, pasándolo como parámetro de
  consulta en la URL de conexión y validándolo en un interceptor de handshake) —
  código mostrado y explicado en la teoría, para que en la Actividad 4.3 el alumnado
  solo tenga que aplicarlo guiado. Documentar (criterio j) sigue siendo parte del
  entregable, pero además de la remediación mínima, no en su lugar.
- Depuración y documentación (criterio j): trazas con nombre de hilo en el camino
  completo (herencia de los Temas 3) y documentar el canal (endpoint, topic, formato del
  mensaje, política de acceso) como se documentó la API REST con OpenAPI — señalando que
  WebSocket no aparece en Swagger y por eso se documenta a mano.

Cierra recapitulando el RA3 (sockets a mano → WebSocket/STOMP → emisión real y
seguridad) y el módulo completo en un párrafo final: servicios en red (RA4) → seguridad
(RA5) → multihilo (RA2) → comunicaciones en red (RA3), todo sobre el mismo GameVault que
el alumnado ha construido también en Acceso a Datos.
```
