# 🧪 Actividad 1.1: Diseccionando peticiones y respuestas HTTP

!!! info "Práctica guiada — sin código, y sin servidor propio todavía"
    Esta semana, en Acceso a Datos, tu propio GameVault solo tiene entidades JPA — todavía no existe ningún endpoint que puedas invocar. Así que hoy no vas a lanzar peticiones contra tu proyecto: vas a diseccionar, con papel y con `curl` contra un objetivo controlado, conversaciones HTTP reales, para que la semana que viene, en cuanto tu propio CRUD esté vivo, ya sepas leer lo que ves.

## Qué vas a practicar

- Diseccionar una petición y una respuesta HTTP reales línea a línea: verbo, ruta, cabeceras, código de estado, cuerpo.
- Relacionar cada trozo de la conversación HTTP con la anotación Java que lo genera (según lo visto en la teoría).
- Practicar la sintaxis de `curl` contra un servidor local mínimo, sin depender de tu propio proyecto.

---

## Requisitos previos

Los apuntes del apartado de teoría de esta semana (el código de `VideojuegoController` que has leído ahí). Si tienes `jq` instalado, lo usarás en el mini-reto (opcional).

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
< Content-Length: 58
<
{"id":3,"titulo":"Hollow Knight","precio":14.99,"fechaLanzamiento":"2017-02-24"}
```

Recuerda: `>` es lo que **envía** el cliente, `<` es lo que **responde** el servidor.

**Responde**, señalando exactamente qué línea o fragmento usas para justificar cada respuesta:

1. ¿Qué **verbo** y qué **ruta** se han pedido?
2. ¿Qué **código de estado** ha devuelto el servidor? ¿A qué familia pertenece (2xx/4xx/5xx)?
3. ¿Qué dice la cabecera `Content-Type` de la respuesta sobre el formato del cuerpo?
4. En el `VideojuegoController` que leíste en la teoría, ¿qué método exacto (con qué anotación) es responsable de generar esta respuesta concreta? Justifica tu respuesta relacionando la ruta pedida con la anotación del método.

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
2. En el método `findById` de `VideojuegoService` que leíste en la teoría (`orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, ...))`), ¿qué parte exacta del código es responsable de que este `404` tenga ese código y no otro (por ejemplo, `400` o `500`)?
3. ¿Este `404` lo genera Tomcat automáticamente, o nace de una decisión explícita del código de la aplicación? Justifica tu respuesta con la línea de código concreta.

---

## Paso 3 — Practicar `curl` de verdad, contra un servidor mínimo

No necesitas tu proyecto para practicar la herramienta en sí. Python trae un servidor HTTP mínimo integrado — suficiente para tener algo real contra lo que lanzar peticiones:

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

Prueba también a pedir un fichero que no existe:

```bash
curl -v http://localhost:8000/no-existe.json
```

**Comprueba**: el código de estado. ¿Coincide con el que esperarías después del Paso 2?

Detén el servidor con `Ctrl+C` cuando termines.

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

`curl` no sabe nada de Java, ni de Spring, ni de cómo está construido un proyecto por dentro — y aun así has podido usarlo contra un servidor de Python en el Paso 3 con la misma sintaxis que usarías contra cualquier API REST. ¿Por qué? Relaciona tu respuesta con la idea de **protocolo estándar** vista en la teoría: ¿qué tendría que pasar para que `curl` (o Postman, o cualquier otro cliente) NO pudiera hablar con un servidor?

---

## ✅ Cierre

Ya sabes leer una conversación HTTP real —verbo, ruta, código de estado, cabeceras, cuerpo— y relacionarla con el código Java que la genera. En cuanto tu propio GameVault tenga su primer endpoint (Acceso a Datos, esta misma semana que viene), vas a lanzarle estas mismas preguntas a peticiones de verdad, contra tu propio proyecto.
