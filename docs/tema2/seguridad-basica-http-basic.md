<a id="seguridad-basica-http-basic"></a>

# 🧩 2. Seguridad básica: usuarios en memoria y HTTP Basic

!!! tip "Vas a construir un estado intermedio, no el final"
    El `SecurityConfig.java` que tendrás al final del módulo usará autenticación con JWT, con HTTP Basic desactivado. Este apartado construye el paso **intermedio** por el que se pasa para llegar ahí — usuarios en memoria y HTTP Basic — y las próximas dos semanas lo van sustituyendo. Recorrer el camino completo (Basic → BCrypt → JWT) es deliberado: cada paso resuelve un problema concreto del anterior, y entenderlo así vale más que llegar directo al resultado final.

---

## 🔒 Lo primero que hace Spring Security: cerrarlo todo

En cuanto añades `spring-boot-starter-security` a un proyecto, sin configurar nada más, **toda** tu API deja de responder libremente — cada petición exige autenticarse. Es el principio de "mínima exposición" que viste el apartado anterior, llevado al extremo por defecto: Spring Security asume que, si no le dices explícitamente qué dejar abierto, todo debe estar cerrado.

---

## 🎭 Autenticación vs. autorización

Dos preguntas distintas, que conviene no confundir:

| | Pregunta que responde | Ejemplo en GameVault |
|---|---|---|
| **Autenticación** | ¿Quién eres? | Comprobar que el usuario y la contraseña son correctos. |
| **Autorización** | ¿Puedes hacer esto? | Comprobar que, siendo tú, tienes permiso para borrar un videojuego. |

Puedes estar autenticado (Spring Security sabe quién eres) y aun así no autorizado para una acción concreta (no tienes el rol necesario). Son dos capas distintas, y las vas a ver aplicadas por separado.

---

## 📦 HTTP Basic: cómo viaja usuario y contraseña

**HTTP Basic** es el mecanismo de autenticación más simple de HTTP: el cliente manda usuario y contraseña en una cabecera `Authorization`, codificados en **Base64**:

```
Authorization: Basic dXNlcjp1c2VyMTIz
```

!!! danger "Base64 NO es cifrado"
    Decodifica tú mismo esa cabecera: `echo "dXNlcjp1c2VyMTIz" | base64 -d` — obtienes `user:user123`, en texto legible. Base64 es solo una **codificación** (una forma de representar bytes como texto), no una forma de ocultar información: cualquiera que intercepte esa cabecera lee la contraseña directamente. HTTP Basic solo es aceptable sobre una conexión cifrada con HTTPS — sin eso, la contraseña viaja prácticamente en claro en cada petición.

---

## 🧑‍🤝‍🧑 Usuarios en memoria y una primera política de rutas

Para empezar, sin base de datos todavía de por medio, Spring Security permite declarar usuarios directamente en código con `InMemoryUserDetailsManager`:

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.withUsername("user")
            .password("{noop}user123")
            .roles("USER")
            .build();

    UserDetails admin = User.withUsername("admin")
            .password("{noop}admin123")
            .roles("ADMIN")
            .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

El prefijo `{noop}` le dice a Spring Security "esta contraseña no está cifrada, compárala tal cual" — una simplificación deliberada para este paso intermedio (la próxima semana la sustituyes por contraseñas de verdad protegidas con BCrypt).

Y una primera política de acceso, con `authorizeHttpRequests`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                    .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
}
```

Esta es una versión simplificada — lectura del catálogo pública, todo lo demás exige estar autenticado — de la matriz de rutas completa que construirás en la semana 11. Los roles `ADMIN` y `USER` que acabas de usar son los mismos que vas a formalizar más adelante como un enum `RolUsuario` — solo esos dos valores, sin más complicación.

---

## ⚠️ Las limitaciones de esta versión — a propósito

Esta configuración tiene dos problemas serios, y son intencionados: sirven de motivación para lo que viene.

1. **Usuarios hardcodeados**: viven en el propio código Java, se pierden cada vez que reinicias la aplicación, y cualquiera con acceso al código ve las contraseñas. La semana 9 los mueve a PostgreSQL, con contraseñas protegidas por BCrypt.
2. **Credenciales que viajan en cada petición**: con HTTP Basic, cada única petición vuelve a mandar usuario y contraseña, apenas ofuscados en Base64. La semana 10 sustituye esto por JWT: te autenticas una vez, y presentas un token en las peticiones siguientes.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Spring Security, nada más añadirse, cierra **todo** por defecto — mínima exposición llevada al extremo.
    - **Autenticación** (¿quién eres?) y **autorización** (¿puedes hacer esto?) son capas distintas.
    - **HTTP Basic** manda usuario:contraseña en Base64 en la cabecera `Authorization` — Base64 no es cifrado, así que Basic solo es seguro sobre HTTPS.
    - `InMemoryUserDetailsManager` declara usuarios directamente en código — útil para empezar, pero se pierden al reiniciar.
    - Esta configuración es un estado **intermedio** deliberado: la semana 9 resuelve los usuarios hardcodeados (BCrypt + PostgreSQL), la semana 10 resuelve las credenciales viajando en cada petición (JWT).
