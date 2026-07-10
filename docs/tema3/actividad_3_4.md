# 🧪 Actividad 3.4: `TaskExecutor` propio

!!! info "Práctica guiada"
    Hoy le das a tu warm-up un pool de hilos propio, con nombre reconocible y prioridad baja, y documentas la decisión.

## Qué vas a practicar

- Configurar un `ThreadPoolTaskExecutor` con parámetros razonados.
- Dirigir `@Async` a un executor concreto por nombre.
- Observar el efecto real de `corePoolSize` con varias tareas seguidas.

---

## Requisitos previos

Tu listener `@Async` del warm-up funcionando (Actividad 3.3).

---

## Paso 1 — El bean, guiado al completo

Crea, en tu paquete `config`, siguiendo el estilo de tu configuración de RabbitMQ:

```java
package com.tunombre.gamevault.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class WarmupExecutorConfig {

    @Bean(name = "warmupExecutor")
    public TaskExecutor warmupExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(20);
        executor.setThreadNamePrefix("warmup-");
        executor.setThreadPriority(Thread.MIN_PRIORITY);
        executor.initialize();
        return executor;
    }
}
```

Cada parámetro tiene un porqué: `corePoolSize(2)` y `maxPoolSize(4)` son deliberadamente pequeños, porque el warm-up es una tarea de fondo ocasional, no el núcleo de tu aplicación; `threadNamePrefix("warmup-")` hace que estos hilos se distingan a simple vista en el log; `setThreadPriority(Thread.MIN_PRIORITY)` los marca como candidatos a ceder el paso ante trabajo más urgente.

---

## Paso 2 — Conectar tu listener a este pool

En `TopNovedadesWarmupListener`, cambia:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async("warmupExecutor")
public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
    // ... tu código ya existente ...
}
```

Reinicia tu aplicación.

---

## Paso 3 — Verificación por el nombre del hilo

Crea un videojuego y mira el log:

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test4","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: que la traza `[WARMUP] Empieza en hilo: ...` ahora muestra un nombre como `warmup-1`, en vez del nombre genérico `task-N` que tenía con el executor por defecto en la Actividad 3.3. **Anota** ambos nombres para comparar.

---

## Paso 4 — Observar con jconsole/thread dump

Repite el procedimiento de la Actividad 3.1 (jconsole, VisualVM, o `jstack`) para localizar los hilos `warmup-*`. **Anota**: cuántos hay activos en este momento, y (si tu herramienta lo muestra) su prioridad configurada.

---

## Mini-reto — el efecto de `corePoolSize`

Crea 3-4 videojuegos seguidos, muy rápido (uno detrás de otro, sin esperar entre peticiones):

```bash
for i in 1 2 3 4; do
  curl -s -X POST http://localhost:8080/api/v1/videojuegos \
    -H "Content-Type: application/json" \
    -d "{\"titulo\":\"Rapido$i\",\"precio\":1,\"fechaLanzamiento\":\"2020-01-01\",\"estudioId\":1}" > /dev/null
done
```

**Observa** en el log cuántos hilos `warmup-*` distintos llegan a aparecer (con `corePoolSize(2)`, no deberías ver muchos más de 2-3 simultáneos, aunque hayas disparado 4 eventos).

Ahora cambia `corePoolSize` a `1` y `maxPoolSize` a `1`, reinicia, y repite el mismo experimento de las 4 peticiones seguidas.

**Describe** la diferencia observada: ¿cuántos hilos `warmup-*` ves esta vez? ¿Qué le pasa a las tareas que no caben de inmediato en ese único hilo (pista: relaciónalo con la `queueCapacity`)? Vuelve a dejar `corePoolSize(2)`/`maxPoolSize(4)` al terminar — es la configuración razonada que documentas a continuación.

---

## Cierre del tema

1. **Documenta tu configuración**: 2-3 frases por parámetro (`corePoolSize`, `maxPoolSize`, `queueCapacity`, prioridad) explicando por qué elegiste esos valores concretos para el warm-up — no "porque sí", con el razonamiento real detrás.
2. **Repaso propio** (4-5 frases) de todo el recorrido de este tema: observar los hilos que ya existían en tu aplicación → construir un evento interno inmutable → un listener asíncrono sincronizado con el commit → un executor propio con nombre y prioridad. ¿Qué pieza te ha costado más entender, y por qué?

---

## ✅ Cierre

En el Tema 4 vuelves a trabajar con hilos, pero esta vez completamente a mano, sin que Spring medie: sockets clásicos primero, WebSocket después.
