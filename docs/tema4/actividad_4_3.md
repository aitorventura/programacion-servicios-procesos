# 🧪 Actividad 4.3: Seguridad del canal

!!! warning "Descarga la plantilla"
    📄 [Plantilla 4.3 — Seguridad del canal](plantillas/Actividad_4_3_PSP_Plantilla.docx){target="_blank" rel="noopener"}

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

**Captura**: la ventana de incógnito mostrando esos registros en vivo, sin haber iniciado sesión.

**Escribe el hallazgo** como una mini-incidencia de seguridad: qué se expone (el contenido concreto), a quién (cualquiera, sin autenticar), y qué opciones de mitigación existen (las tres de la teoría: restringir el handshake, filtrar qué se emite, o documentarlo como decisión consciente).

---

## Paso 2 — Remediación mínima obligatoria

Aplica la solución de la teoría de este apartado: el `JwtHandshakeInterceptor` que ya has visto ahí, letra por letra el mismo código — no hace falta repetirlo aquí. Dos detalles de conexión con tu propio proyecto, que sí son nuevos:

- Convierte también tu `WebSocketConfig` en `@RequiredArgsConstructor`, con un campo `JwtHandshakeInterceptor jwtHandshakeInterceptor` — hasta ahora no tenía ningún campo, y ahora necesita que Spring le inyecte el interceptor para poder registrarlo.
- En `registerStompEndpoints(...)`, añade `.addInterceptors(jwtHandshakeInterceptor)` a la cadena que ya tienes sobre `registry.addEndpoint(...)`.

Con el interceptor y la configuración ya escritos, reinicia tu aplicación — es el único reinicio que hace falta en toda la actividad, y es por ese código Java, que Spring no recarga solo.

Actualiza también tu cliente `actividad.html`: en vez de dejar el token fijo en el código (tendrías que editar el fichero cada vez que quisieras probar con otro), añade un campo de texto y un botón, y pega el token ahí en cada prueba — el mismo patrón que ya conoces del botón "Authorize" de Swagger:

```html
<input type="text" id="token-input" placeholder="Pega tu token JWT...">
<button id="conectar-btn">Conectar</button>
```

```javascript
let client = null;

document.getElementById('conectar-btn').addEventListener('click', () => {
    if (client) {
        client.deactivate(); // cierra cualquier conexión anterior antes de abrir una nueva
    }

    const token = document.getElementById('token-input').value;

    client = new StompJs.Client({
        brokerURL: `ws://localhost:8080/ws-actividad?token=${token}`
    });

    client.onConnect = () => { /* igual que hasta ahora */ };
    client.onWebSocketClose = () => { /* igual que hasta ahora */ };
    client.activate();
});
```

Con esto, cada prueba es tan simple como pegar un valor distinto en el campo y pulsar **Conectar** — sin volver a tocar el fichero ni recargar la página. Fíjate en que `client` ya no se declara dentro del propio *listener*: si pulsas **Conectar** dos veces (por ejemplo, primero con un token erróneo y después con uno válido), sin ese `deactivate()` el primer cliente se queda abierto de fondo, reintentando con el token equivocado — y su propio `onWebSocketClose`, al fallar, puede pisar el indicador del segundo cliente aunque ese sí esté conectado de verdad.

!!! tip "Descarga el cliente ya migrado"
    📄 [actividad_4_3_cliente.html](recursos/actividad_4_3_cliente.html){target="_blank" rel="noopener"} — descárgalo y sustituye con él el contenido de tu `src/main/resources/static/actividad.html`, si prefieres no aplicar el cambio a mano sobre el que ya tienes de la Actividad 4.2.

!!! tip "Así llega el token al servidor"
    Al construir el cliente con ese `brokerURL`, el navegador manda esa URL exacta —con el token ya incrustado— como petición del *handshake*. Es de ahí de donde `beforeHandshake` lo lee, con `request.getURI().getQuery()`. Pulsa **Conectar** con el campo vacío y el *handshake* se rechaza — es justo el caso "sin token" que vas a comprobar primero.

Prueba ahora los dos casos: pulsa **Conectar** con el campo vacío — **comprueba** que el handshake es rechazado. Pega un `accessToken` real (login por Swagger o `curl` como siempre, Actividad 2.4) y pulsa **Conectar** de nuevo — **comprueba** que funciona exactamente igual que antes.

!!! tip "Al rechazarse, no esperes ningún mensaje de error en la página"
    El indicador simplemente pasa a "desconectado" (tu propio `onWebSocketClose`, en `actividad_4_2_cliente.html`) — no hay ningún error visible, ni en la página ni en la consola. El navegador no expone a JavaScript el código de estado real de un *handshake* rechazado, por seguridad; el 401 solo se ve donde ya aprendiste a buscarlo en la Actividad 4.2: **F12 → Network → filtro WS**, en la petición del *handshake*. Ese "desconectado" sin más, sin token, es la prueba de que el rechazo funciona — no una señal de que algo se ha roto.

**Captura**: las dos pruebas — el rechazo sin token, y el funcionamiento normal con token válido.

**Pregunta**: repite la prueba con un token válido pero de un usuario **sin** rol `ADMIN` (registra uno nuevo si no tienes ninguno). ¿Se conecta? Compáralo con lo que exige `GET /api/v1/actividad` por REST, y razona: ¿la remediación que acabas de aplicar deja completamente cerrado el hallazgo del Paso 1, o solo una parte? ¿Qué comprobación le falta a `JwtHandshakeInterceptor` para cerrarlo del todo?

**Captura**: esa misma prueba, con el token de un usuario sin rol `ADMIN`.

**Documenta**: añade la fila de `/ws-actividad` a tu `docs/seguridad.md` (mismo formato que el resto de la tabla, ver la teoría de este apartado).
