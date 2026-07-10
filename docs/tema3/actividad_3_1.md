# 🧪 Actividad 3.1: Cazando hilos en GameVault (análisis, sin código)

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 3.1 del Tema 3 (RA2 - Programación multihilo) del módulo
Programación de Servicios y Procesos (0490), semana real 12 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA de OBSERVACIÓN, no un reto y sin escribir
código de producción (solo trazas temporales de log). El alumnado trabaja sobre su
propia copia de GameVault. Cada paso da los comandos/cambios exactos y la salida
esperada.

Objetivo (RA2, criterios a, b, i): que el alumnado observe y documente los hilos reales
de su GameVault en ejecución.

Estructura sugerida de pasos guiados:
0. Calentamiento con los ejemplos de la teoría (código DADO para copiar y ejecutar, no
   para escribir): el ejemplo del entrelazado de dos hilos (ejecutarlo dos veces y
   comparar salidas) y el del contador compartido con condición de carrera (ejecutarlo
   sin y con synchronized, anotar los resultados de ambos). Objetivo: haber VISTO una
   carrera de verdad antes de buscar hilos en GameVault.
1. Añadir guiadamente una traza temporal con Thread.currentThread().getName() en dos
   puntos dados: VideojuegoService.create() y el método recibir() de
   ActividadVideojuegoEventConsumer (código exacto de la línea de log dado).
2. Ejecutar guiadamente: crear un videojuego vía API y leer en el log los DOS nombres de
   hilo distintos (el del pool HTTP y el del listener de RabbitMQ) — anotarlos y
   explicar con sus palabras por qué no coinciden, apoyándose en el diagrama de la
   teoría.
3. Observación guiada del conjunto de hilos de la JVM con una herramienta (elegir una y
   guiarla paso a paso: jconsole, VisualVM o un thread dump con jstack — comandos
   dados): localizar los hilos de Tomcat (http-nio-*), los del listener de RabbitMQ y el
   hilo main; contar cuántos hilos hay en total y comentar la cifra.
4. Mini-reto (repite el patrón del paso 1): añadir la misma traza en
   ReviewsVideojuegoEventConsumer.recibir(), borrar un videojuego, y explicar por qué
   ese listener también se dispara (la routing key videojuego.eliminado de
   RabbitMQConfig.java) y en qué hilo.
5. Localizar guiadamente el @EnableAsync de GamevaultApplication.java y comprobar (con
   una búsqueda en el proyecto, comando dado) que ningún método usa @Async todavía —
   dejar por escrito la hipótesis de para qué servirá, que se confirmará en la Actividad
   3.2.
6. Retirar las trazas temporales al terminar (recordatorio explícito).
```
