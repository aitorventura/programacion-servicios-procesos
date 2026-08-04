# 🧪 Actividad 4.3: Seguridad del canal

!!! info "Práctica guiada — última actividad del módulo"
    Auditas la seguridad del canal WebSocket que ya tienes funcionando desde la Actividad 4.2, y la corriges — no basta con detectarla.

## Qué vas a practicar

- Detectar **y corregir** una vulnerabilidad real de seguridad.
- Razonar por escrito sobre qué información expone un canal, y a quién.

---

## Requisitos previos

Tu endpoint `/ws-actividad` con la emisión real de `ActividadService` ya conectada (Actividad 4.2).

---

## Paso 1 — La auditoría de seguridad

Abre una ventana de **incógnito** (sin ninguna sesión iniciada, sin token) y conecta tu página `actividad.html` a `/ws-actividad` tal como está ahora mismo (sin la remediación todavía).

**Comprueba**: que, sin haber hecho login, ves en vivo exactamente los mismos registros que `GET /api/v1/actividad` solo permite consultar a `ADMIN`.

**Escribe el hallazgo** como una mini-incidencia de seguridad: qué se expone (el contenido concreto), a quién (cualquiera, sin autenticar), y qué opciones de mitigación existen (las tres de la teoría: restringir el handshake, filtrar qué se emite, o documentarlo como decisión consciente).

---

## Paso 2 — Remediación mínima obligatoria

Aplica la solución de la teoría de este apartado: el `JwtHandshakeInterceptor` que ya has visto ahí, letra por letra el mismo código — no hace falta repetirlo aquí. Dos detalles de conexión con tu propio proyecto, que sí son nuevos:

- Convierte también tu `WebSocketConfig` en `@RequiredArgsConstructor`, con un campo `JwtHandshakeInterceptor jwtHandshakeInterceptor` — hasta ahora no tenía ningún campo, y ahora necesita que Spring le inyecte el interceptor para poder registrarlo.
- En `registerStompEndpoints(...)`, añade `.addInterceptors(jwtHandshakeInterceptor)` a la cadena que ya tienes sobre `registry.addEndpoint(...)`.

Actualiza tu cliente `actividad.html` para pasar el token en la conexión:

```javascript
const client = new StompJs.Client({
    brokerURL: `ws://localhost:8080/ws-actividad?token=${miToken}`
});
```

Repite la prueba de la ventana de incógnito **sin** token: **comprueba** que ahora el handshake es rechazado. Repite con un token válido: **comprueba** que sigue funcionando exactamente igual que antes.

**Captura**: las dos pruebas — el rechazo sin token, y el funcionamiento normal con token válido.
