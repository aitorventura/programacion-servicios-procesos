<a id="listeners-asincronos"></a>

# 🧩 3. Listeners asíncronos: el warm-up de caché (2/2)

En el apartado anterior has construido el evento y su publicación — sin ningún efecto visible todavía. Hoy construyes la pieza que le da sentido a todo: el listener que reacciona, recalentando la caché en un hilo aparte. Aquí es donde entra en juego `@EnableAsync` — una capacidad que tu proyecto todavía no usa en ningún sitio, y que hoy activas tú mismo junto a la anotación `@Async`.

---

## ⚡ `@Async`: qué hace exactamente

`@Async` sobre un método le dice a Spring: "no ejecutes este método en el hilo de quien lo llama — despáchalo a un pool de hilos aparte, y devuelve el control inmediatamente a quien llamó". Es el mismo tipo de mecanismo que ya has visto con `@Cacheable` en el apartado anterior: Spring intercepta la llamada al método anotado y hace algo distinto de lo que harías tú al llamarlo a pelo.

| | Sin `@Async` | Con `@Async` |
|---|---|---|
| ¿En qué hilo se ejecuta el método? | En el mismo hilo de quien lo llama | En un hilo del pool, aparte |
| ¿Cuándo recupera el control quien llamó? | Cuando el método termina de ejecutarse | Inmediatamente, sin esperar a que termine |

Por debajo, en cuanto invocas un método `@Async`, Spring no lo ejecuta ahí mismo: lo envía a un `ExecutorService` (recuerda la escala de abstracción del primer apartado de este tema), y el método original se ejecuta en uno de esos hilos del pool.

Ya tienes el contraste medido en la Actividad 3.2: sin `@Async`, tu listener trivial de prueba corría en el **mismo hilo** de la petición HTTP que publicó el evento. Con `@Async`, va a correr en un hilo **distinto** — lo vas a verificar de nuevo hoy, comparando los nombres de hilo en el log.

!!! danger "La trampa clásica: llamar a un método `@Async` desde dentro de la misma clase"
    Spring solo puede interceptar una llamada que venga **de fuera** de la clase. Si en vez de eso un método llama directamente a otro método `@Async` de esa misma clase (`this.metodoAsync(...)`), no hay ninguna llamada externa que interceptar — se ejecuta como una llamada normal de Java, en el mismo hilo, y `@Async` se ignora **en silencio**, sin ningún error ni aviso. Es uno de los fallos más comunes con `@Async`/`@Transactional` en Spring: en este warm-up no te va a pasar, porque el listener es un bean aparte al que Spring llama desde fuera, pero conviene que lo tengas en la cabeza para el futuro.

---

## ⏱️ `@TransactionalEventListener(phase = AFTER_COMMIT)`

Aquí aparece un problema real, sutil, que un `@EventListener` a secas no resuelve: si el listener se dispara **antes** de que la transacción de `LibroService.create()` haga *commit*, el hilo del warm-up podría leer la base de datos en un estado que **todavía no incluye** el libro recién creado — y recalentaría la caché con datos viejos, justo lo contrario de lo que quieres.

```mermaid
sequenceDiagram
    participant S as LibroService.create()
    participant L as Listener
    participant DB as Base de datos

    S->>DB: INSERT del libro nuevo (sin commit todavía)
    S->>L: publishEvent(...)
    alt EventListener a secas
        L->>DB: Recalienta, leyendo ahora mismo
        Note over L,DB: El commit aún no ha ocurrido — el libro nuevo NO aparece
    else TransactionalEventListener AFTER_COMMIT
        S->>DB: commit
        DB-->>L: Ahora sí, se dispara el listener
        L->>DB: Recalienta, leyendo ya confirmado
        Note over L,DB: El libro nuevo SÍ aparece
    end
```

`@TransactionalEventListener(phase = AFTER_COMMIT)` resuelve esto: en vez de dispararse en el momento de `publishEvent(...)`, espera a que la transacción que publicó el evento haga *commit* con éxito, y solo entonces ejecuta el listener. Es la **técnica específica de sincronización** que necesitas para este caso concreto: sincronizar el arranque del hilo del listener con el momento exacto en que los datos ya son consistentes y visibles. Ya conoces `@Transactional` de Acceso a Datos — esto es la contrapartida del lado "quien reacciona a que una transacción ha terminado", no del lado "quien abre la transacción".

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async
public void onLibrosBaratosInvalidado(LibrosBaratosInvalidadoEvent event) {
    // se ejecuta DESPUÉS del commit, EN OTRO HILO
}
```

---

## 🔄 Estados de un hilo, ahora observables de verdad

Retoma el ciclo de estados del primer apartado de este tema (NEW → RUNNABLE → BLOCKED/WAITING → TERMINATED), esta vez con un caso perfectamente observable: el retraso artificial que simula el coste de recalcular los libros más baratos.

| Momento | Estado del hilo del warm-up |
|---|---|
| El listener empieza a ejecutar el recálculo | `RUNNABLE` |
| Dentro del retraso artificial (el `Thread.sleep` simulado) | `TIMED_WAITING` |
| El retraso termina, el listener sigue calculando | `RUNNABLE` |
| El listener termina | `TERMINATED` |

Fíjate en que aquí no aparece `BLOCKED` — ese estado nace de esperar un *lock* ocupado por otro hilo (como en el `Contador` del primer apartado), y este warm-up no compite por ningún *lock*. Puedes verlo tú mismo con `Thread.getState()`, igual que has hecho con `EstadosDemo` en el primer apartado, capturando el estado del hilo justo durante ese retraso.

---

## ⚠️ Problemas de compartición, en este caso concreto

Dos preguntas que conviene hacerse antes de dar el warm-up por terminado:

- **¿Qué pasa si dos escrituras casi simultáneas publican dos eventos?** Dos hilos distintos intentarían recalentar la misma caché casi a la vez — trabajo duplicado (ambos recalculan lo mismo) y, en el peor caso, una carrera sobre qué resultado queda finalmente guardado en la caché. No es un error grave (el resultado final sigue siendo correcto), pero sí ineficiente.
- **¿Y si el listener leyera una entidad JPA compartida en vez de un evento inmutable?** Si `LibrosBaratosInvalidadoEvent` no fuera un `record` inmutable, sino que el listener leyera directamente un objeto mutable compartido con el hilo que publicó, existiría el riesgo de que ese objeto cambiara mientras el listener aún lo está usando — exactamente el tipo de condición de carrera que has visto en la Actividad 3.1. La inmutabilidad del evento elimina ese riesgo por diseño.

Como panorámica breve: por debajo de `@Async`, Spring usa piezas de `java.util.concurrent` — típicamente un `ExecutorService`/pool de hilos configurable. La configuración fina de ese pool (cuántos hilos, con qué nombre, con qué prioridad) es exactamente lo que vas a construir en el apartado siguiente.

---

## 📈 El resultado que vas a medir

| | Antes (Actividad 3.2, sin warm-up) | Después (hoy, con warm-up) |
|---|---|---|
| Justo después de crear un libro | El siguiente usuario paga el coste completo | El listener ya ha recalentado la caché en segundo plano |
| ¿Quién paga el coste de recalcular? | El primer usuario que pide el listado tras la escritura | Nadie — el warm-up lo paga antes de que nadie lo pida |

Con las dos piezas montadas (evento + listener asíncrono sincronizado con el commit), el "primer usuario tras una escritura" ya no paga el coste de recalcular esa consulta cara — el warm-up lo paga por él, en segundo plano, mientras nadie espera. La Actividad 3.3 lo va a demostrar con mediciones reales, comparando el "antes" (Actividad 3.2) con el "después" (hoy).

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - `@Async` hace que Spring intercepte la llamada y despache la ejecución del método a un pool de hilos aparte — el llamador no espera a que termine.
    - Llamar a un método `@Async` desde dentro de la misma clase (`this.metodo(...)`) hace que Spring no pueda interceptar la llamada: `@Async` se ignora en silencio y el método corre en el mismo hilo, sin ningún error que lo avise.
    - `@TransactionalEventListener(phase = AFTER_COMMIT)` sincroniza el arranque del listener con el momento en que la transacción que publicó el evento ya ha hecho commit — evita leer datos "a medias".
    - El retraso artificial del endpoint lento es un caso observable real del estado **TIMED_WAITING**.
    - Publicaciones casi simultáneas pueden causar trabajo duplicado (no un error grave); un evento **inmutable** evita condiciones de carrera al compartir información entre hilos.
    - Por debajo de `@Async`, Spring usa `java.util.concurrent` — la configuración fina del pool llega en el apartado siguiente.
