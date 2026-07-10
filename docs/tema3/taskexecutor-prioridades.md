<a id="taskexecutor-prioridades"></a>

# 🧩 4. `TaskExecutor` propio y prioridades

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo. Este apartado cierra el RA2.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "TaskExecutor propio y prioridades" del Tema 3 (RA2 -
Programación multihilo) del módulo Programación de Servicios y Procesos (0490), semana
real 16 del calendario — apartado que CIERRA el RA2. Sigue las convenciones de estilo
del README.md del repo.

Criterios de evaluación de RA2 que cubre este apartado (curriculum.md):
- g) Establecimiento y control de la prioridad de los hilos.
- h) Depuración y documentación de los programas.

ESTRUCTURA — teoría primero: retoma y profundiza el concepto de pool de hilos
presentado en el apartado 1, ahora en serio y desde cero: por qué crear un hilo nuevo
por tarea no escala (coste de creación, memoria por hilo, y un pico de tareas puede
tumbar el proceso), y cómo lo resuelve un pool — un grupo fijo de hilos reutilizables
que toman tareas de una cola. Explica sus tres parámetros universales con un diagrama
(hilos base / hilos máximos / capacidad de la cola) y qué pasa cuando la cola se llena.
Explica también qué es la prioridad de un hilo en general (una pista al planificador del
SO sobre a quién dar turno antes — sugerencia, no garantía) antes de tocar la API.

Contenido central: hasta ahora el @Async del warm-up usa el executor por defecto de
Spring — este apartado enseña a definir un TaskExecutor propio para controlar cuántos
hilos, cómo se llaman y con qué prioridad ejecutan las tareas en segundo plano. Es una
MEJORA que no existe en la referencia adjunta (no hay ningún bean TaskExecutor en
config/).

Explica, con código de ejemplo que la Actividad 3.4 construirá guiado:
- Un bean ThreadPoolTaskExecutor en una clase de configuración (siguiendo el estilo del
  paquete config/ del proyecto, como RabbitMQConfig.java): corePoolSize, maxPoolSize,
  queueCapacity y, sobre todo, threadNamePrefix (por ejemplo "warmup-") — el nombre del
  hilo como herramienta de DEPURACIÓN (criterio h): con un prefijo propio, las trazas
  del warm-up se distinguen a simple vista en el log y en jconsole, conectando con lo
  aprendido en la Actividad 3.1.
- Cómo dirigir el @Async a ese executor concreto: @Async("nombreDelBean") — y por qué
  aislar las tareas de fondo en su propio pool pequeño evita que un warm-up masivo
  robe hilos a las peticiones de los usuarios.
- Prioridades (criterio g), con honestidad técnica: ThreadPoolTaskExecutor permite
  setThreadPriority() (Thread.MIN_PRIORITY a MAX_PRIORITY); explica qué significa la
  prioridad de un hilo en Java (una sugerencia al planificador del SO, no una garantía)
  y por qué la práctica moderna prefiere pools separados y tamaños acotados a jugar con
  prioridades — el warm-up es justo el caso de tarea de fondo que debe ceder el paso:
  prioridad baja + pool propio.
- Depuración y documentación (criterio h): ver los hilos "warmup-1..." en un thread
  dump/jconsole, y documentar la decisión de configuración (por qué esos tamaños de
  pool) en un comentario o en el README del proyecto del alumno.

Cierra recapitulando todo RA2 en 4-5 frases (analizar los hilos existentes → evento
interno → listener @Async sincronizado con la transacción → executor propio con nombre y
prioridad) y conectando con el Tema 4 (RA3): en sockets y WebSocket volverán a aparecer
hilos, esta vez gestionados a mano.
```
