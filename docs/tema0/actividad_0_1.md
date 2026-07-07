# 🧪 Actividad 0.1: Relaciones entre clases — sistema de biblioteca

!!! info "Objetivo"
    Demostrar que sabes identificar y programar los distintos tipos de relación entre clases: asociación, agregación, composición, herencia, polimorfismo, clases abstractas e interfaces (incluida una interfaz funcional).

## 🔹 Contexto

Vas a modelar una parte del sistema de gestión de una biblioteca. 

---

## 🔹 Parte A. Asociación, agregación y composición

1. Crea las clases `Socio` y `Prestamo`, relacionadas por asociación bidireccional 1-N: un `Socio` puede tener varios `Prestamo`, y cada `Prestamo` conoce a su `Socio`. Implementa un método `addPrestamo(Prestamo p)` en `Socio` que mantenga la relación en los dos sentidos.
2. Crea la clase `Bibliotecario` y añade un atributo en `Biblioteca` que la relacione por **agregación** (un `Bibliotecario` existe aunque se cierre o elimine la `Biblioteca`).
3. Crea la clase `DatosEdicion` (con `isbn`, `editorial` y `anio`) y añade un atributo en `Libro` que la relacione por **composición** (un `DatosEdicion` no tiene sentido si no pertenece a un `Libro` concreto).

**Pregunta de profundización.** Explica, para tu relación de composición, qué pasaría si `DatosEdicion` pudiera existir sin un `Libro` asociado. ¿Por qué en este caso concreto no tiene sentido?

## 🔹 Parte B. Herencia y polimorfismo

1. Crea la superclase `Publicacion` con un atributo `titulo` (`String`) y un método `mostrar()` que devuelva ese título.
2. Crea las subclases `Libro` y `Revista`, que extiendan `Publicacion` y sobrescriban `mostrar()` (con `@Override`) para que cada una añada su propio texto delante del título (por ejemplo, `"Libro: " + super.mostrar()`).
3. Escribe un fragmento de código que declare una variable de tipo `Publicacion` apuntando a un objeto `Libro`, y llame a `mostrar()` (asignación y ejecución polimorfa).

**Pregunta de profundización.** Añade a `Libro` un método propio, `prestar()`, que no exista en `Publicacion` ni en `Revista`. Intenta llamarlo a través de la variable declarada como `Publicacion`, sin hacer casting. ¿Qué error da Java? Explica con tus palabras por qué el compilador no te deja, si en tiempo de ejecución el objeto real sí tiene ese método.

## 🔹 Parte C. Clases abstractas

Convierte `Publicacion` en abstracta:

- El método `calcularDiasPrestamo()` debe quedar **abstracto** (sin cuerpo): `Libro` lo implementa devolviendo `15` y `Revista` devolviendo `7`.
- El método `mostrar()` de la Parte B se queda **concreto** en `Publicacion` (con cuerpo), y cada subclase lo sigue sobrescribiendo como ya tenías.

**Pregunta de profundización.** ¿Qué pasa si intentas escribir `new Publicacion()`? Pruébalo, pega el error exacto que da Java y explica con tus palabras por qué no lo permite.

## 🔹 Parte D. Interfaces

Crea una interfaz `Notificable` con un único método `enviarAviso(String mensaje)`, e implémentala en `Socio` y en `Bibliotecario` — dos clases que **no tienen ninguna relación de herencia entre sí**.

**Pregunta de profundización.** Explica por qué esto no se podría haber conseguido con herencia (recuerda: Java no permite herencia múltiple de clases, pero sí implementar varias interfaces a la vez). ¿Qué superclase común tendrían que compartir `Socio` y `Bibliotecario` para conseguir lo mismo con herencia, y qué problema tendría esa solución?

## 🔹 Parte E. Interfaz funcional: Comparable

Haz que `Libro` implemente `Comparable<Libro>`, ordenando por el atributo `anio` de su `DatosEdicion` (de más antiguo a más reciente).

**Pregunta de profundización.** Crea una lista con al menos 4 `Libro` de años distintos y ordénala con `Collections.sort()`. ¿Qué habría pasado si tu `compareTo` hubiera devuelto siempre `0`? Razónalo antes de probarlo, y luego compruébalo.

---

??? example "Solución"
    ```java
    // Parte A
    public class Socio {
        private String nombre;
        private List<Prestamo> prestamos = new ArrayList<>();

        public void addPrestamo(Prestamo p) {
            prestamos.add(p);
            p.setSocio(this);
        }
    }

    public class Prestamo {
        private Socio socio;
        private Libro libro;

        public void setSocio(Socio socio) { this.socio = socio; }
    }

    public class Bibliotecario {
        private String nombre;
    }

    public class Biblioteca {
        private String nombre;
        private Bibliotecario encargado; // agregación: existe aunque se cierre la biblioteca
    }

    public class DatosEdicion {
        private String isbn;
        private String editorial;
        private int anio;
    }
    ```

    La agregación es `Biblioteca`–`Bibliotecario`: el bibliotecario sigue existiendo (puede trabajar en otra biblioteca) aunque esta se cierre. La composición es `Libro`–`DatosEdicion`: un ISBN y una editorial no tienen sentido guardados sueltos, solo existen como datos de un libro concreto — si se elimina el libro, no tiene sentido conservar sus datos de edición por separado.

    ```java
    // Parte B y C
    public abstract class Publicacion {
        protected String titulo;

        public String mostrar() { // concreto: heredado y sobrescrito
            return titulo;
        }

        public abstract int calcularDiasPrestamo(); // abstracto: cada subclase decide
    }

    public class Libro extends Publicacion {
        private DatosEdicion datosEdicion;

        @Override
        public String mostrar() {
            return "Libro: " + super.mostrar();
        }

        @Override
        public int calcularDiasPrestamo() { return 15; }

        public void prestar() { /* ... */ }
    }

    public class Revista extends Publicacion {
        @Override
        public String mostrar() {
            return "Revista: " + super.mostrar();
        }

        @Override
        public int calcularDiasPrestamo() { return 7; }
    }
    ```

    ```java
    // Asignación y ejecución polimorfa
    Publicacion publicacion = new Libro();
    System.out.println(publicacion.mostrar()); // ejecuta el mostrar() de Libro
    // publicacion.prestar(); // no compila: Publicacion no declara prestar()
    ```

    `publicacion.prestar()` da un error de compilación (`cannot find symbol`) porque el compilador solo conoce los métodos que declara el tipo de la variable (`Publicacion`), no los del objeto real que contiene en tiempo de ejecución. Haría falta un casting: `((Libro) publicacion).prestar()`.

    `new Publicacion()` da el error `Publicacion is abstract; cannot be instantiated`, porque una clase abstracta puede tener métodos sin implementar (`calcularDiasPrestamo()`) y Java no puede garantizar que un objeto de esa clase sepa responder a todos sus métodos si no está completa.

    ```java
    // Parte D
    public interface Notificable {
        void enviarAviso(String mensaje);
    }

    public class Socio implements Notificable {
        @Override
        public void enviarAviso(String mensaje) { /* enviar email al socio */ }
    }

    public class Bibliotecario implements Notificable {
        @Override
        public void enviarAviso(String mensaje) { /* avisar al bibliotecario */ }
    }
    ```

    Para conseguir esto con herencia, `Socio` y `Bibliotecario` tendrían que compartir una superclase común (por ejemplo, `Persona`) que declarase `enviarAviso()`. El problema es que esa jerarquía no tendría mucho sentido: un `Socio` y un `Bibliotecario` no son en realidad el mismo tipo de cosa más que en que ambos reciben avisos, y forzar una superclase solo para compartir ese método mezclaría conceptos que no tienen por qué estar relacionados.

    ```java
    // Parte E
    public class Libro extends Publicacion implements Comparable<Libro> {
        private DatosEdicion datosEdicion;

        @Override
        public int compareTo(Libro o) {
            return this.datosEdicion.getAnio() - o.datosEdicion.getAnio();
        }
    }
    ```

    Si `compareTo` devolviera siempre `0`, `Collections.sort()` consideraría que todos los libros son "iguales" en orden, así que no reordenaría nada: la lista quedaría exactamente en el mismo orden en que estaba antes de ordenarla.
