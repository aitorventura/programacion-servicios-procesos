# 🧪 Actividad 4.1: Cliente y servidor con sockets clásicos

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 4.1 del Tema 4 (RA3 - Programación de comunicaciones en red) del
módulo Programación de Servicios y Procesos (0490), semana real 17 del calendario. Si
necesita plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. Esta actividad se hace en un
pequeño proyecto de consola FUERA de GameVault (el calendario marca esta semana "sin
proyecto"): dos clases Java sueltas bastan. El enunciado debe guiar paso a paso,
mostrando el código y explicando cada decisión; solo se deja sin guiar, como mini-reto,
lo que repita un patrón idéntico ya mostrado.

Objetivo (RA3, criterios c, d, e, f, h): programar un servidor y un cliente con sockets
TCP clásicos de Java, y hacer que el servidor atienda a varios clientes con hilos
(conectando con lo aprendido en el Tema 3).

Estructura sugerida de pasos guiados:
1. El servidor guiado al completo, código mostrado y explicado línea a línea: un
   ServerSocket escuchando en un puerto (por ejemplo 5000), accept() en bucle, y por
   cada cliente un eco simple (leer línea, responder "ECO: " + línea) usando
   BufferedReader/PrintWriter sobre los streams del socket — versión monohilo primero,
   señalando el problema: mientras atiende a un cliente, accept() no recibe a nadie más.
2. El cliente guiado: un Socket que se conecta, envía líneas leídas de teclado y muestra
   las respuestas — probarlo contra el servidor (dos terminales, comandos dados).
3. Demostración guiada de la limitación monohilo: abrir un segundo cliente mientras el
   primero está conectado y observar que se queda esperando.
4. La versión multihilo guiada: mover la atención al cliente a un Runnable y lanzar un
   hilo por conexión (o un ExecutorService, conectando con el Tema 3) — código mostrado;
   repetir la prueba de dos clientes simultáneos y observar la diferencia, con una traza
   del nombre del hilo por cliente.
5. Mini-reto (repite el patrón ya visto): ampliar el protocolo del eco con un comando
   "SALIR" que cierre limpiamente la conexión de ese cliente (cerrar streams y socket) —
   solo se indica el objetivo.
6. Pregunta de comprensión (criterio h): ¿qué relación hay entre este "un hilo por
   cliente" y lo que hace Tomcat con las peticiones HTTP de GameVault? ¿Y con lo que se
   vio del pool de hilos en el Tema 3? (la respuesta esperada: es el mismo patrón,
   gestionado a mano aquí y por el framework allí).

Cierre explícito para la rúbrica: señala en el propio enunciado que, al terminar esta
actividad, quedan cubiertos de forma LITERAL (con `java.net.Socket`/`ServerSocket`
reales, sin adaptación) los criterios b, c, d, e, f, g y h de RA3 — es la actividad que
sostiene la evaluación de RA3, no un ejercicio preparatorio menor frente a las semanas
de WebSocket.
```
