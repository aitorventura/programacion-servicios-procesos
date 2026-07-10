# 🧪 Actividad 2.2: Primera línea de defensa — HTTP Basic

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 2.2 del Tema 2 (RA5 - Programación segura) del módulo Programación
de Servicios y Procesos (0490), semana real 8 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado en la misma actividad. Recuerda que esta actividad
construye un estado INTERMEDIO deliberado (no presente en la referencia final): usuarios
en memoria + HTTP Basic, que se sustituirán en las actividades 2.3 y 2.4.

Objetivo (RA5, criterios c, d): que el alumnado añada Spring Security a su GameVault con
usuarios en memoria y HTTP Basic, y una primera política de rutas.

Estructura sugerida de pasos guiados:
1. Añadir spring-boot-starter-security al pom (fragmento dado) y arrancar sin configurar
   nada: observar guiadamente que TODO devuelve 401 (curl dado) — la seguridad por
   defecto cierra todo, incluso Swagger.
2. Crear una clase SecurityConfig inicial guiada al completo (código dado y explicado):
   dos usuarios en memoria con InMemoryUserDetailsManager (un USER y un ADMIN, roles
   como en RolUsuario.java de la referencia final), httpBasic activado, y una política
   mínima de rutas con authorizeHttpRequests: GET del catálogo público, resto
   autenticado.
3. Probar guiadamente con curl: petición sin credenciales (401), con credenciales
   correctas (`curl -u`, 200), y observar la cabecera Authorization — decodificar juntos
   el Base64 (comando dado) para comprobar que usuario y contraseña viajan legibles.
4. Mini-reto (repite el patrón del paso 2): añadir una regla más a la política — que el
   POST de videojuegos exija rol ADMIN — y probar con ambos usuarios (403 con USER, 201
   con ADMIN). El patrón de requestMatchers + hasRole ya está mostrado.
5. Pregunta de comprensión: ¿qué dos problemas graves tiene esta configuración para un
   proyecto real? (usuarios hardcodeados en el código; credenciales que viajan en cada
   petición apenas ofuscadas) — son exactamente los que resuelven las dos próximas
   actividades.
```
