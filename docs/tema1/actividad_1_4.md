# 🧪 Actividad 1.4: El DELETE de Estudio y Actuator `/health` — cierre de RA4

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 1.4 del Tema 1 (RA4 - Generación de servicios en red) del módulo
Programación de Servicios y Procesos (0490), semana real 6 del calendario — actividad
que CIERRA el RA4. Si necesita plantilla/solución en .docx, crea antes la skill de
plantilla de PSP clonando /actividad-plantilla-acceso-a-datos con la paleta
marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto — con una excepción deliberada:
el DELETE de Estudio es el reto de repetición de este tema, porque el alumnado ya ha
construido el mismo patrón DOS veces (el DELETE de Videojuego en AD y el PUT de Estudio
en la Actividad 1.3). El resto de la actividad (Actuator) sí va guiado paso a paso.

Objetivo (RA4, criterios d, g — cierre del RA): completar el CRUD de Estudio con el
DELETE (MEJORA: no existe en EstudioController.java de la referencia adjunta) y añadir
la verificación de disponibilidad con Actuator.

Estructura sugerida:
1. Reto de repetición — el DELETE de Estudio: el enunciado solo da la especificación
   (ruta DELETE /api/v1/estudios/{id}, 204 si borra, 404 si no existe, @Transactional en
   el service) y recuerda dónde están los dos ejemplos ya construidos del mismo patrón
   (el DELETE de VideojuegoController y el PUT de Estudio de la Actividad 1.3). El
   alumnado lo escribe solo. Añadir la pregunta de comprensión: al borrar un Estudio,
   ¿qué pasa con sus Videojuego? (conectar con el cascade visto en AD).
2. Verificación del DELETE: manual desde Swagger UI y con un test MockMvc — también como
   reto de repetición (el patrón de test es el de la Actividad 1.3), solo se indica qué
   casos cubrir (204 y 404).
3. Actuator, guiado paso a paso: añadir la dependencia spring-boot-starter-actuator al
   pom (fragmento dado), exponer /actuator/health con detalles en application.yaml
   (configuración dada y explicada), y consultar el endpoint con curl.
4. Experimento guiado de disponibilidad: parar el contenedor de MongoDB
   (`docker compose stop mongo`, comando dado), volver a consultar /actuator/health y
   observar el cambio a DOWN con el detalle del componente mongo; volver a levantarlo y
   comprobar la recuperación — cada paso con su comando y su salida esperada.
5. Cierre de RA4: pedir al alumnado un repaso propio (3-4 frases) del recorrido del
   tema: leer la API → documentarla → probarla → completarla (PUT/DELETE de Estudio) →
   monitorizarla.
```
