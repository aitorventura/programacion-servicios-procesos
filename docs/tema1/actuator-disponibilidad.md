<a id="actuator-disponibilidad"></a>

# 🧩 4. Comunicación simultánea y disponibilidad del servicio

Dos preguntas distintas de las que has visto hasta ahora, sobre el mismo servicio ya funcionando: no "¿responde bien esta petición concreta?", sino "¿aguanta varias peticiones a la vez sin que unas esperen a otras?" y "¿está el servicio, en general, sano y disponible ahora mismo?".

---

## 🧵 Comunicación simultánea de varios clientes

Un servidor real no atiende a un solo cliente a la vez. Spring Web, sobre Tomcat, atiende cada petición HTTP entrante en un **hilo distinto**, tomado de un *pool* — así que varias peticiones pueden procesarse en paralelo sin que unas esperen a que terminen las otras. Esta idea se retomará a fondo en el Tema 3, sobre programación multihilo; de momento, compruébala con un experimento sencillo.

Imagina un método del service con un `Thread.sleep(2000)` puesto a propósito, que simula una consulta lenta (lo montarás así en la actividad). Lanza dos peticiones **simultáneas** contra su endpoint:

```bash
time (curl -s http://localhost:8080/api/v1/libros/top & \
      curl -s http://localhost:8080/api/v1/libros/top & \
      wait)
```

Si las dos peticiones se atendieran una detrás de otra, el conjunto tardaría unos 4 segundos. Si se atienden en paralelo, tardan aproximadamente 2 — porque cada una la procesa un hilo distinto del pool. Puedes confirmarlo añadiendo temporalmente una traza con `Thread.currentThread().getName()` en el método y mirando el log: verás dos nombres de hilo distintos (`http-nio-8080-exec-1`, `http-nio-8080-exec-2`...) para las dos peticiones.

---

## 🩺 Qué es la disponibilidad de un servicio

Que un servicio esté **disponible** no es una única cosa binaria — hay distintos niveles de "estar bien":

1. **El proceso está arrancado**: la aplicación no se ha caído, sigue viva.
2. **Responde peticiones**: acepta conexiones y contesta algo, lo que sea.
3. **Sus dependencias funcionan**: la base de datos, la cola de mensajería, cualquier servicio del que dependa, están accesibles — un proceso vivo que no puede hablar con su base de datos no está realmente "disponible" para hacer su trabajo.

Un **health check** (comprobación de salud) es una forma automática — pensada para que la ejecute una máquina, sin intervención humana — de responder a estas preguntas. Quien consume esa información no eres tú mirando una pantalla: son **monitores de alertas** (que avisan si algo cae), **orquestadores** (que reinician automáticamente un contenedor que no responde) o el propio **CI** (que puede comprobar que el servicio arranca correctamente antes de darlo por bueno).

---

## 🛠️ Spring Boot Actuator

**Spring Boot Actuator** es la implementación de todo esto para una aplicación Spring Boot: un conjunto de endpoints HTTP listos para usar que exponen información operativa sobre tu aplicación — entre ellos, su salud.

Tu propio proyecto no incluye Actuator todavía (revisa tu `pom.xml`: no está la dependencia) — es la mejora que añades esta semana:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<!-- TODO(autor): comprobar si springdoc lista los endpoints de Actuator en Swagger UI una vez añadida esta dependencia (no debería, al no ser un @RestController normal, pero verificarlo con un proyecto real). Si aparecen y ensucian la documentación pensada para consumidores de la API, marcarlos con @Hidden o excluirlos vía springdoc.paths-to-exclude en application.yml, y explicarlo aquí. -->

Con solo esa dependencia, Spring Boot expone automáticamente `/actuator/health`. Para ver el detalle de cada dependencia (y no solo un `UP`/`DOWN` genérico), hace falta una línea de configuración:

```yaml
management:
  endpoint:
    health:
      show-details: always
```

---

## 🟢 El endpoint `/actuator/health`

Con los detalles activados, `/actuator/health` no solo dice si tu aplicación responde — agrega el estado de **cada dependencia real** que Spring Boot detecta en el classpath. En una aplicación con varias piezas de infraestructura (por ejemplo, tu propio GameVault más adelante, cuando tenga PostgreSQL y MongoDB a la vez), eso se ve así:

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "mongo": { "status": "UP" }
  }
}
```

Fíjate en la implicación: si el contenedor de MongoDB se cae, `/actuator/health` pasa a `DOWN` con el detalle del componente `mongo` marcado como el culpable — **aunque la aplicación siga respondiendo peticiones sobre PostgreSQL sin ningún problema**. Es exactamente el matiz del punto 3 de más arriba: "responder" y "estar realmente sano" no son lo mismo.

!!! tip "Es un GET más, pensado para máquinas"
    `/actuator/health` no deja de ser una petición HTTP estándar — el mismo protocolo de siempre. La diferencia es quién lo consulta: normalmente no una persona con un navegador, sino un orquestador o un monitor, cada pocos segundos, de forma automática. Eso es exactamente para lo que sirve: verificar la disponibilidad del servicio.

Actuator trae también otros endpoints útiles, como `/actuator/info` (metadatos de la aplicación) o `/actuator/metrics` (métricas de rendimiento) — no vas a profundizar en ellos ahora, pero conviene que sepas que existen.

---

## 🧭 Recapitulación del tema

Con esto se completa el recorrido: leíste la API y su protocolo, la documentaste y entendiste su semántica de escritura con OpenAPI, la probaste con tests MockMvc, y hoy compruebas cómo atiende varias peticiones a la vez y verificas su disponibilidad de forma automatizable. En la Actividad 1.4 mides ambas cosas sobre tu propio proyecto y pones en marcha Actuator.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Cada petición HTTP la atiende un hilo distinto del *pool* de Tomcat — por eso dos peticiones lentas simultáneas no tardan el doble, sino aproximadamente lo mismo que una sola.
    - La **disponibilidad** de un servicio tiene varios niveles: proceso vivo, responde peticiones, dependencias funcionando — no son lo mismo.
    - Un **health check** es una comprobación automática, pensada para que la consulte una máquina (monitor, orquestador, CI), no una persona.
    - **Spring Boot Actuator** expone `/actuator/health` con la dependencia `spring-boot-starter-actuator`; `management.endpoint.health.show-details: always` muestra el detalle de cada dependencia.
    - `/actuator/health` agrega el estado de cada dependencia real (PostgreSQL, MongoDB...) — si una cae, el estado general pasa a `DOWN` aunque el resto siga funcionando.
