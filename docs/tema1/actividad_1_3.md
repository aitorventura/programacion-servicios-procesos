# 🧪 Actividad 1.3: Tests MockMvc sobre `Videojuego` y `Estudio`

!!! info "Práctica guiada — un patrón, dos entidades"
    Escribes tests MockMvc sobre los endpoints de `Videojuego` (guiado al completo), y repites el mismo patrón sobre `Estudio` (con menos ayuda) — las dos entidades ya tienen su CRUD completo desde la Actividad 1.2 de AD, así que hoy no construyes ningún endpoint nuevo, solo los pones a prueba.

## Qué vas a practicar

- Escribir tests MockMvc que prueben un controller de forma aislada.
- Distinguir qué se mockea de qué se ejecuta de verdad en un test de capa web.
- Repetir el mismo patrón de test sobre un segundo controller, con menos guía cada vez.

---

## Requisitos previos

Tu CRUD completo de `Videojuego` y de `Estudio` (Actividad 1.2 de AD) — las cuatro operaciones de cada uno, funcionando.

---

## Bloque 1 — Tests MockMvc sobre `Videojuego`

### Paso 1 — Primer test, guiado al completo

Crea `VideojuegoControllerTest` en `src/test/java`, en el mismo paquete que tu controller:

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

---

## Bloque 2 — Repite el patrón sobre `Estudio`

`EstudioController` tiene las mismas cinco operaciones que `VideojuegoController` (`getAll`, `getById`, `create`, `update`, `delete`), así que `EstudioControllerTest` es el mismo ejercicio otra vez — pero esta vez sin el primer test dado como ejemplo.

**Escribe `EstudioControllerTest`**, mockeando `EstudioService`, con al menos:

- Un test de `getAll` (o `getById`) en el caso de éxito.
- Un test del caso `404` (con el mock lanzando `ResponseStatusException`).
- Un test de `create` (`201`), uno de `update` (`200`) y uno de `delete` (`204`).

**Pregunta antes de escribir nada**: ¿qué cambia realmente entre testear `VideojuegoController` y testear `EstudioController` — la estructura del test, o solo los nombres de clases y campos? Si tu respuesta es "solo los nombres", ya sabes por qué esta actividad no necesitaba darte otro ejemplo completo.

---

## Pregunta final

Un test MockMvc como los que has escrito no toca la base de datos en ningún momento — el service está mockeado. ¿Qué es lo que SÍ estás verificando entonces? ¿Qué tipo de fallo detectaría uno de estos tests que un test sobre el service real (sin pasar por HTTP) no detectaría?

---

## ✅ Cierre

Tienes tests automatizados sobre los dos controllers completos de tu API. En la próxima actividad cierras el tema: mides cómo tu aplicación atiende varias peticiones a la vez, y activas Actuator para verificar su disponibilidad.
