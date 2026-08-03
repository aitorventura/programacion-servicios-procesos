# 🧪 Actividad 3.2: El evento del warm-up de caché (1/2)

!!! warning "Descarga la plantilla"
    📄 [Plantilla 3.2 — El evento del warm-up de caché (1/2)](plantillas/Actividad_3_2_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — pieza 1 de 2"
    Antes de nada, montas Redis y activas la caché real de `getTopNovedades()`. Con eso listo, construyes el evento interno y su publicación. Todavía no vas a ver ningún efecto sobre la lentitud de `/top` tras una escritura — eso llega en la Actividad 3.3 con el listener.

## Qué vas a practicar

- Añadir Redis a tu proyecto y activar una caché real con `@Cacheable`/`@CacheEvict`.
- Medir el problema real antes de resolverlo.
- Crear un evento interno como `record` inmutable.
- Publicar un evento con `ApplicationEventPublisher`.

---

## Requisitos previos

Tu `VideojuegoService` con `getTopNovedades()` funcionando (sin caché todavía, Actividad 1.4) — la caché con Redis y `@Cacheable` la montas tú mismo en el Paso 0 de hoy. Recuerda que al final de la Actividad 1.4 quitaste el `Thread.sleep(2000)` de ese método porque ya no tenía sentido para una funcionalidad real sin caché — hoy lo recuperas, de forma temporal, para tener algo genuinamente caro que cachear.

---

## Paso 0 — Redis y `@Cacheable`, antes de nada

Añade Redis a `.devcontainer/docker-compose.yml`, junto a tus servicios ya existentes:

```yaml
services:
  # ... tus servicios app, postgres, mongodb y rabbitmq ya existentes ...
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

A diferencia de `postgres`/`mongodb`/`rabbitmq`, aquí no hace falta `volumes`: Redis es una caché, no una fuente de verdad — si el contenedor se reinicia y pierde todo lo guardado, no pasa nada grave, la próxima petición simplemente vuelve a pagar el coste y `@Cacheable` la rellena otra vez.

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Y en `application-dev.yml`:

```yaml
spring:
  data:
    redis:
      host: redis
      port: 6379
```

Otra vez el mismo patrón de siempre: `redis` es el nombre del servicio en `.devcontainer/docker-compose.yml`, no `localhost` — tu aplicación y Redis son contenedores hermanos en la misma red.

Añade `@EnableCaching` a `GamevaultApplication.java`, junto a `@SpringBootApplication`:

```java
@SpringBootApplication
@EnableCaching
public class GamevaultApplication {
    // ...
}
```

`@EnableCaching` es lo que activa, a nivel de toda la aplicación, que Spring se fije en anotaciones como `@Cacheable`/`@CacheEvict` que vas a añadir ahora mismo — sin ella, esas anotaciones se quedarían ahí escritas pero Spring las ignoraría por completo, en silencio, sin ningún error que te avise.

!!! warning "Esto rompe tus tests `@WebMvcTest` ya existentes (Actividad 1.3)"
    En cuanto añadas `@EnableCaching`, `VideojuegoControllerTest`/`EstudioControllerTest` van a fallar con `NoSuchBeanDefinitionException: No qualifying bean of type 'CacheManager'`. El motivo: `@WebMvcTest` carga `GamevaultApplication` (y con ella `@EnableCaching`), pero no trae la autoconfiguración de Redis que normalmente aporta el `CacheManager` — un test de solo el nivel web no toca cachés reales. Arréglalo añadiendo un `CacheManager` de mentira a esos dos tests, junto al `@MockitoBean` del service que ya tenías:
    ```java
    @MockitoBean
    private CacheManager cacheManager;
    ```
    No necesita ningún comportamiento real: ningún código de estos tests llega a usar la caché de verdad (el service ya está mockeado), así que solo hace falta que el bean exista para que el contexto arranque.

Antes de anotarlo, vuelve a añadir el `Thread.sleep(2000)` que quitaste al terminar la Actividad 1.4 — exactamente el mismo bloque `try`/`catch` de entonces. Sin él, `getTopNovedades()` ya es rápido de por sí y no tendrías nada real que cachear. Ahora sí, anótalo con `@Cacheable`, y los tres métodos de escritura de `VideojuegoService` con `@CacheEvict`:

```java
@Cacheable("topNovedades")
@Transactional(readOnly = true)
public List<VideojuegoResponseDTO> getTopNovedades() {
    try {
        Thread.sleep(2000); // de vuelta, temporalmente — lo quitas otra vez al final de la Actividad 3.3
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    // ... tu código ya existente ...
}
```

```java
@CacheEvict(value = "topNovedades", allEntries = true, beforeInvocation = true)
@Transactional
public VideojuegoResponseDTO create(VideojuegoCreateDTO dto) {
    // ... tu lógica de creación ya existente ...
}
```

Repite `@CacheEvict(value = "topNovedades", allEntries = true, beforeInvocation = true)` en `update()` y en `delete()`. `@Cacheable` guarda el resultado la primera vez que se llama y lo devuelve directamente en las siguientes, sin ejecutar el método; `@CacheEvict` borra ese resultado guardado cuando algo cambia, para que la próxima llamada vuelva a calcularlo.

!!! warning "`VideojuegoApiIntegrationTest` necesita Redis a partir de ahora (o la caché deshabilitada en test)"
    A diferencia de `VideojuegoControllerTest` (que ya has arreglado arriba mockeando `CacheManager`), `VideojuegoApiIntegrationTest` (Acceso a Datos, Actividad 2.3) levanta el contexto completo de Spring, sin ningún mock — así que en cuanto `create`/`update`/`delete` lleven `@CacheEvict` de verdad, ese test necesita hablar con un Redis real o falla con `RedisConnectionFailureException`. Y como `beforeInvocation = true` evict *antes* de ejecutar el método, hasta un test que solo comprueba un `404` (que nunca llega a tocar la caché en `getTopNovedades()`) puede fallar por esto.

    Aquí no merece la pena montar un contenedor de Redis solo para este test: no es objeto de `VideojuegoApiIntegrationTest` comprobar el comportamiento de la caché en sí (para eso ya tienes el `curl` a `/top` de más arriba, contra tu Redis real de desarrollo). Deshabilita la caché en el perfil de test, en tu `application-test.yml`:

    ```yaml
    spring:
      cache:
        type: none
    ```

    Con `type: none`, Spring registra un `CacheManager` que no hace absolutamente nada — `@Cacheable`/`@CacheEvict` siguen ahí, pero se convierten en no-ops durante los tests, sin que necesites tocar ni una línea de `VideojuegoApiIntegrationTest`.

    **Ejecuta también** `VideojuegoApiIntegrationTest` y comprueba que sigue pasando (no hace falta captura, esto es solo para que tu batería de tests no se quede rota).

`beforeInvocation = true` no es opcional aquí: sin él, Spring evita **después** de que `create()`/`update()`/`delete()` terminen del todo (incluido el `commit`, que llega en el próximo apartado) — justo el mismo instante en que vas a disparar un aviso de "recalienta ya". Con el orden por defecto, ese aviso podría llegar antes de que el evict haya limpiado nada, encontrar la caché todavía con el valor viejo, y no recalcular nada de verdad.

Antes de probarlo, un detalle que si no lo añades ahora te va a dar un `500` en cuanto pidas `/top`: por defecto, Spring guarda en Redis usando la serialización estándar de Java, que exige que la clase sea `Serializable` — y `VideojuegoResponseDTO`, al ser un `record`, no lo es por defecto. Añádeselo:

```java
public record VideojuegoResponseDTO(
    Long id,
    String titulo,
    BigDecimal precio,
    LocalDate fechaLanzamiento,
    String nombreEstudio,
    Map<String, Object> detallesPlataforma
) implements Serializable {}
```

Un `record` puede implementar `Serializable` sin ningún cambio más en su definición — sigue siendo inmutable, sigue generando `equals`/`hashCode`/`toString` automáticamente, solo que ahora también sabe convertirse a bytes y de vuelta.

**Comprueba** que funciona: pide `/api/v1/videojuegos/top` dos veces seguidas (la segunda debería ser instantánea) y confírmalo mirando dentro de Redis:

```bash
docker exec -it <tu-contenedor-redis> redis-cli
> KEYS *
```

**Captura**: la clave que aparece en Redis tras la segunda petición a `/top`.

---

## Paso 1 — Medir el problema que vas a resolver

Tras las pruebas del Paso 0, la caché ya está caliente — si mides ahora, la "primera llamada" saldría instantánea, sin decir nada del problema real. Vacíala antes de empezar:

```bash
docker exec -it <tu-contenedor-redis> redis-cli
> FLUSHALL
```

```bash
# Primera llamada: paga el Thread.sleep(2000)
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null

# Segunda llamada: sale de caché, instantánea
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null

# Crea un videojuego (invalida la caché con @CacheEvict) — recuerda: exige rol ADMIN
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $TOKEN_ADMIN" \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'

# Tercera llamada: ha vuelto a pagar los 2 segundos
time curl -s http://localhost:8080/api/v1/videojuegos/top > /dev/null
```

**Captura**: la terminal con los tres tiempos medidos (`real` de cada `time curl`). Ese "tercer usuario que paga otra vez los 2 segundos, justo después de una escritura" es exactamente lo que el warm-up de esta y la próxima actividad va a eliminar.

---

## Paso 2 — La clase de evento

Decide junto con el enunciado qué información debe transportar: para este warm-up basta con saber **cuándo** ocurrió la invalidación (no necesitas saber qué operación la causó, ni sobre qué videojuego).

```java
package com.tunombre.gamevault.catalogo.eventos;

import java.time.Instant;

public record TopNovedadesInvalidadoEvent(Instant momento) {}
```

**Pregunta**: ¿por qué este evento vive en un paquete `catalogo.eventos` (interno) y no en `catalogo.api.eventos` (si tu proyecto ya tiene ese paquete, del `VideojuegoEvent` de RabbitMQ)? Relaciona tu respuesta con la distinción que has visto en la teoría entre eventos internos de Spring y mensajería RabbitMQ entre módulos.

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

    @CacheEvict(value = "topNovedades", allEntries = true, beforeInvocation = true)
    @Transactional
    public VideojuegoResponseDTO create(VideojuegoCreateDTO dto) {
        // ... tu lógica de creación ya existente ...
        Videojuego savedVideojuego = videojuegoRepository.save(videojuego);

        eventPublisher.publishEvent(new TopNovedadesInvalidadoEvent(Instant.now()));

        return mapToDTO(savedVideojuego);
    }
}
```

`eventPublisher.publishEvent(...)` dispara el evento. Ahora mismo no pasa nada visible — no hay nadie escuchando todavía.

---

## Paso 4 — Repite en `update()` y `delete()`

Sin más código dado, añade la misma línea de publicación (`eventPublisher.publishEvent(new TopNovedadesInvalidadoEvent(Instant.now()))`) en `update()` y en `delete()` de `VideojuegoService` — en el mismo punto relativo donde ya está en `create()` (justo antes de devolver el resultado).

---

## Paso 5 — Comprobar que se publica, con un listener trivial temporal

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

Para poder comparar de verdad, también necesitas saber en qué hilo se publica el evento — sin eso, no tienes contra qué comparar la traza de arriba. Añade, también **temporalmente**, una segunda traza en `VideojuegoService.create()`, justo antes de `eventPublisher.publishEvent(...)`:

```java
System.out.println("[TRAZA] Publicando evento en hilo: " + Thread.currentThread().getName());
eventPublisher.publishEvent(new TopNovedadesInvalidadoEvent(Instant.now()));
```

Crea un videojuego y mira la consola. **Comprueba** que aparecen las dos trazas, una detrás de otra.

**Captura**: las dos líneas de traza en la consola — `[TRAZA] Publicando evento en hilo: ...` y `[TRAZA] Evento recibido en hilo: ...`.

**Anota** ambos nombres de hilo y compáralos — sin `@Async` todavía, ¿son el mismo hilo, o dos distintos? Este dato es el contraste clave que vas a necesitar la próxima actividad: cuando añadas `@Async`, el nombre del hilo que recibe el evento va a cambiar, mientras que el que publica sigue siendo el mismo (el de la petición HTTP).

Cuando termines de comprobarlo, **retira** la clase `ListenerDePruebaTemporal` y la traza que has añadido en `create()` — son solo para esta verificación, en la Actividad 3.3 construyes el listener real.

---

## Pregunta final

¿Por qué conviene que `TopNovedadesInvalidadoEvent` sea un `record` inmutable, sabiendo que en la próxima actividad va a ser leído desde un hilo distinto al que lo publica? ¿Qué problema evita concretamente la inmutabilidad, comparado con si `TopNovedadesInvalidadoEvent` fuera una clase mutable con setters?

---

## ✅ Cierre

Tu GameVault ya emite un evento cada vez que la caché de novedades se invalida — aunque, de momento, nadie reaccione a él de forma permanente. En la próxima actividad construyes el listener `@Async` que cierra el ciclo: recalentar la caché en un hilo aparte, justo después del commit.
