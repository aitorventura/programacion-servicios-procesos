# 🧪 Actividad 3.1: RabbitMQ, registro de actividad y los hilos reales de tu GameVault

!!! warning "Descarga la plantilla"
    📄 [Plantilla 3.1 — RabbitMQ, registro de actividad y los hilos reales de tu GameVault](plantillas/Actividad_3_1_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — construcción y observación, en una sola sesión"
    Hoy montas RabbitMQ en tu proyecto por primera vez y construyes un registro de actividad del catálogo a partir de eventos: cada vez que se crea, modifica o borra un videojuego, un mensaje viaja por una cola y otro hilo (el del listener) lo registra, sin que la petición HTTP original tenga que esperar a que eso termine. Con eso funcionando, cierras la sesión **observando con tus propios ojos** esos hilos reales ejecutándose en tu aplicación.

## Qué vas a practicar

- Añadir un broker de mensajería (RabbitMQ) a tu Dev Container y a tu proyecto.
- Publicar eventos del catálogo con `RabbitTemplate` y consumirlos con `@RabbitListener`.
- Construir un registro de actividad completo: entidad, servicio, endpoint protegido y consumer.
- Localizar en logs los hilos reales que atienden peticiones HTTP y eventos de RabbitMQ.

---

## Requisitos previos

Tu CRUD de `Videojuego` (Acceso a Datos, Tema 1) y tu `SecurityConfig` con roles ya completo (Actividad 2.5) — vas a añadir una ruta nueva a esa misma configuración.

---

## Paso 1 — El contenedor de RabbitMQ

Añade el servicio a `.devcontainer/docker-compose.yml`, junto a los que ya tengas (`app`, `postgres`, y `mongodb` si ya has llegado a esa actividad de Acceso a Datos):

```yaml
services:
  # ... tus servicios ya existentes ...
  rabbitmq:
    image: rabbitmq:4-management
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

El puerto `5672` es el protocolo AMQP (por el que va a hablar tu aplicación); el `15672` es una interfaz web de administración. Reconstruye o reinicia tu Dev Container desde tu editor para que levante también el nuevo servicio, y comprueba que el contenedor de RabbitMQ está funcionando entrando en `http://localhost:15672` (usuario y contraseña por defecto: `guest`/`guest`) — vas a volver a esa interfaz más adelante para ver las colas y los mensajes pasando por ellas en vivo.

**Captura**: la interfaz de administración de RabbitMQ (`http://localhost:15672`) ya cargada, como prueba de que el contenedor está levantado y accesible.

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

Y en `application-dev.yaml`:

```yaml
spring:
  rabbitmq:
    host: rabbitmq
    port: 5672
    username: guest
    password: guest
```

Como ya viste con `postgres` en Acceso a Datos: `rabbitmq` es el nombre del servicio en `.devcontainer/docker-compose.yml`, no `localhost` — tu aplicación corre dentro de `app`, y RabbitMQ es un contenedor hermano más en la misma red.

!!! tip "¿No debería ir `guest`/`guest` a un fichero que no se commitea?"
    `guest` es la cuenta por defecto de RabbitMQ, y RabbitMQ la bloquea por diseño para cualquier conexión que no venga de `localhost` — no importa qué configures, esa cuenta no sirve en remoto. Como todo esto corre dentro de tu Dev Container, en tu propia máquina, no hay riesgo real en dejarla en `application-dev.yaml`. Es distinto del secreto de JWT o la contraseña de `admin` que ya proteges en `application-dev-local.yml`: si esos se filtraran, servirían para hacerse pasar por alguien en tu API real — `guest`/`guest`, fuera de tu propia máquina, no abre nada.

---

## Paso 2 — El exchange y la cola de actividad

Un **exchange** es quien recibe los mensajes publicados y decide, según su *routing key*, a qué cola (o colas) los reenvía. `RabbitMQConfig` va en tu paquete `config` ya existente, junto a `SecurityConfig` — el mismo paquete `com.tunombre.gamevault.config`, `src/main/java/com/tunombre/gamevault/config/RabbitMQConfig.java`:

```java
package com.tunombre.gamevault.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String CATALOGO_EXCHANGE = "catalogo.exchange";
    public static final String ACTIVIDAD_VIDEOJUEGO_QUEUE = "actividad.videojuego.queue";

    @Bean
    public TopicExchange catalogoExchange() {
        return new TopicExchange(CATALOGO_EXCHANGE);
    }

    @Bean
    public Queue actividadVideojuegoQueue() {
        return new Queue(ACTIVIDAD_VIDEOJUEGO_QUEUE);
    }

    @Bean
    public Binding actividadBinding(Queue actividadVideojuegoQueue, TopicExchange catalogoExchange) {
        return BindingBuilder.bind(actividadVideojuegoQueue).to(catalogoExchange).with("videojuego.*");
    }
}
```

Un `TopicExchange` enruta según un patrón de texto: la cola de actividad se enlaza con `videojuego.*` — el asterisco encaja con `videojuego.creado`, `videojuego.actualizado` o `videojuego.eliminado`, cualquier evento del catálogo. Deja este exchange con un único consumidor por ahora; en Acceso a Datos, cuando conectes las reseñas de MongoDB con el borrado del catálogo, vas a añadir una **segunda** cola sobre este mismo exchange, enlazada solo a `videojuego.eliminado` — no hace falta que te adelantes a eso hoy.

---

## Paso 3 — El evento y quién lo publica

`VideojuegoEvent` va en `src/main/java/com/tunombre/gamevault/catalogo/api/eventos/VideojuegoEvent.java`:

```java
package com.tunombre.gamevault.catalogo.api.eventos;

public record VideojuegoEvent(String tipo, Long videojuegoId) {
    public static final String VIDEOJUEGO_CREADO = "VIDEOJUEGO_CREADO";
    public static final String VIDEOJUEGO_ACTUALIZADO = "VIDEOJUEGO_ACTUALIZADO";
    public static final String VIDEOJUEGO_ELIMINADO = "VIDEOJUEGO_ELIMINADO";
}
```

`VideojuegoEventPublisher`, en cambio, va junto a tu `VideojuegoService`, en el mismo paquete `catalogo` (no en el subpaquete de eventos): `src/main/java/com/tunombre/gamevault/catalogo/VideojuegoEventPublisher.java`:

```java
package com.tunombre.gamevault.catalogo;

import com.tunombre.gamevault.catalogo.api.eventos.VideojuegoEvent;
import com.tunombre.gamevault.config.RabbitMQConfig;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;
import tools.jackson.core.JacksonException;
import tools.jackson.databind.json.JsonMapper;

@Component
@RequiredArgsConstructor
public class VideojuegoEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    private final JsonMapper jsonMapper;

    public void publicar(String tipo, Long videojuegoId) {
        String accion = switch (tipo) {
            case VideojuegoEvent.VIDEOJUEGO_CREADO -> "creado";
            case VideojuegoEvent.VIDEOJUEGO_ACTUALIZADO -> "actualizado";
            case VideojuegoEvent.VIDEOJUEGO_ELIMINADO -> "eliminado";
            default -> throw new IllegalArgumentException("Tipo de evento desconocido: " + tipo);
        };
        try {
            String payload = jsonMapper.writeValueAsString(new VideojuegoEvent(tipo, videojuegoId));
            rabbitTemplate.convertAndSend(RabbitMQConfig.CATALOGO_EXCHANGE, "videojuego." + accion, payload);
        } catch (JacksonException e) {
            throw new RuntimeException("No se ha podido serializar el evento", e);
        }
    }
}
```

`RabbitTemplate` es el equivalente en mensajería de `JdbcTemplate`, que ya conoces de Acceso a Datos: Spring Boot lo configura solo en cuanto detecta `spring-boot-starter-amqp` en el classpath, sin que lo registres tú. `JsonMapper`, igual: es un bean que Spring Boot ya te da configurado (viene con Jackson), así que se inyecta por constructor como todo lo demás — nada de `new JsonMapper()` a mano.

Inyecta `VideojuegoEventPublisher` en `VideojuegoService` (constructor, como todo lo demás) y llama a `publicar(...)` al final de `create`, `update` y `delete`, con el tipo correspondiente y el `id` del videojuego.

**Pregunta**: `VideojuegoService` ahora depende de `VideojuegoEventPublisher`, pero no de nada de RabbitMQ directamente (ni `RabbitTemplate` aparece en `VideojuegoService`). ¿Por qué esa capa intermedia, en vez de inyectar `RabbitTemplate` directamente en `VideojuegoService`?

---

## Paso 4 — El registro de actividad: entidad, servicio y endpoint

El registro de actividad es una funcionalidad autocontenida y nueva —no forma parte del catálogo, ni de la seguridad—, así que le corresponde su propio paquete, `actividad`, creado hoy: las cinco clases de este paso (entidad, repositorio, servicio, controller y el consumer que viene al final) van todas ahí, siguiendo el mismo criterio de "un paquete por funcionalidad" que ya usa el resto de tu proyecto. Empieza por la entidad, `src/main/java/com/tunombre/gamevault/actividad/Actividad.java`:

```java
package com.tunombre.gamevault.actividad;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import java.time.Instant;

@Entity
@Table(name = "actividad")
@Getter
@NoArgsConstructor
public class Actividad {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String tipo;
    private String entidad;
    private String entidadId;
    private Instant fecha;

    public Actividad(String tipo, String entidad, String entidadId) {
        this.tipo = tipo;
        this.entidad = entidad;
        this.entidadId = entidadId;
        this.fecha = Instant.now();
    }
}
```

El repositorio, `ActividadRepository.java`, en el mismo paquete:

```java
package com.tunombre.gamevault.actividad;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface ActividadRepository extends JpaRepository<Actividad, Long> {
    List<Actividad> findAllByOrderByFechaDesc();
}
```

El servicio, `ActividadService.java`:

```java
package com.tunombre.gamevault.actividad;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
@RequiredArgsConstructor
public class ActividadService {
    private final ActividadRepository actividadRepository;

    public void registrar(String tipo, String entidad, String entidadId) {
        actividadRepository.save(new Actividad(tipo, entidad, entidadId));
    }

    public List<Actividad> listar() {
        return actividadRepository.findAllByOrderByFechaDesc();
    }
}
```

Y el controller, `ActividadController.java`, documentado igual que el resto de endpoints del proyecto:

```java
package com.tunombre.gamevault.actividad;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/v1/actividad")
@RequiredArgsConstructor
public class ActividadController {
    private final ActividadService actividadService;

    @Operation(summary = "Listar el registro de actividad del catálogo")
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Listado obtenido correctamente, más reciente primero"),
            @ApiResponse(responseCode = "401", description = "No autenticado"),
            @ApiResponse(responseCode = "403", description = "Autenticado, pero sin rol ADMIN")
    })
    @GetMapping
    public ResponseEntity<List<Actividad>> getAll() {
        return ResponseEntity.ok(actividadService.listar());
    }
}
```

!!! note "Los `401`/`403` aquí sí, en el resto de endpoints todavía no"
    Te habrás fijado en que este endpoint documenta `401` y `403`, algo que los endpoints de `VideojuegoController`/`EstudioController` no hacen todavía, aunque también están protegidos por rol desde la Actividad 2.5. Sería lo correcto añadirlos ahí también — pero es una tarea mecánica y repetitiva (la misma pareja de códigos, endpoint a endpoint) que aquí se deja pasar por no ser el foco de esta actividad. En un proyecto real no se debería dejar así: una documentación desactualizada o incompleta es peor que no tener documentación, porque invita a confiar en algo que no refleja el comportamiento real de la API.

Por último, el consumer, `ActividadVideojuegoEventConsumer.java` — sigue en el mismo paquete `actividad`, aunque conecte con la cola: es aquí donde se registra la actividad, no en `catalogo`, así que se queda junto al resto de piezas que dependen de él. Conecta la cola con el servicio, en un hilo aparte reservado por Spring para esto, no en el de la petición HTTP que originó el evento:

```java
package com.tunombre.gamevault.actividad;

import com.tunombre.gamevault.catalogo.api.eventos.VideojuegoEvent;
import com.tunombre.gamevault.config.RabbitMQConfig;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;
import tools.jackson.databind.json.JsonMapper;

@Service
@RequiredArgsConstructor
public class ActividadVideojuegoEventConsumer {
    private final ActividadService actividadService;
    private final JsonMapper jsonMapper;

    @RabbitListener(queues = RabbitMQConfig.ACTIVIDAD_VIDEOJUEGO_QUEUE)
    public void recibir(String payload) {
        VideojuegoEvent event = jsonMapper.readValue(payload, VideojuegoEvent.class);
        actividadService.registrar(event.tipo(), "Videojuego", event.videojuegoId().toString());
    }
}
```

Por último, en tu `SecurityConfig`, añade la regla de esta ruta nueva junto a las demás de la Actividad 2.5:

```java
.requestMatchers(HttpMethod.GET, "/api/v1/actividad").hasRole("ADMIN")
```

---

## Paso 5 — Trazas temporales, para ver los hilos reales

Antes de comprobar el flujo, añade, **temporalmente**, esta línea al principio de `VideojuegoService.create()`:

```java
System.out.println("[TRAZA] create() en hilo: " + Thread.currentThread().getName());
```

Y esta otra al principio del método `recibir(...)` de `ActividadVideojuegoEventConsumer`:

```java
System.out.println("[TRAZA] recibir() en hilo: " + Thread.currentThread().getName());
```

Con estas dos líneas, la siguiente comprobación te sirve a la vez para validar el flujo **y** para ver los dos hilos reales implicados.

---

## Paso 6 — Comprobar el flujo completo, y leer los dos hilos

Crea, modifica y borra un par de videojuegos con tu API de siempre, y consulta `GET /api/v1/actividad` (con un token de un usuario `ADMIN`). Aquí tienes los comandos con `curl`, pero puedes hacer exactamente lo mismo desde Swagger UI si lo prefieres:

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'

curl http://localhost:8080/api/v1/actividad -H "Authorization: Bearer $TOKEN_ADMIN"
```

**Comprueba**: que aparece un registro por cada operación, más reciente primero.

**Captura**: la respuesta de `GET /api/v1/actividad` con los registros de tus operaciones.

Échale un vistazo también a `http://localhost:15672` (la interfaz de administración) — entra en `actividad.videojuego.queue`, dentro de *Queues and Streams*.

!!! tip "Si no ves ningún mensaje "esperando", es normal"
    Con tu consumer arrancado y escuchando (`Consumers: 1`), RabbitMQ entrega cada mensaje casi al instante en cuanto llega — pasa de `Ready: 0` a `1` y vuelve a `0` en milisegundos, así que lo normal es que "Queued messages" te salga con todo a cero. Eso no es un fallo: es la prueba de que el mensaje se ha entregado y consumido correctamente, no que no haya pasado nada. Para ver el pico igualmente, cambia el desplegable de "Message rates" de *last minute* a **last 10 minutes** — con esa ventana más ancha sí se aprecia el pico de `Publish`/`Deliver` del momento en que has creado el videojuego. La prueba real de que ha funcionado, en cualquier caso, es el `GET /api/v1/actividad` de arriba, con un registro por operación.

**Captura**: la página de `actividad.videojuego.queue` en la interfaz de RabbitMQ (aunque salga todo a cero).

**Anota**: los dos nombres de hilo que han aparecido en la consola junto a las trazas `[TRAZA]` (deberían ser distintos — algo como `http-nio-8080-exec-N` para uno, y un nombre relacionado con el listener de RabbitMQ para el otro).

**Captura**: la consola con las dos trazas `[TRAZA]` visibles, una por cada hilo.

**Explica con tus propias palabras**, apoyándote en el diagrama de la teoría, por qué esos dos nombres no coinciden.

---

## Paso 7 — Retirar las trazas

!!! warning "No dejes las trazas temporales en tu código"
    Elimina las dos líneas `System.out.println("[TRAZA] ...")` que has añadido en el Paso 5 — eran solo para esta comprobación, no forman parte del código final.

---

## Pregunta final

`VideojuegoController` termina de responder (con un `201 Created`, por ejemplo) sin esperar a que `ActividadVideojuegoEventConsumer` haya terminado de guardar el registro de actividad. ¿Qué consecuencia práctica tiene esto si consultases `GET /api/v1/actividad` en el mismísimo instante después de crear un videojuego? ¿Es un problema real, o algo asumible dado lo que este registro representa?
