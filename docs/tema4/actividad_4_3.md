# 🧪 Actividad 4.3: Actividad en vivo desde `ActividadService`

!!! info "Práctica guiada — última actividad del módulo"
    Conectas el canal WebSocket a los datos reales, auditas su seguridad, y la corriges — no basta con detectarla.

## Qué vas a practicar

- Emitir datos reales por el canal WebSocket que construiste en la Actividad 4.2.
- Trazar el viaje completo de un dato a través de todo lo construido en el curso.
- Detectar **y corregir** una vulnerabilidad real de seguridad.

---

## Requisitos previos

Tu endpoint `/ws-actividad` (Actividad 4.2) y tu `ActividadService`/consumer de RabbitMQ (Tema 3).

---

## Paso 1 — La emisión real, guiada al completo

Hasta ahora, `ActividadController.getAll()` devolvía la entidad `Actividad` directamente — cómodo, pero expone tu entidad JPA tal cual por la API, algo que ya sabes que conviene evitar. Aprovecha este paso para introducir un DTO de salida, y reutilízalo también para lo nuevo de hoy: publicar cada registro por WebSocket, además de guardarlo.

```java
package com.tunombre.gamevault.actividad;

import java.time.Instant;

public record ActividadResponseDTO(Long id, String tipo, String entidad, String entidadId, Instant fecha) {}
```

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

**Retira** el `WebSocketTestController` de emisión manual de la Actividad 4.2 — ya no lo necesitas, la emisión real ha ocupado su lugar.

---

## Paso 2 — La demostración completa

Abre tu página `actividad.html` (Actividad 4.2) en **dos** pestañas. Crea un videojuego desde Swagger o `curl`:

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"titulo":"EnVivo","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: que el registro aparece **en vivo**, en las dos pestañas a la vez, sin recargar nada.

**Captura**: una pantalla mostrando ambas pestañas con el registro recién llegado.

**Traza** el viaje completo del dato en tus logs (añade trazas temporales de `Thread.currentThread().getName()` si no las tienes ya de los Temas 3-4): un nombre de hilo para la petición HTTP, otro distinto para el listener de RabbitMQ que ejecuta `registrar()` (y, por tanto, el propio push WebSocket).

---

## Mini-reto — `update` y `delete` en vivo también

Sin tocar nada más, comprueba que actualizar y borrar un videojuego **también** aparecen en vivo en tus pestañas. Deberían funcionar solos, sin ningún cambio adicional.

**Explica por qué**: sigue el flujo completo (RabbitMQ → consumer → `registrar()`) y razona por qué añadir la emisión dentro de `registrar()` en el Paso 1 basta para cubrir los tres tipos de evento (crear, actualizar, borrar), sin tener que tocar tres sitios distintos.

---

## Paso 4 — La auditoría de seguridad

Abre una ventana de **incógnito** (sin ninguna sesión iniciada, sin token) y conecta tu página `actividad.html` a `/ws-actividad` tal como está ahora mismo (sin la remediación todavía).

**Comprueba**: que, sin haber hecho login, ves en vivo exactamente los mismos registros que `GET /api/v1/actividad` solo permite consultar a `ADMIN`.

**Escribe el hallazgo** como una mini-incidencia de seguridad: qué se expone (el contenido concreto), a quién (cualquiera, sin autenticar), y qué opciones de mitigación existen (las tres de la teoría: restringir el handshake, filtrar qué se emite, o documentarlo como decisión consciente).

---

## Paso 5 — Remediación mínima obligatoria

Aplica la solución de la teoría: exige un token válido en el handshake.

```java
@Component
@RequiredArgsConstructor
public class JwtHandshakeInterceptor implements HandshakeInterceptor {

    private final JwtDecoder jwtDecoder;

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                    WebSocketHandler wsHandler, Map<String, Object> attributes) {
        String query = request.getURI().getQuery();
        String token = extraerToken(query);

        if (token == null) {
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return false;
        }

        try {
            jwtDecoder.decode(token);
            return true;
        } catch (JwtException e) {
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return false;
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                WebSocketHandler wsHandler, Exception exception) { }

    private String extraerToken(String query) {
        if (query == null) return null;
        for (String param : query.split("&")) {
            if (param.startsWith("token=")) return param.substring(6);
        }
        return null;
    }
}
```

Regístralo en `WebSocketConfig`:

```java
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws-actividad")
            .setAllowedOriginPatterns("*")
            .addInterceptors(jwtHandshakeInterceptor);
}
```

Actualiza tu cliente `actividad.html` para pasar el token en la conexión:

```javascript
const client = new StompJs.Client({
    brokerURL: `ws://localhost:8080/ws-actividad?token=${miToken}`
});
```

Repite la prueba de la ventana de incógnito **sin** token: **comprueba** que ahora el handshake es rechazado. Repite con un token válido: **comprueba** que sigue funcionando exactamente igual que antes.

---

## Cierre del módulo

Escribe un repaso propio (5-6 líneas) de todo tu recorrido por PSP sobre este mismo GameVault: API REST leída, escrita, probada y monitorizada → asegurada con validación, BCrypt, JWT y roles → con tareas en segundo plano gestionadas con hilos propios → y ahora con comunicación en tiempo real, de sockets a mano a WebSocket seguro. ¿Qué pieza de todo el módulo te ha costado más entender, y por qué? Respuesta personal, no genérica.

---

## ✅ Cierre

Con esto se completa el módulo 0490 entero. Tu GameVault ya tiene servicios en red, seguridad, tareas asíncronas y comunicación en tiempo real — construido pieza a pieza, semana a semana, sobre el mismo proyecto que también trabajaste en Acceso a Datos.
