# 🧪 Actividad 3.2: El evento del warm-up de caché (1/2)

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 3.2 del Tema 3 (RA2 - Programación multihilo) del módulo
Programación de Servicios y Procesos (0490), semana real 14 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado. Esta actividad construye la pieza 1 de 2 del warm-up de
caché — una MEJORA que no existe en la referencia adjunta.

Objetivo (RA2, criterios c, e): crear el evento interno TopNovedadesInvalidadoEvent y
publicarlo desde VideojuegoService tras cada operación que invalida la caché.

Estructura sugerida de pasos guiados:
1. Comprobación guiada del problema que se va a resolver: medir con curl (comandos y
   formato de tiempo dados) la primera llamada a /api/v1/videojuegos/top (~2 s, paga el
   Thread.sleep(2000) simulado) y la segunda (instantánea, sale de la caché); después
   crear un videojuego (el @CacheEvict invalida) y medir de nuevo (~2 s otra vez).
   Anotar las tres mediciones: ese "primer usuario que paga" es lo que el warm-up va a
   eliminar.
2. La clase de evento guiada: un record TopNovedadesInvalidadoEvent (decidir juntos qué
   campos transporta — puede bastar el motivo o el instante de invalidación), en un
   paquete coherente con la estructura del proyecto (por ejemplo,
   catalogo/eventos internos, explicando por qué NO va en catalogo/api/eventos, que es
   el contrato de mensajería RabbitMQ entre módulos).
3. La publicación guiada: inyectar ApplicationEventPublisher en VideojuegoService (con
   @RequiredArgsConstructor, como el resto de dependencias) y publicar el evento en
   create() — código mostrado y explicado.
4. Mini-reto (repite el patrón del paso 3): publicar el mismo evento en update() y
   delete() — solo se indica el objetivo.
5. Verificación guiada de que el evento se publica: como aún no hay listener, añadir un
   listener trivial temporal de prueba (@EventListener que solo hace un log, código
   dado), crear un videojuego y ver la traza; observar también EN QUÉ HILO se ejecuta
   ese listener sin @Async (el mismo hilo de la petición — anotarlo: será el contraste
   clave de la próxima actividad).
6. Pregunta de comprensión: ¿por qué conviene que el evento sea un record inmutable si
   lo van a leer otros hilos? (criterio e).
```
