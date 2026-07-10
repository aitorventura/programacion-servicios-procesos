<a id="seguridad-basica-http-basic"></a>

# 🧩 2. Seguridad básica: usuarios en memoria y HTTP Basic

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Seguridad básica: usuarios en memoria y HTTP Basic" del
Tema 2 (RA5 - Programación segura) del módulo Programación de Servicios y Procesos
(0490), semana real 8 del calendario. Sigue las convenciones de estilo del README.md del
repo.

Criterios de evaluación de RA5 que cubre este apartado (curriculum.md):
- c) Políticas de seguridad para limitar y controlar el acceso de los usuarios.
- d) Esquemas de seguridad basados en roles (primera aproximación).

AVISO IMPORTANTE sobre la referencia: el SecurityConfig.java del GameVault adjunto
muestra el ESTADO FINAL del proyecto (JWT, con `httpBasic(AbstractHttpConfigurer::
disable)`). Este apartado enseña el estado INTERMEDIO por el que se pasa para llegar
ahí: Spring Security recién añadido, usuarios en memoria y HTTP Basic. Esa evolución
está documentada en docs/seguridad/autenticacion-y-autorizacion.md del proyecto — úsala
como fuente. Deja claro al alumnado que esta versión se sustituirá en las semanas 9-10,
y que ver el camino completo (Basic → BCrypt → JWT) es deliberado: entender qué problema
resuelve cada paso.

Contenido central:
- Qué hace spring-boot-starter-security nada más añadirse al pom: todo queda cerrado por
  defecto (contrasta con el principio de "mínima exposición" del apartado 1).
- Conceptos de autenticación vs. autorización, con ejemplos del propio GameVault (¿quién
  eres? vs. ¿puedes borrar un videojuego?).
- HTTP Basic: cómo viaja usuario:contraseña en Base64 en la cabecera Authorization —
  demuéstralo decodificando una cabecera de ejemplo, para dejar claro que Base64 NO es
  cifrado (gancho hacia la necesidad de HTTPS y de mecanismos mejores).
- Usuarios en memoria (InMemoryUserDetailsManager) con dos usuarios y roles distintos
  (user/admin), y una primera política de acceso sencilla con authorizeHttpRequests
  (lectura pública del catálogo, escritura autenticada) — versión simplificada de la
  matriz de rutas final que verán en la semana 11.
- Los roles ADMIN y USER que usa el proyecto final (com/aleroig/gamevault/seguridad/
  RolUsuario.java) como referencia de a dónde se llegará.

Limitaciones a señalar como cierre (y motivación de las semanas siguientes): usuarios
hardcodeados que se pierden al reiniciar y contraseñas que viajan en cada petición —
la semana 9 mueve los usuarios a PostgreSQL con BCrypt, la 10 sustituye Basic por JWT.
```
