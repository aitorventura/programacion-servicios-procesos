# 🧪 Actividad 0.3: Los ejercicios de la 0.2, pero en programación funcional

!!! info "Objetivo"
    Practicar `filter`, `map`, `sorted`, `reduce`, `average`, `collect` y `forEach`.

---

## Ejercicio 1 — Filtrar y transformar

Sobre la lista `List.of("Ana", "Luis", "Marta", "Pedro", "Eva")`, usa `filter()` para quedarte solo con los nombres de más de 3 letras, y `map()` para pasarlos a mayúsculas.

??? example "Solución"
    ```java
    List<String> nombres = List.of("Ana", "Luis", "Marta", "Pedro", "Eva");

    List<String> resultado = nombres.stream()
            .filter(n -> n.length() > 3)
            .map(String::toUpperCase)
            .toList();

    System.out.println(resultado); // [LUIS, MARTA, PEDRO]
    ```

## Ejercicio 2 — Ordenar

Sobre la lista `List.of("banana", "manzana", "cereza")`, usa `sorted()` para ordenarla alfabéticamente.

??? example "Solución"
    ```java
    List<String> frutas = List.of("banana", "manzana", "cereza");

    List<String> ordenadas = frutas.stream()
            .sorted()
            .toList();

    System.out.println(ordenadas); // [banana, cereza, manzana]
    ```

## Ejercicio 3 — Reduce

Sobre la lista `List.of(10, 20, 30)`, usa `reduce()` para calcular la suma de sus elementos.

??? example "Solución"
    ```java
    List<Integer> numeros = List.of(10, 20, 30);

    int suma = numeros.stream()
            .reduce(0, Integer::sum);

    System.out.println(suma); // 60
    ```

## Ejercicio 4 — Average

Sobre la misma lista `List.of(10, 20, 30)`, calcula la media con `mapToInt` y `average()`.

??? example "Solución"
    ```java
    List<Integer> numeros = List.of(10, 20, 30);

    double media = numeros.stream()
            .mapToInt(Integer::intValue)
            .average()
            .getAsDouble();

    System.out.println(media); // 20.0
    ```

## Ejercicio 5 — Collect y forEach

Sobre la lista `List.of("Ana", "Luis", "Marta")`, usa `collect(Collectors.joining(", "))` para unir los nombres en un solo texto, y `forEach(System.out::println)` para imprimirlos uno a uno.

??? example "Solución"
    ```java
    List<String> nombres = List.of("Ana", "Luis", "Marta");

    String listado = nombres.stream()
            .collect(Collectors.joining(", "));
    System.out.println(listado); // Ana, Luis, Marta

    nombres.stream().forEach(System.out::println);
    ```

## Ejercicio 6 — Map con streams

Sobre la lista `List.of("sol", "luna", "estrella", "cometa")`, usa `map()` para transformarla en una lista con la longitud de cada palabra.

??? example "Solución"
    ```java
    List<String> palabras = List.of("sol", "luna", "estrella", "cometa");

    List<Integer> longitudes = palabras.stream()
            .map(String::length)
            .toList();

    System.out.println(longitudes); // [3, 4, 8, 7]
    ```

## Ejercicio 7 — Ordenar por un criterio

Sobre la lista `List.of("sol", "luna", "estrella", "cometa")`, usa `sorted()` con un `Comparator` para ordenarla por longitud de palabra, de más corta a más larga.

??? example "Solución"
    ```java
    List<String> palabras = List.of("sol", "luna", "estrella", "cometa");

    List<String> ordenadas = palabras.stream()
            .sorted(Comparator.comparing(String::length))
            .toList();

    System.out.println(ordenadas); // [sol, luna, cometa, estrella]
    ```

## Ejercicio 8 — Invertir el orden con streams

Dada la lista `List.of(1, 2, 3, 4, 5)`, consigue el mismo resultado que en la actividad 0.2 (`[5, 4, 3, 2, 1]`), pero esta vez usando `sorted()` con `Comparator.reverseOrder()` en vez de una pila.

??? example "Solución"
    ```java
    List<Integer> numeros = List.of(1, 2, 3, 4, 5);

    List<Integer> invertida = numeros.stream()
            .sorted(Comparator.reverseOrder())
            .toList();

    System.out.println(invertida); // [5, 4, 3, 2, 1]
    ```
