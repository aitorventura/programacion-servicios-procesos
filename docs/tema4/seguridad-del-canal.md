<a id="seguridad-del-canal"></a>

# 🧩 3. Seguridad del canal

Último apartado del módulo. Tu canal `/ws-actividad` ya emite datos reales, conectado a `ActividadService` desde la Actividad 4.2. Hoy afrontas algo que has dejado pendiente a propósito: qué implica en seguridad haberlo abierto sin autenticación.

---

## 🔓 El aviso de seguridad del handshake

Aquí llega el momento de mirar atrás con ojo crítico. En la Actividad 4.2 has abierto `/ws-actividad` con `permitAll()` — sin exigir ninguna autenticación. Pero `GET /api/v1/actividad` (la misma información, por REST) exige rol `ADMIN`. Hay una incoherencia real: **¿puede un usuario anónimo suscribirse al topic y ver en vivo exactamente lo que por REST está protegido?**

La respuesta es sí, y el motivo técnico es concreto: el *handshake* de WebSocket, tal como lo has configurado, no lleva el JWT por ningún sitio — un cliente que ni siquiera ha hecho login puede conectar y suscribirse sin problema. Las opciones frente a esto:

1. **Restringir el handshake**: exigir un token válido antes de aceptar la conexión.
2. **Filtrar qué se emite**: dejar el canal abierto, pero no mandar por él nada sensible.
3. **Asumirlo y documentarlo** como decisión consciente para un canal de demostración.

!!! danger "Aquí no vale quedarse solo en detectar y documentar"
    Terminar el módulo con un agujero de seguridad real, detectado y descrito por escrito pero sin corregir, transmite el mensaje contrario a todo lo que has construido en programación segura. La remediación **mínima** es obligatoria, no opcional — la vas a aplicar tú mismo en la Actividad 4.3.

### La remediación mínima: JWT en el handshake

La forma más simple de exigir el token en el handshake es pasarlo como parámetro de consulta en la propia URL de conexión (`/ws-actividad?token=...`), y validarlo con un **interceptor** antes de aceptar — un gancho que Spring ejecuta justo antes (y justo después) de que la conexión se establezca, y en el que tu código decide, con un simple `true`/`false`, si esa conexión sigue adelante o se corta ahí mismo:

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
            return false; // rechaza el handshake
        }

        try {
            jwtDecoder.decode(token); // lanza excepción si no es válido
            return true;
        } catch (JwtException e) {
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return false;
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                WebSocketHandler wsHandler, Exception exception) {
        // no hace falta nada aquí para este caso
    }

    private String extraerToken(String query) {
        if (query == null) return null;
        for (String param : query.split("&")) {
            if (param.startsWith("token=")) return param.substring(6);
        }
        return null;
    }
}
```

Al ser un `@Component`, Spring lo inyecta como cualquier otra dependencia — incluida tu propia `WebSocketConfig`, que hasta ahora no necesitaba ningún campo:

```java
@Configuration
@EnableWebSocketMessageBroker
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final JwtHandshakeInterceptor jwtHandshakeInterceptor;

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-actividad")
                .setAllowedOriginPatterns("*")
                .addInterceptors(jwtHandshakeInterceptor);
    }

    // configureMessageBroker sigue exactamente igual
}
```

`beforeHandshake` se ejecuta **antes** de que la conexión WebSocket se establezca — si devuelve `false`, el *handshake* se rechaza y la conexión nunca llega a completarse. Reutilizas el mismo `JwtDecoder` que ya tenías configurado desde el Tema 2 para validar el token — no hace falta ninguna pieza criptográfica nueva, ni tampoco instanciar el interceptor a mano: al llevar `@RequiredArgsConstructor` en ambas clases, Spring construye y conecta las dos piezas solo.

---

## 📝 Depuración y documentación

Documenta el canal — endpoint, topic, formato del mensaje, política de acceso — de la misma forma en que has documentado la API REST con OpenAPI. Con una salvedad: WebSocket no aparece en Swagger (que solo describe HTTP tradicional), así que este canal se documenta **a mano**, en un fichero aparte.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Un canal abierto sin autenticación puede exponer, por WebSocket, información que por REST está protegida — una incoherencia real que hay que resolver, no solo señalar.
    - Un `HandshakeInterceptor` puede rechazar la conexión WebSocket antes de establecerse, validando un token pasado como parámetro de consulta.
    - Documentar y detectar un agujero de seguridad no sustituye corregirlo — la remediación mínima es parte obligatoria de la entrega.
    - WebSocket no aparece en Swagger — se documenta a mano.
