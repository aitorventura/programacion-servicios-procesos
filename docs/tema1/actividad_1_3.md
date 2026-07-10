# 🧪 Actividad 1.3: Tests MockMvc y el PUT de Estudio

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 1.3 del Tema 1 (RA4 - Generación de servicios en red) del módulo
Programación de Servicios y Procesos (0490), semana real 5 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault (el mismo proyecto compartido con Acceso a Datos). El
enunciado debe guiar paso a paso, mostrando el código y explicando cada decisión; solo
se deja sin guiar, como mini-reto, lo que repita un patrón idéntico ya mostrado (en esta
actividad, el patrón de referencia es el PUT de Videojuego que ya construyeron en AD).

Objetivo (RA4, criterios d, e, f): dos bloques — (1) escribir tests MockMvc guiados
sobre los endpoints de Videojuego, y (2) añadir el PUT de Estudio, una MEJORA que no
existe en la referencia adjunta (EstudioController.java solo tiene GET y POST).

Estructura sugerida de pasos guiados:
1. Primer test MockMvc guiado al completo, código mostrado y explicado línea a línea,
   siguiendo el patrón de
   src/test/java/com/aleroig/gamevault/catalogo/VideojuegoControllerTest.java: el GET de
   lista de videojuegos (petición, status().isOk(), jsonPath sobre el cuerpo).
2. Segundo test guiado: el GET de un videojuego inexistente esperando 404 (nuevo matiz:
   probar el caso de error).
3. Mini-reto de tests (repite el patrón de los pasos 1-2): un test del POST de
   videojuego esperando 201 — solo se indica qué verificar.
4. El PUT de Estudio, guiado por comparación: el enunciado muestra en paralelo el PUT de
   VideojuegoController/VideojuegoService (que ya construyeron en AD la semana 4) y va
   indicando qué adaptar para Estudio (DTO, comprobación de "no encontrado" con
   ResponseStatusException, @Transactional en el service) — el código base se da como
   referencia visible; la adaptación la escriben ellos con el enunciado señalando cada
   cambio.
5. Verificación guiada del PUT nuevo: primero manual desde Swagger UI, después con un
   test MockMvc que repite el patrón ya practicado (mini-reto).
6. Experimento guiado de concurrencia (criterio f): lanzar dos peticiones simultáneas al
   endpoint lento /api/v1/videojuegos/top (getTopNovedades tiene un sleep de 2s) y
   comprobar en los logs que las atienden hilos distintos del pool de Tomcat — pasos y
   comandos dados; anotar el nombre de los hilos observados.
```
