# 🧪 Actividad 0.2: Colecciones

!!! info "Objetivo"
    Practicar el uso de listas, mapas, pilas, colas y conjuntos.

Ejercicios sueltos e independientes, de menos a más complejo. Usa exactamente los datos que indica cada enunciado.

---

## Ejercicio 1 — Listas

Crea una lista de nombres con estos valores, en este orden: `"Ana"`, `"Luis"`, `"Marta"`, `"Pedro"`, `"Eva"`. Inserta `"Nuevo"` en la posición 2. Elimina el elemento de la posición 0. Recorre la lista con un `for-each` e imprime cada nombre.

??? example "Solución"
    ```java
    List<String> nombres = new ArrayList<>();
    nombres.add("Ana");
    nombres.add("Luis");
    nombres.add("Marta");
    nombres.add("Pedro");
    nombres.add("Eva");

    nombres.add(2, "Nuevo");
    nombres.remove(0);

    for (String nombre : nombres) {
        System.out.println(nombre);
    }
    ```

## Ejercicio 2 — Mapas

Crea un mapa que asocie estos códigos de producto con su precio: `"P1"` → `15.0`, `"P2"` → `25.0`, `"P3"` → `9.5`. Elimina `"P2"`. Recorre las claves con un `for-each` e imprime cada clave junto a su precio.

??? example "Solución"
    ```java
    Map<String, Double> precios = new HashMap<>();
    precios.put("P1", 15.0);
    precios.put("P2", 25.0);
    precios.put("P3", 9.5);

    precios.remove("P2");

    for (String clave : precios.keySet()) {
        System.out.println(clave + " -> " + precios.get(clave));
    }
    ```

## Ejercicio 3 — Pila

Usa un `Deque` como pila. Apila, en este orden: `"primero"`, `"segundo"`, `"tercero"`. Saca dos elementos con `pop()` e imprime cada uno.

??? example "Solución"
    ```java
    Deque<String> pila = new ArrayDeque<>();
    pila.push("primero");
    pila.push("segundo");
    pila.push("tercero");

    System.out.println(pila.pop()); // "tercero"
    System.out.println(pila.pop()); // "segundo"
    ```

## Ejercicio 4 — Cola

Usa una `Queue` (con `LinkedList`). Encola, en este orden: `"primero"`, `"segundo"`, `"tercero"`. Saca dos elementos con `poll()` e imprime cada uno.

??? example "Solución"
    ```java
    Queue<String> cola = new LinkedList<>();
    cola.offer("primero");
    cola.offer("segundo");
    cola.offer("tercero");

    System.out.println(cola.poll()); // "primero"
    System.out.println(cola.poll()); // "segundo"
    ```

## Ejercicio 5 — Conjuntos

Añade a un `HashSet` estos elementos, en este orden: `"Ana"`, `"Luis"`, `"Ana"`, `"Marta"`, `"Pedro"`. Imprime cuántos elementos tiene realmente.

??? example "Solución"
    ```java
    Set<String> unicos = new HashSet<>();
    unicos.add("Ana");
    unicos.add("Luis");
    unicos.add("Ana");
    unicos.add("Marta");
    unicos.add("Pedro");

    System.out.println(unicos.size()); // 4, no 5
    ```

## Ejercicio 6 — Lista + mapa

Crea una lista con estas palabras: `"sol"`, `"luna"`, `"estrella"`, `"cometa"`. Recorre la lista con un `for-each` y, por cada palabra, añade un par a un mapa `Map<String, Integer>` donde la clave sea la palabra y el valor su longitud (`palabra.length()`). Al terminar, imprime el mapa completo.

??? example "Solución"
    ```java
    List<String> palabras = List.of("sol", "luna", "estrella", "cometa");
    Map<String, Integer> longitudes = new HashMap<>();

    for (String palabra : palabras) {
        longitudes.put(palabra, palabra.length());
    }

    System.out.println(longitudes); // {sol=3, luna=4, estrella=8, cometa=7}
    ```

## Ejercicio 7 — TreeMap ordenado

Crea un `TreeMap<String, Integer>` y añade estos pares, en este orden: `"banana"` → `3`, `"manzana"` → `1`, `"cereza"` → `2`. Recórrelo con un `for-each` e imprime las claves. Comprueba que salen en orden alfabético aunque no las hayas añadido en ese orden.

??? example "Solución"
    ```java
    Map<String, Integer> frutas = new TreeMap<>();
    frutas.put("banana", 3);
    frutas.put("manzana", 1);
    frutas.put("cereza", 2);

    for (String clave : frutas.keySet()) {
        System.out.println(clave);
    }
    // banana, cereza, manzana (orden alfabético, no el de inserción)
    ```

## Ejercicio 8 — Invertir una lista con una pila

Dada la lista de números `[1, 2, 3, 4, 5]`, usa una pila (`Deque`) para invertir su orden **sin usar `Collections.reverse()`**: apila los 5 números en el orden en que aparecen, luego sácalos uno a uno con `pop()` y ve añadiéndolos a una lista nueva. Imprime la lista resultante y comprueba que queda `[5, 4, 3, 2, 1]`.

??? example "Solución"
    ```java
    List<Integer> numeros = List.of(1, 2, 3, 4, 5);
    Deque<Integer> pila = new ArrayDeque<>();

    for (int n : numeros) {
        pila.push(n);
    }

    List<Integer> invertida = new ArrayList<>();
    while (!pila.isEmpty()) {
        invertida.add(pila.pop());
    }

    System.out.println(invertida); // [5, 4, 3, 2, 1]
    ```
