<a id="usuarios-persistidos-bcrypt"></a>

# 🧩 3. Usuarios persistidos y BCrypt

## 📍 De dónde partimos

Al terminar el apartado anterior, tu API ya distinguía autenticación de autorización, pero con dos piezas deliberadamente provisionales: los usuarios `user` y `admin` declarados a mano en un `InMemoryUserDetailsManager`, con la contraseña en texto plano (el prefijo `{noop}`); y HTTP Basic, mandando esas credenciales en cada petición, solo codificadas en Base64, no cifradas.

Esa versión dejaba dos problemas señalados a propósito. Hoy resuelves el primero: los usuarios vivían en el propio código Java —visibles para cualquiera con acceso al repositorio, y se perdían cada vez que reiniciabas la aplicación—; a partir de ahora viven en PostgreSQL, con la contraseña protegida de verdad. El segundo problema —las credenciales viajando en cada petición— sigue abierto; lo resuelve JWT, más adelante en el tema.

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

Si simplemente aplicaras una función hash (`hash("miPassword123")`) a cada contraseña, dos usuarios con la misma contraseña tendrían **el mismo hash guardado** — y eso es un problema: cualquiera que vea la base de datos sabría que comparten contraseña, y un atacante podría precalcular por adelantado los hashes de las contraseñas más habituales en una **tabla arcoíris**\* y buscar coincidencias directas contra cualquier base de datos robada.

La **sal** (*salt*) resuelve esto: un valor aleatorio distinto para cada usuario, que se combina con la contraseña antes de hashear. Así, dos usuarios con la misma contraseña obtienen hashes completamente distintos — la sal rompe la comparación directa.

Con datos concretos, dos usuarios que eligen exactamente la misma contraseña:

| Usuario | Contraseña | Sal | Hash guardado |
|---|---|---|---|
| `ana` | `miPassword123` | aleatoria, propia de `ana` | `{bcrypt}$2a$10$x7Ff…9kLp` |
| `luis` | `miPassword123` | aleatoria, propia de `luis` | `{bcrypt}$2a$10$q2Rt…3mWx` |

Nadie que mire la tabla `usuarios` puede saber que `ana` y `luis` comparten contraseña, y una tabla arcoíris precalculada para `miPassword123` no sirve de nada aquí: habría que precalcularla de nuevo para cada sal distinta, una por usuario.

<small>\* Una tabla arcoíris es un diccionario gigante, calculado una sola vez por adelantado, que empareja contraseñas comunes con el hash que producirían. El atacante no calcula nada en el momento del ataque: roba una tabla de hashes y busca cada uno directamente en su diccionario ya hecho. Por eso la sal la inutiliza — necesitaría una tabla distinta para cada sal.</small>

---

## 🐌 Por qué BCrypt y no un hash rápido

Podrías pensar en usar **SHA-256** directamente — otra función hash, de propósito general, muy usada para comprobar que un fichero descargado no se ha corrompido o alterado. El problema es justo su virtud en ese otro contexto: **es rápido**, pensado para calcularse millones de veces por segundo sin apenas coste. Aplicado a contraseñas, eso juega en tu contra: un atacante con acceso a una tabla de hashes robada puede probar miles de millones de contraseñas por segundo contra un hash SHA-256, por fuerza bruta.

**BCrypt** está diseñado deliberadamente para ser **lento**, con un coste computacional configurable (el llamado *factor de coste*): cuanto mayor el coste, más tiempo (y recursos) hace falta para calcular un solo hash — insignificante para un login legítimo (una fracción de segundo), pero devastador para un atacante que necesita probar millones de combinaciones. BCrypt incluye además la sal automáticamente, como parte del propio hash resultante.

Una idea de cuánto pesa ese coste en la práctica (tiempos aproximados, en hardware normal):

| Factor de coste | Tiempo aprox. por hash |
|---|---|
| 4 | ~1 ms |
| 10 (el valor por defecto) | ~60-100 ms |
| 12 | ~250-400 ms |

Cien milisegundos son imperceptibles para un usuario que hace login una vez. Pero multiplicados por los millones de intentos que necesitaría un ataque de fuerza bruta, la diferencia entre "factible" e "inviable" se cuenta en años de cómputo.

!!! tip "Un apunte sobre criptografía de clave pública/privada"
    Existe otra familia de criptografía, la de **clave pública/privada**: un par de claves matemáticamente relacionadas donde lo que cifra una (o firma) solo lo puede verificar la otra. No la necesitas todavía — la retomarás en el próximo apartado, cuando veas cómo se firma un JWT.

---

## 🗄️ Usuarios reales en la base de datos

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

El campo `rol` usa el enum que ya se adelantó en el apartado anterior — solo dos valores, sin más complicación:

```java
public enum RolUsuario {
    ADMIN,
    USER
}
```

Y el campo `activo` es una baja lógica, no física: en vez de borrar la fila de un usuario al que le retiras el acceso, lo marcas como `activo = false` y lo conservas — así no pierdes el histórico de qué creó o modificó ese usuario mientras tuvo cuenta. Lo usarás en el siguiente paso.

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

### Un `UserDetailsService` propio: el sustituto del `InMemoryUserDetailsManager`

```java
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

Es el reemplazo directo del `InMemoryUserDetailsManager` del apartado anterior: implementa `UserDetailsService`, y en vez de buscar en una lista fija en memoria, busca en `UsuarioRepository` — la misma base de datos que usas para todo lo demás. Spring Security llama a `loadUserByUsername(...)` automáticamente cuando alguien intenta autenticarse; tú solo tienes que decirle dónde encontrar al usuario. El `.disabled(!usuario.isActivo())` es justo lo que usa el `activo` que has visto al definir la entidad: un usuario con `activo = false` sigue existiendo en la tabla, pero Spring Security lo trata como deshabilitado y rechaza sus intentos de login.

!!! tip "`securityFilterChain` no cambia ni una línea"
    Podría parecer que, al sustituir una pieza tan central, tendrías que volver a tocar `.authorizeHttpRequests(...)`, `.csrf(...)`, `.exceptionHandling(...)` o `.httpBasic(...)` — pero no hace falta tocar nada de eso. Spring Security detecta automáticamente el único bean `UserDetailsService` que hay en el contexto (antes era `InMemoryUserDetailsManager`; ahora es `BdUserDetailsService`) y lo conecta solo con el `PasswordEncoder` que acabas de declarar, usándolos internamente para comparar la contraseña recibida contra el hash guardado. Todo lo que construiste en el apartado anterior —rutas públicas, CSRF desactivado, tu `AuthenticationEntryPoint` a medida— sigue funcionando exactamente igual, sin que toques `securityFilterChain` para nada.

### El registro: `AuthController`

Todo lo de arriba explica cómo se **verifica** un usuario que ya existe — pero no cómo llega a existir el primero. Para eso hace falta un endpoint público que reciba usuario y contraseña, codifique la contraseña con el mismo `PasswordEncoder` de antes, y guarde el `Usuario` resultante:

```java
public record RegisterRequestDTO(
        @NotBlank(message = "El nombre de usuario no puede estar vacío") String username,
        @NotBlank(message = "La contraseña no puede estar vacía") String password
) {}
```

```java
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

`passwordEncoder.encode(...)` es la otra mitad del mismo bean que ya conoces: aquí **codifica**, en `BdUserDetailsService` de arriba se **verificaba** — el mismo `PasswordEncoder`, las dos operaciones opuestas que necesita todo el ciclo de vida de una contraseña. Si alguien intentara registrar un `username` que ya existe, la restricción `unique = true` de la entidad lo bloquea en la base de datos — y ese error ya sabes gestionarlo, es el mismo `DataIntegrityViolationException` que tu `GlobalExceptionHandler` convierte en un `409` desde el primer apartado del tema.

!!! warning "El rol nunca lo decide quien se registra"
    Fíjate en que `RolUsuario.USER` está fijado aquí mismo, en el servidor — no viene del DTO que manda el cliente. Si `RegisterRequestDTO` incluyera un campo `rol` y lo guardaras tal cual, cualquiera podría registrarse a sí mismo como `ADMIN` con solo añadirlo al JSON. La regla es simple: nunca dejes que el cliente decida sus propios privilegios.

Como cualquier otra ruta nueva, hay que abrirla explícitamente en tu política de acceso — nadie tiene todavía una cuenta con la que autenticarse para registrarse:

```java
.requestMatchers(HttpMethod.POST, "/api/v1/auth/register").permitAll()
```

Pero ese mismo aviso deja una pregunta abierta: si nadie puede auto-asignarse `ADMIN`, ¿de dónde sale el primero? Ese usuario tiene que venir de fuera del propio flujo de registro.

### El primer `ADMIN`: un `ApplicationRunner`

La pieza que hace falta es un `ApplicationRunner`: una interfaz de Spring Boot con un único método, `run(...)`, que el propio framework ejecuta automáticamente **una sola vez**, justo después de que la aplicación termine de arrancar — antes de que llegue ninguna petición. Es el sitio natural para insertar datos que tienen que existir desde el primer momento:

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
            Usuario admin = new Usuario();
            admin.setUsername("admin");
            admin.setPassword(passwordEncoder.encode(adminPassword));
            admin.setRol(RolUsuario.ADMIN);
            usuarioRepository.save(admin);
        }
    }
}
```

Igual que con `UserDetailsService`, no hace falta registrar nada a mano: basta con anotar la clase `@Component` e implementar `ApplicationRunner`, y Spring Boot la detecta y la ejecuta sola en cada arranque. El `if (...isEmpty())` es imprescindible por eso mismo —`run(...)` se ejecuta en **cada** arranque, no solo la primera vez—: sin esa comprobación, intentarías insertar el mismo `admin` cada vez que reinicies, y chocarías con la restricción `unique = true` del `username`.

Fíjate en que la contraseña ya no está escrita directamente en el código: viene de `${gamevault.admin.password}`, una propiedad externa. El motivo lo ves en la siguiente sección.

### Esa contraseña tampoco debería estar en tu repositorio

Aunque `admin123` ya no viva en ningún `.java`, si la escribieras tal cual en `application-dev.yaml` seguirías teniendo el mismo problema: ese fichero se sube a Git, así que cualquiera con acceso al repositorio la vería igual. La solución es un fichero de propiedades **aparte, que nunca se versiona**:

```yaml
# application-dev.yaml (SÍ se sube a Git)
spring:
  config:
    import: "optional:application-dev-local.yaml"
```

```yaml
# application-dev-local.yaml (en .gitignore, NO se sube a Git)
gamevault:
  admin:
    password: admin123
```

El prefijo `optional:` le dice a Spring Boot que, si ese fichero no existe (por ejemplo, en la máquina de un compañero que acaba de clonar el repositorio), arranque igualmente en vez de fallar. Y como `@Value("${gamevault.admin.password}")` no lleva ningún valor por defecto, si de verdad falta la propiedad, la aplicación **no arranca** — con un error claro señalando qué falta, en vez de arrancar silenciosamente con una contraseña vacía o adivinada.

Para que cualquiera sepa qué tiene que rellenar sin tener que adivinarlo, se acompaña de una plantilla que sí se sube a Git:

```yaml
# application-dev-local.yaml.example (SÍ se sube a Git — sin valores reales)
gamevault:
  admin:
    password: pon-aqui-tu-propia-contraseña
```

Cada persona que clona el repositorio copia ese `.example` a `application-dev-local.yaml`, rellena sus propios valores, y ese fichero final se queda solo en su máquina. Es el mismo patrón, exactamente, que vas a reutilizar para el secreto de JWT más adelante en el tema.

!!! note "¿No es esto lo mismo que la contraseña de Postgres, que dejaste en el fichero versionado?"
    Buena pregunta, y la respuesta es matizada. Tal y como está tu proyecto **hoy** —corriendo solo en tu Dev Container, sin desplegar en ningún sitio—, el riesgo real de que `admin123` se filtre por GitHub es prácticamente el mismo que el de la contraseña de Postgres del primer apartado del tema: nadie fuera de tu máquina puede usarla ahora mismo, porque no hay ningún servidor público al que dirigirla.

    La diferencia no está en el peligro de hoy, sino en lo fácil que es que este secreto concreto sobreviva sin querer hasta un despliegue real. La contraseña de Postgres solo importaría si algún día reconfiguraras a propósito tu aplicación contra una base de datos en la nube — un paso consciente. Pero es muy típico coger un proyecto que ya funciona como base para algo real y desplegarlo copiando la configuración tal cual, sin darte cuenta de que ese `admin123` o ese secreto de JWT siguen siendo los mismos del primer commit. Protegerlo ahora, cuando no cuesta nada y no urge, es más barato que aprender por qué hacía falta el día que sí urja.

---

## 🧭 Lo que aún falta

Ya no hay usuarios hardcodeados que desaparezcan al reiniciar — están en PostgreSQL, con contraseñas que ni tú mismo puedes leer. Pero sigue quedando el segundo problema del apartado anterior: las credenciales viajan en **cada** petición, aunque sea con HTTP Basic. En el próximo apartado lo resuelve JWT: autenticarte una vez, y presentar un token en las peticiones siguientes.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - **Cifrar** es reversible (con clave); **hashear** no lo es — las contraseñas se hashean, nunca se cifran.
    - La **sal** hace que contraseñas iguales produzcan hashes distintos, evitando comparación directa y tablas arcoíris.
    - **BCrypt** es deliberadamente lento (factor de coste configurable), lo que lo hace resistente a fuerza bruta frente a un hash rápido como SHA-256 a secas.
    - `DelegatingPasswordEncoder` antepone un prefijo (`{bcrypt}`) al hash, indicando el algoritmo usado — el mismo bean codifica y verifica.
    - Un `UserDetailsService` propio sustituye al `InMemoryUserDetailsManager`: busca los usuarios en la base de datos real.
    - Spring Security detecta solo el `UserDetailsService` y el `PasswordEncoder` del contexto y los conecta por su cuenta — `securityFilterChain` no cambia ni una línea.
    - Un endpoint de registro público crea usuarios reales, siempre con rol `USER` fijado en el servidor — nunca confíes en que el cliente decida sus propios privilegios. El primer `ADMIN` no puede salir de ahí: hace falta un *seed* aparte.
    - Ese *seed* no debe llevar la contraseña escrita en el código ni en un fichero versionado: va en `application-dev-local.yaml` (en `.gitignore`), importado con `spring.config.import: optional:...`, acompañado de un `.example` sin valores reales que sí se sube a Git.
