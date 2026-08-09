# 🧪 Actividad 3.4: `TaskExecutor` propio

!!! warning "Descarga la plantilla"
    📄 [Plantilla 3.4 — TaskExecutor propio](plantillas/Actividad_3_4_PSP_Plantilla.docx){target="_blank" rel="noopener"}

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

## Paso 1 — El bean, con tu propia configuración

Crea, en tu paquete `config`, siguiendo el estilo de tu configuración de RabbitMQ, un `TaskExecutor` propio para el warm-up con esta configuración:

- Tamaño base de **2 hilos** (`corePoolSize`): el pool crea hilos a medida que llegan tareas hasta ese número.
- El pool puede crecer hasta **3 hilos** si hace falta, nunca más (`maxPoolSize`).
- Cola para **15 tareas** esperando antes de que el pool necesite crecer (`queueCapacity`).
- Nombre de hilo reconocible, con el prefijo `"warmup-"` (`threadNamePrefix`).
- Prioridad baja (`Thread.MIN_PRIORITY`) — la marcamos como trabajo de fondo; es una pista al planificador, no una garantía de que vaya a ceder CPU a otras tareas.

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
        // Configura aquí los cinco parámetros de la lista de arriba

        executor.initialize();
        return executor;
    }
}
```

Fíjate en que los valores no son los mismos que ves en la teoría — ahí tienes el mismo patrón con otros números, no una plantilla para copiar tal cual.

---

## Paso 2 — Conectar tu listener a este pool

En `TopNovedadesWarmupListener`, dirige `@Async` a tu pool nuevo. Aprovecha también para añadir, **temporalmente**, la traza que quitaste al terminar la Actividad 3.3 — hoy la necesitas otra vez, para comprobar el nombre del hilo en el Paso 3:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async("warmupExecutor")
public void onTopNovedadesInvalidado(TopNovedadesInvalidadoEvent event) {
    System.out.println("[WARMUP] Empieza en hilo: " + Thread.currentThread().getName());
    videojuegoService.getTopNovedades(); // recalienta la caché
}
```

Reinicia tu aplicación.

---

## Paso 3 — Verificación por el nombre del hilo

!!! note "Token de administrador"
    Esta actividad vuelve a usar `ADMIN_TOKEN`. Si has abierto una terminal nueva o el token anterior ha caducado, vuelve a iniciar sesión como `admin` y guarda el nuevo `accessToken` en esa variable antes de continuar.

Crea un videojuego y mira el log:

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"titulo":"Test4","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

**Comprueba**: que la traza `[WARMUP] Empieza en hilo: ...` ahora muestra un nombre como `warmup-1`, en vez del nombre genérico `task-N` que tenía con el executor por defecto en la Actividad 3.3.

**Captura**: la traza `[WARMUP] Empieza en hilo: warmup-...`.

**Anota** ambos nombres (el de hoy y el `task-N` de la Actividad 3.3) para comparar.

---

## Paso 4 — El efecto de `corePoolSize`

Crea 3-4 videojuegos seguidos, muy rápido (uno detrás de otro, sin esperar entre peticiones):

```bash
for i in 1 2 3 4; do
  curl -s -X POST http://localhost:8080/api/v1/videojuegos \
    -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
    -d "{\"titulo\":\"Rapido$i\",\"precio\":1,\"fechaLanzamiento\":\"2020-01-01\",\"estudioId\":1}" > /dev/null
done
```

**Observa** en el log cuántos hilos `warmup-*` distintos llegan a aparecer: con `corePoolSize(2)` y una `queueCapacity(15)` tan holgada, no deberías ver más de **2** hilos distintos, aunque hayas disparado 4 eventos — el pool solo crece por encima de `corePoolSize` cuando la cola se llena, y con sitio de sobra para 15 tareas, eso no llega a pasar con solo 4.

**Captura**: el log mostrando los hilos `warmup-*` de esta primera prueba, con `corePoolSize(2)`.

Ahora cambia `corePoolSize` a `1` y `maxPoolSize` a `1`, reinicia, y repite el mismo experimento de las 4 peticiones seguidas.

**Captura**: el log mostrando los hilos `warmup-*` de esta segunda prueba, con `corePoolSize(1)`/`maxPoolSize(1)`.

**Describe** la diferencia observada: ¿cuántos hilos `warmup-*` ves esta vez? ¿Qué le pasa a las tareas que no caben de inmediato en ese único hilo (pista: relaciónalo con la `queueCapacity`)?

Vuelve a dejar tu configuración original (`corePoolSize(2)`/`maxPoolSize(3)`/`queueCapacity(15)`) al terminar.

Ya has comprobado lo que necesitabas del nombre del hilo — **retira otra vez** la traza `[WARMUP] Empieza en hilo: ...` de `TopNovedadesWarmupListener`, igual que hiciste al terminar la Actividad 3.3.
