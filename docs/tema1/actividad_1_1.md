# 🧪 Actividad 1.1: Diseccionando peticiones y respuestas HTTP

!!! warning "Descarga la plantilla"
    📄 [Plantilla 1.1 — Diseccionando peticiones y respuestas HTTP](plantillas/Actividad_1_1_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — sin código, y sin servidor propio todavía"
    En este punto del curso, en Acceso a Datos tu propio GameVault todavía solo tiene entidades JPA — no existe ningún endpoint que puedas invocar. Así que hoy no vas a lanzar peticiones contra tu proyecto: vas a diseccionar, con papel y con `curl` contra un objetivo controlado, conversaciones HTTP reales, para que en cuanto tu propio CRUD esté vivo (Actividad 1.2), ya sepas leer lo que ves.

## Qué vas a practicar

- Diseccionar una petición y una respuesta HTTP reales línea a línea: verbo, ruta, cabeceras, código de estado, cuerpo.
- Relacionar cada trozo de la conversación HTTP con la anotación Java que lo genera (según lo visto en la teoría).
- Practicar la sintaxis de `curl` contra un servidor local mínimo, sin depender de tu propio proyecto.

---

## Requisitos previos

Los apuntes del apartado de teoría de esta semana. En el Paso 3 puedes instalar opcionalmente una herramienta llamada `jq`, si quieres — se explica ahí.

Vas a necesitar citar el código de `LibroController`/`LibroService` de la teoría (secciones "📖 Leyendo un controlador REST completo" y "❌ ¿Y si el libro no existe?") — tenlo a mano mientras respondes.

---

## Paso 1 — Diseccionar una conversación HTTP dada

Aquí tienes una captura real de una petición y su respuesta, tal como las vería `curl -v`:

```text
> GET /api/v1/videojuegos/3 HTTP/1.1
> Host: localhost:8080
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Content-Length: 80
<
{"id":3,"titulo":"Hollow Knight","precio":14.99,"fechaLanzamiento":"2017-02-24"}
```

Recuerda: `>` es lo que **envía** el cliente, `<` es lo que **responde** el servidor.

**Responde**, señalando exactamente qué línea o fragmento usas para justificar cada respuesta:

1. ¿Qué **verbo** y qué **ruta** se han pedido?
2. ¿Qué **código de estado** ha devuelto el servidor? ¿A qué familia pertenece (2xx/4xx/5xx)?
3. ¿Qué dice la cabecera `Content-Type` de la respuesta sobre el formato del cuerpo?
4. En la teoría leíste `LibroController`, que sigue este mismo patrón sobre libros en vez de videojuegos. ¿Qué método exacto (con qué anotación) sería el responsable de generar esta respuesta concreta si la ruta fuera `/api/v1/libros/3` en vez de `/api/v1/videojuegos/3`? Justifica tu respuesta relacionando la ruta pedida con la anotación del método.

---

## Paso 2 — Ahora un caso de error

Segunda captura:

```text
> GET /api/v1/videojuegos/9999 HTTP/1.1
> Host: localhost:8080
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Content-Type: application/json
<
{"status":404,"error":"Not Found","message":"Videojuego no encontrado"}
```

**Predicción antes de razonar**: si tuvieras que adivinar solo por el código de estado, sin leer el cuerpo, ¿qué habría pasado? Escribe tu respuesta.

**Responde**:

1. ¿Por qué crees que el servidor ha respondido `404` en vez de `200`?
2. En el diagrama y el código de "¿Y si el libro no existe?" que viste en la teoría, ¿qué línea exacta es responsable de que la respuesta sea `404` en vez de `200`? Justifica tu respuesta señalando esa línea.
3. ¿Este `404` lo genera Tomcat automáticamente, o nace de una decisión explícita del código de la aplicación? Justifica tu respuesta con la línea de código concreta.

---

## Paso 3 — Practicar `curl` de verdad, contra un servidor mínimo

No necesitas tu proyecto para practicar la herramienta en sí, pero sí necesitas Python — y tu Dev Container, tal como lo dejaste en la Actividad 1.1 de Acceso a Datos, todavía no lo tiene (solo instalaste la *feature* de Java). Añádela en `devcontainer.json`, junto a las que ya tienes:

```json
"features": {
  "ghcr.io/devcontainers/features/java:1": {
    "version": "21",
    "installMaven": "true"
  },
  "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
  "ghcr.io/devcontainers/features/python:1": {}
}
```

Reconstruye el contenedor para que la nueva *feature* se instale (en VS Code, paleta de comandos → **"Dev Containers: Rebuild Container"**; en IntelliJ IDEA, cierra la conexión desde la ventana **Services** y vuelve a crear el Dev Container desde `devcontainer.json`) — recuerda que editar `devcontainer.json` no tiene efecto hasta que reconstruyes.

Con Python ya disponible, usa su servidor HTTP mínimo integrado — suficiente para tener algo real contra lo que lanzar peticiones:

```bash
mkdir curl-practica && cd curl-practica
echo '{"mensaje": "hola"}' > datos.json
python3 -m http.server 8000
```

En otra terminal:

```bash
curl -v http://localhost:8000/datos.json
```

**Comprueba**: verbo, código de estado y `Content-Type` de la respuesta real que acabas de recibir. Compáralos con los que anotaste en el Paso 1 — ¿tienen la misma forma general, aunque el servidor sea completamente distinto?

**Captura**: la salida completa de `curl -v http://localhost:8000/datos.json` en tu terminal.

Prueba también a pedir un fichero que no existe:

```bash
curl -v http://localhost:8000/no-existe.json
```

**Comprueba**: el código de estado. ¿Coincide con el que esperarías después del Paso 2?

**Captura**: la salida completa de `curl -v http://localhost:8000/no-existe.json` en tu terminal.

!!! tip "Opcional: leer el JSON con más comodidad con `jq`"
    Fíjate en que `curl` te devuelve el JSON del cuerpo tal cual, en una sola línea comprimida — con una respuesta pequeña como `{"mensaje": "hola"}` se lee bien, pero con una respuesta real, más larga y anidada, se vuelve incómodo de leer a simple vista. **`jq`** es una herramienta de línea de comandos que lee JSON de la entrada y lo vuelve a imprimir indentado (y, si tu terminal lo soporta, coloreado) — no cambia el JSON, solo lo hace más legible.

    No viene instalada por defecto en tu Dev Container. Para probarla, instálala dentro del contenedor con:
    ```bash
    sudo apt-get update && sudo apt-get install -y jq
    ```
    Y compara la diferencia:
    ```bash
    curl -s http://localhost:8000/datos.json | jq
    ```
    Es completamente opcional — para esta actividad te basta con leer el JSON tal como lo devuelve `curl -v`.

Detén el servidor con `Ctrl+C` cuando termines, y borra la carpeta `curl-practica` — era solo para esta prueba, no forma parte de tu proyecto:

```bash
cd .. && rm -rf curl-practica
```

---

## Mini-reto — repite la disección con un tercer caso

Aquí tienes una tercera captura, esta vez de un `POST`:

```text
> POST /api/v1/videojuegos HTTP/1.1
> Host: localhost:8080
> Content-Type: application/json
> Content-Length: 71
>
{"titulo":"Celeste","precio":19.99,"fechaLanzamiento":"2018-01-25","estudioId":1}
< HTTP/1.1 201 Created
< Content-Type: application/json
<
{"id":7,"titulo":"Celeste","precio":19.99,"fechaLanzamiento":"2018-01-25"}
```

Sin más indicaciones que las de los Pasos 1-2, diseccionalo tú mismo: verbo, ruta, cabeceras relevantes de la petición (¿por qué esta petición SÍ lleva `Content-Type` y la del Paso 1 no?), código de estado y qué significa exactamente ese código para un `POST` (repasa la tabla de verbos y códigos de la teoría si hace falta).

---

## Pregunta final

`curl` no sabe nada de Java, ni de Spring, ni de cómo está construido un proyecto por dentro — y aun así has podido usarlo contra un servidor de Python en el Paso 3 con la misma sintaxis que usarías contra cualquier API REST. ¿Por qué? Relaciona tu respuesta con la idea de **protocolo estándar** vista en la teoría: ¿qué tendría que pasar para que `curl` (o cualquier otro cliente) NO pudiera hablar con un servidor?

---

## ✅ Cierre

Ya sabes leer una conversación HTTP real —verbo, ruta, código de estado, cabeceras, cuerpo— y relacionarla con el código Java que la genera. En cuanto tu propio GameVault tenga su primer endpoint (Acceso a Datos, Actividad 1.2), vas a lanzarle estas mismas preguntas a peticiones de verdad, contra tu propio proyecto.
