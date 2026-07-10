# 🧪 Actividad 3.3: El listener `@Async` del warm-up (2/2)

!!! info "Práctica guiada — pieza 2 de 2"
    Hoy completas el warm-up: el listener asíncrono que recalienta la caché, y la medición real de que ya no se paga el coste tras cada escritura.

## Qué vas a practicar

- Construir un listener `@Async` + `@TransactionalEventListener(AFTER_COMMIT)`.
- Verificar en el log que el listener corre en un hilo distinto.
- Medir la mejora real de rendimiento.
- Entender por qué `AFTER_COMMIT` importa, provocando el problema que evita.

---

## Requisitos previos

Tu evento `TopNovedadesInvalidadoEvent` y su publicación (Actividad 3.2).

---

## Paso 1 — El listener, guiado al completo

```java
package com.tunombre.gamevault.catalogo.eventos;

import com.tunombre.gamevault.catalogo.VideojuegoService;
import lombok.RequiredArgsConstructor;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Service
@RequiredArgsConstructor
public class TopNovedadesWarmupListener {

    private final VideojuegoService videojuegoService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
        System.out.println("[WARMUP] Empieza en hilo: " + Thread.currentThread().getName()
                + " - evento de: " + event.momento());

        videojuegoService.getTopNovedades(); // recalienta la caché

        System.out.println("[WARMUP] Termina en hilo: " + Thread.currentThread().getName());
    }
}
```

`@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` espera a que la transacción que publicó el evento termine con éxito antes de disparar este método; `@Async` hace que, cuando se dispare, corra en un hilo aparte. Llamar de nuevo a `getTopNovedades()` es lo que recalienta la caché — como sigue anotado `@Cacheable`, este propio hilo va a pagar los 2 segundos del `Thread.sleep`, pero **en segundo plano**, sin que ningún usuario esté esperando esa respuesta.

---

## Paso 2 — Verificar el hilo, y retirar el listener de prueba

Si todavía tienes el `ListenerDePruebaTemporal` de la Actividad 3.2, **retíralo ahora** — ya no lo necesitas.

Crea un videojuego y mira el log:

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test2","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Anota** los dos nombres de hilo que ves en las trazas `[WARMUP] Empieza...`/`Termina...`. **Compara** con el nombre de hilo que anotaste en la Actividad 3.2 (sin `@Async`): ¿son el mismo tipo de hilo, o claramente distintos?

---

## Paso 3 — La medición estrella

Repite el protocolo de medición de la Actividad 3.2, pero ahora con el warm-up completo:

```bash
# Crea un videojuego (dispara el warm-up en segundo plano)
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test3","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'

# Espera unos segundos a que el warm-up termine en segundo plano
sleep 3

# Mide /top — ¿cuánto tarda ahora?
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null
```

**Anota** el tiempo. **Compara** con las mediciones "antes" que hiciste en la Actividad 3.2 (donde el tercer usuario pagaba ~2 segundos). Documenta la diferencia con tus propios números.

---

## Paso 4 — Experimento sobre `AFTER_COMMIT`

Cambia temporalmente tu listener a `@EventListener` a secas (sin `@TransactionalEventListener`, sin fase):

```java
@EventListener
@Async
public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
    // ...
}
```

Añade, también temporalmente, un log que consulte cuántos videojuegos ve el listener justo al empezar (por ejemplo, contando el resultado de `videojuegoService.findAll().size()` antes de recalentar).

**Razona**, sin necesidad de reproducirlo de forma determinista (es una condición de carrera, puede que no falle siempre): si este listener se disparara justo antes de que el `INSERT` del nuevo videojuego se confirmara en la base de datos, ¿qué vería exactamente al consultar? ¿Qué consecuencia tendría eso sobre el contenido de la caché recalentada?

Vuelve a `@TransactionalEventListener(phase = AFTER_COMMIT)` y explica con tus palabras qué diferencia concreta soluciona respecto al `@EventListener` a secas que acabas de probar.

---

## Pregunta final

Si dos profesores del centro crean dos videojuegos casi a la vez (segundos de diferencia), ¿cuántos eventos se publican y cuántos hilos intentan recalentar la caché? ¿Es esto un error grave del sistema, o solo trabajo duplicado sin consecuencias incorrectas? Propón, sin implementarla, alguna forma de evitar ese trabajo duplicado (piensa en si haría falta algún tipo de coordinación entre los hilos, o si bastaría con alguna comprobación previa).

---

## ✅ Cierre

El warm-up está completo y medido: tu GameVault ya no hace pagar al primer usuario los 2 segundos tras cada escritura. En el apartado siguiente configuras manualmente el `TaskExecutor` que hay detrás de este `@Async`, con nombre y prioridad propios.
