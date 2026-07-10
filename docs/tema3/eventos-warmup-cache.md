<a id="eventos-warmup-cache"></a>

# 🧩 2. Eventos internos de Spring: el warm-up de caché (1/2)

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Eventos internos de Spring: el warm-up de caché (1/2)"
del Tema 3 (RA2 - Programación multihilo) del módulo Programación de Servicios y
Procesos (0490), semana real 14 del calendario (primera semana tras exámenes y
Navidad — arranca recordando en 2-3 frases dónde se quedó el tema). Sigue las
convenciones de estilo del README.md del repo.

Criterios de evaluación de RA2 que cubre este apartado (curriculum.md):
- c) Aplicaciones que implementan varios hilos (primera mitad de la construcción).
- e) Mecanismos para compartir información entre hilos.

ESTRUCTURA — teoría primero: antes de Spring, explica desde cero el concepto de evento
y el patrón observador/publicador-suscriptor en general: qué es un evento ("algo ha
pasado", empaquetado como objeto), quién publica y quién escucha, y qué gana el diseño —
el que publica no conoce a los que reaccionan (desacoplamiento), y pueden añadirse
reacciones nuevas sin tocar al emisor. Ilustra con un ejemplo cotidiano (una
notificación a la que se suscriben varias apps) y conecta con lo que el alumnado ya usa
sin saberlo: los eventos de interfaz (un clic de botón) y el RabbitMQ analizado la
semana 12 son el mismo patrón a distintas escalas.

Contenido central: los eventos internos de Spring (ApplicationEventPublisher +
listeners) como la implementación de ese patrón DENTRO de la aplicación, y como vehículo
para pasar información entre hilos — la pieza 1 de 2 del warm-up de caché. Esta semana
el evento y su publicación; la que viene (apartado 3), el listener @Async. Explica
también aquí, desde cero, qué es una caché y qué es "invalidar" y "recalentar" una caché
(el alumnado ha usado @Cacheable en getTopNovedades sin que nadie le haya explicado el
concepto): guardar un resultado caro para reutilizarlo, borrarlo cuando deja de ser
válido, y volver a calcularlo antes de que alguien lo pida.

IMPRESCINDIBLE explicar aquí también, y no antes ni después: qué es Redis y por qué
aparece en este proyecto. El alumnado ya ha visto el contenedor `redis` en el
`docker-compose.yaml` (Tema 0 de AD) y en el `/actuator/health` (Tema 1 de PSP), pero
nadie le ha dicho qué es — y es precisamente aquí, con la caché, donde encaja. Explica
en 3-4 frases: Redis es una base de datos en memoria clave-valor, muy rápida, que se usa
típicamente como caché compartida (a diferencia de una caché solo en la memoria de la
propia aplicación, una caché en Redis sobrevive a un reinicio de la aplicación y puede
compartirse si algún día hay varias instancias de GameVault corriendo a la vez). Con las
dependencias `spring-boot-starter-cache` y `spring-boot-starter-data-redis` presentes en
el pom.xml, Spring Boot autoconfigura Redis como el almacén real detrás de
`@CacheManager`/`@Cacheable` — es decir, cuando `getTopNovedades()` guarda su resultado
en la caché "topNovedades", ese resultado se guarda físicamente en el contenedor Redis
del docker-compose, no en un mapa en memoria de la propia aplicación. Verifícalo con el
alumnado conectando a Redis con `redis-cli` (comando dado) y viendo la clave
`topNovedades` aparecer tras la primera petición.

IMPORTANTE — esto es una MEJORA que NO existe en la referencia adjunta: no hay ningún
evento interno de Spring en GameVault (los VideojuegoEvent de
catalogo/api/eventos/ son eventos de RabbitMQ entre módulos, no eventos internos de
Spring). Distingue explícitamente ambos desde el principio, porque el alumnado los va a
confundir: RabbitMQ = mensajería entre procesos/módulos a través de un broker externo
(lo analizado en la semana 12); ApplicationEventPublisher = eventos DENTRO de la misma
JVM, entre beans, sin broker.

Explica con código de ejemplo (que luego la Actividad 3.2 construye guiado):
- El diseño del warm-up: cuando VideojuegoService hace @CacheEvict de "topNovedades"
  (create/update/delete), publicar un evento interno TopNovedadesInvalidadoEvent; un
  listener (la semana que viene) lo recibirá y llamará a getTopNovedades() en un hilo
  aparte para recalentar la caché antes de que llegue el siguiente usuario.
- La clase de evento como record simple (qué información transporta y por qué conviene
  que sea inmutable — criterio e: la inmutabilidad como forma segura de compartir
  información entre hilos, sin locks).
- La publicación: inyectar ApplicationEventPublisher en VideojuegoService (igual que se
  inyecta cualquier otra dependencia, con @RequiredArgsConstructor como en todo el
  proyecto) y publicar tras cada operación de escritura.
- Aclara que, sin listener todavía, publicar el evento no hace nada visible — el sistema
  queda "emitiendo" a la espera de la pieza 2.

Cierra con el mapa de las dos piezas (semana 14: evento + publicación; semana 15:
listener @Async + sincronización con la transacción) para que se vea el plan completo.
```
