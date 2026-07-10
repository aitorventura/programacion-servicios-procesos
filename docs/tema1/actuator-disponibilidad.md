<a id="actuator-disponibilidad"></a>

# 🧩 4. Disponibilidad del servicio: Actuator

Un servicio en red, una vez desplegado, corre sin que nadie lo esté mirando en pantalla constantemente. Este apartado responde a una pregunta distinta de todo lo visto hasta ahora: no "¿funciona esta petición concreta?", sino "¿está el servicio, en general, sano y disponible ahora mismo?".

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

Tu propio GameVault no incluye Actuator todavía (revisa tu `pom.xml`: no está la dependencia) — es la mejora que añades esta semana:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Con solo esa dependencia, Spring Boot expone automáticamente `/actuator/health`. Para ver el detalle de cada dependencia (y no solo un `UP`/`DOWN` genérico), hace falta una línea de configuración:

```yaml
management:
  endpoint:
    health:
      show-details: always
```

---

## 🟢 El endpoint `/actuator/health`

Con los detalles activados, `/actuator/health` no solo dice si tu aplicación responde — agrega el estado de **cada dependencia real** que Spring Boot detecta en el classpath. En GameVault, eso significa PostgreSQL, MongoDB, RabbitMQ y Redis (los cuatro servicios de `docker-compose.yaml` — de Redis basta con que sepas, por ahora, que es "una base de datos en memoria más" cuya salud también se agrega aquí; verás para qué la usa realmente el proyecto en el Tema 3, con la caché de novedades):

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "mongo": { "status": "UP" },
    "rabbit": { "status": "UP" },
    "redis": { "status": "UP" }
  }
}
```

Fíjate en la implicación: si el contenedor de MongoDB se cae, `/actuator/health` pasa a `DOWN` con el detalle del componente `mongo` marcado como el culpable — **aunque la aplicación siga respondiendo peticiones sobre PostgreSQL sin ningún problema**. Es exactamente el matiz del punto 3 de más arriba: "responder" y "estar realmente sano" no son lo mismo.

!!! tip "Es un GET más, pensado para máquinas"
    `/actuator/health` no deja de ser una petición HTTP estándar — el mismo protocolo de siempre. La diferencia es quién lo consulta: normalmente no una persona con un navegador, sino un orquestador o un monitor, cada pocos segundos, de forma automática. Eso es exactamente para lo que sirve: verificar la disponibilidad del servicio.

Actuator trae también otros endpoints útiles, como `/actuator/info` (metadatos de la aplicación) o `/actuator/metrics` (métricas de rendimiento) — no vas a profundizar en ellos ahora, pero conviene que sepas que existen.

---

## 🧭 Recapitulación del tema

Con esto se completa el recorrido: leíste la API y su protocolo (apartado 1), la documentaste y entendiste su semántica de escritura con OpenAPI (apartado 2), la probaste con MockMvc y con varios clientes simultáneos (apartado 3), y hoy verificas su disponibilidad de forma automatizable. En la Actividad 1.4 completas el CRUD de `Estudio` con el `DELETE` que falta y pones en marcha Actuator sobre tu propio proyecto.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - La **disponibilidad** de un servicio tiene varios niveles: proceso vivo, responde peticiones, dependencias funcionando — no son lo mismo.
    - Un **health check** es una comprobación automática, pensada para que la consulte una máquina (monitor, orquestador, CI), no una persona.
    - **Spring Boot Actuator** expone `/actuator/health` con la dependencia `spring-boot-starter-actuator`; `management.endpoint.health.show-details: always` muestra el detalle de cada dependencia.
    - GameVault agrega en `/actuator/health` el estado de PostgreSQL, MongoDB, RabbitMQ y Redis — si uno cae, el estado general pasa a `DOWN` aunque el resto siga funcionando.
