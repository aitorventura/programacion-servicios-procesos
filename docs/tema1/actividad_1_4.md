# 🧪 Actividad 1.4: Comunicación simultánea y disponibilidad del servicio

!!! warning "Descarga la plantilla"
    📄 [Plantilla 1.4 — Comunicación simultánea y disponibilidad del servicio](plantillas/Actividad_1_4_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Dos bloques, los dos guiados"
    Cierras el tema con las dos últimas piezas de la teoría: comprobar que tu aplicación atiende varias peticiones a la vez sin que se pisen, y verificar de forma automatizable que sigue "sana" cuando una de sus dependencias falla.

## Qué vas a practicar

- Construir un endpoint deliberadamente lento para poder medir concurrencia.
- Comprobar experimentalmente que Spring atiende cada petición en un hilo distinto.
- Añadir Actuator y activar el detalle de salud de las dependencias.
- Observar en vivo cómo cae la disponibilidad de un componente concreto.

---

## Requisitos previos

Tu CRUD completo de `Videojuego` (con las cuatro operaciones) y los tests MockMvc de la Actividad 1.3.

---

## Bloque 1 — Comunicación simultánea

### Paso 1 — Un método lento, para tener algo que medir

Antes del experimento necesitas un endpoint que tarde de verdad — la teoría lo daba por hecho, así que lo construyes ahora. Añade a `VideojuegoRepository`:

```java
List<Videojuego> findTop5ByOrderByFechaLanzamientoDesc();
```

Y a `VideojuegoService`:

```java
@Transactional(readOnly = true)
public List<VideojuegoResponseDTO> getTopNovedades() {
    try {
        Thread.sleep(2000); // simula una consulta costosa — lo quitas al final de la actividad
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return videojuegoRepository.findTop5ByOrderByFechaLanzamientoDesc()
            .stream()
            .map(this::mapToDTO)
            .toList();
}
```

Y en `VideojuegoController`, documentado igual que el resto de endpoints del proyecto:

```java
@Operation(summary = "Listar los 5 videojuegos más recientes")
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Listado obtenido correctamente")
})
@GetMapping("/top")
public ResponseEntity<List<VideojuegoResponseDTO>> getTopNovedades() {
    return ResponseEntity.ok(videojuegoService.getTopNovedades());
}
```

!!! tip "¿Y si `/top` choca con `/{id}`?"
    Podría parecer un problema: `GET /api/v1/videojuegos/top` encaja tanto con `@GetMapping("/top")` como con `@GetMapping("/{id}")` (que aceptaría, literalmente, cualquier texto en esa posición, incluida la palabra `top`). Pero Spring no decide por el orden en que declares los métodos en la clase: prioriza automáticamente la ruta más específica —una ruta literal, sin variables, como `/top`— sobre la que solo tiene un `{parámetro}` genérico. Declara los métodos en el orden que te resulte más cómodo de leer.

Arranca tu aplicación y comprueba con una sola llamada que `GET /api/v1/videojuegos/top` tarda efectivamente unos 2 segundos.

### Paso 2 — Medir la concurrencia

Repite el experimento de la teoría con tu propio proyecto:

```bash
time (curl -s http://localhost:8080/api/v1/videojuegos/top & \
      curl -s http://localhost:8080/api/v1/videojuegos/top & \
      wait)
```

**Anota** el tiempo total mostrado por `time`. Añade temporalmente una línea `System.out.println(Thread.currentThread().getName())` al principio de `getTopNovedades()` en tu service, repite la prueba, y anota los dos nombres de hilo que aparecen en la consola. Cuando termines, retira esa línea — era solo para observar, no para quedarse en el código.

**Captura**: la salida de la terminal con el resultado de `time`, y el fragmento del log de tu aplicación donde se ven los dos nombres de hilo distintos.

**Pregunta**: ¿qué relación hay entre este "una petición, un hilo" que acabas de observar y el resultado de `time`? Si en vez de dos lanzaras cincuenta peticiones simultáneas, ¿crees que el comportamiento sería exactamente el mismo, o hay algún límite? (No hace falta que sepas la respuesta exacta todavía — se trabaja a fondo en el Tema 3).

---

## Bloque 2 — Actuator, guiado paso a paso

### Paso 3 — Añadir la dependencia

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Paso 4 — Exponer el detalle de salud

En `application.yml` (la configuración común, la que se carga siempre — no la confundas con `application-dev.yaml`, que solo se activa con el perfil `dev`):

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

**Captura**: la respuesta completa de `curl -s http://localhost:8080/actuator/health | jq`, con el bloque `components` visible.

---

## Experimento guiado de disponibilidad

Este experimento necesita que tengas más de un servicio en tu `.devcontainer/docker-compose.yml` — si en este punto del curso solo tienes PostgreSQL, hazlo con ese mismo servicio (parándolo tendrás el mismo efecto sobre el componente `db`).

!!! tip "Dónde ejecutar estos comandos"
    Puedes lanzarlos desde la propia terminal integrada de tu editor (VS Code o IntelliJ IDEA), dentro del Dev Container: el `docker-outside-of-docker` que configuraste en la Actividad 1.1 de Acceso a Datos hace que `docker compose` vea y controle los mismos contenedores que tu sistema operativo, aunque la terminal esté dentro de `app`. Como el fichero ya no está en la raíz del proyecto, apunta a él con `-f`; y como `docker compose` no adivina solo qué contenedores son "los tuyos", indícale también el proyecto con `-p`.

!!! warning "No des por hecho el nombre del proyecto"
    `gamevault_devcontainer` es el nombre que usa VS Code, pero IntelliJ no siempre lo nombra igual (puede llamarlo simplemente `devcontainer`, entre otras variantes). Antes de ejecutar nada, comprueba el nombre real:
    ```bash
    docker compose ls
    ```
    Si usas el nombre equivocado, `docker compose ... stop postgres` no encontrará el contenedor y no parará nada — sin avisarte de que ha fallado. Es exactamente lo que comprueba la actividad de disponibilidad: si `/actuator/health` sigue en `UP` después de "parar" PostgreSQL, sospecha primero de esto antes que de Actuator.

### Paso 5 — Parar una dependencia

```bash
docker compose -f .devcontainer/docker-compose.yml -p <proyecto> stop postgres
```

Vuelve a consultar:

```bash
curl -s http://localhost:8080/actuator/health | jq
```

**Predicción**: antes de ejecutar el comando, escribe qué esperas ver en el campo `status` general y en el componente correspondiente a PostgreSQL.

**Captura**: la respuesta real de `/actuator/health` con PostgreSQL parado, con el `status` general y el componente `db` en `DOWN` — compárala con tu predicción.

### Paso 6 — Recuperar el servicio

```bash
docker compose -f .devcontainer/docker-compose.yml -p <proyecto> start postgres
```

Espera unos segundos y vuelve a consultar `/actuator/health`.

**Comprueba**: que el estado ha vuelto a `UP` sin que hayas tenido que reiniciar tu aplicación Spring Boot — solo el contenedor de la base de datos.

**Captura**: la respuesta de `/actuator/health` ya recuperada, con `status: UP` de nuevo.

**Pregunta**: si tu aplicación estuviera respondiendo peticiones GET normales mientras PostgreSQL estaba caído (por ejemplo, un endpoint que no toca la base de datos), ¿el servicio estaría "disponible" o no? Justifica tu respuesta con los tres niveles de disponibilidad vistos en la teoría.

---

## Paso 7 — Deja el endpoint listo de verdad

El `Thread.sleep(2000)` era solo para poder medir concurrencia — `getTopNovedades()` se queda en tu proyecto de forma permanente (mostrar las novedades del catálogo es una funcionalidad real), pero no tiene sentido que responda con un retraso artificial para siempre. Quítalo de `VideojuegoService.getTopNovedades()` y comprueba que el endpoint sigue funcionando, ahora sin demora.

---

## ✅ Cierre

En el Tema 2 entras en Programación Segura: hoy tu API está completamente abierta — nada te impide borrar cualquier recurso sin identificarte. Eso empieza a cambiar ahí.
