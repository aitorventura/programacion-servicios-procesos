<a id="roles-y-rutas-protegidas"></a>

# 🧩 5. Roles, rutas protegidas y tests de seguridad

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo. Este apartado cierra el RA5.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Roles, rutas protegidas y tests de seguridad" del Tema 2
(RA5 - Programación segura) del módulo Programación de Servicios y Procesos (0490),
semana real 11 del calendario — apartado que CIERRA el RA5. Sigue las convenciones de
estilo del README.md del repo.

Criterios de evaluación de RA5 que cubre este apartado (curriculum.md):
- d) Esquemas de seguridad basados en roles (versión completa).
- h) Depuración y documentación de las aplicaciones.

Contenido central: la política de autorización completa del proyecto, ruta a ruta, y
cómo se prueba automáticamente que la seguridad hace lo que debe.

Apóyate en el proyecto GameVault (com.aleroig.gamevault) como ejemplo real:
- El bloque authorizeHttpRequests completo de SecurityConfig.java, leído línea a línea
  como una TABLA de política de seguridad (preséntalo también como tabla en el
  apartado): login público; GET del catálogo (videojuegos y estudios) público; POST de
  reviews para USER o ADMIN; POST/PUT/DELETE de videojuegos y POST de estudios solo
  ADMIN; GET /api/v1/actividad solo ADMIN; Swagger y /error abiertos; y la línea más
  importante de todas: `anyRequest().denyAll()` — todo lo no listado queda cerrado
  (retoma el principio de mínima exposición de la semana 7). Comenta también por qué
  las rutas nuevas que el alumnado ha ido añadiendo (PUT/DELETE de Estudio del Tema 1,
  el ranking de AD...) necesitan su regla explícita o quedarán bloqueadas — un error
  típico que conviene saber depurar (criterio h).
- La diferencia entre 401 (no autenticado) y 403 (autenticado sin permiso), con ejemplos
  de qué petición produce cada uno según la tabla.
- Tests de seguridad: cómo probar con MockMvc que un endpoint responde 401 sin token,
  403 con rol insuficiente y 200/201 con el rol correcto — apóyate en los tests del
  proyecto (src/test/java/com/aleroig/gamevault/, revisa cómo gestionan la seguridad:
  anotaciones tipo @WithMockUser o construcción del token en el test de integración
  GamevaultApiTest.java) y en lo aprendido en el Tema 1 con MockMvc.
- Documentación (criterio h): la propia tabla de rutas/roles como documentación de la
  política de seguridad, al estilo de docs/seguridad/autenticacion-y-autorizacion.md del
  proyecto.

Cierra recapitulando todo RA5 en 4-5 frases (validación y errores → Basic en memoria →
usuarios BCrypt en PostgreSQL → JWT → roles y política completa con tests) y recordando
que AD reutilizará este JWT en la semana real 16 (PUT de reseñas con autoría).
```
