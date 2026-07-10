# 🧪 Actividad 4.3: Actividad en vivo desde `ActividadService` — cierre de RA3

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo. Es la actividad final del módulo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 4.3 del Tema 4 (RA3 - Programación de comunicaciones en red) del
módulo Programación de Servicios y Procesos (0490), semana real 19 del calendario —
actividad que CIERRA el RA3 y es la ÚLTIMA del módulo 0490. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado. Completa la MEJORA del canal de actividad en vivo
(Actividad 4.2).

Objetivo (RA3, criterios g, h, j — cierre del RA y del módulo): emitir el registro de
actividad real por /topic/actividad, evaluar la seguridad del canal Y CORREGIRLA con la
remediación mínima (exigir JWT en el handshake) — no basta con detectar y documentar el
agujero, hay que cerrarlo antes de dar el módulo por terminado.

Estructura sugerida de pasos guiados:
1. La emisión guiada: inyectar SimpMessagingTemplate en ActividadService (con
   @RequiredArgsConstructor, como todo el proyecto) y, en registrar(), publicar en
   /topic/actividad un DTO con los datos del registro (reutilizar o adaptar
   ActividadResponseDTO) — código mostrado y explicado. Retirar el emisor de prueba de
   la Actividad 4.2.
2. La demostración completa, guiada: abrir el cliente STOMP de la Actividad 4.2 en dos
   pestañas, crear un videojuego por Swagger/curl, y ver aparecer el registro EN VIVO en
   ambas pestañas — documentar con una captura. Trazar el viaje completo del dato con
   los nombres de hilo en el log (traza dada): hilo HTTP → hilo del listener de
   RabbitMQ → push WebSocket.
3. Mini-reto (repite el patrón del paso 2): comprobar que el update y el delete de
   videojuegos también aparecen en vivo (ya funcionan solos si el paso 1 se hizo en
   registrar() — el reto es explicar POR QUÉ funcionan sin tocar nada más, siguiendo el
   flujo RabbitMQ → consumer → registrar()).
4. La auditoría de seguridad guiada (el "aviso del handshake" del calendario): con una
   ventana de incógnito (sin login), conectar al WebSocket y suscribirse — comprobar que
   un anónimo ve en vivo lo que GET /api/v1/actividad solo permite a ADMIN. Con ayuda
   del enunciado, escribir el hallazgo como una mini-incidencia de seguridad (qué se
   expone, a quién, opciones de mitigación).
5. REMEDIACIÓN MÍNIMA OBLIGATORIA (entregable, no opcional — cambio respecto a versiones
   anteriores de este prompt): aplicar guiadamente la solución mostrada en la teoría de
   actividad-en-vivo-cierre.md — exigir un token JWT válido en el handshake de
   /ws-actividad (por ejemplo, pasado como parámetro de consulta y validado en un
   interceptor de handshake antes de aceptar la conexión), con el código mostrado y
   explicado paso a paso. Repetir la prueba con la ventana de incógnito del paso 4 y
   comprobar que ahora el handshake es rechazado sin token válido. El cierre del módulo
   de Programación Segura no puede terminar con un agujero documentado pero sin
   corregir.
6. Cierre del módulo: un repaso propio (5-6 líneas) del recorrido completo de PSP sobre
   su GameVault — API REST probada y monitorizada (RA4), asegurada con JWT y roles
   (RA5), con tareas en segundo plano (RA2) y ahora con comunicación en vivo (RA3) — y
   qué pieza le ha costado más y por qué (respuesta personal, no genérica).
```
