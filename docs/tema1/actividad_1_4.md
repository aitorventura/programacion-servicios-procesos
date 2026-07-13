# 🧪 Actividad 1.4: El DELETE de Estudio y Actuator `/health`

!!! info "Reto de repetición + práctica guiada"
    El endpoint `DELETE` de Estudio lo escribes tú solo — es la tercera vez que conectas este patrón (`DELETE` de Videojuego en AD, `PUT` de Estudio en la Actividad 1.3), así que no llevas la mano guiada esta vez. Actuator sí va paso a paso, es contenido nuevo.

## Qué vas a practicar

- Completar un CRUD reutilizando un patrón ya practicado dos veces, sin guía paso a paso.
- Añadir Actuator y activar el detalle de salud de las dependencias.
- Observar en vivo cómo cae la disponibilidad de un componente concreto.

---

## Requisitos previos

Tu `PUT` de `Estudio` de la Actividad 1.3 funcionando, y el `DELETE` de `Videojuego` de Acceso a Datos como segundo patrón ya construido para guiarte.

---

## Reto — el DELETE de Estudio

Igual que con el `PUT` de la semana pasada, no partes de cero: `EstudioService.delete(Long id)` ya existe (Acceso a Datos, Actividad 1.2), completo y funcionando — solo que, hasta hoy, ningún endpoint lo invoca. Sin más guía que esta especificación, añade el endpoint que falta:

- Ruta: `DELETE /api/v1/estudios/{id}`
- `204 No Content` si borra correctamente.
- `404 Not Found` si el `id` no existe.

Tienes **dos** ejemplos ya construidos del mismo patrón (controller que delega en un service que ya gestiona el "no encontrado"): el `DELETE` de `VideojuegoController` (Acceso a Datos) y el `PUT` de `EstudioController` que tú mismo escribiste en la Actividad 1.3, conectando un método de `EstudioService` que ya existía. Escribe el `@DeleteMapping` de `Estudio` siguiendo ese mismo patrón, sin que se te dé el código.

**Pregunta de comprensión**: al borrar un `Estudio` que tiene videojuegos asociados, ¿qué pasa con esos videojuegos? Relaciona tu respuesta con `cascade = CascadeType.ALL` y `orphanRemoval = true`, que viste al crear la entidad `Estudio` en Acceso a Datos (Actividad 1.1).

---

## Verificación del DELETE — también como reto

Comprueba tu `DELETE` de dos formas, sin que se te dé el código de ninguna de las dos:

1. **Manual**, desde Swagger UI (Actividad 1.2): crea un estudio de prueba, bórralo, comprueba el `204`, y comprueba después que un `GET` sobre ese mismo `id` da `404`.
2. **Automatizada**, con un test MockMvc que siga el mismo patrón que ya usaste en la Actividad 1.3 (mockear el service, `mockMvc.perform(delete(...))`, afirmar el código de estado) — cubre los dos casos: `204` cuando existe, `404` cuando no.

---

## Actuator, guiado paso a paso

### Paso 1 — Añadir la dependencia

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Paso 2 — Exponer el detalle de salud

En `application.yaml` (la configuración común, no la de un perfil concreto):

```yaml
management:
  endpoint:
    health:
      show-details: always
```

Reinicia tu aplicación y consulta:

```bash
curl -s http://localhost:8080/actuator/health | jq
```

**Comprueba**: que la respuesta incluye un bloque `components` con, al menos, tu base de datos PostgreSQL (`db`) en estado `UP`.

---

## Experimento guiado de disponibilidad

Este experimento necesita que tengas más de un servicio en tu `.devcontainer/docker-compose.yml` — si en este punto del curso solo tienes PostgreSQL, hazlo con ese mismo servicio (parándolo tendrás el mismo efecto sobre el componente `db`).

!!! tip "Dónde ejecutar estos comandos"
    Puedes lanzarlos desde la propia terminal integrada de VS Code, dentro del Dev Container: el `docker-outside-of-docker` que configuraste en la Actividad 1.1 de Acceso a Datos hace que `docker compose` vea y controle los mismos contenedores que tu sistema operativo, aunque la terminal esté dentro de `app`. Como el fichero ya no está en la raíz del proyecto, apunta a él con `-f`; y como `docker compose` no adivina solo qué contenedores son "los tuyos", indícale también el proyecto con `-p gamevault_devcontainer` (el mismo nombre que ya usaste en la Actividad 1.1 de AD).

### Paso 3 — Parar una dependencia

```bash
docker compose -f .devcontainer/docker-compose.yml -p gamevault_devcontainer stop postgres
```

Vuelve a consultar:

```bash
curl -s http://localhost:8080/actuator/health | jq
```

**Predicción**: antes de ejecutar el comando, escribe qué esperas ver en el campo `status` general y en el componente correspondiente a PostgreSQL.

### Paso 4 — Recuperar el servicio

```bash
docker compose -f .devcontainer/docker-compose.yml -p gamevault_devcontainer start postgres
```

Espera unos segundos y vuelve a consultar `/actuator/health`.

**Comprueba**: que el estado ha vuelto a `UP` sin que hayas tenido que reiniciar tu aplicación Spring Boot — solo el contenedor de la base de datos.

**Pregunta**: si tu aplicación estuviera respondiendo peticiones GET normales mientras PostgreSQL estaba caído (por ejemplo, un endpoint que no toca la base de datos), ¿el servicio estaría "disponible" o no? Justifica tu respuesta con los tres niveles de disponibilidad vistos en la teoría.

---

## Repaso del tema

Escribe un repaso propio (3-4 frases) del recorrido completo de este tema: leer la API y su protocolo → documentarla y entender su semántica de escritura → probarla con tests y varios clientes simultáneos → completarla (`PUT`/`DELETE` de Estudio) → verificar su disponibilidad. ¿Qué pieza te ha costado más entender, y por qué?

---

## ✅ Cierre

La semana que viene entras en Programación Segura: hoy tu API está completamente abierta — nada te impide borrar cualquier recurso sin identificarte. Eso empieza a cambiar en el Tema 2.
