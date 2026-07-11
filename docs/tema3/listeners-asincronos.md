<a id="listeners-asincronos"></a>

# 🧩 3. Listeners asíncronos: el warm-up de caché (2/2)

La semana pasada construiste el evento y su publicación — sin ningún efecto visible todavía. Hoy construyes la pieza que le da sentido a todo: el listener que reacciona, recalentando la caché en un hilo aparte. Aquí es donde el `@EnableAsync` "durmiente" que localizaste en la Actividad 3.1 entra por fin en juego.

---

## ⚡ `@Async`: qué hace exactamente

`@Async` sobre un método le dice a Spring: "no ejecutes este método en el hilo de quien lo llama — despáchalo a un pool de hilos aparte, y devuelve el control inmediatamente a quien llamó". Por debajo, Spring construye un **proxy** alrededor de tu bean: cuando alguien invoca un método `@Async`, en realidad no lo ejecuta directamente — el proxy intercepta la llamada, la envía a un `ExecutorService` (recuerda la escala de abstracción de la Actividad 3.1), y el método original se ejecuta en uno de esos hilos del pool.

Ya tienes el contraste medido en la Actividad 3.2: sin `@Async`, tu listener trivial de prueba corría en el **mismo hilo** de la petición HTTP que publicó el evento. Con `@Async`, va a correr en un hilo **distinto** — lo vas a verificar de nuevo hoy, comparando los nombres de hilo en el log.

---

## ⏱️ `@TransactionalEventListener(phase = AFTER_COMMIT)`

Aquí aparece un problema real, sutil, que un `@EventListener` a secas no resuelve: si el listener se dispara **antes** de que la transacción de `LibroService.create()` haga *commit*, el hilo del warm-up podría leer la base de datos en un estado que **todavía no incluye** el libro recién creado — y recalentaría la caché con datos viejos, justo lo contrario de lo que quieres.

`@TransactionalEventListener(phase = AFTER_COMMIT)` resuelve esto: en vez de dispararse en el momento de `publishEvent(...)`, espera a que la transacción que publicó el evento haga *commit* con éxito, y solo entonces ejecuta el listener. Es la **técnica específica de sincronización** que necesitas para este caso concreto: sincronizar el arranque del hilo del listener con el momento exacto en que los datos ya son consistentes y visibles. Ya conoces `@Transactional` de Acceso a Datos — esto es la contrapartida del lado "quien reacciona a que una transacción ha terminado", no del lado "quien abre la transacción".

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async
public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
    // se ejecuta DESPUÉS del commit, EN OTRO HILO
}
```

---

## 🔄 Estados de un hilo, ahora observables de verdad

Retoma el ciclo de estados de la Actividad 3.1 (NEW → RUNNABLE → BLOCKED/WAITING → TERMINATED), esta vez con un caso perfectamente observable: el `Thread.sleep(2000)` dentro de `getTopNovedades()`. Mientras ese `sleep` está activo, el hilo que ejecuta el warm-up está en estado **TIMED_WAITING** — puedes verlo tú mismo con jconsole o VisualVM (las mismas herramientas de la Actividad 3.1), capturando el estado del hilo justo durante esos 2 segundos.

---

## ⚠️ Problemas de compartición, en este caso concreto

Dos preguntas que conviene hacerse antes de dar el warm-up por terminado:

- **¿Qué pasa si dos escrituras casi simultáneas publican dos eventos?** Dos hilos distintos intentarían recalentar la misma caché casi a la vez — trabajo duplicado (ambos recalculan lo mismo) y, en el peor caso, una carrera sobre qué resultado queda finalmente guardado en la caché. No es un error grave (el resultado final sigue siendo correcto), pero sí ineficiente.
- **¿Y si el listener leyera una entidad JPA compartida en vez de un evento inmutable?** Si `TopNovedadesInvalidadoEvent` no fuera un `record` inmutable, sino que el listener leyera directamente un objeto mutable compartido con el hilo que publicó, existiría el riesgo de que ese objeto cambiara mientras el listener aún lo está usando — exactamente el tipo de condición de carrera que viste en la Actividad 3.1. La inmutabilidad del evento elimina ese riesgo por diseño.

Como panorámica breve: por debajo de `@Async`, Spring usa piezas de `java.util.concurrent` — típicamente un `ExecutorService`/pool de hilos configurable. La configuración fina de ese pool (cuántos hilos, con qué nombre, con qué prioridad) es exactamente lo que vas a construir en el apartado siguiente.

---

## 📈 El resultado que vas a medir

Con las dos piezas montadas (evento + listener asíncrono sincronizado con el commit), el "primer usuario tras una escritura" ya no paga los 2 segundos de `getTopNovedades()` — el warm-up los paga por él, en segundo plano, mientras nadie espera. La Actividad 3.3 lo va a demostrar con mediciones reales, comparando el "antes" (Actividad 3.2) con el "después" (hoy).

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - `@Async` despacha la ejecución de un método a un pool de hilos aparte, mediante un proxy — el llamador no espera a que termine.
    - `@TransactionalEventListener(phase = AFTER_COMMIT)` sincroniza el arranque del listener con el momento en que la transacción que publicó el evento ya ha hecho commit — evita leer datos "a medias".
    - El `Thread.sleep(2000)` de `getTopNovedades()` es un caso observable real del estado **TIMED_WAITING**.
    - Publicaciones casi simultáneas pueden causar trabajo duplicado (no un error grave); un evento **inmutable** evita condiciones de carrera al compartir información entre hilos.
    - Por debajo de `@Async`, Spring usa `java.util.concurrent` — la configuración fina del pool llega en el apartado siguiente.
