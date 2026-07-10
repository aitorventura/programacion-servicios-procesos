<a id="listeners-asincronos"></a>

# 🧩 3. Listeners asíncronos: el warm-up de caché (2/2)

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Listeners asíncronos: el warm-up de caché (2/2)" del
Tema 3 (RA2 - Programación multihilo) del módulo Programación de Servicios y Procesos
(0490), semana real 15 del calendario. Sigue las convenciones de estilo del README.md
del repo.

Criterios de evaluación de RA2 que cubre este apartado (curriculum.md):
- d) Estados de ejecución de un hilo y aplicaciones que los gestionan.
- f) Hilos sincronizados mediante técnicas específicas.
- j) Librerías específicas del lenguaje para multihilo.
- k) Problemas derivados de la compartición de información entre hilos.

Contenido central: la pieza 2 del warm-up — el listener que reacciona al
TopNovedadesInvalidadoEvent (Actividad 3.2) y recalienta la caché en un hilo aparte.
Aquí es donde el @EnableAsync "durmiente" de GamevaultApplication.java entra por fin en
juego.

Explica, con código de ejemplo que la Actividad 3.3 construirá guiado:
- @Async sobre el método listener: qué hace exactamente Spring (proxy que despacha la
  ejecución a un pool de hilos) y el contraste medido en la Actividad 3.2: sin @Async el
  listener corría en el hilo de la petición; con @Async corre en otro (verificable por
  el nombre del hilo en el log).
- @TransactionalEventListener(phase = AFTER_COMMIT) frente a @EventListener a secas
  (criterio f — sincronización entre hilos con técnicas específicas): el problema real
  que resuelve — si el listener se lanza ANTES de que la transacción de
  VideojuegoService.create() haga commit, el hilo del warm-up puede leer de la base de
  datos un estado sin el videojuego nuevo y recalentar la caché con datos viejos.
  Sincronizar el arranque del hilo con el commit de la transacción es la técnica
  específica de este caso. (El alumnado conoce @Transactional de AD — apóyate en eso.)
- Estados de un hilo (criterio d): el ciclo NEW → RUNNABLE → TIMED_WAITING (el
  Thread.sleep(2000) del getTopNovedades es un ejemplo perfecto y observable) →
  TERMINATED, conectado con lo que verían en jconsole/VisualVM en la Actividad 3.1.
- Problemas de compartición (criterio k), aterrizados en este caso concreto: ¿qué pasa
  si dos escrituras casi simultáneas publican dos eventos y dos hilos recalientan a la
  vez? (trabajo duplicado, carreras sobre la caché); ¿y si el listener leyera una
  entidad JPA compartida en vez de un record inmutable? Menciona también, como
  panorámica breve (criterio j), las piezas de java.util.concurrent que Spring usa por
  debajo (ExecutorService, pools) — la configuración fina llega en el apartado 4.

Cierra con el resultado final medible: tras el warm-up, el "primer usuario tras una
escritura" ya no paga los 2 segundos — la medición de la Actividad 3.3 lo demostrará.
```
