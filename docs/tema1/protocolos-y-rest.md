<a id="protocolos-y-rest"></a>

# 🧩 1. Protocolos estándar y servicios REST: leyendo el controlador

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Protocolos estándar y servicios REST: leyendo el
controlador" del Tema 1 (RA4 - Generación de servicios en red) del módulo Programación
de Servicios y Procesos (0490), semana real 3 del calendario. Sigue las convenciones de
estilo del README.md del repo (tabs-colored, admonitions, densidad visual, pretérito
perfecto compuesto). Este módulo usa la paleta marrón/ámbar.

Criterios de evaluación de RA4 que cubre este apartado (curriculum.md):
- a) Protocolos estándar de comunicación para la implementación de servicios en red.
- c) Librerías que permiten implementar servicios en red con protocolos estándar.

ESTRUCTURA OBLIGATORIA — teoría primero, proyecto después. El alumnado NO sabe qué es
HTTP, ni una API, ni REST: llega sabiendo Java, Git/Docker y bases de datos. Explica
todo desde cero. (Qué es Spring Boot y cómo se estructura GameVault se explica esa
misma semana en Acceso a Datos, Tema 1, apartado "Spring Boot: el chasis de GameVault" —
remite a ese apartado y no lo dupliques aquí.)

PARTE 1 — Teoría general, desde cero y sin GameVault todavía:
- Cómo funciona la web en una petición: un CLIENTE (navegador, aplicación) pide y un
  SERVIDOR responde; anatomía de una URL (protocolo, host, puerto, ruta) pieza a pieza.
- Qué es HTTP: un protocolo de texto de petición-respuesta. Muestra una petición y una
  respuesta HTTP REALES y crudas (texto completo con su primera línea, cabeceras y
  cuerpo) y disecciónalas etiquetando cada parte: método/verbo, ruta, versión, código de
  estado, Content-Type, cuerpo.
- Los verbos (GET/POST/PUT/DELETE: qué expresa cada uno) y los códigos de estado por
  familias (2xx bien, 3xx redirección, 4xx culpa del cliente, 5xx culpa del servidor)
  con los 5-6 códigos que se usarán todo el curso (200, 201, 204, 400, 401/403, 404).
- Qué es JSON como formato del cuerpo (repaso breve con un ejemplo — lo han tocado, no
  lo dominan).
- Qué es una API y qué es REST: exponer los DATOS de tu aplicación como RECURSOS con
  direcciones (URLs) sobre los que se actúa con los verbos — el ejemplo genérico de una
  API de biblioteca (GET /libros, GET /libros/3, POST /libros...) antes de ver la de
  videojuegos.

PARTE 2 — Aterrizaje en GameVault. Esta semana NO se escribe código nuevo: se aprende a
LEER el controlador REST que ya existe, entendiendo qué papel juega cada pieza en la
comunicación en red. Contexto de coordinación: el alumnado trabaja sobre su propia copia
de GameVault, el MISMO proyecto compartido con Acceso a Datos (0486) — esta misma
semana, en AD, están creando las entidades Videojuego/Estudio; aquí se mira el mismo
proyecto desde el ángulo de la comunicación en red. Apóyate en:
- com/aleroig/gamevault/catalogo/VideojuegoController.java, solo los métodos GET:
  explica anotación a anotación qué significa cada cosa desde la óptica HTTP —
  `@RestController` (respuestas serializadas al cuerpo HTTP), `@RequestMapping("/api/v1/
  videojuegos")` (la ruta como identificador del recurso, el versionado /v1),
  `@GetMapping` y `@GetMapping("/{id}")` (verbo + ruta = operación), `@PathVariable`,
  `ResponseEntity.ok(...)` (código de estado 200 explícito).
- El flujo completo de una petición: cliente → puerto 8080 → Spring Web (Tomcat
  embebido) → controller → service → respuesta JSON — con un diagrama mermaid sencillo.
- Contrasta HTTP/REST con alternativas no estándar (un protocolo binario propio sobre
  sockets, que se verá en el Tema 4/RA3) para justificar el criterio a): por qué usar un
  protocolo estándar hace que cualquier cliente (navegador, curl, Postman, otra
  aplicación) pueda hablar con el servicio sin acordar nada más.
- Menciona que Spring Web viene de la dependencia spring-boot-starter-web del pom.xml, y
  que el servidor embebido (Tomcat) es quien escucha en el puerto — la "librería que
  implementa el servicio en red" del criterio c).

No entres todavía en los endpoints de escritura (POST/PUT/DELETE) ni en Swagger: eso es
el siguiente apartado, escritura-y-openapi.md.
```
