# 🧪 Actividad 1.3: Tests MockMvc y el PUT de Estudio

!!! info "Práctica guiada"
    Dos bloques hoy: primero escribes tests MockMvc sobre los endpoints de `Videojuego` que ya tienes; después añades el `PUT` de `Estudio`, que todavía no existe en tu proyecto.

## Qué vas a practicar

- Escribir tests MockMvc que prueben un controller de forma aislada.
- Distinguir qué se mockea de qué se ejecuta de verdad en un test de capa web.
- Añadir una operación nueva (`PUT`) a un controller existente, siguiendo un patrón ya conocido.
- Comprobar experimentalmente la atención simultánea de varios clientes.

---

## Requisitos previos

Tu CRUD de `Videojuego` (Actividad 1.2 de AD) y el `GET`/`POST` de `Estudio` ya funcionando en tu proyecto.

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

### Mini-reto — test del POST

Repite el mismo patrón (preparar el mock con `when(...).thenReturn(...)`, actuar con `mockMvc.perform(post(...).contentType(...).content(...))`, afirmar con `.andExpect(status().isCreated())`) para probar que un `POST /api/v1/videojuegos` válido devuelve `201`. Guíate por el test de `create` que ya viste en la teoría para el cuerpo JSON de la petición.

---

## Bloque 2 — El PUT de Estudio

### Paso 3 — Por comparación con el PUT de Videojuego

Ya construiste el `PUT` de `Videojuego` en Acceso a Datos (Actividad 1.2). Tenlo a la vista mientras haces esto — el de `Estudio` es el mismo patrón, adaptado:

```java
// Lo que ya tienes en VideojuegoService (tenlo delante como patrón)
@Transactional
public VideojuegoResponseDTO update(Long id, VideojuegoCreateDTO dto) {
    Videojuego v = videojuegoRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Videojuego no encontrado"));
    v.setTitulo(dto.titulo());
    // ...
    return mapToDTO(videojuegoRepository.save(v));
}
```

Adáptalo a `EstudioService`, teniendo en cuenta lo que cambia:

- El DTO es `EstudioDTO` (no hace falta un `EstudioCreateDTO` separado — reutiliza el mismo para crear y actualizar).
- El "no encontrado" se comprueba igual, con `ResponseStatusException(HttpStatus.NOT_FOUND, ...)`.
- `@Transactional` en el método de escritura, igual que en `Videojuego`.

Escribe tú el método `update` en `EstudioService` y el endpoint `@PutMapping("/{id}")` correspondiente en `EstudioController`, usando el fragmento de arriba como plantilla de qué cambiar y qué no.

**Pregunta antes de continuar**: ¿qué parte del patrón de `Videojuego` NO tiene equivalente en `Estudio` (pista: `Videojuego.update` también busca y valida un `Estudio` relacionado — `Estudio` no tiene ninguna relación de ese tipo que validar)?

### Paso 4 — Verificar el PUT nuevo

Primero a mano, desde Swagger UI (Actividad 1.2): despliega el nuevo `PUT /api/v1/estudios/{id}`, pruébalo con un cambio de `nombre` o `pais`, y comprueba el `200`.

Después, con un test MockMvc — repite el mismo patrón del Bloque 1 (mockear el service, `mockMvc.perform(put(...))`, afirmar el `200` y el cuerpo actualizado). Solo se indica el objetivo: escríbelo tú.

---

## Bloque 3 — Comunicación simultánea

Repite el experimento de la teoría con tu propio proyecto:

```bash
time (curl -s http://localhost:8080/api/v1/videojuegos/top & \
      curl -s http://localhost:8080/api/v1/videojuegos/top & \
      wait)
```

**Anota** el tiempo total mostrado por `time`. Añade temporalmente una línea `System.out.println(Thread.currentThread().getName())` al principio de `getTopNovedades()` en tu service, repite la prueba, y anota los dos nombres de hilo que aparecen en la consola. Cuando termines, retira esa línea — era solo para observar, no para quedarse en el código.

---

## Pregunta final

¿Qué relación hay entre este "una petición, un hilo" que acabas de observar y lo que has hecho al lanzar dos `curl` a la vez? Si en vez de dos lanzaras cincuenta peticiones simultáneas, ¿crees que el comportamiento sería exactamente el mismo, o hay algún límite? (No hace falta que sepas la respuesta exacta todavía — se trabaja a fondo en el Tema 3).

---

## ✅ Cierre

Tienes tests automatizados sobre tu API y el `PUT` de `Estudio` ya funcionando. La semana que viene cierras el CRUD de `Estudio` con el `DELETE` y activas Actuator para verificar la disponibilidad del servicio.
