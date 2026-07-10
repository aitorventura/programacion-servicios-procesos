# 🧪 Actividad 3.3: El listener `@Async` del warm-up (2/2)

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 3.3 del Tema 3 (RA2 - Programación multihilo) del módulo
Programación de Servicios y Procesos (0490), semana real 15 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado. Completa la MEJORA del warm-up iniciada en la Actividad
3.2.

Objetivo (RA2, criterios d, f, j, k): construir el listener asíncrono que recalienta la
caché de topNovedades y demostrar el resultado con mediciones.

Estructura sugerida de pasos guiados:
1. El listener guiado al completo, código mostrado y explicado: una clase
   TopNovedadesWarmupListener (@Service) con un método anotado
   @TransactionalEventListener(phase = AFTER_COMMIT) + @Async que recibe el
   TopNovedadesInvalidadoEvent y llama a videojuegoService.getTopNovedades() para
   recalentar la caché; con una traza de log del nombre del hilo al empezar y al
   terminar.
2. Verificación guiada del hilo: crear un videojuego y comparar en el log el hilo de la
   petición con el hilo del listener (con @Async debe ser distinto — contrastar con la
   anotación tomada en la Actividad 3.2 sin @Async). Retirar el listener trivial de
   prueba de la 3.2 si sigue ahí.
3. La medición estrella, guiada (repite el protocolo de medición de la Actividad 3.2):
   crear un videojuego, esperar ~3 segundos, y medir /api/v1/videojuegos/top — ahora
   responde al instante, porque el warm-up ya pagó los 2 segundos en su propio hilo.
   Comparar con las mediciones "antes" de la 3.2 y documentar la mejora.
4. Experimento guiado sobre AFTER_COMMIT (criterio f): cambiar temporalmente a
   @EventListener a secas (sin fase), añadir un log en el listener que consulte cuántos
   videojuegos ve, y razonar con ayuda del enunciado qué podría salir mal (leer datos
   pre-commit); volver a @TransactionalEventListener y explicar con sus palabras la
   diferencia.
5. Pregunta de comprensión (criterio k): si dos profesores del centro crean dos
   videojuegos casi a la vez, ¿cuántos eventos se publican y cuántos hilos recalientan?
   ¿Es un error grave o solo trabajo duplicado? ¿Se les ocurre alguna forma de evitarlo?
   (razonar, no implementar).
```
