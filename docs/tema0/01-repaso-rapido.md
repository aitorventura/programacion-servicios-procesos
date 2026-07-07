<a id="repaso-rapido"></a>

# 1. Repaso rápido: variables, condicionales y bucles

![Diapositivas](diapositivas/01-repaso-rapido.pdf){ type=application/pdf style="width:100%;min-height:80vh" }

!!!info "Descarga de diapositivas"
    [Descarga las diapositivas](diapositivas/01-repaso-rapido.pptx){target="_blank" rel="noopener"}

Esto ya lo has visto en Programación, así que va a ser un repaso exprés, solo para refrescar la memoria antes de entrar en materia nueva.

---

## 1.1 Variables y tipos

Java es un lenguaje **fuertemente tipado**: cada variable declara de qué tipo es y ese tipo no cambia durante la ejecución. Los tipos primitivos (`int`, `double`, `boolean`, `char`...) guardan el valor directamente, mientras que los tipos referencia (`String`, `ArrayList`, cualquier clase propia) guardan una dirección de memoria que apunta al objeto.

| Categoría | Ejemplos | Guarda |
|---|---|---|
| Primitivo | `int`, `double`, `boolean`, `char` | el valor directamente |
| Referencia | `String`, `Persona`, `ArrayList<T>` | una dirección al objeto |

Dentro de los primitivos, no todos los números son iguales: `int` no es lo mismo que `float`, y no es solo cuestión de gustos. Cada tipo numérico ocupa un tamaño distinto en memoria, y eso limita el rango de valores que puede guardar y si admite decimales o no:

| Tipo | Tamaño | Admite decimales | Cuándo usarlo |
|---|---|---|---|
| `int` | 32 bits | No | el tipo por defecto para enteros; el que más vas a usar |
| `long` | 64 bits | No | cuando un `int` se queda corto (contadores muy grandes, timestamps) |
| `double` | 64 bits | Sí | el tipo por defecto para decimales |
| `float` | 32 bits | Sí | decimales cuando te sobra precisión de `double` y quieres ahorrar memoria |

```java
int cantidad = 25;
long poblacionMundial = 8_100_000_000L; // la "L" indica que es long, no int
double precio = 19.99;
float precioAproximado = 19.99f;        // la "f" indica que es float, no double
```

!!! warning "Cuidado"
    Si escribes `19.99` sin más, Java lo interpreta como `double` por defecto. Para guardarlo en un `float` hace falta el sufijo `f` (`19.99f`); si lo omites, el código no compila porque estarías intentando meter un `double` en una variable más pequeña sin decírselo explícitamente al compilador. Con `long` pasa lo mismo con la `L`, aunque solo hace falta si el número no cabe en un `int` (más de unos 2.147 millones).

El tipo referencia que más vas a usar con diferencia es `String`, así que merece un repaso aparte. Un `String` es **inmutable**: ningún método lo modifica, todos devuelven un `String` nuevo y dejan el original intacto.

| Método | Qué hace | Ejemplo |
|---|---|---|
| `length()` | número de caracteres | `"Hola".length()` → `4` |
| `charAt(i)` | carácter en la posición `i` | `"Hola".charAt(0)` → `'H'` |
| `substring(i, j)` | trozo de texto entre las posiciones `i` y `j` (sin incluir `j`) | `"Hola".substring(1, 3)` → `"ol"` |
| `toUpperCase()` / `toLowerCase()` | pasa a mayúsculas o minúsculas | `"Hola".toUpperCase()` → `"HOLA"` |
| `trim()` | quita los espacios del principio y el final | `"  Hola  ".trim()` → `"Hola"` |
| `contains(texto)` | comprueba si contiene ese texto | `"Hola".contains("ol")` → `true` |
| `split(separador)` | trocea la cadena en un array de `String` | `"a,b,c".split(",")` → `{"a", "b", "c"}` |
| `equals(otro)` | compara el contenido, letra a letra | `"Hola".equals("hola")` → `false` |

!!! tip "Recuerda"
    `"Hola".equals("hola")` da `false` porque `equals()` distingue mayúsculas de minúsculas. Si no te interesa esa diferencia, usa `equalsIgnoreCase()`. Y ojo: para comparar dos `String` usa siempre `equals()`, nunca `==` — el porqué lo ves en el siguiente apartado.

Cuando necesitas guardar varios valores del mismo tipo juntos, usas un **array**: un bloque de memoria de tamaño fijo, declarado con corchetes.

```java
int[] numeros = {2, 4, 6, 8};
System.out.println(numeros[0]); // 2, el primer elemento
```

Esa palabra, "fijo", es la clave: una vez creado un array con 4 huecos, no puede pasar a tener 5. Esa limitación es justo la que resuelven las colecciones, que vemos en el siguiente apartado.

## 1.2 Condicionales

`if`/`else` y `switch` deciden qué bloque de código se ejecuta según una condición booleana. Nada distinto a lo que ya conoces:

```java
int edad = 17;

if (edad >= 18) {
    System.out.println("Es mayor de edad");
} else {
    System.out.println("Es menor de edad");
}
```

Cuando hay más de dos posibilidades, los `if` se anidan: dentro del `else` de uno metes otro `if`/`else` completo, y así sucesivamente. Es habitual escribirlo como una cadena de `else if` para no acumular llaves de más:

```java
int nota = 6;

if (nota >= 9) {
    System.out.println("Sobresaliente");
} else if (nota >= 7) {
    System.out.println("Notable");
} else if (nota >= 5) {
    System.out.println("Suficiente");
} else {
    System.out.println("Insuficiente");
}
```

!!! tip "Recuerda"
    Cada `else if` solo se comprueba si todos los anteriores han sido falsos, y en cuanto uno se cumple, el resto ya no se evalúa. Con `nota = 6`, Java comprueba `nota >= 9` (falso), luego `nota >= 7` (falso), luego `nota >= 5` (cierto) e imprime "Suficiente" sin llegar a mirar el `else` final.

!!! warning "Cuidado"
    Para comparar dos `String` nunca uses `==`: compara si son el mismo objeto en memoria, no si tienen el mismo contenido. Usa siempre `.equals()`, como en la tabla de arriba: `if (nombre.equals("Ana"))`. Con tipos primitivos (`int`, `double`, `boolean`) sí puedes usar `==` sin problema, porque ahí compara el valor directamente. Este fallo va a reaparecer más adelante en el tema, cuando comparemos objetos dentro de colecciones.

## 1.3 Bucles

`for`, `while` y `do-while` repiten un bloque de código mientras se cumpla una condición.

```java
// for clásico: se usa cuando sabes cuántas veces quieres repetir
for (int i = 0; i < 4; i++) {
    System.out.println("Vuelta número " + i);
}

// while: se usa cuando no sabes de antemano cuántas veces va a repetirse
int intentos = 0;
while (intentos < 3) {
    intentos++;
}

// do-while: como el while, pero comprueba la condición al final
int numero;
do {
    numero = leerNumero(); // se ejecuta al menos una vez, pase lo que pase
} while (numero < 0);
```

!!! tip "Recuerda"
    La diferencia entre `while` y `do-while` está en cuándo se comprueba la condición. En un `while`, si la condición ya es falsa la primera vez, el bloque no se ejecuta ni una sola vez. En un `do-while`, el bloque se ejecuta siempre al menos una vez, y solo después se comprueba si hay que repetir. Por eso se usa mucho para pedir datos al usuario: quieres leer el dato como mínimo una vez, y repetir solo si no es válido.

El `for-each` (`for (Tipo elemento : coleccion)`) es el que más vas a usar a partir de ahora para recorrer arrays, listas y colecciones sin manejar índices a mano:

```java
int[] numeros = {2, 4, 6, 8};

for (int n : numeros) {
    System.out.println(n);
}
```

!!! tip "Recuerda"
    A partir de aquí vamos a preferir el `for-each` y, más adelante, los streams frente al `for` clásico con índice: son más legibles y dejan menos margen para errores de índices fuera de rango.
