# Programación de Servicios y Procesos (0490) — Apuntes y actividades

Sitio de apuntes y actividades del módulo **Programación de Servicios y Procesos** (0490, ciclo DAM), construido con [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Teoría y actividades se apoyan en el mismo proyecto compartido que Acceso a Datos, **GameVault** (catálogo de videojuegos con Spring Boot), trabajando aquí la capa de servicios REST, seguridad, concurrencia y comunicaciones en red.

## Puesta en marcha

```bash
pip install -r requirements.txt
mkdocs serve
```

Abre la URL que te indique la terminal (por defecto `http://127.0.0.1:8000`).

## Temario

- **Tema 0 — Repaso**: relaciones entre clases, colecciones, programación funcional.
- **Tema 1 — Servicios en red**: protocolos estándar y REST, escritura en la API y OpenAPI/Swagger, tests con MockMvc, comunicación simultánea y disponibilidad con Actuator.
- **Tema 2 — Programación segura**: principios de programación segura, seguridad básica con HTTP Basic, usuarios persistidos con BCrypt, autenticación JWT, roles y rutas protegidas.
- **Tema 3 — Programación multihilo**: hilos en una aplicación real, eventos internos y *warm-up* de caché, listeners asíncronos, `TaskExecutor` propio.
- **Tema 4 — Comunicaciones en red**: sockets cliente/servidor, WebSocket con STOMP, actividad en vivo.
