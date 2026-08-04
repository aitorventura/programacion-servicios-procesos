# 🧪 Actividad 4.2: El canal `/ws-actividad`, conectado a `ActividadService`

!!! warning "Descarga la plantilla"
    📄 [Plantilla 4.2 — El canal /ws-actividad, conectado a ActividadService](plantillas/Actividad_4_2_PSP_Plantilla.docx){target="_blank" rel="noopener"}

## Qué vas a practicar

- Configurar un endpoint WebSocket con STOMP en Spring.
- Abrir esa ruta en tu política de seguridad.
- Conectar el canal a un dato real de tu proyecto — nada de mensajes de prueba.
- Construir el cliente STOMP que lo recibe en vivo.

---

## Requisitos previos

Tu `SecurityConfig` con `anyRequest().denyAll()` (Tema 2), y el módulo `actividad` con `ActividadService.registrar()` ya funcionando.

---

## Paso 1 — La dependencia y la configuración

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```java
package com.tunombre.gamevault.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-actividad").setAllowedOriginPatterns("*");
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

`setAllowedOriginPatterns("*")` es la autorización de origen que ya has visto en la teoría de este apartado — aquí, `*` acepta cualquier origen, cómodo para probar hoy desde tu propia máquina.

---

## Paso 2 — Abrir la ruta en `SecurityConfig`

Con tu `anyRequest().denyAll()` del Tema 2, el handshake a `/ws-actividad` va a quedar bloqueado si no le añades su propia regla:

```java
.requestMatchers("/ws-actividad/**").permitAll()
.requestMatchers(HttpMethod.GET, "/actividad.html").permitAll()
```

Necesitas las dos reglas, no solo la primera: `/ws-actividad/**` abre el *handshake* del canal WebSocket, pero el propio fichero `actividad.html` que vas a crear en el Paso 4 es una petición GET normal y corriente, distinta del *handshake* — y sigue cayendo bajo `anyRequest().denyAll()` si no la abres también. Sin esta segunda regla, el navegador ni siquiera llega a cargar la página: un 401 antes de que el cliente STOMP tenga ocasión de conectar.

!!! tip "La discusión de fondo llega en la Actividad 4.3"
    Abrir esta ruta sin autenticación es, a propósito, una simplificación para poder probar el canal hoy. Si te preguntas si esto es sensato dejarlo así — buena intuición: es exactamente lo que vas a auditar y valorar en la última actividad del módulo.

---

## Paso 3 — La emisión real, guiada al completo

Nada de mensajes de prueba: vas a conectar el canal a un dato real de tu proyecto. Hasta ahora, `ActividadController.getAll()` devolvía la entidad `Actividad` directamente — el mismo problema que ya viste en Acceso a Datos (Tema 1, DTOs): expone tal cual algo pensado para que lo entienda Hibernate, no un cliente.

No es que WebSocket lo exija técnicamente —es una mala práctica que ya arrastrabas—, pero ya que hoy tocas `registrar()` para añadir la emisión, es el momento de arreglarla: introduce un DTO de salida propio, y reutilízalo también para publicar cada registro por WebSocket.

```java
package com.tunombre.gamevault.actividad;

import java.time.Instant;

public record ActividadResponseDTO(Long id, String tipo, String entidad, String entidadId, Instant fecha) {}
```

`id` y `entidadId` no son lo mismo: `id` es el identificador propio de ese registro de actividad (autogenerado, uno distinto por cada evento que se guarda); `entidadId` es el id de la entidad sobre la que trata ese evento —el videojuego que se ha creado, actualizado o borrado—. `Actividad` es un registro genérico que sirve para cualquier tipo de entidad, así que `entidadId` se guarda como texto, no como una relación de verdad; `entidad` (el campo de al lado) es el que dice de qué tipo se trata en cada caso.

Modifica `ActividadService`:

```java
@Service
@RequiredArgsConstructor
public class ActividadService {

    private final ActividadRepository actividadRepository;
    private final SimpMessagingTemplate messagingTemplate;

    public void registrar(String tipo, String entidad, String entidadId) {
        Actividad actividad = actividadRepository.save(new Actividad(tipo, entidad, entidadId));
        messagingTemplate.convertAndSend("/topic/actividad", mapToDTO(actividad));
    }

    public List<ActividadResponseDTO> listar() {
        return actividadRepository.findAllByOrderByFechaDesc()
                .stream()
                .map(this::mapToDTO)
                .toList();
    }

    private ActividadResponseDTO mapToDTO(Actividad a) {
        return new ActividadResponseDTO(a.getId(), a.getTipo(), a.getEntidad(), a.getEntidadId(), a.getFecha());
    }
}
```

Fíjate en que `registrar(...)` sigue recibiendo los mismos tres parámetros de siempre (`tipo`, `entidad`, `entidadId`) — no le añades ninguno nuevo, así que `ActividadVideojuegoEventConsumer` (Actividad 3.1) sigue llamándolo exactamente igual, sin que tengas que tocarlo. Lo único que cambia es que, tras guardar, el mismo método también publica el registro en `/topic/actividad`, ya convertido a `ActividadResponseDTO`.

Como `listar()` ahora devuelve `List<ActividadResponseDTO>` en vez de `List<Actividad>`, actualiza también el tipo genérico en `ActividadController.getAll()` (`ResponseEntity<List<ActividadResponseDTO>>`) — el cuerpo del método no cambia, sigue delegando en `listar()`.

**Pregunta**: sin tocar nada más, crear, actualizar y borrar un videojuego deberían empezar los tres a aparecer en vivo por el canal. Sigue el flujo completo (RabbitMQ → consumer → `registrar()`, visto en el Tema 3) y razona por qué añadir la emisión dentro de `registrar()` basta para cubrir los tres tipos de evento, sin tener que tocar tres sitios distintos.

---

## Paso 4 — El cliente, guiado al completo

### El mecanismo, sin nada de estilo

Esto es todo lo que hace falta para que funcione — tres piezas: conectar, suscribirte a un destino, y pintar cada mensaje que llegue:

```html
<!DOCTYPE html>
<html>
<head><title>Actividad en vivo</title></head>
<body>
    <h1>Actividad en vivo</h1>
    <ul id="mensajes"></ul>

    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <script>
        const client = new StompJs.Client({
            brokerURL: 'ws://localhost:8080/ws-actividad'  // conexión al endpoint
        });

        client.onConnect = () => {
            client.subscribe('/topic/actividad', (mensaje) => {  // suscripción al topic
                const li = document.createElement('li');
                li.textContent = mensaje.body;
                document.getElementById('mensajes').appendChild(li);  // pintar lo recibido
            });
        };

        client.activate();
    </script>
</body>
</html>
```

`new StompJs.Client({ brokerURL: ... })` crea el cliente apuntando a tu endpoint. `onConnect` es el callback que se dispara en cuanto el *handshake* termina — dentro, `client.subscribe(destino, callback)` te suscribe a `/topic/actividad`, y ese `callback` se ejecuta **cada vez** que llega un mensaje nuevo a ese destino, con el mensaje ya en `mensaje.body` (una cadena de texto en JSON — recuerda convertirla con `JSON.parse(...)` si quieres leer sus campos, en vez de pintarla tal cual). Lo único que hace este mecanismo mínimo es crear un `<li>` con ese texto y añadirlo a la lista — nada más.

Guarda esto ya como `src/main/resources/static/actividad.html`. Con esto ya tienes todo el código en su sitio —el endpoint (Paso 1), la ruta abierta (Paso 2), la emisión real (Paso 3) y ahora el cliente—, así que es el momento de reiniciar tu aplicación por fin: hasta ahora no tenías con qué comprobar nada de lo anterior. Ábrelo en `http://localhost:8080/actividad.html`.

### La misma página, con algo más de cuidado visual

!!! tip "Descarga el cliente con estilo"
    📄 [actividad_4_2_cliente.html](recursos/actividad_4_2_cliente.html){target="_blank" rel="noopener"} — descárgalo y sustituye con él el contenido de tu `src/main/resources/static/actividad.html`.

El HTML/CSS de ese fichero no es contenido nuevo de la actividad —es solo para que la demostración se vea mejor—: sigue siendo exactamente el mismo `client`/`onConnect`/`subscribe` de arriba, palabra por palabra; lo único que cambia es que `pintar(...)` hace algo más elaborado que un `textContent` a secas (colorea según el tipo de evento, y muestra cuánto hace que ha llegado — y ya interpreta el JSON, no hace falta que lo hagas tú). La parte que de verdad importa sigue señalada en los comentarios, marcada con "El mismo mecanismo de la versión sin estilo, sin cambios".

Recarga `http://localhost:8080/actividad.html`. Con el cliente conectado y suscrito, crea un videojuego desde Swagger o `curl` (con tu token de `ADMIN`):

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"titulo":"EnVivo","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: que el registro aparece en tu cliente (la página HTML) casi al instante, sin recargar nada.

**Captura**: la respuesta del `curl` (o Swagger) junto al registro recién llegado en `actividad.html`.

---

## Paso 5 — Demostración del pub-sub

Abre tu página `actividad.html` en **dos** pestañas del navegador a la vez (ambas deberían conectarse y suscribirse).

**Antes de crear nada, predícelo**: cuando crees un nuevo videojuego, ¿crees que el aviso va a llegar solo a la primera pestaña que se ha conectado, o a las dos? Razona tu respuesta apoyándote en el diagrama de "Cola de RabbitMQ vs. topic de STOMP" de la teoría de este apartado, antes de comprobarlo.

Crea otro videojuego.

**Comprueba**: que el aviso llega a **ambas** pestañas, tal y como has predicho (o no) — esta es la diferencia clave con petición-respuesta, observada en directo: nadie ha vuelto a pedir nada, y aun así ambos clientes se han enterado.

**Captura**: las dos pestañas mostrando el mismo registro, una junto a la otra.

---

## Pregunta de comprensión

Abre las herramientas de desarrollador del navegador (F12) **antes** de cargar la página — el *handshake* solo ocurre una vez, al conectar, así que si abres la pestaña de red después ya no lo verás salvo que recargues. Con las herramientas ya abiertas, ve a la pestaña **Network** (Red) y recarga `actividad.html`: verás la lista habitual de peticiones (el propio HTML, el script de `stomp.umd.min.js`...) más una con nombre `ws-actividad`. Filtra por **WS** (arriba de la lista de peticiones, junto a "All"/"Fetch/XHR"/etc.) si te cuesta encontrarla entre el resto — ese filtro deja solo las conexiones WebSocket. Haz clic en esa petición y mira su pestaña **Headers**: ahí están el código de estado y las cabeceras que te pide la pregunta.

**Anota**: ¿qué código de estado tiene? ¿Qué cabecera de la respuesta lo delata como distinto de una petición HTTP normal? ¿En qué se diferencia esta petición de todas las que has hecho en el curso hasta hoy (piensa en cuánto dura la conexión después de recibir esa respuesta)?
