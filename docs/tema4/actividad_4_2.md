# 🧪 Actividad 4.2: El endpoint `/ws-actividad` con STOMP

!!! warning "Se construye completamente guiado, desde cero"
    No hay ningún ejemplo previo con el que comparar tu resultado — vas a seguir el enunciado paso a paso. Es una ampliación de profundidad sobre lo ya construido con sockets clásicos en la Actividad 4.1, no un punto de partida distinto.

## Qué vas a practicar

- Configurar un endpoint WebSocket con STOMP en Spring.
- Abrir esa ruta en tu política de seguridad.
- Probar el canal con un cliente STOMP mínimo.

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

Reinicia tu aplicación.

---

## Paso 2 — Abrir la ruta en `SecurityConfig`

Con tu `anyRequest().denyAll()` del Tema 2, el handshake a `/ws-actividad` va a quedar bloqueado si no le añades su propia regla:

```java
.requestMatchers("/ws-actividad/**").permitAll()
```

!!! tip "La discusión de fondo llega en la Actividad 4.3"
    Abrir esta ruta sin autenticación es, a propósito, una simplificación para poder probar el canal hoy. Si te preguntas si esto es sensato dejarlo así — buena intuición: es exactamente lo que vas a auditar y valorar en la última actividad del módulo.

---

## Paso 3 — Un endpoint de prueba para emitir algo

Para comprobar que el canal funciona antes de conectarlo al flujo real, añade un endpoint temporal:

```java
@RestController
@RequestMapping("/api/v1/test")
@RequiredArgsConstructor
public class WebSocketTestController {

    private final SimpMessagingTemplate messagingTemplate;

    @PostMapping("/emitir")
    public ResponseEntity<Void> emitir() {
        messagingTemplate.convertAndSend("/topic/actividad", "Mensaje de prueba: " + java.time.Instant.now());
        return ResponseEntity.ok().build();
    }
}
```

`SimpMessagingTemplate` es la pieza que te permite publicar manualmente en un destino STOMP, sin que tenga que originarse en un cliente. Añade también esta ruta a tu política de seguridad (`permitAll()`, temporal, solo para esta prueba).

---

## Paso 4 — El cliente, guiado al completo

Crea `src/main/resources/static/actividad.html`:

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

Ábrela en `http://localhost:8080/actividad.html`.

Con el cliente conectado y suscrito, dispara el mensaje de prueba:

```bash
curl -X POST http://localhost:8080/api/v1/test/emitir
```

**Comprueba**: que el mensaje aparece en tu cliente (la página HTML) casi al instante.

---

## Paso 5 — Demostración del pub-sub

Abre tu página `actividad.html` en **dos** pestañas del navegador a la vez (ambas deberían conectarse y suscribirse). Dispara de nuevo el mensaje de prueba.

**Comprueba**: que el mensaje llega a **ambas** pestañas — esta es la diferencia clave con petición-respuesta, observada en directo: nadie ha vuelto a pedir nada, y aun así ambos clientes se han enterado.

---

## Pregunta de comprensión

Abre las herramientas de desarrollador del navegador (F12), pestaña de red, y localiza la petición del *handshake* hacia `/ws-actividad`. **Anota**: ¿qué código de estado tiene? ¿Qué cabecera de la respuesta lo delata como distinto de una petición HTTP normal? ¿En qué se diferencia esta petición de todas las que has hecho en el curso hasta hoy (piensa en cuánto dura la conexión después de recibir esa respuesta)?

---

## ✅ Cierre

Tienes un canal WebSocket funcionando, probado con un mensaje manual. En la última actividad lo conectas al flujo real de `ActividadService` y auditas la seguridad del handshake que has dejado abierta hoy.
