<a id="websocket-gamevault"></a>

# 🧩 2. WebSocket en GameVault: el canal de actividad en vivo

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "WebSocket en GameVault: el canal de actividad en vivo"
del Tema 4 (RA3 - Programación de comunicaciones en red) del módulo Programación de
Servicios y Procesos (0490), semana real 18 del calendario. Sigue las convenciones de
estilo del README.md del repo.

Este apartado NO cubre criterios de RA3 que estén pendientes: los criterios c) Librerías
y mecanismos del lenguaje para programar aplicaciones en red, e) Sockets para programar
un cliente que se comunica con un servidor, y f) Aplicación servidor en red verificada
ya quedaron cubiertos de forma LITERAL en la semana 17 con sockets Java reales
(sockets-cliente-servidor.md + Actividad 4.1) — no dependas de este apartado para
justificarlos en la rúbrica. Este apartado es una AMPLIACIÓN de profundidad sobre el
mismo RA3 (mismo concepto de "canal bidireccional persistente" presentado la semana
pasada, ahora sobre un caso de aplicación real) y un puente hacia RA4 (servicios en
red): profundiza c, e, f sobre un protocolo de nivel más alto, pero no es necesario para
tenerlos cubiertos.

AVISO para quien redacte: GameVault (la referencia adjunta) no tiene NINGUNA
configuración WebSocket (no hay `spring-boot-starter-websocket` en el pom ni
`@EnableWebSocketMessageBroker`). A diferencia de todos los demás apartados del curso,
que enseñan algo comparando con un estado final ya construido en la referencia, aquí no
hay ningún `WebSocketConfig.java` de referencia contra el que contrastar lo que el
alumnado construye. Dilo explícitamente en el texto («en este apartado no vas a
encontrar en el proyecto adjunto un ejemplo ya hecho como en los temas anteriores: lo
construimos completamente guiado, paso a paso, sin red de seguridad de comparación»)
para que el alumnado no lo busque y crea que se ha perdido algo.

ESTRUCTURA — teoría primero: arranca con el problema general, sin WebSocket todavía:
¿cómo puede una página web enterarse de algo que acaba de pasar en el servidor? Explica
las soluciones históricas en orden (recargar la página; polling — preguntar cada X
segundos, con su coste; long polling en una frase) para que WebSocket aparezca como
respuesta a un problema real y no como tecnología caída del cielo.

Contenido central: WebSocket como el canal bidireccional persistente presentado en el
apartado 1 — un socket de verdad entre navegador y servidor, que empieza como una
petición HTTP (el handshake con la cabecera Upgrade) y se convierte en una conexión
permanente donde el SERVIDOR puede enviar sin que el cliente pregunte. Explica también
qué es STOMP desde cero: WebSocket solo da un tubo de bytes/mensajes sin formato — STOMP
es un protocolo simple por encima que añade la semántica de mensajería (destinos
/topic/..., suscribirse, enviar), reutilizando el modelo pub-sub que el alumnado ya
conoce de RabbitMQ y de los eventos del Tema 3.

IMPORTANTE — esto es una MEJORA que NO existe en la referencia adjunta: GameVault no
tiene ninguna configuración WebSocket (no hay spring-boot-starter-websocket en el pom ni
configuración @EnableWebSocketMessageBroker). El caso de uso elegido: emitir en vivo el
registro de actividad del módulo `actividad` (com/aleroig/gamevault/actividad/ —
ActividadService.registrar() ya guarda cada evento del catálogo; hoy solo puede
consultarse con GET /api/v1/actividad, en frío y solo para ADMIN).

Explica, con el código de ejemplo que la Actividad 4.2 construirá guiado:
- La dependencia spring-boot-starter-websocket y la clase de configuración
  (@Configuration + @EnableWebSocketMessageBroker, siguiendo el estilo del paquete
  config/): registerStompEndpoints con el endpoint /ws-actividad, y
  configureMessageBroker con el broker simple y el prefijo /topic.
- El flujo completo con un diagrama: cliente se conecta a /ws-actividad (handshake
  HTTP→WebSocket) → se suscribe a /topic/actividad → cuando alguien crea/borra un
  videojuego, el servidor publica en ese topic → TODOS los suscritos lo reciben al
  instante.
- El contraste con REST en una tabla (quién inicia, cuántas respuestas por petición,
  duración de la conexión, caso de uso típico) — y el paralelismo con RabbitMQ: mismo
  modelo pub-sub, distinto alcance (RabbitMQ entre módulos del backend; WebSocket entre
  backend y navegadores).
- Cómo probar un WebSocket sin escribir un frontend entero (criterio e): una página
  HTML mínima con un cliente STOMP en JavaScript, o Postman (soporta WebSocket/STOMP) —
  la Actividad 4.2 guiará una de las dos vías.

No entres todavía en la emisión desde ActividadService ni en la seguridad del handshake:
eso es el siguiente apartado, actividad-en-vivo-cierre.md.
```
