# 🧪 Actividad 4.1: Cliente y servidor con sockets clásicos

!!! info "Práctica guiada — sin proyecto GameVault en esta actividad"
    Dos clases Java sueltas, fuera de tu GameVault. Al terminar esta actividad programas un servidor y un cliente TCP reales, con sockets Java clásicos (`java.net.Socket`/`ServerSocket`), sin ningún framework de por medio.

## Qué vas a practicar

- Programar un servidor TCP con `ServerSocket` y un cliente con `Socket`.
- Ver la limitación de un servidor monohilo con tus propios ojos.
- Atender varios clientes simultáneos con un hilo por conexión.

---

## Requisitos previos

Ninguno específico de GameVault — solo un IDE con Java.

---

## Paso 1 — El servidor, monohilo, guiado al completo

```java
import java.io.*;
import java.net.*;

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
        try (BufferedReader in = new BufferedReader(new InputStreamReader(cliente.getInputStream()));
             PrintWriter out = new PrintWriter(cliente.getOutputStream(), true)) {

            String linea;
            while ((linea = in.readLine()) != null) {
                System.out.println("Recibido: " + linea);
                out.println("ECO: " + linea);
            }
        } finally {
            cliente.close();
        }
    }
}
```

`new ServerSocket(5000)` hace el *bind* al puerto 5000 y empieza a escuchar. `accept()` **bloquea** hasta que llega un cliente — cuando llega, devuelve un `Socket` para hablar con ese cliente concreto. `BufferedReader`/`PrintWriter` sobre los streams del socket son la forma de leer y escribir línea a línea, igual que ya conoces de la entrada/salida estándar.

**Fíjate en el problema**: mientras `atenderCliente(cliente)` está ocupado (dentro del `while` leyendo líneas de ese cliente), el bucle principal no vuelve a llamar a `accept()` — nadie más puede conectarse hasta que ese cliente termine.

---

## Paso 2 — El cliente

```java
import java.io.*;
import java.net.*;

public class ClienteEco {
    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket("localhost", 5000);
             BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader teclado = new BufferedReader(new InputStreamReader(System.in))) {

            String linea;
            while ((linea = teclado.readLine()) != null) {
                out.println(linea);
                System.out.println("Respuesta: " + in.readLine());
            }
        }
    }
}
```

`new Socket("localhost", 5000)` hace el *connect* hacia el servidor. Arranca `ServidorEco` en una terminal, y `ClienteEco` en otra: escribe líneas y observa el eco.

---

## Paso 3 — Ver la limitación monohilo

Con el servidor y un cliente ya conectados (el cliente esperando en el bucle, sin cerrar), abre una **tercera** terminal y arranca un segundo `ClienteEco`.

**Observa**: el segundo cliente se queda esperando, sin conectar — porque el servidor sigue "atrapado" atendiendo al primero dentro de `atenderCliente(...)`, y no ha vuelto a llamar a `accept()`.

---

## Paso 4 — La versión multihilo, guiada

Modifica el servidor para lanzar un hilo por cada conexión aceptada:

```java
import java.io.*;
import java.net.*;
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
        try (BufferedReader in = new BufferedReader(new InputStreamReader(cliente.getInputStream()));
             PrintWriter out = new PrintWriter(cliente.getOutputStream(), true)) {

            String linea;
            while ((linea = in.readLine()) != null) {
                out.println("ECO: " + linea);
            }
        } catch (IOException e) {
            System.out.println("Cliente desconectado: " + e.getMessage());
        } finally {
            try { cliente.close(); } catch (IOException ignored) {}
        }
    }
}
```

Un `ExecutorService` (el pool de hilos que ya conoces del Tema 3) atiende cada conexión en su propio hilo, en vez de bloquear el bucle principal. Repite la prueba del Paso 3: arranca `ServidorEcoMultihilo`, conecta dos clientes a la vez, y comprueba que **ambos** funcionan simultáneamente — cada uno con su propia traza de hilo en la consola del servidor.

---

## Mini-reto — comando `SALIR`

Amplía el protocolo del eco: si el cliente envía la línea exacta `SALIR`, el servidor debe cerrar limpiamente esa conexión concreta (cerrar los streams y el socket de ese cliente, sin afectar a los demás clientes conectados) en vez de seguir haciendo eco. Solo se indica el objetivo — repite el patrón ya usado en `atenderClienteConTraza`.

---

## Pregunta de comprensión

¿Qué relación hay entre este "un hilo por cliente" que acabas de programar a mano y lo que hace Tomcat con las peticiones HTTP de tu GameVault? ¿Y con el pool de hilos configurado que construiste en el Tema 3 (`ThreadPoolTaskExecutor`)? Explica con tus palabras por qué es, en el fondo, el mismo patrón — gestionado a mano aquí, y por el framework allí.

---

## ✅ Cierre

Has programado un servidor y un cliente TCP reales, y resuelto a mano el mismo problema que Tomcat resuelve por ti en cada petición HTTP. Las próximas actividades amplían este mismo tema con WebSocket, sobre un caso de aplicación real de tu GameVault.
