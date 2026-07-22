# 🧪 Actividad 2.3: Usuarios reales en PostgreSQL con BCrypt

!!! warning "Descarga la plantilla"
    📄 [Plantilla 2.3 — Usuarios reales en PostgreSQL con BCrypt](plantillas/Actividad_2_3_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada"
    Hoy sustituyes los usuarios en memoria de la Actividad 2.2 por usuarios reales en PostgreSQL, con la contraseña protegida por BCrypt — el estado final de este apartado.

## Qué vas a practicar

- Crear una entidad `Usuario` persistida (territorio conocido de AD).
- Codificar contraseñas con BCrypt al crear usuarios.
- Construir un endpoint de registro real, con el rol fijado en el servidor.
- Sustituir `InMemoryUserDetailsManager` por un `UserDetailsService` real.

---

## Requisitos previos

Tu `SecurityConfig` con usuarios en memoria de la Actividad 2.2.

!!! warning "Revisa tu `data.sql`, o perderás los usuarios que crees hoy"
    Si desde Acceso a Datos sigues teniendo `spring.sql.init.mode: always` en tu `application-dev.yaml`, tu `data.sql` se vuelve a ejecutar en **cada** arranque de la aplicación — y si sigue el patrón habitual de borrar e insertar datos de prueba para dejar la base de datos en un estado repetible, cualquier usuario que crees hoy con `/register` (o el `admin` del *seed*) desaparecerá la próxima vez que reinicies, sin que hayas hecho nada mal.

    Para evitarlo, cambia en tu `application-dev.yaml`:
    ```yaml
    spring:
      sql:
        init:
          mode: never
    ```
    Con `never`, `data.sql` deja de ejecutarse en cada arranque — la base de datos conserva lo que de verdad tenga guardado, en vez de resetearse cada vez. Si en algún momento necesitas volver a poblarla desde cero, cambia el valor a `always` temporalmente, arranca una vez, y vuelve a dejarlo en `never`.

---

## Paso 1 — La entidad `Usuario` y su repositorio

```java
package com.tunombre.gamevault.seguridad;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "usuarios")
@Getter
@Setter
@NoArgsConstructor
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RolUsuario rol;

    @Column(nullable = false)
    private boolean activo = true;
}
```

```java
package com.tunombre.gamevault.seguridad;

public enum RolUsuario {
    ADMIN,
    USER
}
```

```java
package com.tunombre.gamevault.seguridad;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByUsername(String username);
}
```

Nada de esto es nuevo — es una entidad JPA más, como las que ya conoces de Acceso a Datos. La única diferencia real: el campo `password` nunca va a contener texto legible.

---

## Paso 2 — El `PasswordEncoder` y la semilla del primer `admin`

Añade el bean a tu `SecurityConfig`:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

Antes de crear el `ApplicationRunner`, prepara dónde va a vivir la contraseña de ese primer `admin` — **nunca** escrita en un `.java` ni en un fichero versionado. Añade a tu `.gitignore`:

```
application-dev-local.yml
```

Crea `application-dev-local.yml.example` (este sí se sube a Git, sin valores reales):

```yaml
gamevault:
  admin:
    password: pon-aqui-tu-propia-contraseña
```

Cópialo como `application-dev-local.yml` (este ya no se sube — está en `.gitignore`) y rellena tu propio valor:

```yaml
gamevault:
  admin:
    password: admin123
```

Y en tu `application-dev.yml`, importa ese fichero:

```yaml
spring:
  config:
    activate:
      on-profile: dev
    import: "optional:application-dev-local.yml"
```

Ahora sí, crea el `ApplicationRunner` que has visto en la teoría, que inserte **solo** el usuario `admin`, leyendo la contraseña de esa propiedad en vez de escribirla en el código:

```java
@Component
@RequiredArgsConstructor
public class UsuariosSeed implements ApplicationRunner {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    @Value("${gamevault.admin.password}")
    private String adminPassword;

    @Override
    public void run(ApplicationArguments args) {
        if (usuarioRepository.findByUsername("admin").isEmpty()) {
            Usuario a = new Usuario();
            a.setUsername("admin");
            a.setPassword(passwordEncoder.encode(adminPassword));
            a.setRol(RolUsuario.ADMIN);
            usuarioRepository.save(a);
        }
    }
}
```

`passwordEncoder.encode(...)` es donde ocurre la magia: la contraseña en claro entra, y lo que se guarda en la base de datos es el hash BCrypt, nunca el texto original.

**¿Por qué solo `admin`, y no también `user`?** Porque, como vas a ver en el paso siguiente, nadie puede registrarse a sí mismo como `ADMIN` — así que ese primer usuario con privilegios tiene que existir de antemano, fuera del propio flujo de la aplicación. Al usuario `user` lo vas a crear tú mismo dentro de un momento, a través de un endpoint de registro real.

---

## Paso 3 — El endpoint de registro: `AuthController`

```java
package com.tunombre.gamevault.seguridad;

public record RegisterRequestDTO(
        @NotBlank(message = "El nombre de usuario no puede estar vacío") String username,
        @NotBlank(message = "La contraseña no puede estar vacía") String password
) {}
```

```java
package com.tunombre.gamevault.seguridad;

@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    @Operation(summary = "Registrar un nuevo usuario")
    @ApiResponses({
            @ApiResponse(responseCode = "201", description = "Usuario creado correctamente"),
            @ApiResponse(responseCode = "400", description = "El cuerpo de la petición no supera las validaciones"),
            @ApiResponse(responseCode = "409", description = "Ya existe un usuario con ese nombre")
    })
    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequestDTO dto) {
        Usuario usuario = new Usuario();
        usuario.setUsername(dto.username());
        usuario.setPassword(passwordEncoder.encode(dto.password()));
        usuario.setRol(RolUsuario.USER);
        usuarioRepository.save(usuario);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

Abre esta ruta en tu política de acceso — nadie tiene todavía una cuenta con la que autenticarse para registrarse:

```java
.requestMatchers(HttpMethod.POST, "/api/v1/auth/register").permitAll()
```

Reinicia y crea el usuario `user` a través de tu propio endpoint:

```bash
curl -i -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"user","password":"user123"}'
```

**Comprueba**: `201 Created`.

**Captura**: la respuesta completa de este `curl`.

**Pregunta**: `RolUsuario.USER` está fijado en el propio método `register(...)`, no en el `RegisterRequestDTO`. ¿Qué pasaría si añadieras un campo `rol` al DTO y guardaras ese valor tal cual, sin fijarlo tú? Explica por qué eso sería un fallo de seguridad grave, no un simple detalle de diseño.

---

## Paso 4 — Comprobar los hashes en la base de datos

Tienes dos vías, igual que en Acceso a Datos — usa la que prefieras:

- **Herramienta gráfica** (pgAdmin, DBeaver): conecta a `localhost:5432` con las credenciales de tu `docker-compose.yml`, y ejecuta `SELECT username, password FROM usuarios;` sobre `gamevault_db`.
- **`psql` desde terminal**:
  ```bash
  docker exec -it <tu-contenedor-postgres> psql -U gamevault_user -d gamevault_db \
    -c "SELECT username, password FROM usuarios;"
  ```

**Anota** los hashes completos de `admin` y `user`. **Comprueba** que ambos empiezan por `{bcrypt}$2a$` (o `$2b$`, según la versión).

**Captura**: el resultado de esta consulta, con las dos filas.

Ahora regístrate una segunda vez, con un usuario distinto pero **la misma contraseña** que `user`:

```bash
curl -i -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"user2","password":"user123"}'
```

Repite la consulta a la base de datos y **compara** los hashes de `user` y `user2`. **Comprueba** que, pese a tener exactamente la misma contraseña, sus hashes son completamente distintos — es la sal en acción, no un error.

**Captura**: la consulta repetida, con las tres filas, para que se vean los hashes de `user` y `user2` uno junto al otro.

---

## Paso 5 — `BdUserDetailsService`, guiado al completo

```java
package com.tunombre.gamevault.seguridad;

import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class BdUserDetailsService implements UserDetailsService {

    private final UsuarioRepository usuarioRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Usuario usuario = usuarioRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Usuario no encontrado: " + username));

        return User.withUsername(usuario.getUsername())
                .password(usuario.getPassword())
                .roles(usuario.getRol().name())
                .disabled(!usuario.isActivo())
                .build();
    }
}
```

Ahora **retira** el bean `userDetailsService()` con `InMemoryUserDetailsManager` de tu `SecurityConfig` (el de la Actividad 2.2) — ya no lo necesitas. Spring Security detecta automáticamente esta nueva clase (anotada `@Service`, implementa `UserDetailsService`) y la usa en su lugar.

**Pregunta**: ¿qué pieza exacta sustituye a cuál? Nombra el bean/clase que desaparece y la clase que lo reemplaza.

---

## Paso 6 — Probar el login con los usuarios reales

`GET /api/v1/videojuegos` no sirve para esta prueba: es `permitAll()`, así que respondería `200` con `user:user123`, con una contraseña inventada, o sin ninguna credencial — el código no distingue si se ha validado algo de verdad. La prueba que sí demuestra una autenticación real es la misma que ya usaste en la Actividad 2.2: el `POST` protegido con `hasRole("ADMIN")`, donde un `403` (no un `401`) solo puede darse si las credenciales eran correctas y el rol insuficiente:

```bash
curl -i -u user:user123 -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
curl -i -u admin:admin123 -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: `403` con `user` (autenticado de verdad contra su hash real, pero sin el rol `ADMIN`), `201` con `admin` — exactamente el mismo comportamiento que con los usuarios en memoria de la Actividad 2.2, sin que hayas tocado esa parte de `SecurityConfig`. Prueba también con `user2:user123` — debería dar el mismo `403` que `user`, aunque su hash guardado sea distinto: BCrypt vuelve a calcular el hash con la sal guardada y compara, no le importa que el resultado final "se vea" distinto.

**Captura**: las dos respuestas del bloque anterior (`user` y `admin`).

---

## Pregunta final

Si un atacante consiguiera robar tu tabla `usuarios` completa, ¿qué obtendría exactamente? ¿Por qué eso no le serviría directamente para iniciar sesión como uno de tus usuarios? ¿Y por qué le costaría más romper esos hashes que si hubieras usado SHA-256 a secas?

---

## ✅ Cierre

Tus usuarios ya son reales, se pueden crear a través de un endpoint real, y sus contraseñas están protegidas. Sigue quedando un problema: cada petición sigue mandando las credenciales completas (aunque protegidas al guardarse, viajan por HTTP Basic en cada llamada). En la próxima actividad lo resuelves con JWT.
