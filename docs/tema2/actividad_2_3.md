# 🧪 Actividad 2.3: Usuarios reales en PostgreSQL con BCrypt

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 2.3 del Tema 2 (RA5 - Programación segura) del módulo Programación
de Servicios y Procesos (0490), semana real 9 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado.

Objetivo (RA5, criterios b, e): que el alumnado sustituya los usuarios en memoria de la
Actividad 2.2 por usuarios persistidos en PostgreSQL con contraseña BCrypt, replicando
el estado final de la referencia (paquete com/aleroig/gamevault/seguridad/).

Estructura sugerida de pasos guiados:
1. La entidad Usuario y su RolUsuario (enum) + UsuarioRepository, guiados con el código
   de la referencia mostrado — señalar que esto es territorio conocido de AD (una
   entidad JPA más); la novedad es el campo de contraseña, que NUNCA guardará texto
   claro.
2. El bean PasswordEncoder en SecurityConfig
   (PasswordEncoderFactories.createDelegatingPasswordEncoder()), guiado, y la creación
   de los usuarios iniciales (semilla en DataInitializer o equivalente de su GameVault)
   codificando la contraseña con el encoder — código mostrado.
3. Comprobación guiada en la base de datos (psql/DBeaver, consulta dada): ver el hash
   almacenado con su prefijo {bcrypt} y diseccionarlo con la ayuda del enunciado.
   Verificar que dos usuarios con la misma contraseña tienen hashes distintos (la sal).
4. GamevaultUserDetailsService guiado con el código de la referencia: implementar
   UserDetailsService y retirar el InMemoryUserDetailsManager de la Actividad 2.2 —
   explicar qué pieza sustituye a cuál.
5. Prueba guiada: login por HTTP Basic con los usuarios de la base de datos (curl dado),
   y comprobación de que los roles siguen funcionando (la regla ADMIN del mini-reto de
   la 2.2).
6. Pregunta de comprensión: si un atacante roba la tabla de usuarios, ¿qué obtiene
   exactamente y por qué no le sirve directamente para iniciar sesión? ¿Y por qué BCrypt
   se lo pone más difícil que un SHA-256 a secas?
```
