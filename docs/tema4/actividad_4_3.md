# 🧪 Actividad 4.3: Seguridad del canal

!!! warning "Descarga la plantilla"
    📄 [Plantilla 4.3 — Seguridad del canal](plantillas/Actividad_4_3_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — última actividad del módulo"
    Auditas la seguridad del canal WebSocket que ya tienes funcionando desde la Actividad 4.2, corriges su acceso anónimo y compruebas después qué problema de autorización sigue pendiente.

## Qué vas a practicar

- Detectar y corregir la exposición de un canal a usuarios anónimos.
- Distinguir **autenticación** (tener un JWT válido) de **autorización** (tener el rol necesario).
- Razonar por escrito sobre qué información expone un canal, y a quién.

---

## Requisitos previos

Tu endpoint `/ws-actividad` con la emisión real de `ActividadService` ya conectada (Actividad 4.2).

---

## Paso 1 — La auditoría de seguridad

Abre una ventana de **incógnito** (sin ninguna sesión iniciada, sin token) y carga `actividad.html` tal como está ahora mismo, antes de aplicar la remediación.

Con esa ventana abierta y conectada al canal, crea o modifica un videojuego desde otra ventana o desde `curl`, utilizando allí tu `ADMIN_TOKEN`.

**Comprueba**: que la ventana de incógnito recibe en vivo el nuevo registro de actividad aunque ella no haya enviado ningún token ni haya iniciado sesión.

**Captura**: la ventana de incógnito mostrando el nuevo registro recibido, junto con la prueba de que ese cliente no se ha autenticado.

**Escribe el hallazgo** como una mini-incidencia de seguridad: qué información se expone, a quién y qué opciones de mitigación existen.

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

!!! tip "Dónde comprobar un handshake rechazado"
    La página puede limitarse a mostrar que la conexión no se ha establecido y la consola del navegador puede mostrar únicamente un error genérico de WebSocket. Para comprobar el código HTTP concreto del rechazo, utiliza **F12 → Network → filtro WS** y abre la petición del *handshake*. Ahí podrás distinguir claramente el rechazo de una conexión que sí ha llegado a establecerse.

**Captura**: las dos pruebas — el rechazo sin token, y el funcionamiento normal con token válido.

**Pregunta**: repite la prueba con un token válido pero de un usuario **sin** rol `ADMIN` (registra uno nuevo si no tienes ninguno). ¿Se conecta? Compáralo con lo que exige `GET /api/v1/actividad` por REST, y razona: ¿la remediación que acabas de aplicar deja completamente cerrado el hallazgo del Paso 1, o solo una parte? ¿Qué comprobación le falta a `JwtHandshakeInterceptor` para cerrarlo del todo?

**Captura**: esa misma prueba, con el token de un usuario sin rol `ADMIN`.

**Documenta**: añade la fila de `/ws-actividad` a tu `docs/seguridad.md` (mismo formato que el resto de la tabla, ver la teoría de este apartado).
