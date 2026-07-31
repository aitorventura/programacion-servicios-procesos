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

`new StompJs.Client({ brokerURL: ... })` crea el cliente apuntando a tu endpoint. `onConnect` es el callback que se dispara en cuanto el *handshake* termina — dentro, `client.subscribe(destino, callback)` te suscribe a `/topic/actividad`, y ese `callback` se ejecuta **cada vez** que llega un mensaje nuevo a ese destino, con el mensaje ya en `mensaje.body`. Lo único que haces con él aquí es crear un `<li>` y añadirlo a la lista — nada más.

Guarda esto ya como `src/main/resources/static/actividad.html`, ábrelo en `http://localhost:8080/actividad.html`, y compruébalo con el endpoint de prueba del Paso 3 antes de seguir — así sabes que el mecanismo funciona antes de complicar el fichero con nada más.

### La misma página, con algo más de cuidado visual

El HTML/CSS de abajo no es contenido nuevo de la actividad —es solo para que la demostración se vea mejor—: sigue siendo exactamente el mismo `client`/`onConnect`/`subscribe` de arriba, palabra por palabra; lo único que cambia es que `pintar(...)` hace algo más elaborado que un `textContent` a secas (colorea según el tipo de evento, y muestra cuánto hace que ha llegado). La parte que de verdad importa sigue señalada en los comentarios. Sustituye el contenido del fichero por esta versión:

```html
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Actividad en vivo</title>
<style>
    :root {
        --bg-0: #0a0c10;
        --panel: rgba(24, 28, 38, 0.72);
        --border: rgba(255,255,255,.08);
        --texto: #eef0f5;
        --tenue: #6f7686;
        --tenue-2: #9aa1b3;
        --brand: #7c8cff;
        --brand-glow: rgba(124,140,255,.35);
        --verde: #34d399;
        --azul: #60a5fa;
        --rojo: #fb7185;
        --gris: #9aa1b3;
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
        margin: 0; min-height: 100vh; padding: 3rem 1.25rem;
        font-family: -apple-system, 'Segoe UI', Inter, system-ui, sans-serif;
        color: var(--texto);
        background:
            radial-gradient(600px 400px at 15% -10%, rgba(124,140,255,.16), transparent 60%),
            radial-gradient(500px 380px at 100% 0%, rgba(52,211,153,.10), transparent 55%),
            var(--bg-0);
        display: flex; flex-direction: column; align-items: center;
        -webkit-font-smoothing: antialiased;
    }

    .panel {
        width: 100%; max-width: 560px;
        background: var(--panel);
        backdrop-filter: blur(18px);
        border: 1px solid var(--border);
        border-radius: 1.1rem;
        padding: 1.6rem 1.5rem 1.1rem;
        box-shadow: 0 30px 60px -20px rgba(0,0,0,.6), inset 0 1px 0 rgba(255,255,255,.04);
    }

    header { display: flex; align-items: center; gap: .75rem; }
    .icono-app {
        width: 2.35rem; height: 2.35rem; border-radius: .7rem; flex-shrink: 0;
        display: flex; align-items: center; justify-content: center;
        background: linear-gradient(155deg, var(--brand), #4c56c7);
        box-shadow: 0 6px 16px -4px var(--brand-glow);
    }
    .icono-app svg { width: 1.15rem; height: 1.15rem; }
    .titulos { flex: 1; min-width: 0; }
    h1 { font-size: 1.05rem; font-weight: 650; margin: 0; letter-spacing: -.01em; }
    .subtitulo { font-size: .78rem; color: var(--tenue); margin-top: .1rem; }

    .en-vivo {
        display: flex; align-items: center; gap: .4rem;
        font-size: .68rem; font-weight: 700; letter-spacing: .05em; text-transform: uppercase;
        color: var(--gris); background: rgba(154,161,179,.1);
        border: 1px solid rgba(154,161,179,.2);
        padding: .3rem .6rem .3rem .5rem; border-radius: 999px; flex-shrink: 0;
        transition: color .2s ease, background .2s ease, border-color .2s ease;
    }
    .en-vivo.conectado { color: var(--verde); background: rgba(52,211,153,.12); border-color: rgba(52,211,153,.25); }
    .punto { width: .4rem; height: .4rem; border-radius: 50%; background: currentColor; }
    .en-vivo.conectado .punto { box-shadow: 0 0 0 0 rgba(52,211,153,.65); animation: pulso 1.8s infinite; }
    @keyframes pulso {
        0%   { box-shadow: 0 0 0 0 rgba(52,211,153,.55); }
        70%  { box-shadow: 0 0 0 .45rem rgba(52,211,153,0); }
        100% { box-shadow: 0 0 0 0 rgba(52,211,153,0); }
    }

    .separador { height: 1px; background: var(--border); margin: 1.25rem 0 .9rem; }

    #mensajes { list-style: none; margin: 0; padding: 0; display: flex; flex-direction: column; gap: .35rem; }
    #mensajes li {
        display: flex; align-items: center; gap: .8rem;
        padding: .65rem .4rem; border-radius: .6rem;
        animation: entra .4s cubic-bezier(.16,1,.3,1);
        transition: background .15s ease;
    }
    #mensajes li:hover { background: rgba(255,255,255,.035); }
    @keyframes entra { from { opacity: 0; transform: translateY(-6px); } to { opacity: 1; transform: none; } }

    .chip {
        width: 2.1rem; height: 2.1rem; border-radius: .6rem; flex-shrink: 0;
        display: flex; align-items: center; justify-content: center;
        background: color-mix(in srgb, var(--tipo-color) 16%, transparent);
        border: 1px solid color-mix(in srgb, var(--tipo-color) 30%, transparent);
    }
    .chip svg { width: 1rem; height: 1rem; stroke: var(--tipo-color); }

    .cuerpo { flex: 1; min-width: 0; }
    .linea-1 { display: flex; align-items: baseline; gap: .45rem; flex-wrap: wrap; }
    .entidad { font-size: .88rem; font-weight: 600; color: var(--texto); }
    .entidad-id { font-size: .78rem; color: var(--tenue); font-weight: 400; }
    .tipo-label { font-size: .72rem; font-weight: 600; color: var(--tipo-color); }
    .detalle { font-size: .74rem; color: var(--tenue); margin-top: .1rem; }

    .vacio { color: var(--tenue); font-style: italic; text-align: center; padding: 2rem 0; font-size: .85rem; }

    footer { display: flex; justify-content: space-between; align-items: center; margin-top: .9rem; padding-top: .1rem; }
    .contador { font-size: .72rem; color: var(--tenue); }
    .contador b { color: var(--tenue-2); font-weight: 600; }
</style>
</head>
<body>
<div class="panel">
    <header>
        <div class="icono-app">
            <svg viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <rect x="2" y="6" width="20" height="12" rx="6"/>
                <circle cx="8.5" cy="12" r="1.5" fill="white" stroke="none"/>
                <circle cx="15.5" cy="10.5" r="1" fill="white" stroke="none"/>
                <circle cx="17" cy="13" r="1" fill="white" stroke="none"/>
            </svg>
        </div>
        <div class="titulos">
            <h1>Actividad en vivo</h1>
            <div class="subtitulo">Cambios en el catálogo de GameVault</div>
        </div>
        <div class="en-vivo" id="indicador"><span class="punto"></span><span id="texto-estado">conectando</span></div>
    </header>

    <div class="separador"></div>

    <ul id="mensajes"><li class="vacio">Esperando el primer evento…</li></ul>

    <footer>
        <span class="contador"><b id="num">0</b> eventos recibidos</span>
    </footer>
</div>

<script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
<script>
    // Todo este bloque es decoración: iconos, colores por tipo y tiempo relativo.
    // La parte que enseña la actividad de verdad empieza en "A partir de aquí" más abajo.
    const ICONOS = {
        CREADO:      '<path d="M12 5v14M5 12h14"/>',
        ACTUALIZADO: '<path d="M12 20h9"/><path d="M16.5 3.5a2.12 2.12 0 0 1 3 3L7 19l-4 1 1-4 12.5-12.5z"/>',
        BORRADO:     '<path d="M3 6h18"/><path d="M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/><path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6"/><path d="M10 11v6"/><path d="M14 11v6"/>',
    };
    const COLORES = { CREADO: 'var(--verde)', ACTUALIZADO: 'var(--azul)', BORRADO: 'var(--rojo)' };

    const lista = document.getElementById('mensajes');
    const numEl = document.getElementById('num');
    const indicador = document.getElementById('indicador');
    const textoEstado = document.getElementById('texto-estado');
    let esPrimero = true;
    let total = 0;

    function tiempoRelativo(fecha) {
        const s = Math.max(0, Math.round((Date.now() - fecha.getTime()) / 1000));
        if (s < 5) return 'justo ahora';
        if (s < 60) return `hace ${s} s`;
        return `hace ${Math.round(s / 60)} min`;
    }

    function pintar(cuerpoMensaje) {
        if (esPrimero) { lista.innerHTML = ''; esPrimero = false; }
        const li = document.createElement('li');

        let datos = null;
        try { datos = JSON.parse(cuerpoMensaje); } catch { /* el mensaje de prueba de hoy es solo texto plano */ }

        if (datos && datos.tipo) {
            const color = COLORES[datos.tipo] || 'var(--gris)';
            const icono = ICONOS[datos.tipo] || '<circle cx="12" cy="12" r="3"/>';
            const fecha = new Date(datos.fecha);
            li.style.setProperty('--tipo-color', color);
            li.innerHTML = `
                <span class="chip"><svg viewBox="0 0 24 24" fill="none" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">${icono}</svg></span>
                <span class="cuerpo">
                    <span class="linea-1">
                        <span class="tipo-label">${datos.tipo}</span>
                        <span class="entidad">${datos.entidad}</span>
                        <span class="entidad-id">#${datos.entidadId}</span>
                    </span>
                    <div class="detalle" data-fecha="${fecha.toISOString()}">${tiempoRelativo(fecha)}</div>
                </span>`;
        } else {
            li.style.setProperty('--tipo-color', 'var(--gris)');
            li.innerHTML = `<span class="chip"><svg viewBox="0 0 24 24" fill="none" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"/></svg></span>
                <span class="cuerpo">${cuerpoMensaje}</span>`;
        }

        lista.prepend(li);
        total++;
        numEl.textContent = total;
    }

    setInterval(() => {
        document.querySelectorAll('.detalle[data-fecha]').forEach(el => {
            el.textContent = tiempoRelativo(new Date(el.dataset.fecha));
        });
    }, 1000);

    // ── El mismo mecanismo de la versión sin estilo, sin cambios ───────────
    const client = new StompJs.Client({
        brokerURL: 'ws://localhost:8080/ws-actividad'  // conexión al endpoint
    });

    client.onConnect = () => {
        indicador.classList.add('conectado');
        textoEstado.textContent = 'en vivo';
        client.subscribe('/topic/actividad', (mensaje) => {  // suscripción al topic
            pintar(mensaje.body);  // pintar lo recibido
        });
    };

    client.onWebSocketClose = () => {
        indicador.classList.remove('conectado');
        textoEstado.textContent = 'desconectado';
    };

    client.activate();
</script>
</body>
</html>
```

Recarga `http://localhost:8080/actividad.html`.

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
