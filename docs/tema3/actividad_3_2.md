# 🧪 Actividad 3.2: Cazando hilos en GameVault (análisis, sin código)

!!! info "Práctica guiada de observación — sin código de producción, misma sesión que la Actividad 3.1"
    Continúa directamente donde has dejado la Actividad 3.1: hoy no escribes ninguna funcionalidad nueva. Vas a ver, con tus propios ojos, los hilos reales que acabas de crear con RabbitMQ ejecutándose en tu aplicación — con trazas temporales que retiras al terminar.

## Qué vas a practicar

- Ver una condición de carrera de verdad, no solo leer sobre ella.
- Localizar en logs los distintos hilos que atienden peticiones y eventos.
- Observar el conjunto de hilos de tu JVM con una herramienta de monitorización.

---

## Requisitos previos

Tu GameVault funcionando con RabbitMQ levantado y el consumer de actividad registrando eventos (Actividad 3.1) — el resto de la actividad de hoy depende de tener esos dos hilos reales para observar.

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

!!! warning "jconsole y VisualVM necesitan una ventana gráfica"
    Tu aplicación corre dentro de tu Dev Container, y `jconsole`/`VisualVM` son programas con interfaz gráfica — si los lanzas desde la terminal integrada de VS Code, no van a tener ninguna pantalla donde abrirse. Si quieres usar alguno de los dos igualmente, tienes que instalarlo en tu propio equipo (fuera del contenedor) y conectarlo por red al proceso de dentro, algo que no vas a montar hoy. La opción que **sí** funciona sin nada adicional, directamente desde la terminal del contenedor, es `jstack` — sigue esa pestaña si no quieres complicarte.

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

## Paso 4 — Una capacidad que todavía no existe en tu proyecto

Busca en todo el proyecto (con tu IDE, "Find in Files", o desde la terminal) cualquier uso de la anotación `@Async`:

```bash
grep -r "@Async" src/main/java
```

**Comprueba**: no debería aparecer ningún resultado — es una pieza que Spring ofrece para ejecutar un método en un hilo aparte, pero que tu proyecto todavía no usa en ningún sitio. Escribe, como hipótesis (no hace falta que sea la respuesta exacta), para qué situación de tu propio GameVault crees que serviría esta capacidad — la vas a construir tú mismo en la Actividad 3.4.

---

## Paso 5 — Retirar las trazas

!!! warning "No dejes las trazas temporales en tu código"
    Elimina las dos líneas `System.out.println("[TRAZA] ...")` que has añadido en el Paso 1 — eran solo para esta observación, no forman parte del código final.

---

## ✅ Cierre

Has visto una condición de carrera de verdad y has identificado, con nombres de hilo reales, que tu GameVault ya reparte trabajo entre varios hilos sin que tú lo hayas programado explícitamente. En la próxima actividad empiezas a construir tu propia pieza multihilo: el evento interno del warm-up de caché.
