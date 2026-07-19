# 🧪 Actividad 3.1: RabbitMQ y el registro de actividad del catálogo

!!! info "Práctica guiada — misma sesión que la Actividad 3.2"
    Antes de observar hilos reales en la Actividad 3.2 (que hacéis en esta misma sesión de dos horas, a continuación), hace falta que existan. Hoy montas RabbitMQ en tu proyecto por primera vez, y construyes un registro de actividad del catálogo a partir de eventos: cada vez que se crea, modifica o borra un videojuego, un mensaje viaja por una cola y otro hilo (el del listener) lo registra, sin que la petición HTTP original tenga que esperar a que eso termine.

## Qué vas a practicar

- Añadir un broker de mensajería (RabbitMQ) a tu Dev Container y a tu proyecto.
- Publicar eventos del catálogo con `RabbitTemplate` y consumirlos con `@RabbitListener`.
- Construir un registro de actividad completo: entidad, servicio, endpoint protegido y consumer.

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

El puerto `5672` es el protocolo AMQP (por el que va a hablar tu aplicación); el `15672` es una interfaz web de administración — levanta el contenedor desde la terminal integrada (comprueba primero el nombre real de tu proyecto con `docker compose ls` — no siempre es `gamevault_devcontainer`, depende de tu editor — y sustitúyelo en `docker compose -f .devcontainer/docker-compose.yml -p <proyecto> up -d rabbitmq`), o reconstruyendo el Dev Container: "Dev Containers: Rebuild Container" en VS Code, o recreándolo desde `devcontainer.json` en IntelliJ IDEA. Después entra en `http://localhost:15672` (usuario y contraseña por defecto: `guest`/`guest`) para ver, más adelante, las colas y los mensajes pasando por ellas en vivo.

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

Como ya viste con `postgres` y `mongodb` en Acceso a Datos: `rabbitmq` es el nombre del servicio en `.devcontainer/docker-compose.yml`, no `localhost` — tu aplicación corre dentro de `app`, y RabbitMQ es un contenedor hermano más en la misma red.

---

## Paso 2 — El exchange y la cola de actividad

Un **exchange** es quien recibe los mensajes publicados y decide, según su *routing key*, a qué cola (o colas) los reenvía:

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

```java
package com.tunombre.gamevault.catalogo.api.eventos;

public record VideojuegoEvent(String tipo, Long videojuegoId) {
    public static final String VIDEOJUEGO_CREADO = "VIDEOJUEGO_CREADO";
    public static final String VIDEOJUEGO_ACTUALIZADO = "VIDEOJUEGO_ACTUALIZADO";
    public static final String VIDEOJUEGO_ELIMINADO = "VIDEOJUEGO_ELIMINADO";
}
```

```java
package com.tunombre.gamevault.catalogo;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.tunombre.gamevault.catalogo.api.eventos.VideojuegoEvent;
import com.tunombre.gamevault.config.RabbitMQConfig;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class VideojuegoEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    private final ObjectMapper objectMapper;

    public void publicar(String tipo, Long videojuegoId) {
        String accion = switch (tipo) {
            case VideojuegoEvent.VIDEOJUEGO_CREADO -> "creado";
            case VideojuegoEvent.VIDEOJUEGO_ACTUALIZADO -> "actualizado";
            case VideojuegoEvent.VIDEOJUEGO_ELIMINADO -> "eliminado";
            default -> throw new IllegalArgumentException("Tipo de evento desconocido: " + tipo);
        };
        try {
            String payload = objectMapper.writeValueAsString(new VideojuegoEvent(tipo, videojuegoId));
            rabbitTemplate.convertAndSend(RabbitMQConfig.CATALOGO_EXCHANGE, "videojuego." + accion, payload);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("No se ha podido serializar el evento", e);
        }
    }
}
```

`RabbitTemplate` es el equivalente en mensajería de `JdbcTemplate`, que ya conoces de Acceso a Datos: Spring Boot lo configura solo en cuanto detecta `spring-boot-starter-amqp` en el classpath, sin que lo registres tú.

Inyecta `VideojuegoEventPublisher` en `VideojuegoService` (constructor, como todo lo demás) y llama a `publicar(...)` al final de `create`, `update` y `delete`, con el tipo correspondiente y el `id` del videojuego.

**Pregunta**: `VideojuegoService` ahora depende de `VideojuegoEventPublisher`, pero no de nada de RabbitMQ directamente (ni `RabbitTemplate` aparece en `VideojuegoService`). ¿Por qué esa capa intermedia, en vez de inyectar `RabbitTemplate` directamente en `VideojuegoService`?

---

## Paso 4 — El registro de actividad: entidad, servicio y endpoint

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

```java
package com.tunombre.gamevault.actividad;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface ActividadRepository extends JpaRepository<Actividad, Long> {
    List<Actividad> findAllByOrderByFechaDesc();
}
```

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

```java
package com.tunombre.gamevault.actividad;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/v1/actividad")
@RequiredArgsConstructor
public class ActividadController {
    private final ActividadService actividadService;

    @GetMapping
    public ResponseEntity<List<Actividad>> getAll() {
        return ResponseEntity.ok(actividadService.listar());
    }
}
```

Y el consumer que conecta la cola con el servicio, en un hilo del contenedor de listeners de AMQP, no en el de la petición HTTP que originó el evento:

```java
package com.tunombre.gamevault.actividad;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.tunombre.gamevault.catalogo.api.eventos.VideojuegoEvent;
import com.tunombre.gamevault.config.RabbitMQConfig;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ActividadVideojuegoEventConsumer {
    private final ActividadService actividadService;
    private final ObjectMapper objectMapper;

    @RabbitListener(queues = RabbitMQConfig.ACTIVIDAD_VIDEOJUEGO_QUEUE)
    public void recibir(String payload) throws Exception {
        VideojuegoEvent event = objectMapper.readValue(payload, VideojuegoEvent.class);
        actividadService.registrar(event.tipo(), "Videojuego", event.videojuegoId().toString());
    }
}
```

Por último, en tu `SecurityConfig`, añade la regla de esta ruta nueva junto a las demás de la Actividad 2.5:

```java
.requestMatchers(HttpMethod.GET, "/api/v1/actividad").hasRole("ADMIN")
```

---

## Paso 5 — Comprobar el flujo completo

Crea, modifica y borra un par de videojuegos con tu API de siempre, y consulta `GET /api/v1/actividad` (con un token de un usuario `ADMIN`):

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'

curl http://localhost:8080/api/v1/actividad -H "Authorization: Bearer $TOKEN_ADMIN"
```

**Comprueba**: que aparece un registro por cada operación, más reciente primero. Échale un vistazo también a `http://localhost:15672` (la interfaz de administración) — en la sección *Queues* deberías ver `actividad.videojuego.queue` con mensajes entregados.

---

## Pregunta final

`VideojuegoController` termina de responder (con un `201 Created`, por ejemplo) sin esperar a que `ActividadVideojuegoEventConsumer` haya terminado de guardar el registro de actividad. ¿Qué consecuencia práctica tiene esto si consultases `GET /api/v1/actividad` en el mismísimo instante después de crear un videojuego? ¿Es un problema real, o algo asumible dado lo que este registro representa?

---

## ✅ Cierre

Tu GameVault ya tiene RabbitMQ funcionando y un registro de actividad completo, construido sobre un hilo distinto al de las peticiones HTTP. En la próxima actividad vas a **observar** esos hilos reales — los que ya has creado hoy sin necesidad de escribir `new Thread()` en ningún sitio.
