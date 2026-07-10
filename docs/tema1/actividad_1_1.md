# 🧪 Actividad 1.1: Explorando la API de GameVault con un cliente HTTP

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 1.1 del Tema 1 (RA4 - Generación de servicios en red) del módulo
Programación de Servicios y Procesos (0490), semana real 3 del calendario. Si la
actividad necesita plantilla/solución en .docx, no existe todavía skill específica de
PSP: créala primero clonando /actividad-plantilla-acceso-a-datos con la paleta
marrón/ámbar de este módulo (igual que pptx-psp es el clon marrón/ámbar de
pptx-acceso-a-datos).

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault (el mismo proyecto compartido con Acceso a Datos, construido
individualmente durante el curso). El enunciado debe guiar paso a paso; solo se deja sin
guiar, como mini-reto, lo que repita un patrón idéntico ya mostrado en la misma
actividad. En esta actividad NO se escribe código: es exploración guiada de la API en
marcha.

Objetivo (RA4, criterios a, c, e): que el alumnado arranque su GameVault y explore sus
endpoints GET con un cliente HTTP (curl y/o Postman), aprendiendo a leer peticiones y
respuestas HTTP reales.

Estructura sugerida de pasos guiados:
1. Arrancar GameVault (Docker Compose + aplicación, ya saben hacerlo del Tema 0 de AD) y
   comprobar que escucha en el puerto 8080.
2. Peticiones GET guiadas con los comandos exactos dados: `GET /api/v1/videojuegos`
   (lista), `GET /api/v1/videojuegos/1` (detalle) — observar con `curl -v` (o la pestaña
   de Postman) el verbo, la ruta, el código de estado 200, la cabecera Content-Type y el
   cuerpo JSON, con cada parte señalada y explicada en el enunciado.
3. Petición guiada a un recurso inexistente (`GET /api/v1/videojuegos/9999`) y análisis
   de la respuesta 404 — conectar con VideojuegoService.findById() y su
   ResponseStatusException, que el alumnado puede abrir y leer.
4. Mini-reto (repite el patrón de los pasos 2-3): explorar por su cuenta
   `GET /api/v1/estudios` y un GET de detalle de estudio, anotando verbo, código y
   cuerpo — mismo procedimiento, distinto recurso.
5. Pregunta de comprensión: ¿por qué un mismo cliente (curl) puede hablar con GameVault
   y con cualquier otra API REST del mundo sin instalar nada específico? Conectar con la
   idea de protocolo estándar del apartado de teoría.
```
