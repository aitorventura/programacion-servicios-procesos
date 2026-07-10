# 🧩 Tema 4: Programación de comunicaciones en red

> **RA3**: Programa mecanismos de comunicación en red empleando sockets y analizando el escenario de ejecución.

---

## 🎯 Criterios de evaluación

✅ Se han identificado escenarios que precisan establecer comunicación en red entre varias aplicaciones.
✅ Se han identificado los roles de cliente y de servidor y sus funciones asociadas.
✅ Se han reconocido librerías y mecanismos del lenguaje de programación que permiten programar aplicaciones en red.
✅ Se ha analizado el concepto de socket, sus tipos y características.
✅ Se han utilizado sockets para programar una aplicación cliente que se comunique con un servidor.
✅ Se ha desarrollado una aplicación servidor en red y verificado su funcionamiento.
✅ Se han desarrollado aplicaciones que utilizan sockets para intercambiar información.
✅ Se han utilizado hilos para posibilitar la comunicación simultánea de varios clientes con el servidor.
✅ Se han caracterizado los modelos de comunicación más usuales en las arquitecturas de aplicaciones distribuidas.
✅ Se han depurado y documentado las aplicaciones desarrolladas.

---

## 📘 Índice de contenidos

1. [Sockets: la base de toda comunicación en red](sockets-cliente-servidor.md)
2. [WebSocket en GameVault: el canal de actividad en vivo](websocket-gamevault.md)
3. [Actividad en vivo y seguridad del canal — cierre de RA3](actividad-en-vivo-cierre.md)

**Actividades:**

- [Actividad 4.1 — Cliente y servidor con sockets clásicos](actividad_4_1.md)
- [Actividad 4.2 — El endpoint `/ws-actividad` con STOMP](actividad_4_2.md)
- [Actividad 4.3 — Actividad en vivo desde `ActividadService` — cierre de RA3](actividad_4_3.md)

---

!!! info "¿Cómo avanzar por el contenido?"
    Utiliza el índice o las flechas de navegación al final de cada página para desplazarte por los distintos apartados de este tema.

!!! note "Nota para el profesorado"
    Este tema corresponde a las semanas reales 17-19 del calendario (RA3, bloque cerrado que también cierra el módulo).

    **Cobertura literal de RA3 (revisado):** los criterios b, c, d, e, f, g, h del currículo (roles cliente/servidor, librerías del lenguaje, tipos de socket, cliente que se comunica con un servidor, servidor verificado, intercambio de información, hilos para varios clientes simultáneos) quedan cubiertos de forma LITERAL, con `java.net.Socket`/`ServerSocket` reales, en la semana 17 (`sockets-cliente-servidor.md` + Actividad 4.1). No dependen de WebSocket para estar satisfechos. Antes se marcaban como "Adaptado" dando a entender que solo se cumplían vía WebSocket en las semanas 18-19 — eso subestimaba el trabajo real ya hecho en la 4.1 y generaba un riesgo innecesario si se auditara el RA3 contra el currículo literal.

    Las semanas 18-19 (WebSocket/STOMP sobre GameVault, el canal `/ws-actividad`) se reencuadran como una AMPLIACIÓN de profundidad sobre el mismo RA3 (mismo concepto de "canal bidireccional persistente" ya presentado en la teoría de la semana 17, esta vez sobre un escenario real de aplicación) y como puente hacia RA4 (servicios en red), no como sustituto necesario de los criterios ya cubiertos. Consecuencia práctica: si un alumno completara solo la Actividad 4.1 y no llegara a las Actividades 4.2/4.3, RA3 seguiría estando cubierto en su literal — las últimas dos semanas añaden valor pero no son la única vía de cumplimiento.

    Aviso aparte, distinto del anterior: el canal `/ws-actividad` de las semanas 18-19 es una MEJORA que no existe en la referencia adjunta (no hay `spring-boot-starter-websocket` ni configuración WebSocket en el `pom.xml`/código entregado). A diferencia del resto del curso, donde el alumnado siempre puede comparar su trabajo con un estado final ya construido en la referencia, aquí construye sin ese punto de comparación — coméntalo explícitamente en la teoría y en las actividades para que no se sorprendan al no encontrar un `WebSocketConfig.java` en el proyecto adjunto.

    Los apartados de teoría y las actividades están pendientes de redactar: cada `.md` contiene el prompt que debe usarse con `/improve-notes` para generarlo, apoyándose en el proyecto GameVault adjunto.
