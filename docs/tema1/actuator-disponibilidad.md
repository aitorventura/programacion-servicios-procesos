<a id="actuator-disponibilidad"></a>

# 🧩 4. Disponibilidad del servicio: Actuator

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo. Este apartado cierra el RA4.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Disponibilidad del servicio: Actuator" del Tema 1 (RA4 -
Generación de servicios en red) del módulo Programación de Servicios y Procesos (0490),
semana real 6 del calendario — apartado que CIERRA el RA4. Sigue las convenciones de
estilo del README.md del repo.

Criterios de evaluación de RA4 que cubre este apartado (curriculum.md):
- d) Desarrollo y prueba de servicios de comunicación en red (segunda parte: el DELETE
  de Estudio, en la actividad).
- g) Verificación de la disponibilidad del servicio.

ESTRUCTURA — teoría primero: explica desde cero qué es la disponibilidad de un servicio
y por qué se monitoriza: un servicio en red corre sin nadie mirándolo, así que hace
falta una forma automática de saber si está vivo y sano; qué es un health check como
concepto general (una comprobación periódica que una máquina puede hacer sola), la
diferencia entre "el proceso está arrancado", "responde peticiones" y "sus dependencias
funcionan" (tres niveles distintos de "estar bien"), y quién consume esa información en
el mundo real (monitores de alertas, orquestadores que reinician contenedores caídos, el
CI). Después, Spring Boot Actuator como la implementación de todo eso.

Contenido central: monitorización de un servicio en red con Spring Boot Actuator — qué
significa que un servicio esté "disponible" (responde, y además sus dependencias
funcionan), y cómo se comprueba de forma automatizable.

Apóyate en el proyecto GameVault (com.aleroig.gamevault) como ejemplo real:
- Comprueba primero si el pom.xml de la referencia incluye spring-boot-starter-actuator;
  si no lo incluye, preséntalo como MEJORA que se añade esta semana (dependencia +
  configuración mínima en application.yaml para exponer /actuator/health).
- El endpoint /actuator/health: qué devuelve (UP/DOWN) y, con detalles activados
  (management.endpoint.health.show-details), cómo agrega el estado de las dependencias
  reales de GameVault — PostgreSQL, MongoDB, RabbitMQ y Redis, los cuatro servicios del
  docker-compose.yaml — de modo que si se para el contenedor de Mongo, el health pasa a
  DOWN aunque la aplicación siga respondiendo. Para Redis basta con nombrarlo aquí como
  "una base de datos en memoria más, cuya salud también se agrega" — su explicación de
  fondo (qué es y para qué lo usa el proyecto) llega en el Tema 3 (RA2, semana 14, el
  warm-up de caché), no la dupliques aquí.
- Conecta con lo ya visto: el health check es "un GET más" (protocolo estándar otra
  vez), pensado para que lo consulte una máquina (un orquestador, un monitor, el CI) y
  no una persona — cierra así el criterio g).
- Menciona brevemente otros endpoints útiles de Actuator (info, metrics) sin
  profundizar.

Cierra el apartado recapitulando todo RA4 en 3-4 frases (leer la API y su protocolo →
escribir y documentar con OpenAPI → probar con MockMvc y varios clientes → verificar
disponibilidad) y presentando la Actividad 1.4, que completa la pareja de mejoras sobre
Estudio con el DELETE (el PUT se hizo en la Actividad 1.3).
```
