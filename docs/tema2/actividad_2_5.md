# 🧪 Actividad 2.5: Roles y rutas protegidas — cierre de RA5

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 2.5 del Tema 2 (RA5 - Programación segura) del módulo Programación
de Servicios y Procesos (0490), semana real 11 del calendario — actividad que CIERRA el
RA5. Si necesita plantilla/solución en .docx, crea antes la skill de plantilla de PSP
clonando /actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso; solo se deja sin guiar,
como mini-reto, lo que repita un patrón idéntico ya mostrado.

Objetivo (RA5, criterios d, h — cierre del RA): que el alumnado complete en su GameVault
la política de autorización por roles al nivel de la referencia (el bloque
authorizeHttpRequests de SecurityConfig.java) y la verifique con tests de seguridad.

Estructura sugerida de pasos guiados:
1. Construcción guiada de la tabla de política de seguridad: el enunciado presenta la
   tabla objetivo (ruta × verbo × quién puede) tomada de SecurityConfig.java de la
   referencia, Y AMPLIADA con las rutas que el alumnado ha añadido durante el curso y
   que la referencia no contempla (PUT/DELETE de Estudio del Tema 1 — decidir juntos:
   ¿solo ADMIN, como el resto de escrituras del catálogo?). Rellenar SecurityConfig
   regla a regla siguiendo la tabla, con las primeras reglas explicadas y las restantes
   como repetición del patrón.
2. La regla final `anyRequest().denyAll()` y su verificación guiada: probar una ruta no
   listada y observar el bloqueo — entender por qué "cerrado por defecto" es la opción
   segura.
3. Depuración guiada de un caso real (criterio h): con denyAll activo, alguna ruta
   legítima olvidada dejará de funcionar (el enunciado provoca el caso a propósito, por
   ejemplo el GET del resumen de reviews) — diagnosticar con los logs de Spring Security
   (configuración de logging dada) y arreglar añadiendo la regla que falta.
4. Tests de seguridad guiados: un test MockMvc completo mostrado y explicado (endpoint
   de escritura sin token → 401; con rol USER → 403; con rol ADMIN → éxito), siguiendo
   el enfoque de los tests de la referencia.
5. Mini-reto (repite el patrón del paso 4): los mismos tres casos sobre otro endpoint
   (por ejemplo, el DELETE de Estudio que crearon en el Tema 1) — solo se indica qué
   cubrir.
6. Cierre de RA5: completar la tabla de política como documento final (la documentación
   del criterio h) y un repaso propio (4-5 frases) de la evolución completa: validación →
   Basic → BCrypt → JWT → roles.
```
