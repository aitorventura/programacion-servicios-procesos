<a id="usuarios-persistidos-bcrypt"></a>

# 🧩 3. Usuarios persistidos y BCrypt

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Usuarios persistidos y BCrypt" del Tema 2 (RA5 -
Programación segura) del módulo Programación de Servicios y Procesos (0490), semana real
9 del calendario. Sigue las convenciones de estilo del README.md del repo.

Criterios de evaluación de RA5 que cubre este apartado (curriculum.md):
- b) Principales técnicas y prácticas criptográficas.
- e) Algoritmos criptográficos para proteger el acceso a la información almacenada.

Contenido central: sustituir los usuarios en memoria de la semana anterior por usuarios
reales guardados en PostgreSQL, con la contraseña protegida por un hash BCrypt. Este SÍ
es ya el estado final de la referencia adjunta.

Parte criptográfica (criterio b), explicada con los pies en el suelo antes de tocar
código: diferencia entre cifrar (reversible, con clave) y hashear (irreversible);
hash con sal y por qué dos usuarios con la misma contraseña deben tener hashes
distintos; por qué BCrypt y no un hash rápido tipo SHA-256 a secas (coste computacional
configurable contra fuerza bruta). Menciona también, en 2-3 frases, la criptografía de
clave pública/privada como concepto (se retomará con la firma del JWT la próxima
semana).

Apóyate en el proyecto GameVault (com.aleroig.gamevault, paquete `seguridad`) como
ejemplo real:
- Usuario.java y UsuarioRepository.java: la entidad JPA de usuario (con su rol
  RolUsuario) y su repositorio — conecta con lo que el alumnado ya domina de AD: es
  una entidad más en PostgreSQL.
- GamevaultUserDetailsService.java: el puente entre "mi tabla de usuarios" y Spring
  Security — implementa UserDetailsService cargando el usuario desde el repositorio;
  explícalo como el sustituto directo del InMemoryUserDetailsManager de la semana
  anterior.
- El bean PasswordEncoder de SecurityConfig.java
  (PasswordEncoderFactories.createDelegatingPasswordEncoder()): qué es el
  DelegatingPasswordEncoder (el prefijo {bcrypt} en el hash almacenado) y cómo se usa
  tanto al crear usuarios como al verificar credenciales.
- Muestra un hash BCrypt real de ejemplo y disecciónalo (versión, coste, sal, hash) para
  que se vea que la sal va incluida.

Cierra señalando lo que aún falta: las credenciales siguen viajando en cada petición
(HTTP Basic) — la semana 10 lo resuelve con JWT.
```
