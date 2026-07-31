# 🧪 Actividad 4.1: Cliente y servidor con sockets clásicos

!!! warning "Descarga la plantilla"
    📄 [Plantilla 4.1 — Cliente y servidor con sockets clásicos](plantillas/Actividad_4_1_PSP_Plantilla.docx){target="_blank" rel="noopener"}

!!! info "Práctica guiada — sin proyecto GameVault en esta actividad"
    Un proyecto Java nuevo y suelto, fuera de tu GameVault. Al terminar esta actividad tienes un servidor y un cliente TCP reales, con sockets Java clásicos (`java.net.Socket`/`ServerSocket`), sin ningún framework de por medio.

## Qué vas a practicar

- Entender un servidor TCP con `ServerSocket` y un cliente con `Socket`, código en mano.
- Ver la limitación de un servidor monohilo con tus propios ojos, y entender por qué ocurre.
- Atender varios clientes simultáneos con un hilo por conexión.
- Relacionar, con la teoría ya leída, por qué el mismo patrón de *streams* sirve igual en el cliente que en el servidor.

---

## Requisitos previos

Ninguno específico de GameVault — solo un JDK 17+ instalado y un IDE con el que puedas ejecutar clases Java sueltas.

---

## Antes de empezar — un proyecto nuevo, sin Dev Container

**¿Hace falta el Dev Container de GameVault?** No. Ese Dev Container existe para darte PostgreSQL, MongoDB, RabbitMQ y las versiones fijas de Java/Maven que **ese** proyecto necesita. Esta actividad no toca ninguna base de datos ni ningún framework — solo `java.net`/`java.io` de la propia JDK. Te basta con el Java que ya tengas instalado en tu máquina, fuera de cualquier contenedor, en un proyecto nuevo y vacío.

Crea un proyecto Java simple (sin Maven, sin Spring, sin nada — en IntelliJ: *File → New → Project → Java*, sin marcar ningún framework) con esta estructura de paquetes, y ve creando cada clase en el paso correspondiente:

```
sockets-actividad-4-1/
└── src/
    └── com/tunombre/sockets/
        ├── ConexionEco.java
        ├── ServidorEco.java
        ├── ClienteEco.java
        └── ServidorEcoMultihilo.java
```

---

## Paso 1 — Un helper para leer y escribir

Como ya sabes de la teoría de este apartado, una vez la conexión existe, cada lado —cliente y servidor— tiene su propio `Socket`, y ambos lo usan exactamente igual: streams de entrada y salida para leer y escribir línea a línea. Aquí tienes esa idea ya escrita en una clase, que vas a usar en el resto de la actividad, tanto en el servidor como en el cliente:

```java
package com.tunombre.sockets;

import java.io.*;
import java.net.Socket;

public class ConexionEco implements Closeable {
    private final Socket socket;
    private final BufferedReader in;
    private final PrintWriter out;

    public ConexionEco(Socket socket) throws IOException {
        this.socket = socket;
        this.in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        this.out = new PrintWriter(socket.getOutputStream(), true);
    }

    public String recibir() throws IOException {
        return in.readLine();
    }

    public void enviar(String mensaje) {
        out.println(mensaje);
    }

    @Override
    public void close() throws IOException {
        socket.close();
    }
}
```

El constructor recibe un `Socket` ya conectado, sin preguntar de dónde viene — le da igual si es el que ha devuelto `accept()` en el servidor, o el que ha devuelto `new Socket(...)` en el cliente. Por debajo sigue siendo exactamente lo que ya conoces: un `BufferedReader` y un `PrintWriter` envolviendo los *streams* del socket; `recibir()` y `enviar(...)` son solo nombres propios para `in.readLine()` y `out.println(...)`. Al implementar `Closeable`, además, puedes usarla directamente en un *try-with-resources*.

---

## Paso 2 — El servidor, monohilo

```java
package com.tunombre.sockets;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class ServidorEco {
    public static void main(String[] args) throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(5000)) {
            System.out.println("Servidor escuchando en el puerto 5000...");

            while (true) {
                Socket cliente = serverSocket.accept();
                System.out.println("Cliente conectado: " + cliente.getInetAddress());
                atenderCliente(cliente);
            }
        }
    }

    private static void atenderCliente(Socket cliente) throws IOException {
        try (ConexionEco conexion = new ConexionEco(cliente)) {
            String linea;
            while ((linea = conexion.recibir()) != null) {
                System.out.println("Recibido: " + linea);
                conexion.enviar("ECO: " + linea);
            }
        }
    }
}
```

`new ServerSocket(5000)` hace el *bind* al puerto 5000 y empieza a escuchar. `accept()` **bloquea** hasta que llega un cliente — cuando llega, devuelve un `Socket` para hablar con ese cliente concreto, que envuelves en tu `ConexionEco`. Al ser un *try-with-resources*, la conexión se cierra sola en cuanto el `while` termina (el cliente se desconecta), sin que tengas que acordarte de cerrarla a mano.

**Fíjate en el problema**: mientras `atenderCliente(cliente)` está ocupado (dentro del `while` leyendo líneas de ese cliente), el bucle principal no vuelve a llamar a `accept()` — nadie más puede conectarse hasta que ese cliente termine.

**Responde**:

1. El `while ((linea = conexion.recibir()) != null)` es el que mantiene viva la conversación con un cliente. ¿En qué momento exacto devuelve `null` — es decir, qué tiene que hacer el cliente para que ese `while` termine?
2. El `while (true)` de `main` es distinto: no tiene ninguna condición de salida. ¿Por qué tiene sentido que sea así en un servidor, y no, por ejemplo, un bucle que acepte solo 5 conexiones y termine? ¿Qué le pasaría a tu servidor si ese `while` sí tuviera un final?

Ejecuta `ServidorEco` ahora mismo (déjalo corriendo) antes de pasar al siguiente paso.

---

## Paso 3 — El cliente

```java
package com.tunombre.sockets;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.Socket;

public class ClienteEco {
    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket("localhost", 5000);
             ConexionEco conexion = new ConexionEco(socket);
             BufferedReader teclado = new BufferedReader(new InputStreamReader(System.in))) {

            String linea;
            while ((linea = teclado.readLine()) != null) {
                conexion.enviar(linea);
                System.out.println("Respuesta: " + conexion.recibir());
            }
        }
    }
}
```

`new Socket("localhost", 5000)` hace el *connect* hacia el servidor, y ese mismo `Socket` es el que envuelve tu `ConexionEco` — el mismo helper, el mismo constructor, sin ningún cambio respecto al servidor. Ejecuta `ClienteEco` (con `ServidorEco` del Paso 2 ya corriendo) y escribe un par de líneas.

**Captura**: las dos ejecuciones una junto a otra — la consola de `ServidorEco` mostrando `Recibido: ...`, y la de `ClienteEco` mostrando `Respuesta: ECO: ...`, después de haber escrito al menos una línea.

**Responde**: acabas de usar `ConexionEco` en el servidor (Paso 2) y en el cliente (Paso 3), sin tocar ni una línea de la clase. ¿Por qué el mismo código sirve igual para los dos lados? Relaciónalo con la teoría de este apartado — una vez la conexión existe, ¿en qué se diferencian de verdad el `Socket` del cliente y el que ha devuelto `accept()` en el servidor?

---

## Paso 4 — Ver la limitación monohilo

!!! warning "Ejecutar un segundo cliente a la vez, no reiniciar el primero"
    Desde aquí en adelante vas a necesitar **dos** `ClienteEco` corriendo al mismo tiempo. Volver a pulsar *Run* sobre la misma clase, por defecto, no lanza una segunda instancia — **reinicia** la que ya tenías corriendo, cerrando su conexión sin que te des cuenta. En IntelliJ: **Run → Edit Configurations**, selecciona la configuración de `ClienteEco`, y en el desplegable **"Modify options"** activa **"Allow multiple instances"** (el nombre exacto varía un poco entre versiones, pero es esa opción). En cuanto la actives aparece como una casilla marcada en la propia configuración, y a partir de ahí cada *Run* abre una pestaña nueva e independiente, sin tocar la anterior. Si tu IDE no tiene esa opción, la alternativa segura es compilar una vez y ejecutar `java com.tunombre.sockets.ClienteEco` a mano, en dos terminales distintas.

Con `ServidorEco` y un `ClienteEco` ya conectados (el cliente esperando en el bucle, sin cerrar), ejecuta un **segundo** `ClienteEco` en paralelo, como acabas de ver arriba.

**Observa**: el segundo cliente se queda esperando, sin conectar — porque el servidor sigue "atrapado" atendiendo al primero dentro de `atenderCliente(...)`, y no ha vuelto a llamar a `accept()`.

**Captura**: la ejecución del segundo `ClienteEco`, sin ninguna respuesta, mientras el primero sigue conectado.

---

## Paso 5 — La versión multihilo

```java
package com.tunombre.sockets;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ServidorEcoMultihilo {
    public static void main(String[] args) throws IOException {
        ExecutorService pool = Executors.newCachedThreadPool();

        try (ServerSocket serverSocket = new ServerSocket(5000)) {
            System.out.println("Servidor escuchando en el puerto 5000...");

            while (true) {
                Socket cliente = serverSocket.accept();
                System.out.println("Cliente conectado: " + cliente.getInetAddress());
                pool.submit(() -> atenderClienteConTraza(cliente));
            }
        }
    }

    private static void atenderClienteConTraza(Socket cliente) {
        System.out.println("Atendiendo en hilo: " + Thread.currentThread().getName());
        try (ConexionEco conexion = new ConexionEco(cliente)) {
            String linea;
            while ((linea = conexion.recibir()) != null) {
                conexion.enviar("ECO: " + linea);
            }
        } catch (IOException e) {
            System.out.println("Cliente desconectado: " + e.getMessage());
        }
    }
}
```

Un `ExecutorService` (el pool de hilos que ya conoces del Tema 3) atiende cada conexión en su propio hilo, en vez de bloquear el bucle principal — y como `ConexionEco` sigue siendo *try-with-resources*, cada hilo cierra su propia conexión al terminar, sin pisarse con los demás.

`Executors.newCachedThreadPool()` es un tipo concreto de `ExecutorService` que no habías visto en código todavía. En el Tema 3 lo pensaste como "una plantilla fija de operarios" — pero fija es solo una de las formas posibles de montar un pool, no la única. `newCachedThreadPool()` no tiene un tamaño fijo: crea un hilo nuevo cada vez que hace falta uno y no hay ninguno libre, reutiliza los que quedan libres, y cierra los que llevan un rato sin trabajo. Encaja bien aquí porque no sabes de antemano cuántos clientes se van a conectar a la vez — a diferencia de la cola de tareas de warm-up del Tema 3, donde un número fijo de operarios ya bastaba.

Cierra `ServidorEco` (Paso 2) y arranca `ServidorEcoMultihilo` en su lugar (mismo puerto 5000). Repite la prueba del Paso 4: conecta dos `ClienteEco` a la vez, y comprueba que **ambos** funcionan simultáneamente.

**Captura**: la consola de `ServidorEcoMultihilo` mostrando dos trazas `Atendiendo en hilo: ...` con nombres de hilo distintos, y los dos `ClienteEco` recibiendo eco a la vez.

**Responde**: fíjate en `pool.submit(() -> atenderClienteConTraza(cliente))` — la variable `cliente` se declara dentro del `while (true)`, una vez por cada vuelta, y se usa dentro de una lambda que no se ejecuta necesariamente de inmediato. Si dos clientes se conectan casi a la vez, ¿podría el hilo que atiende al primero acabar usando, por error, el `Socket` del segundo? Explica por qué sí o por qué no, en función de dónde está declarada la variable `cliente`.

---

## Paso 6 — Comando `SALIR`

Amplía el protocolo: si el cliente envía la línea exacta `SALIR`, el servidor cierra esa conexión concreta en vez de seguir haciendo eco. Sustituye el `while` de `atenderClienteConTraza` por este:

```java
String linea;
while ((linea = conexion.recibir()) != null) {
    if (linea.equals("SALIR")) {
        break;
    }
    conexion.enviar("ECO: " + linea);
}
```

Con dos `ClienteEco` conectados a la vez, escribe `SALIR` en uno de ellos.

**Captura**: el cliente que ha enviado `SALIR` desconectado (o a punto de cerrarse), mientras el otro cliente sigue funcionando con normalidad, sin verse afectado.

**Responde**:

1. ¿Por qué cerrar esta `conexion` no afecta a los demás clientes que puedan estar conectados a la vez? Relaciónalo con lo que sabes sobre `ServerSocket` y los `Socket` nuevos que crea cada `accept()`.
2. Este `while` sustituye al que ya tenías dentro de `atenderClienteConTraza`. ¿Por qué no hace falta añadir ningún cierre explícito para que `conexion.close()` se ejecute al salir del `while` con el `break`?

---

## Pregunta de comprensión

¿Qué relación hay entre este "un hilo por cliente" que acabas de ver y lo que hace Tomcat con las peticiones HTTP de tu GameVault? ¿Y con el pool de hilos configurado que construiste en el Tema 3 (`ThreadPoolTaskExecutor`)? Explica con tus palabras por qué es, en el fondo, el mismo patrón — gestionado a mano aquí, y por el framework allí.

---

## ✅ Cierre

Has visto un servidor y un cliente TCP reales funcionando, y el mismo problema que Tomcat resuelve por ti en cada petición HTTP resuelto a mano, con hilos. Las próximas actividades amplían este mismo tema con WebSocket, sobre un caso de aplicación real de tu GameVault.
