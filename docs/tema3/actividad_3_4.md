# 🧪 Actividad 3.4: `TaskExecutor` propio — cierre de RA2

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 3.4 del Tema 3 (RA2 - Programación multihilo) del módulo
Programación de Servicios y Procesos (0490), semana real 16 del calendario — actividad
que CIERRA el RA2. Si necesita plantilla/solución en .docx, crea antes la skill de
plantilla de PSP clonando /actividad-plantilla-acceso-a-datos con la paleta
marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado. Completa la MEJORA del warm-up (Actividades 3.2 y 3.3).

Objetivo (RA2, criterios g, h — cierre del RA): configurar un TaskExecutor propio para
el warm-up, con nombre de hilos y prioridad controlados, y documentarlo.

Estructura sugerida de pasos guiados:
1. El bean guiado al completo: una clase de configuración en config/ (siguiendo el
   estilo de RabbitMQConfig.java) con un ThreadPoolTaskExecutor — corePoolSize/
   maxPoolSize/queueCapacity con valores razonados en el propio enunciado,
   threadNamePrefix "warmup-" y prioridad baja con setThreadPriority
   (Thread.MIN_PRIORITY) — código mostrado, cada parámetro explicado.
2. Conexión guiada: cambiar el @Async del listener de la Actividad 3.3 por
   @Async("nombreDelBean") y arrancar.
3. Verificación guiada por el nombre: crear un videojuego y comprobar en el log que el
   warm-up ahora corre en un hilo "warmup-1" (antes era un hilo genérico task-N del
   executor por defecto — comparar con la anotación de la 3.3).
4. Observación guiada con jconsole/thread dump (repite el procedimiento de la Actividad
   3.1): localizar los hilos warmup-* y anotar su prioridad.
5. Mini-reto (repite el patrón del paso 1): provocar varios eventos seguidos (crear 3-4
   videojuegos rápido) y observar cuántos hilos warmup-* llegan a existir según el
   corePoolSize configurado; probar a cambiar corePoolSize a 1 y repetir — describir la
   diferencia observada.
6. Cierre de RA2 (criterio h): documentar la configuración elegida (2-3 frases por
   parámetro: por qué ese tamaño de pool, por qué prioridad baja) y un repaso propio
   (4-5 frases) del recorrido completo del tema: observar hilos → evento → listener
   @Async → executor propio.
```
