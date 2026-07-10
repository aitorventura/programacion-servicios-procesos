# 🧪 Actividad 3.2: El evento del warm-up de caché (1/2)

!!! info "Práctica guiada — pieza 1 de 2"
    Hoy construyes el evento interno y su publicación. Todavía no vas a ver ningún efecto sobre la lentitud de `/top` — eso llega la semana que viene con el listener.

## Qué vas a practicar

- Medir el problema real antes de resolverlo.
- Crear un evento interno como `record` inmutable.
- Publicar un evento con `ApplicationEventPublisher`.

---

## Requisitos previos

Tu `VideojuegoService` con `getTopNovedades()` (`@Cacheable`) y los métodos de escritura (`@CacheEvict`) ya funcionando.

---

## Paso 1 — Medir el problema que vas a resolver

```bash
# Primera llamada: paga el Thread.sleep(2000)
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null

# Segunda llamada: sale de caché, instantánea
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null

# Crea un videojuego (invalida la caché con @CacheEvict)
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'

# Tercera llamada: ha vuelto a pagar los 2 segundos
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null
```

**Anota** los tres tiempos medidos. Ese "tercer usuario que paga otra vez los 2 segundos, justo después de una escritura" es exactamente lo que el warm-up de las próximas dos semanas va a eliminar.

---

## Paso 2 — La clase de evento

Decide junto con el enunciado qué información debe transportar: para este warm-up basta con saber **cuándo** ocurrió la invalidación (no necesitas saber qué operación la causó, ni sobre qué videojuego).

```java
package com.tunombre.gamevault.catalogo.eventos;

import java.time.Instant;

public record TopNovedadesInvalidadoEvent(Instant momento) {}
```

**Pregunta**: ¿por qué este evento vive en un paquete `catalogo.eventos` (interno) y no en `catalogo.api.eventos` (si tu proyecto ya tiene ese paquete, del `VideojuegoEvent` de RabbitMQ)? Relaciona tu respuesta con la distinción que viste en la teoría entre eventos internos de Spring y mensajería RabbitMQ entre módulos.

---

## Paso 3 — La publicación, guiada al completo

En `VideojuegoService`, añade `ApplicationEventPublisher` a tus dependencias:

```java
@Service
@RequiredArgsConstructor
public class VideojuegoService {
    private final VideojuegoRepository videojuegoRepository;
    private final EstudioRepository estudioRepository;
    private final ApplicationEventPublisher eventPublisher;

    @CacheEvict(value = "topNovedades", allEntries = true)
    @Transactional
    public VideojuegoResponseDTO create(VideojuegoCreateDTO dto) {
        // ... tu lógica de creación ya existente ...
        Videojuego saved = videojuegoRepository.save(v);

        eventPublisher.publishEvent(new TopNovedadesInvalidadoEvent(Instant.now()));

        return mapToDTO(saved);
    }
}
```

`eventPublisher.publishEvent(...)` dispara el evento. Ahora mismo no pasa nada visible — no hay nadie escuchando todavía.

---

## Mini-reto — repite en `update()` y `delete()`

Sin más código dado, añade la misma línea de publicación (`eventPublisher.publishEvent(new TopNovedadesInvalidadoEvent(Instant.now()))`) en `update()` y en `delete()` de `VideojuegoService` — en el mismo punto relativo donde ya está en `create()` (justo antes de devolver el resultado).

---

## Paso 4 — Comprobar que se publica, con un listener trivial temporal

Añade, **temporalmente**, esta clase para verificar que tu evento realmente se dispara:

```java
package com.tunombre.gamevault.catalogo.eventos;

import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class ListenerDePruebaTemporal {

    @EventListener
    public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
        System.out.println("[TRAZA] Evento recibido en hilo: " + Thread.currentThread().getName()
                + " - momento: " + event.momento());
    }
}
```

Crea un videojuego y mira la consola. **Comprueba** que la traza aparece. **Anota** el nombre del hilo en el que se ejecuta este listener — sin `@Async` todavía, ¿es el mismo hilo que procesó la petición HTTP, o uno distinto? Este dato es el contraste clave que vas a necesitar la próxima actividad: cuando añadas `@Async`, ese nombre de hilo va a cambiar.

Cuando termines de comprobarlo, **retira** esta clase — es solo para esta verificación, la semana que viene construyes el listener real.

---

## Pregunta final

¿Por qué conviene que `TopNovedadesInvalidadoEvent` sea un `record` inmutable, sabiendo que la semana que viene va a ser leído desde un hilo distinto al que lo publica? ¿Qué problema evita concretamente la inmutabilidad, comparado con si `TopNovedadesInvalidadoEvent` fuera una clase mutable con setters?

---

## ✅ Cierre

Tu GameVault ya emite un evento cada vez que la caché de novedades se invalida — aunque, de momento, nadie reaccione a él de forma permanente. La semana que viene construyes el listener `@Async` que cierra el ciclo: recalentar la caché en un hilo aparte, justo después del commit.
