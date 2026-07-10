# 🧪 Actividad 4.2: El endpoint `/ws-actividad` con STOMP

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 4.2 del Tema 4 (RA3 - Programación de comunicaciones en red) del
módulo Programación de Servicios y Procesos (0490), semana real 18 del calendario. Si
necesita plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado. Es una MEJORA que no existe en la referencia adjunta: a
diferencia de otras actividades del curso, aquí no hay ningún fichero de referencia con
el que comparar el resultado — dilo explícitamente en el enunciado para que el
alumnado no lo busque.

Objetivo (RA3, criterios c, e, f — de profundización, ya cubiertos literalmente en la
Actividad 4.1 con sockets reales; esta actividad los amplía sobre un protocolo de más
alto nivel, no los sustituye): montar en su GameVault el endpoint WebSocket
/ws-actividad con STOMP y verificarlo con un cliente, enviando un primer mensaje de
prueba.

Estructura sugerida de pasos guiados:
1. La dependencia spring-boot-starter-websocket en el pom (fragmento dado) y la clase de
   configuración WebSocketConfig en config/ (código mostrado y explicado:
   @EnableWebSocketMessageBroker, endpoint /ws-actividad en registerStompEndpoints,
   broker simple con prefijo /topic en configureMessageBroker).
2. Atención a la seguridad, guiado: con el anyRequest().denyAll() del Tema 2, el
   handshake a /ws-actividad quedará bloqueado — añadir la regla que lo permite en
   SecurityConfig (dada y explicada; la discusión de fondo sobre qué implica abrirlo
   llega en la Actividad 4.3).
3. Un endpoint de prueba guiado para poder emitir algo: un pequeño método (por ejemplo
   en un controller de prueba, o directamente con SimpMessagingTemplate desde un
   endpoint temporal) que publique un mensaje fijo en /topic/actividad — código dado.
4. El cliente guiado (elegir UNA vía y guiarla al completo): página HTML mínima con
   STOMP.js servida en estático (código completo dado, con cada parte comentada:
   conexión a /ws-actividad, suscripción a /topic/actividad, pintar lo recibido), o
   Postman si se prefiere sin código. Conectar, lanzar el mensaje de prueba del paso 3 y
   verlo llegar.
5. Demostración guiada del pub-sub (criterio f): abrir el cliente en DOS pestañas a la
   vez, lanzar un mensaje y verlo llegar a ambas — la diferencia clave con
   petición-respuesta, observada en directo.
6. Pregunta de comprensión: en la pestaña de red del navegador (F12), localizar el
   handshake de /ws-actividad — ¿qué código de estado tiene y qué cabecera lo delata?
   (101 Switching Protocols, Upgrade) ¿En qué se diferencia de todas las peticiones que
   habéis hecho en el curso hasta hoy?
```
