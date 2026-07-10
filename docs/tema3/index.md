# 🧩 Tema 3: Programación multihilo

> **RA2**: Desarrolla aplicaciones compuestas por varios hilos de ejecución analizando y aplicando librerías específicas del lenguaje de programación.

---

## 🎯 Criterios de evaluación

✅ Se han identificado situaciones en las que resulte útil la utilización de varios hilos en un programa.
✅ Se han reconocido los mecanismos para crear, iniciar y finalizar hilos.
✅ Se han programado aplicaciones que implementen varios hilos.
✅ Se han identificado los posibles estados de ejecución de un hilo y programado aplicaciones que los gestionen.
✅ Se han utilizado mecanismos para compartir información entre varios hilos de un mismo proceso.
✅ Se han desarrollado programas formados por varios hilos sincronizados mediante técnicas específicas.
✅ Se ha establecido y controlado la prioridad de cada uno de los hilos de ejecución.
✅ Se han depurado y documentado los programas desarrollados.
✅ Se ha analizado el contexto de ejecución de los hilos.
✅ Se han analizado librerías específicas del lenguaje de programación que permiten la programación multihilo.
✅ Se han reconocido los problemas derivados de la compartición de información entre los hilos de un mismo proceso.

---

## 📘 Índice de contenidos

1. [Hilos en una aplicación real: dónde ya los estás usando](hilos-en-gamevault.md)
2. [Eventos internos de Spring: el warm-up de caché (1/2)](eventos-warmup-cache.md)
3. [Listeners asíncronos: el warm-up de caché (2/2)](listeners-asincronos.md)
4. [`TaskExecutor` propio y prioridades](taskexecutor-prioridades.md)

**Actividades:**

- [Actividad 3.1 — Cazando hilos en GameVault (análisis, sin código)](actividad_3_1.md)
- [Actividad 3.2 — El evento del warm-up de caché (1/2)](actividad_3_2.md)
- [Actividad 3.3 — El listener `@Async` del warm-up (2/2)](actividad_3_3.md)
- [Actividad 3.4 — `TaskExecutor` propio — cierre de RA2](actividad_3_4.md)

---

!!! info "¿Cómo avanzar por el contenido?"
    Utiliza el índice o las flechas de navegación al final de cada página para desplazarte por los distintos apartados de este tema.

!!! note "Nota para el profesorado"
    Este tema corresponde a las semanas reales 12, 14, 15 y 16 del calendario (RA2; la semana 13 natural es de exámenes y entre la 12 y la 14 median las vacaciones de Navidad). El hilo conductor es el **warm-up de la caché de `topNovedades`**: en la referencia adjunta, `GamevaultApplication.java` declara `@EnableAsync` pero ningún método lo usa todavía, y `VideojuegoService.getTopNovedades()` (`@Cacheable`, con un retardo simulado de 2 s) paga la lentitud en la primera petición tras cada invalidación — la mejora que construye el alumnado (evento interno + listener `@Async` + `TaskExecutor` propio) resuelve exactamente eso. Cada `.md` contiene el prompt para generarlo con `/improve-notes`.
