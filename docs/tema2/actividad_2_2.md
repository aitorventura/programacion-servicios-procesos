# 🧪 Actividad 2.2: Primera línea de defensa — HTTP Basic

!!! info "Práctica guiada — estado intermedio deliberado"
    Hoy añades Spring Security con usuarios en memoria y HTTP Basic. No es la versión final (eso llega en las Actividades 2.3 y 2.4) — es un paso intermedio para entender qué problema resuelve cada pieza.

## Qué vas a practicar

- Observar el efecto de Spring Security "sin configurar nada".
- Configurar usuarios en memoria y una primera política de rutas.
- Comprobar cómo viaja una credencial HTTP Basic, decodificándola tú mismo.

---

## Requisitos previos

Tu GameVault con el CRUD de videojuegos y `GlobalExceptionHandler` (Actividad 2.1) funcionando.

---

## Paso 1 — Añadir Spring Security y ver el efecto inmediato

En tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Reinicia tu aplicación **sin añadir ninguna configuración propia todavía** y prueba:

```bash
curl -i http://localhost:8080/api/v1/videojuegos
```

**Predicción**: antes de ejecutar, ¿qué código de estado esperas? Escríbelo.

Deberías obtener un `401 Unauthorized` — incluso en un `GET` que hasta ahora era público. **Comprueba también** que Swagger UI (`/swagger-ui.html`) ha dejado de ser accesible sin autenticarte.

---

## Paso 2 — `SecurityConfig` inicial, guiada al completo

Crea `SecurityConfig` en tu paquete `seguridad`:

```java
package com.tunombre.gamevault.seguridad;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

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

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(HttpMethod.GET, "/api/v1/videojuegos/**").permitAll()
                        .requestMatchers(HttpMethod.GET, "/api/v1/estudios/**").permitAll()
                        .anyRequest().authenticated()
                )
                .httpBasic(Customizer.withDefaults())
                .build();
    }
}
```

Dos usuarios en memoria (`user`/`user123` con rol `USER`, `admin`/`admin123` con rol `ADMIN`), `GET` del catálogo público, el resto exige estar autenticado, y `httpBasic(Customizer.withDefaults())` activa HTTP Basic como mecanismo de autenticación.

Reinicia y comprueba de nuevo el `GET` del Paso 1 — debería volver a funcionar sin credenciales, porque ahora está explícitamente marcado como público.

---

## Paso 3 — Probar con curl y decodificar la cabecera

Sin credenciales, sobre una ruta protegida:

```bash
curl -i -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" -d '{}'
```

**Comprueba**: `401`.

Con credenciales:

```bash
curl -i -u user:user123 http://localhost:8080/api/v1/videojuegos
```

**Comprueba**: `200`. Repite con `curl -v` (no `-i`) para ver la cabecera `Authorization` que `curl` ha generado por ti, y decodifícala a mano:

```bash
echo "dXNlcjp1c2VyMTIz" | base64 -d
```

(sustituye esa cadena por la que veas realmente en tu propia petición, tras `Basic `).

**Comprueba**: que el resultado es tu usuario y contraseña en texto legible — exactamente lo que viste en la teoría.

---

## Mini-reto — proteger la creación de videojuegos por rol

Repite el patrón de `requestMatchers(...)` ya mostrado en el Paso 2 para añadir una regla nueva: el `POST /api/v1/videojuegos` debe exigir el rol `ADMIN` específicamente (no basta con estar autenticado con cualquier usuario).

**Pista de sintaxis** (ya la has visto en el `requestMatchers` del catálogo, solo cambia el final):

```java
.requestMatchers(HttpMethod.POST, "/api/v1/videojuegos").hasRole("ADMIN")
```

Prueba con los dos usuarios:

```bash
curl -i -u user:user123 -X POST http://localhost:8080/api/v1/videojuegos -H "Content-Type: application/json" -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
curl -i -u admin:admin123 -X POST http://localhost:8080/api/v1/videojuegos -H "Content-Type: application/json" -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: `403` con `user`, `201` con `admin`.

---

## Pregunta final

Esta configuración tiene dos problemas serios para un proyecto real. Nómbralos con tus propias palabras, relacionando cada uno con algo concreto que hayas visto en esta actividad (uno tiene que ver con dónde viven `user`/`admin` en tu código; el otro con lo que has visto al decodificar la cabecera `Authorization`). ¿Qué actividad crees que resolverá cada uno, según lo que se ha adelantado en la teoría?

---

## ✅ Cierre

Tu API ya distingue quién puede hacer qué — pero con usuarios que se pierden al reiniciar y contraseñas casi en claro en cada petición. En la próxima actividad resuelves el primero: usuarios reales en PostgreSQL, con contraseñas protegidas por BCrypt.
