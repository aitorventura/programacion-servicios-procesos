<a id="sockets-cliente-servidor"></a>

# 🧩 1. Sockets: la base de toda comunicación en red

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Sockets: la base de toda comunicación en red" del Tema 4
(RA3 - Programación de comunicaciones en red) del módulo Programación de Servicios y
Procesos (0490), semana real 17 del calendario. Sigue las convenciones de estilo del
README.md del repo. Este apartado es principalmente teórico y sus ejercicios se hacen en
pequeños programas de consola FUERA de GameVault (así lo marca el calendario: "sin
proyecto") — pero usa GameVault como referencia constante de "esto ya lo estás usando
sin saberlo".

Criterios de evaluación de RA3 que cubre este apartado (curriculum.md):
- a) Escenarios que precisan comunicación en red entre aplicaciones.
- b) Roles de cliente y de servidor y sus funciones.
- d) Concepto de socket, tipos y características.
- i) Modelos de comunicación en arquitecturas distribuidas.

Contenido central:
- Qué es un socket: el extremo programable de una conexión de red (IP + puerto),
  la pieza que hay DEBAJO de todo lo que el alumnado ya ha usado — cuando curl habla con
  GameVault en el puerto 8080, cuando la aplicación se conecta a PostgreSQL (5432),
  MongoDB (27017) o RabbitMQ (5672), hay sockets TCP por debajo. Usa los puertos reales
  del docker-compose.yaml de GameVault para hacerlo tangible, y una comprobación en vivo
  con netstat/ss (comando dado) mostrando las conexiones establecidas de la aplicación.
- Roles cliente y servidor (criterio b): quién escucha (bind/listen/accept — el servidor,
  como Tomcat en el 8080) y quién inicia (connect — el cliente); en Java clásico:
  ServerSocket (servidor) y Socket (cliente), con los streams de entrada/salida para
  intercambiar datos.
- Tipos de socket (criterio d): stream/TCP (orientado a conexión, fiable — todo lo que
  usa GameVault) frente a datagrama/UDP (sin conexión, sin garantía — dónde se usa:
  streaming, juegos en tiempo real), en una tabla comparativa.
- Modelos de comunicación (criterio i), cada uno anclado a algo que el alumnado ya ha
  visto en el curso: petición-respuesta síncrono (REST, Temas 1 de PSP y todo AD),
  publicación-suscripción asíncrono (RabbitMQ, Tema 3), y el que falta por conocer:
  el canal bidireccional persistente (WebSocket, próxima semana). Un diagrama comparando
  los tres.
- Escenarios (criterio a): ¿cuándo NO basta petición-respuesta? Cuando el SERVIDOR tiene
  que avisar al cliente de algo que acaba de pasar sin que este pregunte — presenta el
  caso concreto del proyecto: el panel de actividad en vivo de GameVault que se
  construirá las semanas 18-19.

Cierra anticipando la práctica (Actividad 4.1): un mini cliente/servidor de consola con
ServerSocket/Socket e hilos, para tocar con las manos lo que Spring normalmente esconde.
Deja explícito que la Actividad 4.1, con sockets reales y multihilo, cubre de forma
LITERAL los criterios b, c, d, e, f, g y h de RA3 — no son un "aperitivo" antes de la
cobertura real vía WebSocket: son ya la cobertura real. Las semanas 18-19 (WebSocket)
amplían el mismo RA3 sobre un caso de aplicación real, pero no son necesarias para tener
el RA3 cubierto.
```
