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

La forma más simple de exigir el token en el handshake es pasarlo como parámetro de consulta en la propia URL de conexión (`/ws-actividad?token=...`), y validarlo con un **interceptor** antes de aceptar.

Ese parámetro no lo añade el navegador por su cuenta —no hay ninguna cookie ni ninguna sesión de por medio—: lo pone el propio cliente, al construir la URL de conexión. Es el mismo `new StompJs.Client({ brokerURL: ... })` que ya conoces, solo que ahora la URL lleva el token pegado al final:

```javascript
const client = new StompJs.Client({
    brokerURL: `ws://localhost:8080/ws-actividad?token=${token}`
});
```

Esa es la URL exacta que el navegador manda como petición del *handshake* — y del lado del servidor hace falta algo que la lea y decida si esa conexión sigue adelante. Ahí entra el **interceptor**: un gancho que Spring ejecuta justo antes (y justo después) de que la conexión se establezca, y en el que tu código decide, con un simple `true`/`false`, si esa conexión sigue adelante o se corta ahí mismo.

Spring representa ese gancho con una interfaz, `HandshakeInterceptor`, con dos métodos que tienes que implementar: `beforeHandshake` (donde decides si la conexión sigue) y `afterHandshake` (que se ejecuta después de establecerse, útil para *logging* o limpieza — hoy no la necesitas para nada). Así queda una clase que la implementa, inyectando el mismo `JwtDecoder` que ya usas para validar tokens en el resto de la API:

```java
package com.tunombre.gamevault.seguridad; // el mismo paquete donde ya tienes JwtService, SecurityConfig y el resto de piezas de seguridad

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
        if (query == null) return null; // sin query string, no hay token que buscar
        for (String param : query.split("&")) { // "token=abc&otro=xyz" -> ["token=abc", "otro=xyz"]
            if (param.startsWith("token=")) return param.substring(6); // se queda solo con lo que va después de "token="
        }
        return null; // había query string, pero sin ningún "token="
    }
}
```

Aquí no tienes ningún `@RequestParam` que te lo resuelva solo, como en un controller REST normal: `beforeHandshake` recibe la petición en crudo, así que `extraerToken` recorre la *query string* a mano, param a param, hasta encontrar el que empieza por `"token="`.

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

!!! tip "Por qué esto es una solución sencilla para la práctica"
    Pasar el token como parámetro de la URL (`?token=...`) tiene un inconveniente conocido: la URL completa puede quedar registrada en logs del servidor, proxies o herramientas de monitorización. Para esta práctica local es una solución sencilla y fácil de observar; en una aplicación real convendría utilizar un mecanismo de autenticación del canal que no exponga el token en la propia URL.

Fíjate también en qué comprueba exactamente `jwtDecoder.decode(token)`: que el token es válido, no que su dueño tiene rol `ADMIN`. Cierra la puerta a quien no ha hecho login en absoluto —que era el agujero real que has detectado en el Paso 1 de la Actividad 4.3—, pero no reproduce del todo la misma política que `GET /api/v1/actividad`: cualquier usuario autenticado, no solo uno con rol `ADMIN`, sigue viendo el canal en vivo. Es exactamente lo que vas a razonar en esa misma actividad.

---

## 📝 Depuración y documentación

WebSocket no aparece en Swagger (que solo describe HTTP tradicional), así que la nueva ruta no queda documentada en ningún sitio automáticamente. Pero no hace falta inventar un fichero nuevo solo para esto: `/ws-actividad` es, ante todo, una ruta más con su propia política de acceso — y para eso ya existe `docs/seguridad.md`, creado en la Actividad 2.5. Añade una fila igual que las demás:

```markdown
## WebSocket

| Ruta | Verbo | Quién puede |
|---|---|---|
| `/ws-actividad` (handshake, `Upgrade: websocket`) | GET | Cualquiera con un JWT válido — no distingue rol |
```

Es el mismo documento en el que ya clasificaste el resto de rutas del proyecto — no hace falta ningún criterio distinto para esta, aunque no sea HTTP tradicional.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Un canal abierto sin autenticación puede exponer, por WebSocket, información que por REST está protegida — una incoherencia real que hay que resolver, no solo señalar.
    - Un `HandshakeInterceptor` puede rechazar la conexión WebSocket antes de establecerse, validando un token pasado como parámetro de consulta.
    - Esa validación comprueba que el token es válido, no el rol de su dueño: cierra el agujero de autenticación, pero no reproduce la política de solo-`ADMIN` del REST — es una remediación mínima, con ese límite consciente.
    - Documentar y detectar un agujero de seguridad no sustituye corregirlo — la remediación mínima es parte obligatoria de la entrega.
    - WebSocket no aparece en Swagger — documentas la ruta a mano, en el mismo `docs/seguridad.md` creado en el Tema 2, no en un fichero aparte.
