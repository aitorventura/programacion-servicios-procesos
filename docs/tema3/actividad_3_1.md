# 🧪 Actividad 3.1: Cazando hilos en GameVault (análisis, sin código)

!!! info "Práctica guiada de observación — sin código de producción"
    Hoy no escribes ninguna funcionalidad nueva. Vas a ver, con tus propios ojos, hilos reales ejecutándose en tu aplicación — con trazas temporales que retiras al terminar.

## Qué vas a practicar

- Ver una condición de carrera de verdad, no solo leer sobre ella.
- Localizar en logs los distintos hilos que atienden peticiones y eventos.
- Observar el conjunto de hilos de tu JVM con una herramienta de monitorización.

---

## Requisitos previos

Tu GameVault funcionando con RabbitMQ levantado (el consumer de actividad ya debe estar registrando eventos).

---

## Paso 0 — Calentamiento: ver el entrelazado y la condición de carrera

Copia y ejecuta este ejemplo de consola (no forma parte de tu GameVault, es solo para observar):

```java
public class EntrelazadoDemo {
    public static void main(String[] args) {
        Runnable tarea = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + i);
            }
        };
        new Thread(tarea, "Hilo-A").start();
        new Thread(tarea, "Hilo-B").start();
    }
}
```

Ejecútalo dos veces seguidas. **Anota**: ¿la salida es idéntica en ambas ejecuciones?

Ahora el contador con condición de carrera:

```java
public class ContadorDemo {
    private static int valor = 0;

    public static void main(String[] args) throws InterruptedException {
        Runnable incrementar = () -> {
            for (int i = 0; i < 10000; i++) valor++;
        };

        Thread t1 = new Thread(incrementar);
        Thread t2 = new Thread(incrementar);
        t1.start(); t2.start();
        t1.join(); t2.join();

        System.out.println("Valor final: " + valor);
    }
}
```

Ejecútalo varias veces. **Anota** los valores finales obtenidos — ¿alguna vez da exactamente 20000?

Ahora cambia `valor++` dentro del `Runnable` por una llamada a un método `synchronized`:

```java
private static synchronized void incrementar() { valor++; }
```

(ajusta el `Runnable` para llamar a `incrementar()` en vez de `valor++` directamente). Ejecuta varias veces más. **Anota**: ¿da siempre 20000 ahora?

---

## Paso 1 — Trazas temporales en dos puntos de tu GameVault

Añade, **temporalmente**, esta línea al principio de `VideojuegoService.create()`:

```java
System.out.println("[TRAZA] create() en hilo: " + Thread.currentThread().getName());
```

Y esta otra al principio del método `recibir(...)` de `ActividadVideojuegoEventConsumer`:

```java
System.out.println("[TRAZA] recibir() en hilo: " + Thread.currentThread().getName());
```

---

## Paso 2 — Crear un videojuego y leer los dos hilos

```bash
curl -X POST http://localhost:8080/api/v1/videojuegos \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Test","precio":1,"fechaLanzamiento":"2020-01-01","estudioId":1}'
```

Mira la consola de tu aplicación. **Anota** los dos nombres de hilo que aparecen (deberían ser distintos — algo como `http-nio-8080-exec-N` para uno, y un nombre relacionado con el contenedor de listeners de AMQP para el otro).

**Explica con tus propias palabras**, apoyándote en el diagrama de la teoría, por qué esos dos nombres no coinciden.

---

## Paso 3 — Observar el conjunto de hilos de la JVM

`jconsole`, `VisualVM` y `jstack` son herramientas de inspección de la JVM que vienen incluidas con el propio JDK (o, en el caso de VisualVM, se instalan aparte como complemento) — no hace falta instalar nada especial para usarlas: muestran en tiempo real, o en una foto puntual, qué hilos existen dentro de un proceso Java en ejecución.

Elige **una** de estas herramientas y síguela hasta el final:

<div class="tabs-colored" markdown>

=== "🟢 jconsole"
    Con tu aplicación arrancada, ejecuta `jconsole` desde la terminal, conéctate al proceso de tu aplicación Java, y ve a la pestaña **Threads**. Localiza y anota: cuántos hilos `http-nio-8080-exec-*` ves, si hay algún hilo relacionado con el contenedor de listeners de RabbitMQ, y el hilo `main`.

=== "🔵 VisualVM"
    Arranca VisualVM, selecciona el proceso de tu aplicación en el panel izquierdo, y abre la pestaña **Threads**. Mismo objetivo: localizar y anotar los grupos de hilos mencionados arriba.

=== "🟠 jstack (thread dump)"
    Averigua el PID de tu proceso Java (`jps`, o el gestor de tareas) y ejecuta:
    ```bash
    jstack <PID> > dump.txt
    ```
    Abre `dump.txt` y busca (Ctrl+F) las cadenas `http-nio` y `AMQP` o `SimpleAsyncTaskExecutor`.

</div>

**Anota**: el número total aproximado de hilos que ves en tu JVM en este momento. **Comenta** en una frase por qué ese número es mucho mayor que "1" (que sería lo esperable si pensaras en tu aplicación como un único programa secuencial).

---

## Mini-reto — repite el patrón con el consumer de reviews

Añade la misma traza del Paso 1 (adaptada) al principio de `recibir(...)` en `ReviewsVideojuegoEventConsumer`. Borra un videojuego que tenga alguna reseña asociada:

```bash
curl -X DELETE http://localhost:8080/api/v1/videojuegos/{id}
```

**Comprueba** que la traza de `ReviewsVideojuegoEventConsumer` también se dispara. **Explica**, revisando `RabbitMQConfig.java`, por qué ese listener concreto se activa al borrar (pista: busca la *routing key* `videojuego.eliminado` y a qué cola está enlazada) — y en qué hilo se ejecuta.

---

## Paso 5 — El `@EnableAsync` que no hace nada todavía

Localiza `@EnableAsync` en `GamevaultApplication.java`. Busca en todo el proyecto (con tu IDE, "Find in Files") cualquier uso de la anotación `@Async`:

```bash
grep -r "@Async" src/main/java
```

**Comprueba**: ¿aparece algún resultado? Escribe, como hipótesis (no hace falta que sea la respuesta exacta), para qué crees que servirá esta capacidad sin estrenar — la confirmarás en la Actividad 3.2.

---

## Paso 6 — Retirar las trazas

!!! warning "No dejes las trazas temporales en tu código"
    Elimina las tres líneas `System.out.println("[TRAZA] ...")` que has añadido en los Pasos 1 y 4 — eran solo para esta observación, no forman parte del código final.

---

## ✅ Cierre

Has visto una condición de carrera de verdad y has identificado, con nombres de hilo reales, que tu GameVault ya reparte trabajo entre varios hilos sin que tú lo hayas programado explícitamente. En la próxima actividad empiezas a construir tu propia pieza multihilo: el evento interno del warm-up de caché.
