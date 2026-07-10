<a id="tests-mockmvc"></a>

# 🧩 3. Probar servicios con MockMvc

Hasta ahora has probado la API a mano: con `curl` (Actividad 1.1) y con Swagger UI (Actividad 1.2). Los dos funcionan, pero comparten un problema — tienes que repetir los mismos clics o comandos cada vez que quieres comprobar que todo sigue funcionando. Hoy conoces un tercer cliente, uno que se ejecuta solo: un **test automatizado**.

---

## 🧪 Qué es un test automatizado

Un **test automatizado** es código que comprueba el comportamiento de otro código, sin intervención humana. En vez de abrir Postman y pulsar "Enviar" cada vez que cambias algo, escribes una vez el test y lo ejecutas las veces que haga falta — en segundos, sin volver a hacer los mismos clics, y de forma que cualquiera (incluido un pipeline de integración continua, como el que viste en el Tema 0 de AD con GitHub Actions) pueda ejecutarlo automáticamente en cada cambio.

Un test JUnit sigue casi siempre la misma estructura, conocida como **preparar-actuar-afirmar** (*Arrange-Act-Assert*):

```java
@Test
void sumar_DebeDevolverLaSumaCorrecta() {
    Calculadora calc = new Calculadora();      // preparar

    int resultado = calc.sumar(2, 3);          // actuar

    assertEquals(5, resultado);                // afirmar
}
```

`@Test` marca el método como un test que JUnit debe ejecutar. Primero **preparas** lo que necesitas, luego **actúas** (llamas al método que quieres probar), y por último **afirmas** (compruebas que el resultado es el esperado) — si la afirmación falla, el test falla y te avisa exactamente qué esperaba y qué obtuvo.

---

## 🎭 Test unitario vs. test de integración

No todos los tests prueban lo mismo ni al mismo nivel:

| | Test unitario | Test de integración |
|---|---|---|
| **Qué prueba** | Una pieza aislada (una clase, un método) | Varias piezas reales trabajando juntas |
| **Colaboradores** | Simulados (*mocks*) | Reales |
| **Velocidad** | Muy rápido | Más lento |

Un **mock** es un objeto falso que sustituye a una dependencia real durante el test, y que tú programas para que se comporte exactamente como tú digas ("cuando te llamen con este parámetro, devuelve este resultado"). Sirve para aislar la pieza que quieres probar de todo lo demás: si estás probando un controller, no necesitas que la base de datos esté levantada de verdad — mockeas el service que hay detrás y decides tú qué devuelve.

---

## 🌐 MockMvc: un cliente HTTP programable

**MockMvc** es la herramienta de Spring para probar controllers REST sin necesitar un servidor HTTP real arrancado: simula peticiones HTTP contra tus controladores, dentro del propio test, y te permite comprobar el código de estado y el cuerpo de la respuesta con código Java.

Es, en esencia, un tercer cliente — como `curl` o Swagger UI — pero con una diferencia clave: es **repetible** y se puede ejecutar automáticamente (en tu máquina o en un pipeline de CI) cada vez que cambias algo, sin que nadie tenga que abrir el navegador.

---

## 🎮 Aterrizaje en GameVault: leyendo `VideojuegoControllerTest`

```java
@WebMvcTest(VideojuegoController.class)
@AutoConfigureMockMvc(addFilters = false)
class VideojuegoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private VideojuegoService videojuegoService;

    @Test
    void create_DebeDevolver400_CuandoElDtoNoEsValido() throws Exception {
        String body = """
                {"titulo": "", "precio": -5, "fechaLanzamiento": null, "estudioId": -1}
                """;

        mockMvc.perform(post("/api/v1/videojuegos")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.status").value(400));
    }
}
```

Pieza a pieza:

- `@WebMvcTest(VideojuegoController.class)`: arranca solo la capa web de Spring (el controller y lo relacionado con MVC), sin levantar toda la aplicación ni conectar con una base de datos real — mucho más rápido que un arranque completo.
- `@MockitoBean private VideojuegoService videojuegoService`: sustituye el service real por un mock — el test prueba el controller de forma aislada, sin que la lógica del service (ni la base de datos que hay detrás) entre en juego.
- `mockMvc.perform(post(...).contentType(...).content(...))`: construye y envía una petición simulada — el equivalente, en código, al `curl -X POST` que ya conoces.
- `.andExpect(status().isBadRequest())`: afirma el código de estado esperado (aquí, `400`).
- `.andExpect(jsonPath("$.status").value(400))`: `jsonPath` navega el cuerpo JSON de la respuesta como si fuera un mini-selector, para afirmar valores concretos dentro de él.

Este test concreto es de **capa web**: prueba que el controller valida correctamente el DTO de entrada (gracias a `@Valid`, que ya viste el apartado anterior) sin necesitar el service real ni una base de datos.

### ¿Y el test de integración completo?

En `src/test/java/com/aleroig/gamevault/integration/GamevaultApiTest.java` hay otro tipo de test, con `@Testcontainers`, que levanta bases de datos **reales** en contenedores Docker solo para la duración del test — no mockea nada. Lo trabajarás a fondo en Acceso a Datos; aquí basta con que sitúes los dos niveles: `@WebMvcTest` prueba una capa aislada y rápida, un test de integración prueba varias piezas reales trabajando juntas y es más lento pero da más confianza sobre el sistema completo.

---

## 🧵 Comunicación simultánea de varios clientes

Un servidor real no atiende a un solo cliente a la vez. Spring Web, sobre Tomcat, atiende cada petición HTTP entrante en un **hilo distinto**, tomado de un *pool* — así que varias peticiones pueden procesarse en paralelo sin que unas esperen a que terminen las otras. Esta idea se retomará a fondo en el Tema 3, sobre programación multihilo; de momento, compruébala con un experimento sencillo.

`VideojuegoService.getTopNovedades()` tiene, a propósito, un `Thread.sleep(2000)` que simula una consulta lenta. Lanza dos peticiones **simultáneas**:

```bash
time (curl -s http://localhost:8080/api/v1/videojuegos/top & \
      curl -s http://localhost:8080/api/v1/videojuegos/top & \
      wait)
```

Si las dos peticiones se atendieran una detrás de otra, el conjunto tardaría unos 4 segundos. Si se atienden en paralelo, tardan aproximadamente 2 — porque cada una la procesa un hilo distinto del pool. Puedes confirmarlo añadiendo temporalmente una traza con `Thread.currentThread().getName()` en el método y mirando el log: verás dos nombres de hilo distintos (`http-nio-8080-exec-1`, `http-nio-8080-exec-2`...) para las dos peticiones.

---

## 🎯 Lo que viene: el PUT de Estudio

En tu propio `EstudioController` (Acceso a Datos, Actividad 1.2) solo tienes `GET` y `POST` — todavía no existe `PUT` ni `DELETE`. Esa es la práctica de esta semana y la siguiente: en la Actividad 1.3 construyes el `PUT` de `Estudio`, replicando el mismo patrón que ya construiste para `Videojuego` en Acceso a Datos la semana pasada; el `DELETE` llega en la Actividad 1.4.

---

## ✅ Ideas clave

??? tip "Abrir resumen"

    - Un **test automatizado** comprueba comportamiento sin intervención humana, y se puede repetir en cada cambio; sigue el patrón preparar-actuar-afirmar.
    - Un test **unitario** aísla una pieza con **mocks**; un test de **integración** prueba varias piezas reales juntas.
    - **MockMvc** simula peticiones HTTP contra tus controladores sin arrancar un servidor real — un cliente HTTP programable y repetible.
    - `@WebMvcTest` arranca solo la capa web; `@MockitoBean` sustituye una dependencia por un mock; `mockMvc.perform(...).andExpect(...)` construye la petición y afirma el resultado; `jsonPath` navega el cuerpo JSON.
    - Cada petición HTTP la atiende un hilo distinto del pool de Tomcat — por eso dos peticiones lentas simultáneas no tardan el doble, sino aproximadamente lo mismo que una sola.
    - Esta semana toca construir el `PUT` de `Estudio` (Actividad 1.3), replicando el patrón del `PUT` de `Videojuego` ya construido en AD.
