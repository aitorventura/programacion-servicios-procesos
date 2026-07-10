<a id="usuarios-persistidos-bcrypt"></a>

# 🧩 3. Usuarios persistidos y BCrypt

!!! tip "Esto sí es ya el estado final"
    A diferencia del apartado anterior (usuarios en memoria, un paso intermedio), lo que construyes hoy es la solución definitiva: usuarios reales en PostgreSQL, con la contraseña protegida por BCrypt.

---

## 🔐 Cifrar vs. hashear

Dos operaciones criptográficas que se confunden con facilidad, pero son opuestas en una propiedad clave:

| | Cifrar | Hashear |
|---|---|---|
| **Reversible** | Sí, con la clave correcta | No, nunca |
| **Para qué sirve** | Proteger datos que necesitas recuperar tal cual | Proteger datos que solo necesitas **comparar**, nunca leer de vuelta |

Una contraseña no necesitas nunca "leerla de vuelta" — solo necesitas comprobar si la que te acaban de dar coincide con la que el usuario registró. Por eso las contraseñas se **hashean**, no se cifran: ni siquiera tú, con acceso total a la base de datos, deberías poder recuperar la contraseña original a partir del hash guardado.

---

## 🧂 Hash con sal

Si simplemente aplicaras una función hash (`hash("miPassword123")`) a cada contraseña, dos usuarios con la misma contraseña tendrían **el mismo hash guardado** — y eso es un problema: cualquiera que vea la base de datos sabría que comparten contraseña, y un atacante podría precalcular hashes de contraseñas comunes (una "tabla arcoíris") y buscar coincidencias directas.

La **sal** (*salt*) resuelve esto: un valor aleatorio distinto para cada usuario, que se combina con la contraseña antes de hashear. Así, dos usuarios con la misma contraseña obtienen hashes completamente distintos — la sal rompe la comparación directa.

---

## 🐌 Por qué BCrypt y no un hash rápido

Podrías pensar en usar SHA-256 directamente. El problema es justo su virtud en otros contextos: **es rápido**. Un atacante con acceso a una tabla de hashes robada puede probar miles de millones de contraseñas por segundo contra un hash SHA-256, por fuerza bruta.

**BCrypt** está diseñado deliberadamente para ser **lento**, con un coste computacional configurable (el llamado *factor de coste*): cuanto mayor el coste, más tiempo (y recursos) hace falta para calcular un solo hash — insignificante para un login legítimo (una fracción de segundo), pero devastador para un atacante que necesita probar millones de combinaciones. BCrypt incluye además la sal automáticamente, como parte del propio hash resultante.

!!! tip "Un apunte sobre criptografía de clave pública/privada"
    Existe otra familia de criptografía, la de **clave pública/privada**: un par de claves matemáticamente relacionadas donde lo que cifra una (o firma) solo lo puede verificar la otra. No la necesitas todavía — la retomarás la semana que viene, cuando veas cómo se firma un JWT.

---

## 🎮 Aterrizaje en GameVault: usuarios reales

### La entidad `Usuario`

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    private RolUsuario rol;

    private boolean activo = true;
}
```

Territorio ya conocido de Acceso a Datos: una entidad JPA más, con su repositorio (`UsuarioRepository extends JpaRepository<Usuario, Long>`, con un `findByUsername` añadido). La única novedad real es el campo `password`: nunca va a contener texto legible, solo el hash BCrypt.

### El `PasswordEncoder`

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

`createDelegatingPasswordEncoder()` construye un `DelegatingPasswordEncoder`: un encoder que antepone un prefijo al hash almacenado (por ejemplo, `{bcrypt}`) indicando qué algoritmo se usó — así, si en el futuro cambiaras de algoritmo, los hashes antiguos con prefijos distintos seguirían siendo reconocibles y verificables, sin migrar toda la tabla de golpe. Este mismo bean se usa tanto para **codificar** una contraseña al crear un usuario como para **verificar** una contraseña al hacer login (BCrypt no "descifra" el hash — vuelve a calcularlo con la contraseña recibida y compara).

Un hash BCrypt real tiene esta forma:

```
{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
```

Diseccionado: `{bcrypt}` (el prefijo del `DelegatingPasswordEncoder`), `$2a$` (la versión del algoritmo), `10$` (el factor de coste — aquí, 2¹⁰ iteraciones), y el resto combina la sal y el hash resultante, ambos codificados en la misma cadena — no hace falta guardar la sal en una columna aparte, viaja incluida.

### `GamevaultUserDetailsService`: el sustituto del `InMemoryUserDetailsManager`

```java
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

Es el reemplazo directo del `InMemoryUserDetailsManager` de la semana pasada: implementa `UserDetailsService`, y en vez de buscar en una lista fija en memoria, busca en `UsuarioRepository` — la misma base de datos que usas para todo lo demás. Spring Security llama a `loadUserByUsername(...)` automáticamente cuando alguien intenta autenticarse; tú solo tienes que decirle dónde encontrar al usuario.

---

## 🧭 Lo que aún falta

Ya no hay usuarios hardcodeados que desaparezcan al reiniciar — están en PostgreSQL, con contraseñas que ni tú mismo puedes leer. Pero sigue quedando el segundo problema del apartado anterior: las credenciales viajan en **cada** petición, aunque sea con HTTP Basic. La semana que viene lo resuelve JWT: autenticarte una vez, y presentar un token en las peticiones siguientes.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - **Cifrar** es reversible (con clave); **hashear** no lo es — las contraseñas se hashean, nunca se cifran.
    - La **sal** hace que contraseñas iguales produzcan hashes distintos, evitando comparación directa y tablas arcoíris.
    - **BCrypt** es deliberadamente lento (factor de coste configurable), lo que lo hace resistente a fuerza bruta frente a un hash rápido como SHA-256 a secas.
    - `DelegatingPasswordEncoder` antepone un prefijo (`{bcrypt}`) al hash, indicando el algoritmo usado — el mismo bean codifica y verifica.
    - `GamevaultUserDetailsService` sustituye al `InMemoryUserDetailsManager`: implementa `UserDetailsService` buscando en la base de datos real.
