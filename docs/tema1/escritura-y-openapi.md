<a id="escritura-y-openapi"></a>

# 🧩 2. Escritura en la API y documentación con OpenAPI

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Escritura en la API y documentación con OpenAPI" del
Tema 1 (RA4 - Generación de servicios en red) del módulo Programación de Servicios y
Procesos (0490), semana real 4 del calendario. Sigue las convenciones de estilo del
README.md del repo.

Criterios de evaluación de RA4 que cubre este apartado (curriculum.md):
- b) Ventajas de la utilización de protocolos estándar.
- h) Depuración y documentación de las aplicaciones.

Contexto de coordinación: esta MISMA semana, en Acceso a Datos, el alumnado está
construyendo el CRUD completo de Videojuego (controller + service + DTOs) en su
GameVault. Aquí NO se repite esa construcción: se explica la vertiente HTTP de esos
mismos endpoints de escritura, y se añade la documentación OpenAPI. Deja esa división
clara al principio del apartado.

ESTRUCTURA — teoría primero: antes del proyecto, cubre los conceptos generales con
ejemplos genéricos (la API de biblioteca del apartado anterior sirve): la semántica
completa de cada verbo de escritura y su código de respuesta natural (POST→201,
PUT→200, DELETE→204), qué es la idempotencia con una definición clara y ejemplos (¿qué
pasa si repito esta petición dos veces?), y qué es el CONTRATO de una API — la
descripción formal de qué rutas existen, qué reciben y qué devuelven — y por qué hace
falta cuando el consumidor es otro programa u otra persona; presenta OpenAPI como el
formato estándar de ese contrato y Swagger UI como su visor interactivo, ANTES de tocar
la configuración concreta.

Apóyate en el proyecto GameVault (com.aleroig.gamevault) como ejemplo real:
- com/aleroig/gamevault/catalogo/VideojuegoController.java, métodos POST/PUT/DELETE,
  leídos desde la óptica HTTP: la semántica de cada verbo (POST crea → 201 Created con
  `ResponseEntity.status(HttpStatus.CREATED)`, PUT reemplaza → 200, DELETE elimina → 204
  No Content con `ResponseEntity.noContent()`), `@RequestBody` (el cuerpo JSON de la
  petición mapeado a un DTO), `@Valid` (validación de entrada — se profundizará en el
  Tema 2/RA5), e idempotencia: por qué repetir un PUT o un DELETE no cambia el resultado
  pero repetir un POST sí crea duplicados.
- Ventajas del protocolo estándar (criterio b) con ejemplos concretos: los códigos de
  estado son universales (cualquier cliente sabe qué significa un 201 o un 404 sin leer
  tu documentación), la caché y las herramientas (Postman, Swagger) funcionan sin
  configuración específica.
- Documentación con OpenAPI/Swagger (criterio h):
  com/aleroig/gamevault/config/OpenApiConfig.java y la dependencia springdoc del
  pom.xml — explica qué genera automáticamente (la especificación en /v3/api-docs y la
  interfaz Swagger UI en /swagger-ui.html), cómo Swagger UI sirve a la vez de
  documentación y de cliente de pruebas interactivo, y por qué documentar el contrato de
  la API importa cuando el consumidor es otra aplicación (o un compañero de equipo).

No entres todavía en tests automatizados con MockMvc: eso es el siguiente apartado,
tests-mockmvc.md.
```
