# 🧪 Actividad 1.3: Tests MockMvc sobre `Videojuego` y `Estudio`

!!! warning "Descarga la plantilla"
    📄 [Plantilla 1.3 — Tests MockMvc sobre Videojuego y Estudio](plantillas/Actividad_1_3_PSP_Plantilla.docx){target="_blank" rel="noopener"}

## Qué vas a practicar

- Escribir tests MockMvc que prueben un controller de forma aislada.
- Distinguir qué se mockea de qué se ejecuta de verdad en un test de capa web.
- Repetir el mismo patrón de test sobre un segundo controller, con menos guía cada vez.
- Ejecutar tus tests automáticamente en cada `push`, con un pipeline de CI en GitHub Actions.

---

## Requisitos previos

Tu CRUD completo de `Videojuego` y de `Estudio` (Actividad 1.2 de AD) — las cuatro operaciones de cada uno, funcionando.

!!! note "Esto es una muestra representativa, no la batería de tests completa"
    Por tiempo, aquí solo escribes un test por caso (éxito y `404` en lectura; éxito en escritura) sobre cada método. En un proyecto real, cada método merece más de un test: casos límite (una lista vacía en `getAll`, un DTO que falla la validación en `create`/`update`, un `id` que no existe también en `update`/`delete`, no solo en `getById`...). Aquí basta con que el patrón quede claro y aplicado una vez por caso — la cobertura exhaustiva es una decisión de cada proyecto, no algo que se pueda enseñar de una sola vez.

---

## Bloque 1 — Tests MockMvc sobre `Videojuego`

### Paso 1 — Primer test, guiado al completo

Crea `VideojuegoControllerTest.java` en `src/test/java/com/tunombre/gamevault/catalogo/` — el mismo paquete que tu controller, esta vez bajo `src/test/java` en vez de `src/main/java`:

```java
package com.tunombre.gamevault.catalogo;

import com.tunombre.gamevault.catalogo.dto.VideojuegoResponseDTO;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(VideojuegoController.class)
@AutoConfigureMockMvc(addFilters = false)
class VideojuegoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private VideojuegoService videojuegoService;

    @Test
    void getAll_DebeDevolverListaDeVideojuegos() throws Exception {
        var dto = new VideojuegoResponseDTO(1L, "Hades", new BigDecimal("24.99"), LocalDate.of(2020, 9, 17), "Supergiant Games");
        when(videojuegoService.findAll()).thenReturn(List.of(dto));

        mockMvc.perform(get("/api/v1/videojuegos"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].titulo").value("Hades"));
    }
}
```

Léelo con calma: `when(videojuegoService.findAll()).thenReturn(...)` es la parte de **preparar** — le dices al mock qué debe devolver cuando lo llamen; `mockMvc.perform(get(...))` es **actuar**; los dos `.andExpect(...)` son **afirmar**. No se ha tocado la base de datos en ningún momento — `videojuegoService` es un mock, no el service real.

**Predicción**: si cambiaras `dto.titulo()` en la afirmación a `"Otro título"` sin cambiar el mock, ¿el test pasaría o fallaría? ¿Por qué?

### Paso 2 — Test del caso 404

Añade un segundo test, esta vez para el caso de error:

```java
@Test
void getById_DebeDevolver404_CuandoNoExiste() throws Exception {
    when(videojuegoService.findById(999L))
            .thenThrow(new ResponseStatusException(HttpStatus.NOT_FOUND, "Videojuego no encontrado"));

    mockMvc.perform(get("/api/v1/videojuegos/999"))
            .andExpect(status().isNotFound());
}
```

Aquí el mock, en vez de devolver un valor, **lanza una excepción** — así reproduces, sin tocar la base de datos, exactamente el mismo escenario que viviste con `curl` en la Actividad 1.1 cuando pediste un id inexistente.

### Paso 3 — Escritura: `POST`, `PUT` y `DELETE`

Repite el mismo patrón (preparar el mock con `when(...).thenReturn(...)`, actuar con `mockMvc.perform(post(...).contentType(...).content(...))`, afirmar con `.andExpect(status().isCreated())`) para probar:

- Que un `POST /api/v1/videojuegos` válido devuelve `201`. Guíate por el test de `create` que ya viste en la teoría para el cuerpo JSON de la petición.
- Que un `PUT /api/v1/videojuegos/{id}` válido devuelve `200`.
- Que un `DELETE /api/v1/videojuegos/{id}` devuelve `204`.

Tres tests, mismo patrón, sin más pistas que las de los Pasos 1-2.

Con los cinco tests de `VideojuegoControllerTest` escritos, ejecútalos desde tu entorno de desarrollo (el botón ▷ junto a la clase, o junto a cada test).

**Captura**: los cinco tests de `VideojuegoControllerTest` en verde, ejecutados desde el propio IDE.

---

## Bloque 2 — Repite el patrón sobre `Estudio`

`EstudioController` tiene las mismas cinco operaciones que `VideojuegoController` (`getAll`, `getById`, `create`, `update`, `delete`), así que `EstudioControllerTest` es el mismo ejercicio otra vez — pero esta vez sin el primer test dado como ejemplo.

**Escribe `EstudioControllerTest.java`** en `src/test/java/com/tunombre/gamevault/catalogo/`, mockeando `EstudioService`, con al menos:

- Un test de `getAll` (o `getById`) en el caso de éxito.
- Un test del caso `404` (con el mock lanzando `ResponseStatusException`).
- Un test de `create` (`201`), uno de `update` (`200`) y uno de `delete` (`204`).

**Pregunta antes de escribir nada**: ¿qué cambia realmente entre testear `VideojuegoController` y testear `EstudioController` — la estructura del test, o solo los nombres de clases y campos? Si tu respuesta es "solo los nombres", ya sabes por qué esta actividad no necesitaba darte otro ejemplo completo.

Con `EstudioControllerTest` completo, ejecútalo también desde tu entorno de desarrollo.

**Captura**: los tests de `EstudioControllerTest` en verde, ejecutados desde el propio IDE.

---

## Bloque 3 — CI: tus tests, en GitHub Actions

Ya montaste un pipeline de CI en Acceso a Datos (Actividad 0.5, sobre `validador-dni-ci`). Hoy aplicas exactamente el mismo patrón a tu propio GameVault — con la diferencia de que ahora sí tienes algo real que automatizar: los tests que acabas de escribir.

Si tu GameVault todavía no está subido a un repositorio de GitHub, súbelo ahora. Crea `.github/workflows/ci.yml`:

```yaml
name: CI — GameVault

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar el código fuente
        uses: actions/checkout@v4

      - name: Instalar Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Dar permiso de ejecución al wrapper
        run: chmod +x mvnw

      - name: Ejecutar los tests
        run: ./mvnw test -B -Dtest='*ControllerTest'

      - name: Guardar el informe de tests
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: informe-tests
          path: target/surefire-reports/
```

!!! tip "Fíjate en lo que NO lleva este workflow"
    No hay ningún servicio de PostgreSQL levantado, ni ninguna configuración de base de datos — a diferencia de lo que quizá esperabas. Es la consecuencia directa de lo que ya sabes de la teoría: `@WebMvcTest` mockea el service, así que estos tests nunca tocan una base de datos real. Un futuro test de integración con `@Testcontainers` sí necesitaría un job bastante más complicado.

!!! warning "`./mvnw: Permission denied`"
    Si el workflow falla con este error, `mvnw` ha llegado a GitHub sin el bit de ejecución — algo muy habitual si generaste el proyecto en Windows, donde Git no siempre conserva ese permiso al hacer commit (Windows no tiene el mismo concepto de "permiso de ejecución" que Linux). El paso "Dar permiso de ejecución al wrapper" de arriba (`chmod +x mvnw`) lo arregla sin que dependa de qué permisos traiga el fichero desde tu máquina — es la forma robusta de resolverlo, en vez de intentar arreglar el permiso a mano en tu repositorio.

!!! warning "`GamevaultApplicationTests.contextLoads` falla con `Failed to load ApplicationContext`"
    `mvn test`, sin más, ejecuta **todos** los tests del proyecto — incluido `GamevaultApplicationTests`, el test que Spring Initializr generó automáticamente cuando creaste el proyecto (Actividad 1.1). Ese test arranca la aplicación **completa**, con `DataSource` incluido, y necesita una PostgreSQL real corriendo — algo que este runner no tiene (recuerda el aviso de arriba: los tests MockMvc de hoy no la necesitan, pero ese test de arranque sí). `-Dtest='*ControllerTest'` le dice a Maven que ejecute solo las clases cuyo nombre termine en `ControllerTest` — exactamente `VideojuegoControllerTest` y `EstudioControllerTest`, los tests MockMvc que has escrito hoy, sin tocar el test de arranque completo. Levantar una PostgreSQL real dentro del propio workflow (con un servicio de GitHub Actions) es posible, pero es una pieza más compleja que queda fuera del alcance de esta actividad.

**Pregunta**: tu `pom.xml` de GameVault viene con un *wrapper* (`mvnw`/`mvnw.cmd`), generado automáticamente por Spring Initializr — a diferencia de `validador-dni-ci`, donde no había ninguno. El workflow de arriba usa `./mvnw test`, no `mvn test`. ¿Qué garantiza usar el *wrapper* en vez del `mvn` que esté instalado en el runner de GitHub? Relaciónalo con por qué el Dev Container fija siempre las mismas versiones (Actividad 1.1).

**Predicción**: antes de hacer push, ¿cuántos tests esperas que se ejecuten en total, sumando `VideojuegoControllerTest` y `EstudioControllerTest`?

Haz commit y push. Ve a la pestaña **Actions** de tu repositorio y observa el workflow ejecutarse.

**Captura**: el workflow terminado en verde, con el número de tests ejecutados visible en el log del step "Ejecutar los tests".

### Mini-reto — rómpelo a propósito

Cambia, en cualquiera de tus tests, un valor esperado por uno incorrecto (por ejemplo, en una afirmación `jsonPath`, o un código de estado que no es el que de verdad devuelve tu endpoint). Haz commit y push.

**Captura**: el workflow terminado en rojo, con el test que ha fallado visible en el log.

Corrige el cambio, vuelve a hacer push, y comprueba que el workflow recupera el verde.

---

## Pregunta final

Un test MockMvc como los que has escrito no toca la base de datos en ningún momento — el service está mockeado. ¿Qué es lo que SÍ estás verificando entonces? ¿Qué tipo de fallo detectaría uno de estos tests que un test sobre el service real (sin pasar por HTTP) no detectaría?

---

## ✅ Cierre

Tienes tests automatizados sobre los dos controllers completos de tu API, y un pipeline de CI que los ejecuta solo en cada `push` — si alguien (tú u otro compañero) rompe algo sin darse cuenta, GitHub Actions lo va a decir antes de que llegue a producción. En la próxima actividad cierras el tema: mides cómo tu aplicación atiende varias peticiones a la vez, y activas Actuator para verificar su disponibilidad.
