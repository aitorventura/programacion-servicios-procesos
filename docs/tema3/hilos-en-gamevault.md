<a id="hilos-en-gamevault"></a>

# 🧩 1. Hilos en una aplicación real: dónde ya los estás usando

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Hilos en una aplicación real: dónde ya los estás usando"
del Tema 3 (RA2 - Programación multihilo) del módulo Programación de Servicios y
Procesos (0490), semana real 12 del calendario. Sigue las convenciones de estilo del
README.md del repo. Esta semana es de ANÁLISIS: no se escribe código nuevo, se aprende a
ver los hilos que ya existen en el proyecto.

Criterios de evaluación de RA2 que cubre este apartado (curriculum.md):
- a) Situaciones en las que resulta útil utilizar varios hilos.
- b) Mecanismos para crear, iniciar y finalizar hilos.
- i) Análisis del contexto de ejecución de los hilos.

ESTRUCTURA OBLIGATORIA — teoría primero, proyecto después. El alumnado NO ha visto
nunca programación concurrente: la PARTE 1 es la base teórica de TODO el tema y debe ser
la más extensa del apartado.

PARTE 1 — Teoría fundamental de hilos, desde cero y con ejemplos genéricos de consola
(sin GameVault todavía):
- Proceso vs. hilo: qué es un proceso (un programa en ejecución con su memoria propia —
  la JVM entera es uno; compruébalo con el administrador de tareas) y qué es un hilo
  (una línea de ejecución DENTRO del proceso, que comparte la memoria con los demás
  hilos). Diagrama proceso-con-varios-hilos.
- Concurrencia vs. paralelismo: turnarse en un núcleo vs. ejecutar a la vez en varios;
  el planificador del sistema operativo decide, tu programa no.
- Crear hilos en Java, con un ejemplo de consola completo y mínimo: la interfaz Runnable
  y new Thread(runnable).start() — qué hace start() frente a llamar a run() a mano, y
  una salida por consola donde se VEA el entrelazado no determinista de dos hilos
  imprimiendo (ejecutar dos veces, salida distinta).
- El ciclo de vida de un hilo: NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING →
  TERMINATED, con diagrama de estados y qué lleva a cada estado (sleep, join, esperar un
  lock).
- El problema central de compartir memoria — la condición de carrera, con EL ejemplo
  clásico completo: dos hilos incrementando un contador compartido 10.000 veces cada
  uno; el resultado no da 20.000. Explica por qué (contador++ son tres operaciones:
  leer, sumar, escribir — y se intercalan) y la solución básica: synchronized (qué es un
  lock/sección crítica). Menciona deadlock en 2-3 frases como el peligro opuesto
  (bloquear demasiado).
- Cierra la parte teórica con la escala de abstracción: Thread/Runnable a mano →
  ExecutorService y pools de hilos (qué es un pool: hilos reutilizables que cogen tareas
  de una cola, y por qué es mejor que crear hilos sin parar) → las abstracciones de
  Spring (@Async, listeners) que delegan todo eso en el framework.

PARTE 2 — Aterrizaje: el descubrimiento de que el GameVault del alumnado YA es una
aplicación multihilo aunque nunca haya escrito `new Thread()`. Tres "avistamientos" de
hilos reales:
- El pool de Tomcat: cada petición HTTP se atiende en un hilo distinto (ya se comprobó
  experimentalmente en PSP Tema 1 con las dos peticiones simultáneas a
  /api/v1/videojuegos/top) — retómalo ahora con nombre técnico: pool de hilos,
  contexto de ejecución por petición.
- Los listeners de RabbitMQ como hilos reales — PERO ANTES de entrar en los hilos,
  explica RabbitMQ en sí desde cero (2-3 frases por pieza, esta es la PRIMERA vez que
  el alumnado ve el término en el curso, y AD también lo da por conocido más adelante en
  su Tema 4, así que este es el primer que lo debe explicar bien): qué es un broker de
  mensajería (un servidor intermediario que recibe mensajes de quien los produce y los
  entrega a quien los consume, sin que productor y consumidor se conozcan ni tengan que
  estar activos a la vez); qué es una cola (una lista de mensajes pendientes de
  procesar) y un exchange (a quién le llega cada mensaje publicado, según reglas de
  enrutado); y qué hace `@RabbitListener` (marca un método como consumidor de una cola:
  Spring AMQP lo invoca automáticamente cuando llega un mensaje, en un hilo del propio
  contenedor de listeners, no en el hilo que lo publicó). Con esa base, aterriza en
  com/aleroig/gamevault/actividad/mensajeria/ActividadVideojuegoEventConsumer.java y
  com/aleroig/gamevault/reviews/mensajeria/ReviewsVideojuegoEventConsumer.java — sus
  métodos @RabbitListener NO se ejecutan en el hilo de la petición HTTP que originó el
  evento, sino en hilos del contenedor de listeners de Spring AMQP. Explica el flujo
  completo con un diagrama: petición HTTP (hilo A) → VideojuegoService.create() →
  VideojuegoEventPublisher publica en RabbitMQ → el consumer procesa (hilo B), y qué
  gana la aplicación con ello (la petición no espera al registro de actividad). Cierra
  este punto anotando que esta es la explicación de referencia de RabbitMQ para todo el
  curso: cualquier otra actividad (de PSP o de AD) que use RabbitMQ puede remitir aquí
  en vez de volver a explicarlo desde cero.
- El @EnableAsync "durmiente": GamevaultApplication.java declara @EnableAsync pero
  NINGÚN método del proyecto usa @Async todavía — señálalo como el hueco que este tema
  va a rellenar: las semanas 14-16 construyen el warm-up de la caché de topNovedades
  usando precisamente esa capacidad sin estrenar.
- Presenta el problema motivador del warm-up (criterio a): getTopNovedades() en
  VideojuegoService.java es @Cacheable("topNovedades") y simula 2 s de lentitud con
  Thread.sleep(2000); cada create/update/delete hace @CacheEvict — así que tras cada
  escritura, el siguiente usuario paga los 2 segundos. Un hilo en segundo plano que
  recaliente la caché es la situación "útil" de libro.

Nota: la escala de mecanismos de creación (criterio b) ya se presentó al final de la
PARTE 1 — aquí solo recuérdala en una frase, señalando que Thread/Runnable a mano se
practicará también en el Tema 4 con sockets.
```
