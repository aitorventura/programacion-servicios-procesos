# 🧪 Actividad 2.1: Validación y `GlobalExceptionHandler`

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 2.1 del Tema 2 (RA5 - Programación segura) del módulo Programación
de Servicios y Procesos (0490), semana real 7 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault (el mismo proyecto compartido con Acceso a Datos). El
enunciado debe guiar paso a paso, mostrando el código y explicando cada decisión; solo
se deja sin guiar, como mini-reto, lo que repita un patrón idéntico ya mostrado en la
misma actividad.

Objetivo (RA5, criterio a): que el alumnado construya en su GameVault la validación de
entrada y la gestión centralizada de errores tal y como existen en la referencia.

Estructura sugerida de pasos guiados:
1. Añadir las anotaciones de validación a VideojuegoCreateDTO, guiado: cada anotación
   (las que use la referencia — revisar VideojuegoCreateDTO.java y replicar) mostrada y
   explicada, junto con el `@Valid` en el controller si aún no lo tienen.
2. Provocar guiadamente un error de validación (petición POST con cuerpo inválido dada
   en el enunciado) y observar la respuesta de Spring por defecto — analizar juntos qué
   información devuelve y por qué es mejorable.
3. Construir GlobalExceptionHandler y ErrorResponse, guiado con el código de la
   referencia (com/aleroig/gamevault/exception/) mostrado y explicado parte a parte:
   @RestControllerAdvice, @ExceptionHandler para los errores de validación y para
   ResponseStatusException, y el formato uniforme de ErrorResponse.
4. Repetir la petición inválida del paso 2 y comparar la nueva respuesta con la
   anterior: qué información ya no se filtra y qué gana el cliente con un formato
   coherente.
5. Mini-reto (repite el patrón del paso 1): añadir validación al DTO de reseñas
   (ReviewRequestDTO — por ejemplo, puntuación entre 1 y 5) y provocar el error para
   comprobar que el handler central también lo captura — solo se indica el objetivo.
6. Pregunta de comprensión: ¿por qué conviene que TODAS las excepciones pasen por un
   único punto antes de convertirse en respuesta HTTP? Nombra un dato interno concreto
   que el handler evita filtrar a un atacante.
```
