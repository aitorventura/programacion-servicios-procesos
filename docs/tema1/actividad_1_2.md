# 🧪 Actividad 1.2: Swagger/OpenAPI — documenta y prueba tu API

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 1.2 del Tema 1 (RA4 - Generación de servicios en red) del módulo
Programación de Servicios y Procesos (0490), semana real 4 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault (el mismo proyecto compartido con Acceso a Datos). El
enunciado debe guiar paso a paso, mostrando el código y explicando cada decisión; solo
se deja sin guiar, como mini-reto, lo que repita un patrón idéntico ya mostrado en la
misma actividad.

Objetivo (RA4, criterios b, e, h): que el alumnado añada a su GameVault la documentación
OpenAPI/Swagger tal y como existe en la referencia, y la use como cliente para probar
los endpoints de escritura que está construyendo esa misma semana en AD.

Estructura sugerida de pasos guiados:
1. Añadir la dependencia springdoc al pom.xml (fragmento dado, tomado del pom de la
   referencia) y la clase com/aleroig/gamevault/config/OpenApiConfig.java (código
   mostrado y explicado: título, versión y descripción de la API).
2. Arrancar y recorrer guiadamente Swagger UI (/swagger-ui.html): localizar los
   endpoints de videojuegos, ver los esquemas de los DTOs generados automáticamente, y
   entender qué parte de la pantalla viene de dónde (rutas de los @GetMapping/@PostMapping,
   modelos de los DTOs).
3. Prueba guiada desde Swagger UI: un POST de videojuego con el cuerpo JSON dado
   (observar el 201), un PUT (observar el 200) — cada paso con capturas/indicaciones de
   qué botón usar y qué respuesta esperar.
4. Mini-reto (repite el patrón del paso 3): probar desde Swagger UI el DELETE y verificar
   el 204, y después el GET del recurso borrado para verificar el 404 — solo se indica el
   objetivo.
5. Pregunta de comprensión: Swagger UI y curl (Actividad 1.1) han hablado con la misma
   API sin que hayas tenido que adaptar nada en el servidor, ¿qué ventaja del uso de
   protocolos estándar demuestra esto? (criterio b).
```
