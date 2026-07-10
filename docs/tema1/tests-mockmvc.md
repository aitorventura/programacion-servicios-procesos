<a id="tests-mockmvc"></a>

# 🧩 3. Probar servicios con MockMvc

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Probar servicios con MockMvc" del Tema 1 (RA4 -
Generación de servicios en red) del módulo Programación de Servicios y Procesos (0490),
semana real 5 del calendario. Sigue las convenciones de estilo del README.md del repo.

Criterios de evaluación de RA4 que cubre este apartado (curriculum.md):
- d) Desarrollo y prueba de servicios de comunicación en red.
- e) Clientes de comunicaciones para verificar el funcionamiento de los servicios.
- f) Comunicación simultánea de varios clientes con el servicio.

ESTRUCTURA — teoría primero: antes de MockMvc, asegura la base de testing desde cero
(en Entornos de Desarrollo de 1º vieron pruebas, pero repásalo sin darlo por sabido):
qué es un test automatizado y por qué existe (comprobar comportamiento sin repetir
clics), la anatomía de un test JUnit (@Test, preparar-actuar-afirmar, los asserts), y la
diferencia entre test unitario (una pieza aislada, colaboradores simulados/mocks — qué
es un mock, en 2-3 frases) y test de integración (varias piezas reales juntas).

Contenido central: MockMvc como "cliente HTTP programable" para probar los endpoints
automáticamente — el tercer cliente que conoce el alumnado tras curl (Actividad 1.1) y
Swagger UI (Actividad 1.2), con la diferencia clave de que este es repetible y se
ejecuta en el CI.

Apóyate en el proyecto GameVault (com.aleroig.gamevault) como ejemplo real:
- src/test/java/com/aleroig/gamevault/catalogo/VideojuegoControllerTest.java: analiza su
  estructura — cómo se construye una petición (get/post/put con contenido JSON), cómo se
  afirman código de estado y cuerpo (andExpect, status(), jsonPath()), y qué se mockea
  frente a qué se ejecuta de verdad.
- Explica la diferencia entre este tipo de test (capa web, service mockeado o contexto
  parcial) y el test de integración completo
  (src/test/java/com/aleroig/gamevault/integration/GamevaultApiTest.java, con
  Testcontainers) que el alumnado conocerá a fondo en AD — aquí basta con situar ambos
  niveles.
- Sobre el criterio f) (comunicación simultánea de varios clientes): explica que Spring
  Web/Tomcat atiende cada petición HTTP en un hilo distinto de un pool — el alumnado
  puede comprobarlo imprimiendo Thread.currentThread().getName() o mirando los logs, y
  esta idea se retomará a fondo en el Tema 3 (RA2, multihilo). Un experimento simple:
  dos peticiones lentas simultáneas (el método getTopNovedades() de
  VideojuegoService.java tiene un Thread.sleep(2000) simulado, perfecto para esto) no
  tardan 4 segundos, sino ~2, porque las atienden hilos distintos.

Cierra presentando la práctica de la semana: el PUT de Estudio. En la referencia
adjunta, com/aleroig/gamevault/catalogo/EstudioController.java SOLO tiene GET y POST —
el PUT y el DELETE de Estudio son una MEJORA que el alumnado añadirá en PSP (el PUT
esta semana en la Actividad 1.3, el DELETE la próxima en la 1.4), replicando el patrón
de VideojuegoController que ya construyó en AD.
```
