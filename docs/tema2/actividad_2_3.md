# 🧪 Actividad 2.3: Usuarios reales en PostgreSQL con BCrypt

!!! info "Práctica guiada"
    Hoy sustituyes los usuarios en memoria de la Actividad 2.2 por usuarios reales en PostgreSQL, con la contraseña protegida por BCrypt — el estado final de este apartado.

## Qué vas a practicar

- Crear una entidad `Usuario` persistida (territorio conocido de AD).
- Codificar contraseñas con BCrypt al crear usuarios.
- Sustituir `InMemoryUserDetailsManager` por un `UserDetailsService` real.

---

## Requisitos previos

Tu `SecurityConfig` con usuarios en memoria de la Actividad 2.2.

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

## Paso 2 — El `PasswordEncoder` y la semilla de usuarios

Añade el bean a tu `SecurityConfig`:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

Ahora crea (o adapta, si ya tienes un `DataInitializer` de AD) un componente que inserte los dos usuarios de prueba **codificando** la contraseña con este mismo encoder:

```java
@Component
@RequiredArgsConstructor
public class UsuariosSeed implements ApplicationRunner {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    public void run(ApplicationArguments args) {
        if (usuarioRepository.findByUsername("user").isEmpty()) {
            Usuario u = new Usuario();
            u.setUsername("user");
            u.setPassword(passwordEncoder.encode("user123"));
            u.setRol(RolUsuario.USER);
            usuarioRepository.save(u);
        }
        if (usuarioRepository.findByUsername("admin").isEmpty()) {
            Usuario a = new Usuario();
            a.setUsername("admin");
            a.setPassword(passwordEncoder.encode("admin123"));
            a.setRol(RolUsuario.ADMIN);
            usuarioRepository.save(a);
        }
    }
}
```

`passwordEncoder.encode(...)` es donde ocurre la magia: la contraseña en claro (`"user123"`) entra, y lo que se guarda en la base de datos es el hash BCrypt, nunca el texto original.

---

## Paso 3 — Comprobar el hash en la base de datos

```bash
docker exec -it <tu-contenedor-postgres> psql -U gamevault_user -d gamevault_db \
  -c "SELECT username, password FROM usuarios;"
```

**Anota** el hash completo de uno de los usuarios. **Comprueba**:

1. Que empieza por `{bcrypt}$2a$` (o `$2b$`, según la versión).
2. Que, si tuvieras dos usuarios con exactamente la misma contraseña, sus hashes serían distintos entre sí — puedes comprobarlo creando un tercer usuario de prueba con la misma contraseña que `user` y comparando los dos hashes en la tabla.

---

## Paso 4 — `GamevaultUserDetailsService`, guiado al completo

```java
package com.tunombre.gamevault.seguridad;

import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class GamevaultUserDetailsService implements UserDetailsService {

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

## Paso 5 — Probar el login con los usuarios reales

```bash
curl -i -u user:user123 http://localhost:8080/api/v1/videojuegos
curl -i -u admin:admin123 -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: que ambas peticiones funcionan exactamente igual que con los usuarios en memoria de la actividad anterior — la regla de rol `ADMIN` del mini-reto de la 2.2 debe seguir funcionando sin que hayas tocado esa parte de `SecurityConfig`.

---

## Pregunta final

Si un atacante consiguiera robar tu tabla `usuarios` completa, ¿qué obtendría exactamente? ¿Por qué eso no le serviría directamente para iniciar sesión como uno de tus usuarios? ¿Y por qué le costaría más romper esos hashes que si hubieras usado SHA-256 a secas?

---

## ✅ Cierre

Tus usuarios ya son reales y sus contraseñas están protegidas. Sigue quedando un problema: cada petición sigue mandando las credenciales completas (aunque protegidas al guardarse, viajan por HTTP Basic en cada llamada). La semana que viene lo resuelves con JWT.
