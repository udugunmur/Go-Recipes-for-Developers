# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 2: Trabajo con Cadenas de Texto (*Strings*)

El tipo *string* es uno de los tipos de datos fundamentales en Go.

Go utiliza *strings* inmutables codificados en **UTF-8**. Esto puede resultar confuso para un desarrollador nuevo; después de todo, esto funciona:

```go
x := "Hello"
x += " World"
fmt.Println(x) // Prints Hello World
```

¿No acabamos de cambiar `x`? Sí, lo hicimos. Lo que es inmutable aquí son los *strings* `"Hello"` y `" World"`. Por lo tanto, el *string* en sí mismo es inmutable, pero la variable de tipo *string*, `x`, es mutable. Para modificar variables de tipo *string*, creas *slices* de bytes o de *runes* (que sí son mutables), trabajas con ellos y luego los vuelves a convertir en un *string*.

UTF-8 es la codificación más común utilizada en tecnologías web e Internet. Esto significa que cada vez que manejas texto en un programa Go, estás tratando con *strings* UTF-8. Si necesitas procesar datos en una codificación diferente, primero los conviertes a UTF-8, los procesas y los vuelves a codificar a su codificación original.

UTF-8 es una codificación de longitud variable que utiliza de uno a cuatro bytes para cada punto de código (*codepoint*). La mayoría de los puntos de código representan un carácter, pero hay algunos que representan otra información, como formato. Esto puede causar algunas sorpresas. Por ejemplo, la longitud de un *string* (es decir, el número de bytes que ocupa) es diferente del número de caracteres. Hallar el número de caracteres en un *string* requiere contarlos secuencialmente. Cuando realizas un *slice* de un *string*, debes tener cuidado con los límites de los puntos de código.

Go utiliza el tipo `rune` para denotar puntos de código. Por lo tanto, un *string* puede verse tanto como una secuencia de bytes como una secuencia de *runes*. Esto se ilustra en la Figura 2.1. Aquí, `x` es una variable de tipo *string* que tiene un puntero al *string* inmutable, que es una secuencia de bytes y también puede verse como una secuencia de *runes*. Aunque UTF-8 es una codificación de longitud variable, `rune` es un tipo de longitud fija de 32 bits (`uint32`). Los puntos de código más pequeños, como el carácter `H`, son un decimal de 32 bits, `72`, mientras que el byte `H` es un valor de 8 bits.

#### Figura 2.1 – Un string, byte y rune

En este capítulo, examinaremos algunas operaciones comunes que involucran *strings* y texto. Las recetas incluidas en este capítulo son las siguientes:
- Creación de *strings*
- Formateo de *strings*
- Combinación de *strings*
- Comparaciones en mayúsculas, minúsculas y formato título (*title case*)
- Manejo de *strings* internacionalizados
- Trabajo con codificaciones (*encodings*)
- Iteración de bytes y *runes* en *strings*
- División (*splitting*)
- Expresiones regulares
- Lectura de *strings* línea por línea o palabra por palabra
- Recorte (*trimming*)
- Plantillas (*templates*)

---

### Sección 1: Creación de *Strings*

En esta receta, veremos cómo crear *strings* en un programa.

#### Cómo hacerlo...

Usa un literal de cadena (*string literal*). Hay dos tipos de literales de cadena en Go:

1. Utiliza literales de cadena interpretados, entre comillas dobles:

```go
x := "Hello world"
```

Con los literales de cadena interpretados, debes escapar ciertos caracteres:

```go
x := "This is how you can include a \" in your string literal"
y := "You can also use a newline \n, tab \t"
```

Puedes incluir puntos de código Unicode o bytes hexadecimales, escapados con `\`:

```go
w := "\u65e5本\U00008a9e"
x := "\xff"
```

No puedes incluir saltos de línea ni comillas dobles sin escapar en un *string* interpretado.

2. Utiliza literales de cadena sin procesar (*raw string literals*), usando comillas invertidas (*backticks*). Un literal sin procesar puede incluir cualquier carácter (incluidos saltos de línea) excepto una comilla invertida. No hay forma de escapar comillas invertidas en un literal sin procesar:

```go
x := `This is a multiline raw string literal. Backslash will print as backslash \`
```

Si necesitas incluir una comilla invertida en tu literal sin procesar, haz lo siguiente:

```go
x := `This is a raw string literal with ` + "`" + ` in it`
```

---

### Sección 2: Formateo de *Strings*

La biblioteca estándar de Go ofrece múltiples formas de sustituir valores en una plantilla de texto. Aquí analizaremos las utilidades de formato de texto en el paquete `fmt`. Ofrecen una forma simple y conveniente de sustituir valores en una plantilla de texto.

#### Cómo hacerlo...

- Utiliza la familia de funciones `fmt.Print` para formatear valores.
- `fmt.Print` imprimirá un valor utilizando su formato predeterminado.
- Un valor de tipo *string* se imprimirá tal cual.
- Un valor numérico se convertirá primero en un *string* como un entero, un número decimal o utilizando notación científica para exponentes grandes.
- Un valor booleano se imprimirá como `true` o `false`.
- Los valores estructurados se imprimirán como una lista de campos.
- Si una función `Print` termina con `ln` (como `fmt.Println`), se generará un salto de línea después del *string*.
- Si una función `Print` termina con `f`, la función aceptará un argumento de formato, que se utilizará como la plantilla en la que se sustituirán los valores.
- `fmt.Sprintf` formateará un *string* y lo retornará.
- `fmt.Fprintf` formateará un *string* y lo escribirá en un `io.Writer`, que puede ser un archivo, una conexión de red, etc.
- `fmt.Printf` formateará un *string* y lo escribirá en la salida estándar (*standard output*).

#### Cómo funciona...

Todas estas funciones utilizan el formato `%[opciones]<verbo>` para consumir un argumento de la lista de argumentos. Para producir un carácter `%` en la salida, usa `%%`:

```go
func main() {
	fmt.Printf("Print integers using %%d: %d|\n", 10) // Print integers using %d: 10|
	fmt.Printf("You can set the width of the printed number, left aligned: %5d|\n", 10) // You can set the width of the printed number, left
	// aligned: 10|
	fmt.Printf("You can make numbers right-aligned with a given width: %-5d|\n", 10) // You can make numbers right-aligned with a given width: 10 |
	fmt.Printf("The width can be filled with 0s: %05d|\n", 10) // The width can be filled with 0s: 00010|
	fmt.Printf("You can use multiple arguments: %d %s %v\n", 10, "yes", true) // You can use multiple arguments: 10 yes true
	fmt.Printf("You can refer to the same argument multiple times : %d %s %[2]s %v\n", 10, "yes", true) // You can refer to the same argument multiple times : 10 yes
	// yes true
	fmt.Printf("But if you use an index n, the next argument will be selected from n+1 : %d %s %[2]s %[1]v %v\n", 10, "yes", true) // But if you use an index n, the next argument will be selected
	// from n+1 : 10 yes yes 10 yes
	fmt.Printf("Use %%v to use the default format for the type: %v %v %v\n", 10, "yes", true) // Use %v to use the default format for the type: 10 yes true
	fmt.Printf("For floating point, you can specify precision: %5.2f\n", 12.345657) // For floating point, you can specify precision: 12.35
	fmt.Printf("For floating point, you can specify precision: %5.2f\n", 12.0) // For floating point, you can specify precision: 12.00
	type S struct {
		IntValue    int
		StringValue string
	}
	s := S{
		IntValue:    1,
		StringValue: `foo "bar"`,
	}
	// Print the field values of a structure, in the order they are
	// declared
	fmt.Printf("%v\n", s) // {1 foo "bar"}
	// Print the field names and values of a structure
	fmt.Printf("%+v\n", s) //{IntValue:1 StringValue:foo "bar"}
}
```

---

### Sección 3: Combinación de *Strings*

La biblioteca estándar de Go ofrece múltiples formas de construir *strings* a partir de componentes. La mejor manera depende del tipo de *strings* con los que estés tratando y de su longitud. Esta sección muestra varias formas de construir *strings*.

#### Cómo hacerlo...

- Para combinar una cantidad fija y pequeña de *strings*, o para agregar *runes* a otro *string*, usa los operadores `+` o `+=`, o `strings.Builder`.
- Para construir un *string* algorítmicamente, usa `strings.Builder`.
- Para combinar un *slice* de *strings*, usa `strings.Join`.
- Para combinar partes de rutas URL, usa `path.Join`.
- Para construir rutas del sistema de archivos a partir de segmentos de ruta, usa `filepath.Join`.

#### Cómo funciona...

Para construir valores constantes o para concatenaciones simples, usa los operadores `+` o `+=`:

```go
var TwoLines = "This is the first line \n" + "This is the second line"

func ThreeLines(newLine string) string {
	return TwoLines + "\n" + newLine
}
```

Puedes agregar *runes* a un *string* de la misma manera:

```go
func AddNewLine(line string) string {
	return line + string('\n')
}
```

> **Consejo**  
> El uso del operador `+` para *strings* puede generar controversia en equipos preocupados por el rendimiento. Es cierto que el operador `+` puede volverse ineficiente porque múltiples adiciones pueden crear *strings* temporales innecesarios para almacenar resultados intermedios. También es cierto que, para algunos casos de uso, el compilador puede generar mejor código del que puedes escribir manualmente. Sin embargo, a menos que uses el operador `+` para crear *strings* dentro de bucles `for`, rara vez será la causa de tus problemas de rendimiento. Por ejemplo, `x+y` casi siempre superará a `fmt.Sprintf("%s%s", x, y)`. En caso de duda, escribe un *benchmark* y mide. Así es como se ve en mi equipo portátil:
>
> ```text
> BenchmarkXPlusY-12     98628536    11.31 ns/op
> BenchmarkSprintf-12    12120278    97.70 ns/op
> BenchmarkBuilder-12    33077902    34.89 ns/op
> ```

Para casos no triviales donde debes agregar muchos *strings* cortos para construir uno más largo, usa `strings.Builder`. Aunque `strings.Builder` parece una interfaz conveniente sobre un *slice* de bytes, hace mucho más que eso. Crea *strings* a partir del *slice* de bytes subyacente sin copiar memoria, por lo que casi siempre supera el rendimiento de usar un *slice* de bytes y luego crear un *string* a partir de él.

> **Consejo**  
> Este es un ejemplo que muestra por qué deberías preferir las funciones de la biblioteca estándar sobre bibliotecas de terceros u optimizaciones manuales. Estas funciones están fuertemente optimizadas y se basan en partes internas de Go sin generar problemas de portabilidad:

```go
builder := strings.Builder{} // Zero-value is ready to use
for i := 0; i < 10000; i++ {
	builder.WriteString(getShortString(i))
}
fmt.Println(builder.String())
```

Usa `strings.Join` para combinar un *slice* de *strings*. Si estás tratando con nombres de archivo y necesitas combinar múltiples niveles de directorios, usa `filepath.Join` para evitar caracteres separadores específicos de la plataforma. `filepath.Join` utilizará `\` en plataformas Windows y `/` en plataformas basadas en Linux. Si estás tratando con URLs y necesitas combinar múltiples segmentos, usa `path.Join`, que siempre utilizará `/` para combinar partes:

```go
package main

import (
	"fmt"
	"path"
	"path/filepath"
	"strings"
)

func main() {
	words := []string{"foo", "bar", "baz"}
	fmt.Println(strings.Join(words, " ")) // foo bar baz
	fmt.Println(strings.Join(words, ""))  // foobarbaz
	fmt.Println(path.Join(words...))      // foo/bar/baz
	fmt.Println(filepath.Join(words...))  // foo/bar/baz or foo\bar\baz, depending on the host system
	paths := []string{"/foo", "//bar", "baz"}
	fmt.Println(strings.Join(paths, " ")) // /foo //bar baz
	fmt.Println(path.Join(paths...))      // /foo/bar/baz
	fmt.Println(filepath.Join(paths...))  // /foo/bar/baz or \foo\bar\baz depending on the host system
}
```

---

### Sección 4: Trabajo con Mayúsculas y Minúsculas (*String Cases*)

Al trabajar con datos textuales, surgen a menudo problemas relacionados con el tamaño de las letras (*cases*). ¿Debería una búsqueda de texto distinguir mayúsculas de minúsculas (*case-sensitive*) o ser insensible a ellas (*case-insensitive*)? ¿Cómo convertimos un *string* a minúsculas o mayúsculas? En esta sección, veremos algunas recetas para abordar estos problemas comunes de manera portable.

#### Cómo hacerlo...

- Convierte *strings* a mayúsculas y minúsculas usando las funciones `strings.ToUpper` y `strings.ToLower`, respectivamente.
- Al tratar con texto en idiomas con asignaciones especiales de mayúsculas/minúsculas (como el turco, donde "İ" es la versión en mayúscula de "i"), usa `strings.ToUpperSpecial` y `strings.ToLowerSpecial`.
- Para convertir texto a mayúsculas para su uso en títulos, usa `strings.ToTitle`.
- Para comparar *strings* lexicográficamente, utiliza los operadores de comparación.
- Para verificar la equivalencia de *strings* ignorando mayúsculas y minúsculas, usa `strings.EqualFold`.

#### Cómo funciona...

Convertir un *string* a mayúsculas o minúsculas es simple:

```go
greet := "Hello World!"
fmt.Println(strings.ToUpper(greet))
fmt.Println(strings.ToLower(greet))
```

Este programa genera la siguiente salida:

```text
HELLO WORLD!
hello world!
```

Sin embargo, las mayúsculas/minúsculas pueden variar según el idioma. Por ejemplo, existen casos especiales para algunos de los idiomas túrquicos:

```go
word := "ilk"
fmt.Println(strings.ToUpper(word))
```

Esto imprimirá:

```text
ILK
```

Sin embargo, ese no es el uso correcto de mayúsculas en turco. Probemos lo siguiente:

```go
import (
	"fmt"
	"strings"
	"unicode"
)

func main() {
	word := "ilk"
	fmt.Println(strings.ToUpperSpecial(unicode.TurkishCase, word))
}
```

El programa anterior imprimirá:

```text
İLK
```

El formato título (*title case*) difiere de las mayúsculas o minúsculas principalmente al tratar con ligaduras y dígrafos, es decir, más de un carácter representado como un único carácter, como Ǉ (U+01C7):

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	fmt.Println(strings.ToTitle("Ǉ")) // U+01C7
	fmt.Println(strings.ToUpper("Ǉ"))
	fmt.Println(strings.ToLower("Ǉ"))
}
```

Este programa imprime:

```text
ǈ
Ǉ
ǉ
```

Mayúsculas (*uppercase*), minúsculas (*lowercase*) y formato título (*title case*) definen cómo imprimir un *string* utilizando una asignación de caso particular (*case mapping*). El plegado de mayúsculas y minúsculas (*case folding*) es el proceso de transformar texto al mismo caso con fines de comparación.

Para comparaciones lexicográficas sensibles a mayúsculas y minúsculas, utiliza los operadores relacionales:

```go
fmt.Println("a" < "b") // true
```

Para comparar dos *strings* Unicode sin distinción de mayúsculas y minúsculas, usa `strings.EqualFold`:

```go
fmt.Println(strings.EqualFold("here", "Here")) // true
fmt.Println(strings.EqualFold("here", "Here")) // true
fmt.Println(strings.EqualFold("GÖ", "gö"))     // true
```

#### Y mucho más...

Aunque el paquete `strings` de la biblioteca estándar incluye la mayoría de las funciones de comparación de cadenas que necesitas, pueden no ser suficientes al manejar texto internacionalizado. Por ejemplo, en muchos casos, querrías que "Montréal" y "montreal" se consideren iguales. `strings.EqualFold` no hará eso. Muchas de las funciones auxiliares para manejar el procesamiento de texto internacionalizado se encuentran en los paquetes bajo `golang.org/x/text`.

Unicode ofrece múltiples formas de representar un *string* determinado. La `é` en "Montréal" puede representarse como una sola *rune*, `\u00e9`, o como `e` seguida de un acento agudo, `e\u0301`. `\u0301` es el "acento agudo combinatorio" (*combining acute accent*), ◌́, y modifica el punto de código que lo precede. Según el estándar Unicode, `é` y `e + ◌́` son "canónicamente equivalentes". También existe una equivalencia de compatibilidad, como `\ufb00`, que representa `ff` como un solo punto de código, y la secuencia `ff`. Las secuencias canónicamente equivalentes también son compatibles, pero no todas las secuencias compatibles son canónicamente equivalentes.

Por lo tanto, si necesitas eliminar diacríticos (es decir, marcas sin espaciado) del texto, puedes descomponerlo, eliminar los diacríticos y luego componerlo de la siguiente manera:

```go
// Based on the blog post https://go.dev/blog/normalization
package main

import (
	"fmt"
	"io"
	"strings"
	"unicode"

	"golang.org/x/text/transform"
	"golang.org/x/text/unicode/norm"
)

func main() {
	isMn := func(r rune) bool {
		return unicode.Is(unicode.Mn, r) // Mn: nonspacing marks
	}
	t := transform.Chain(norm.NFD, transform.RemoveFunc(isMn), norm.NFC)
	rd := transform.NewReader(strings.NewReader("Montréal"), t)
	str, _ := io.ReadAll(rd)
	fmt.Println(string(str))
}
```

El programa anterior imprimirá:

```text
Montreal
```

---

### Sección 5: Trabajo con Codificaciones (*Encodings*)

Si existe la posibilidad de que tu programa tenga que trabajar con datos producidos por sistemas dispares, debes tener en cuenta las diferentes codificaciones de texto. Este es un tema muy amplio, pero esta sección ofrece pautas para comenzar a explorarlo.

#### Cómo hacerlo...

- Usa el paquete `golang.org/x/text/encoding` para manejar diferentes codificaciones.
- Para encontrar una codificación por nombre, utiliza uno de los siguientes paquetes:
  - `golang.org/x/text/encoding/ianaindex`
  - `golang.org/x/text/encoding/htmlindex`
- Una vez que tengas una codificación, úsala para traducir texto hacia y desde UTF-8.

#### Cómo funciona...

Usa uno de los índices para encontrar una codificación. Luego, usa esa codificación para leer o escribir datos:

```go
package main

import (
	"fmt"
	"os"

	"golang.org/x/text/encoding/ianaindex"
)

func main() {
	enc, err := ianaindex.MIME.Encoding("US-ASCII")
	if err != nil {
		panic(err)
	}
	b, err := os.ReadFile("ascii.txt")
	if err != nil {
		panic(err)
	}
	decoder := enc.NewDecoder()
	encoded, err := decoder.Bytes(b)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(encoded))
}
```

---

### Sección 6: Iteración de Bytes y *Runes* en *Strings*

Los *strings* en Go pueden verse como una secuencia de bytes o como una secuencia de *runes*. Esta sección muestra cómo puedes iterar un *string* de ambas formas.

#### Cómo hacerlo...

- Para iterar los bytes de un *string*, utiliza índices:

```go
for i := 0; i < len(str); i++ {
	fmt.Print(str[i], " ")
}
```

- Para iterar las *runes* de un *string*, utiliza `range`:

```go
for index, c := range str {
	fmt.Print(c, " ")
}
```

#### Cómo funciona...

Un *string* en Go es un *slice* de bytes, por lo que esperarías poder escribir un bucle `for` para iterar los bytes y *runes* de un *string*. Podrías pensar en hacer lo siguiente:

```go
strBytes := []byte(str)
strRunes := []rune(str)
```

Sin embargo, convertir un *string* en un *slice* de bytes o en un *slice* de *runes* es una operación costosa. La primera crea una copia modificable de los bytes del *string* `str`, y la segunda crea una copia modificable de las *runes* de `str`. Recuerda que `rune` es `uint32`.

Existen dos formas de bucle `for` para iterar los elementos de un *string*. El siguiente bucle `for` iterará los bytes de un *string*:

```go
str := "Hello 世界"
for i := 0; i < len(str); i++ {
	fmt.Print(str[i], " ")
}
```

La salida es la siguiente:

```text
72 101 108 108 111 32 228 184 150 231 149 140
```

Además, ten en cuenta que `str[i]` te dará el $i$-ésimo byte, no la $i$-ésima *rune*.

La siguiente forma itera las *runes* de un *string*:

```go
for i, r := range str {
	fmt.Printf("( %d %d %s)", i, r, string(r))
}
```

La salida es la siguiente:

```text
(0 72 H)(1 101 e)(2 108 l)(3 108 l)(4 111 o)(5 32 )(6 19990 世)(9 30028 界)
```

Observa los índices: van en una secuencia de `0, 1, 2, 3, 4, 5, 6, 9`. Esto se debe a que `str[6]` contiene una *rune* de 3 bytes, al igual que `str[9]`.

Cuando estés trabajando con `[]byte` en lugar de un *string*, puedes emular la iteración de *runes* de la siguiente manera:

```go
import (
	"fmt"
	"unicode/utf8"
)

str := []byte("Hello 世界")
for i := 0; i < len(str); {
	r, n := utf8.DecodeRune(str[i:])
	fmt.Print("(", i, r, " ", string(r), ")")
	i += n
}
```

La función `utf8.DecodeRune` decodifica la siguiente *rune* del *slice* de bytes y retorna esa *rune* y el número de bytes consumidos. De esta manera, puedes decodificar las *runes* de un *slice* de bytes sin convertirlo previamente en un *string*.

---

### Sección 7: División (*Splitting*)

El paquete `strings` ofrece funciones convenientes para dividir un *string* y obtener un *slice* de *strings*.

#### Cómo hacerlo...

- Para dividir un *string* en componentes utilizando un delimitador, usa `strings.Split`.
- Para dividir los componentes separados por espacios de un *string*, usa `strings.Fields`.

#### Cómo funciona...

Si necesitas procesar un *string* delimitado con un delimitador fijo, usa `strings.Split`. Si necesitas procesar secciones separadas por espacios de un *string*, usa `strings.Fields`:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	fmt.Println(strings.Split("a,b,c,d", ","))        // ["a", "b", "c", "d"]
	fmt.Println(strings.Split("a, b, c, d", ","))     // ["a", " b", " c", " d"]
	fmt.Println(strings.Fields("a b c d "))           // ["a", "b", "c", "d"]
	fmt.Println(strings.Split("a---b---c--d--", "-")) // ["a", "", "", "b", "", "", "c", "", "d", "", ""]
}
```

Ten en cuenta que `strings.Split` puede causar algunas sorpresas cuando el delimitador se repite. Por ejemplo, con `"-"` como delimitador, `"a---b"` se divide en `"a"`, `""`, `""` y `"b"`. Los dos *strings* vacíos son aquellos entre el primer y segundo `"-"`, y entre el segundo y tercer `"-"`.

---

### Sección 8: Lectura de *Strings* Línea por Línea o Palabra por Palabra

Existen muchos casos de uso para procesar *strings* en flujo (*stream*), como al trabajar con textos grandes o entradas de usuario. Esta receta muestra el uso de `bufio.Scanner` para este propósito.

#### Cómo hacerlo...

Usa `bufio.Scanner` para leer líneas, palabras o bloques personalizados:
1. Crea una instancia de `bufio.Scanner`.
2. Establece el método de división (*split method*).
3. Lee los *tokens* escaneados en un bucle `for`.

#### Cómo funciona...

El `Scanner` funciona como un iterador: cada llamada al método `Scan()` devolverá `true` si procesó el siguiente *token*, o `false` si no hay más *tokens*. El *token* se puede obtener mediante el método `Text()`:

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

const input = `This is a string that has 3 lines.`

func main() {
	lineScanner := bufio.NewScanner(strings.NewReader(input))
	line := 0
	for lineScanner.Scan() {
		text := lineScanner.Text()
		line++
		fmt.Printf("Line %d: %s\n", line, text)
	}
	if err := lineScanner.Err(); err != nil {
		panic(err)
	}
	wordScanner := bufio.NewScanner(strings.NewReader(input))
	wordScanner.Split(bufio.ScanWords)
	word := 0
	for wordScanner.Scan() {
		text := wordScanner.Text()
		word++
		fmt.Printf("word %d: %s\n", word, text)
	}
	if err := wordScanner.Err(); err != nil {
		panic(err)
	}
}
```

La salida es la siguiente:

```text
Line 1: This is a string
Line 2: that has 3
Line 3: lines.
word 1: This
word 2: is
word 3: a
word 4: string
word 5: that
word 6: has
word 7: 3
word 8: lines.
```

---

### Sección 9: Recorte de los Extremos de un *String* (*Trimming*)

La entrada del usuario suele ser desordenada, incluyendo espacios adicionales antes o después del texto relevante. Esta receta muestra cómo usar las funciones de recorte de cadenas para este propósito.

#### Cómo hacerlo...

Usa la familia de funciones `strings.Trim`, como se muestra aquí:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	fmt.Println(strings.TrimRight("Break-------", "-"))             // Break
	fmt.Println(strings.TrimRight("Break with spaces-- -- --", "- ")) // Break with spaces
	fmt.Println(strings.TrimSuffix("file.txt", ".txt"))              // file
	fmt.Println(strings.TrimLeft(" \t Indented text", " \t"))       // Indented text
	fmt.Println(strings.TrimSpace(" \t \n Indented text \n\t"))       // Indented text
}
```

---

### Sección 10: Expresiones Regulares

Una expresión regular ofrece métodos eficientes para verificar que los datos textuales coincidan con un patrón determinado, buscar patrones, extraer y reemplazar partes del texto. Por lo general, compilas una expresión regular una vez y luego utilizas esa expresión regular compilada muchas veces para validar, buscar, extraer o reemplazar partes de *strings* de manera eficiente.

#### Validación de Entrada

La validación de formato es el proceso de garantizar que los datos procedentes de la entrada del usuario u otras fuentes tengan un formato reconocido. Las expresiones regulares pueden ser una herramienta eficaz para dicha validación.

##### Cómo hacerlo...

Usa expresiones regulares precompiladas para validar valores de entrada que deben ajustarse a un patrón:

```go
package main

import (
	"fmt"
	"regexp"
)

var integerRegexp = regexp.MustCompile("^[0-9]+$")

func main() {
	fmt.Println(integerRegexp.MatchString("123"))   // true
	fmt.Println(integerRegexp.MatchString(" 123 ")) // false
}
```

Para asegurar una coincidencia exacta, asegúrate de incluir los marcadores de inicio (`^`) y fin de texto (`$`); de lo contrario, terminarás aceptando entradas que contengan *strings* que coincidan con la expresión regular.

No todos los tipos de entrada son adecuados para la validación con expresiones regulares. Algunas entradas tienen expresiones regulares complicadas (como la de correos electrónicos o políticas de contraseñas), por lo que validaciones personalizadas pueden funcionar mejor en esos casos.

#### Búsqueda de Patrones

Puedes usar una expresión regular para buscar a través de datos textuales y localizar cadenas que coincidan con un patrón.

##### Cómo hacerlo...

Usa la familia de métodos `regexp.Find` para buscar subcadenas que coincidan con un patrón:

```go
package main

import (
	"fmt"
	"regexp"
)

func main() {
	re := regexp.MustCompile(`[0-9]+`)
	fmt.Println(re.FindAllString("This regular expression find numbers, like 1, 100, 500, etc.", -1))
}
```

Aquí está la salida:

```text
[1 100 500]
```

---

### Sección 11: Extracción de Datos de *Strings*

Puedes utilizar una expresión regular para localizar y extraer texto que se encuentre dentro de un patrón.

#### Cómo hacerlo...

Utiliza grupos de captura (*capture groups*) para extraer subcadenas que coincidan con un patrón.

#### Cómo funciona...

```go
package main

import (
	"fmt"
	"regexp"
)

func main() {
	re := regexp.MustCompile(`^(\w+)=(\w+)$`)
	result := re.FindStringSubmatch(`property=12`)
	fmt.Printf("Key: %s value: %s\n", result[1], result[2])
	result = re.FindStringSubmatch(`x=y`)
	fmt.Printf("Key: %s value: %s\n", result[1], result[2])
}
```

Aquí está la salida:

```text
Key: property value: 12
Key: x value: y
```

Analicemos esta expresión regular:
- `^(\w+)`: Un *string* compuesto por uno o más caracteres de palabra al principio de la línea (grupo de captura 1).
- `=`: Un signo `=`.
- `(\w+)$`: Un *string* compuesto por uno o más caracteres de palabra (grupo de captura 2) seguido del final de línea.

Observa que los grupos de captura se encuentran entre paréntesis.

El método `FindStringSubmatch` devuelve el *string* coincidente completo como el elemento `0` del *slice*, y luego cada uno de los grupos de captura. Usando los grupos de captura, puedes extraer datos como se mostró anteriormente.

---

### Sección 12: Reemplazo de Partes de un *String*

Puedes usar una expresión regular para buscar a través del texto, reemplazando partes que coincidan con un patrón por otros *strings*.

#### Cómo hacerlo...

Usa la familia de funciones `Replace` para reemplazar los patrones en un *string* por otro valor:

```go
package main

import (
	"fmt"
	"regexp"
)

func main() {
	// Find numbers, capture the first digit
	re := regexp.MustCompile(`([0-9])[0-9]*`)
	fmt.Println(re.ReplaceAllString("This example replaces numbers with 'x': 1, 100, 500.", "x"))
	// This example replaces numbers with 'x': x, x, x.
	fmt.Println(re.ReplaceAllString("This example replaces all numbers with their first digits: 1, 100, 500.", "${1}"))
	// This example replaces all numbers with their first digits: 1,
	// 1, 5.
}
```

---

### Sección 13: Plantillas (*Templates*)

Las plantillas son útiles para generar salidas textuales basadas en datos. El paquete `text/template` se puede utilizar en los siguientes contextos:
- **Archivos de configuración**: Puedes aceptar plantillas en archivos de configuración, como el siguiente ejemplo que utiliza una variable de mapa `env` para crear configuraciones dependientes del entorno:
  ```text
  logfile: {{.env.logDir}}/log.json
  ```
- **Informes (*Reporting*)**: Utiliza plantillas para generar salidas para aplicaciones de línea de comandos e informes.
- **Aplicaciones web**: El paquete `html/template` proporciona funcionalidad de plantillas segura para HTML en la generación de HTML basada en plantillas para construir aplicaciones web.

#### Sustitución de Valores

El uso principal de las plantillas es insertar elementos de datos en texto estructurado. Esta sección describe cómo puedes insertar valores calculados en un programa dentro de una plantilla.

##### Cómo hacerlo...

Usa la sintaxis `{{.name}}` para sustituir un valor en una plantilla.

El siguiente fragmento de código ejecuta una plantilla utilizando diferentes entradas:

```go
package main

import (
	"os"
	"text/template"
)

type Book struct {
	Title   string
	Author  string
	PubYear int
}

const tp = `The book "{{.Title}}" by {{.Author}} was published in {{.PubYear}}. `

func main() {
	book1 := Book{
		Title:   "Pride and Prejudice",
		Author:  "Jane Austen",
		PubYear: 1813,
	}
	book2 := Book{
		Title:   "The Lord of the Rings",
		Author:  "J.R.R. Tolkien",
		PubYear: 1954,
	}
	tmpl, err := template.New("book").Parse(tp)
	if err != nil {
		panic(err)
	}
	tmpl.Execute(os.Stdout, book1)
	tmpl.Execute(os.Stdout, book2)
}
```

El programa anterior genera la siguiente salida:

```text
The book "Pride and Prejudice" by Jane Austen was published in 1813. The book "The Lord of the Rings" by J.R.R. Tolkien was published in 1954.
```

La llamada `template.New(name)` crea una plantilla vacía con el nombre proporcionado (habrá más sobre esto más adelante). El objeto de plantilla devuelto representa un cuerpo de plantilla (que está vacío después de la llamada a `New()`). El motor de plantillas de Go utiliza una plantilla que representa el cuerpo, así como cero o más plantillas nombradas que están asociadas con ese cuerpo. La llamada `tmpl.Parse(tp)` analiza la plantilla `tp` como el cuerpo de la plantilla nombrada indicada. Si hay otras definiciones de plantilla en `tp` que se definen utilizando la construcción `{{define}}`, también se mantienen dentro de `tmpl`.

`tmpl.Execute(os.Stdout, book1)` ejecuta la plantilla, escribiendo la salida en `os.Stdout`. El segundo argumento, `book1`, son los datos utilizados para evaluar la plantilla. Accedes a ellos mediante `.`. Así, por ejemplo, cuando se evalúa `{{.Author}}`, el motor de plantillas lee `book1.Author`, utilizando reflexión, y genera su valor. En otras palabras, `.` es `book1` para la primera llamada a `tmpl.Execute`, y `.` es `book2` para la segunda llamada a `tmpl.Execute` en el ejemplo anterior.

Dado que esto se hace utilizando reflexión, lo siguiente produce la misma salida:

```go
tmpl.Execute(os.Stdout, map[string]any{
	"Title":   "Pride and Prejudice",
	"Author":  "Jane Austen",
	"PubYear": 1813,
})
```

#### Iteración

Una plantilla puede incluir datos tabulares o listas que se completan utilizando *slices* o mapas calculados en un programa. Las plantillas proporcionan un mecanismo de iteración para renderizar dicho contenido.

##### Cómo hacerlo...

Para *slices*/*arrays*, haz lo siguiente:

```go
{{ range <slice> }} 
// Here, {{.}} refers the subsequent elements of the slice/array 
{{end}}
```

Para mapas, haz lo siguiente:

```go
{{ range <map> }} 
// Here, {{.}} refers to the subsequent values (not keys) of the map 
// The iteration order of the map is not guaranteed 
{{end}}
```

Alternativamente, haz lo siguiente:

```go
{{ range $key, $value := <map> }} 
// Here, {{$key}} and {{$value}} are variables that are set to 
// subsequent key-value pairs of the map 
{{end}}
```

##### Cómo funciona...

Usa `range` para recorrer *slices* y mapas.

Modifica el ejemplo anterior con lo siguiente:

```go
const tpIter = `{{range .}} The book "{{.Title}}" by {{.Author}} was published in {{.PubYear}}. {{end}}`
```

Luego, modifícalo también con lo siguiente:

```go
... 
tmpl, err = template.New("bookIter").Parse(tpIter) 
if err != nil { 
    panic(err) 
} 
tmpl.Execute(os.Stdout, []Book{book1, book2})
```

Aquí está la salida:

```text
The book "Pride and Prejudice" by Jane Austen was published in 1813. The book "The Lord of the Rings" by J.R.R. Tolkien was published in 1954.
```

Ahora, ten en cuenta que `.` es un *slice* de libros, por lo que podemos iterar a través de sus elementos. Al evaluar la sección dentro de `{{range .}}`, `.` se establece en los elementos sucesivos del *slice*: durante la primera iteración, `.` es `book1`, y durante la segunda iteración, `.` es `book2`.

Trataremos las líneas vacías en breve.

Lo mismo ocurre con los mapas:

```go
tmpl.Execute(os.Stdout, map[int]Book{
	1: book1,
	2: book2,
})
```

#### Variables y Ámbito (*Scope*)

A menudo es necesario definir variables locales dentro de las plantillas para almacenar valores calculados. Las variables definidas en las plantillas siguen reglas de ámbito similares a las variables definidas en funciones: los bloques `{{range}}`, `{{if}}`, `{{with}}` y `{{define}}` crean un nuevo ámbito.

Una variable definida en un ámbito es accesible en todos los ámbitos contenidos dentro de ese ámbito, pero no es accesible fuera de él.

##### Cómo hacerlo...

`.` (punto) se refiere al "objeto actual", de la siguiente manera:
- En el ámbito de nivel superior, `.` se refiere al objeto pasado como argumento de datos del método `Execute`.
- Dentro de un `{{range}}`, `.` se refiere al elemento actual del *slice*/*array*/*map*.
- Dentro de un `{{with <expr>}}`, `.` se refiere al valor de `<expr>`.
- Dentro de un bloque `{{define}}`, `.` se refiere al valor del objeto pasado a `{{template "name" <object>}}`.

`.X` se refiere al miembro llamado `X` en el objeto actual:
- Si `.` es un mapa, entonces `.X` evalúa al elemento con la clave `X`.
- Si `.` es un *struct*, entonces `.X` evalúa a la variable miembro `X` exportada.

> **Consejo**  
> Nota el énfasis en **exportada**. El motor de plantillas utiliza reflexión para encontrar el valor de `X` en el objeto actual. Si el objeto actual es un *struct*, la reflexión solo puede acceder a los campos exportados, por lo que no puedes acceder a variables no exportadas. Sin embargo, si el objeto actual es un mapa, esto se convierte en una búsqueda por clave y no existe tal restricción. En otras palabras, `{{.name}}` solo funcionará si `.` es un mapa, pero `{{.Name}}` funcionará tanto para un `.` *struct* como para un `.` mapa.

Define una nueva variable local que sea visible en el ámbito actual utilizando lo siguiente:

```go
$name := value
```

##### Cómo funciona...

Usa la notación `$name` para asignar un valor calculado a una variable en lugar de recalcularlo cada vez:

```html
{{ $disabled := false }} 
{{ if eq .Selection "1"}} 
    {{ $disabled = true }} 
{{ end }} 
<input type="text" value="{{.Value1}}" {{if $disabled}}disabled{{end}}> 
<input type="text" value="{{.Value2}}" {{if $disabled}}disabled{{end}}>
```

La primera sección de esta plantilla es equivalente a lo siguiente:

```go
disabled := false
if data.Selection == "1" {
	disabled = true
}
```

`$` es necesario como el primer carácter del nombre de la variable. Sin él, el motor de plantillas pensará que `name` es una función.

#### Y más: Bucles Anidados y Condicionales

Cuando trabajas con bucles o condiciones anidadas, el ámbito puede convertirse en un desafío. Cada `{{range}}`, `{{if}}` y `{{with}}` crea un nuevo ámbito. Las variables definidas dentro de un ámbito solo son accesibles en ese ámbito y en todos los ámbitos incluidos en él. Puedes usar esto para crear bucles anidados y aun así acceder a las variables definidas en el ámbito contenedor:

```go
type Book struct {
	Title    string
	Author   string
	Editions []Edition
}
type Edition struct {
	Edition int
	PubYear int
}

const tp = `{{range $bookIndex, $book := .}}
{{$book.Author}}
{{range $book.Editions}}
{{$book.Title}} Edition: {{.Edition}} {{.PubYear}}
{{end}}
{{end}}`
```

En esta plantilla, el primer `range` define el índice del bucle, `$bookIndex`, y la variable del bucle, `$book`, que se pueden usar en los ámbitos anidados. En esta etapa, `.` apunta al *slice* de campos `Book`. El siguiente `range` itera el `$book.Editions` actual; es decir, `.` ahora apunta a los elementos sucesivos del *slice* `Book.Editions`. La plantilla anidada accede tanto a los campos de `Edition` como a los campos de `Book` desde el ámbito contenedor.

---

### Sección 14: Manejo de Líneas Vacías

Las acciones de plantilla (es decir, los elementos de código colocados en una plantilla) pueden generar espacios y líneas vacías no deseadas. El sistema de plantillas de Go ofrece algunos mecanismos para lidiar con estos espacios no deseados.

#### Cómo hacerlo...

Usa `-` junto al delimitador de plantilla:
- `{{-` eliminará todos los espacios/tabulaciones/saltos de línea que se hayan generado antes de este elemento de plantilla.
- `-}}` eliminará todos los espacios/tabulaciones/saltos de línea que vengan después de este elemento de plantilla.

Si una directiva de plantilla produce una salida, como el valor de una variable, se escribirá en el flujo de salida. Pero si una directiva de plantilla no genera ninguna salida, como una sentencia `{{range}}` o `{{if}}`, se reemplazará con cadenas vacías. Y si esas sentencias están en una línea por sí solas, esas líneas también se escribirán en la salida, como esto:

```text
{{range .}} 
{{if gt . 1}} 
{{.}} 
{{end}} 
{{end}}
```

Esta plantilla producirá una salida cada cuatro líneas. Cuando no haya nada que mostrar, imprimirá tres líneas vacías.

Corrige esto usando `"-"` dentro de las construcciones `{{ }}`. `{{ -}}` eliminará todo el espacio vacío (incluidas las líneas) que venga después, y `{{- }}` eliminará todos los espacios vacíos anteriores, como sigue:

```text
{{range . -}} 
{{ if gt . 1 }} 
{{- . }} 
{{end -}} 
{{end -}}
```

Aquí está la salida:

```text
2 3 4 5
```

¿Cómo podemos deshacernos de los espacios al principio de cada línea? Primero, tenemos que averiguar por qué están allí, lo cual se muestra aquí:

```text
{{- . }} __{{end -}}
```

El primer `"-"` eliminará todos los espacios antes del valor. No podemos poner `-}}` en esta línea, ni `{{- end}}`, ya que estas soluciones también eliminarían los saltos de línea. Pero podemos hacer esto:

```text
{{range . -}} 
{{ if gt . 1 }} 
{{- . }} 
{{end -}} 
{{end -}}
```

Esto producirá lo siguiente:

```text
2 3 4 5
```

---

### Sección 15: Composición de Plantillas

A medida que las plantillas crecen, pueden volverse repetitivas. Para reducir dicha repetición, el sistema de plantillas de Go ofrece bloques con nombre (componentes) que se pueden reutilizar dentro de una plantilla, al igual que las funciones en un programa. Luego, la plantilla final se puede componer a partir de estos componentes.

#### Cómo hacerlo...

Puedes crear "componentes" de plantilla que puedes reutilizar en múltiples contextos. Para definir una plantilla nombrada, usa la construcción `{{define "name"}}`:

```text
{{define "template1"}} 
... 
{{end}} 
{{define "template2"}} 
... 
{{end}}
```

Luego, llama a esa plantilla utilizando la construcción `{{template "name" .}}` como si fuera una función con un único argumento:

```text
{{template "template1" .}} 
{{range .List}} 
{{template "template2" .}} 
{{end}}
```

#### Cómo funciona...

El siguiente ejemplo imprime una lista de libros utilizando una plantilla nombrada:

```go
package main

import (
	"os"
	"text/template"
)

const tp = `{{define "line"}} 
{{.Title}} {{.Author}} {{.PubYear}} 
{{end}} 
Book list: 
{{range . -}} 
{{template "line" .}} 
{{end -}} 
`

type Book struct {
	Title   string
	Author  string
	PubYear int
}

var books = []Book{
	{
		Title:   "Pride and Prejudice",
		Author:  "Jane Austen",
		PubYear: 1813,
	},
	{
		Title:   "To Kill a Mockingbird",
		Author:  "Harper Lee",
		PubYear: 1960,
	},
	{
		Title:   "The Great Gatsby",
		Author:  "F. Scott Fitzgerald",
		PubYear: 1925,
	},
	{
		Title:   "The Lord of the Rings",
		Author:  "J.R.R. Tolkien",
		PubYear: 1954,
	},
}

func main() {
	tmpl, err := template.New("body").Parse(tp)
	if err != nil {
		panic(err)
	}
	tmpl.Execute(os.Stdout, books)
}
```

La plantilla `tmpl` contiene dos plantillas en este ejemplo: la plantilla llamada `"body"` (porque se creó con `template.New("body")`), y la plantilla llamada `"line"` (porque la plantilla contiene `{{define "line"}}`). Para cada elemento del *slice*, la plantilla `"body"` crea una instancia de `"line"` con los elementos sucesivos del *slice* `books`.

Esto es equivalente a lo siguiente:

```go
const lineTemplate = `{{.Title}} {{.Author}} {{.PubYear}}`
const bodyTemplate = `Book list: 
{{range . -}} 
{{template "line" .}} 
{{end -}}`

func main() {
	tmpl, err := template.New("body").Parse(bodyTemplate)
	if err != nil {
		panic(err)
	}
	_, err = tmpl.New("line").Parse(lineTemplate)
	if err != nil {
		panic(err)
	}
	tmpl.Execute(os.Stdout, books)
}
```

---

### Sección 16: Composición de Plantillas – Plantillas de Diseño (*Layout Templates*)

Al desarrollar aplicaciones web, suele ser deseable contar con algunas plantillas que especifiquen los diseños de página (*layouts*). Las páginas web completas se construyen combinando componentes de página, desarrollados como plantillas independientes utilizando este diseño. Desafortunadamente, el motor de plantillas de Go te obliga a pensar en soluciones alternativas porque las referencias de plantillas de Go son estáticas. Esto significa que necesitarías una plantilla de diseño separada para cada página.

Pero existen alternativas.

Te mostraré una idea básica que demuestra cómo se puede utilizar la composición de plantillas para que puedas extenderla según tu caso de uso, o cómo utilizar una biblioteca de terceros disponible que haga esto. La idea crucial en la composición mediante plantillas de diseño es que si defines una nueva plantilla utilizando un nombre de plantilla ya definido, la nueva definición anula la anterior.

#### Cómo hacerlo...

1. Crea una plantilla de diseño. Usa plantillas vacías o plantillas con contenido predeterminado para las secciones que redefinirás para cada ocasión.
2. Crea un sistema de configuración donde definas cada composición posible. Cada composición incluye la plantilla de diseño, así como las plantillas que definen las secciones en la plantilla de diseño.
3. Compila cada composición como una plantilla separada.

#### Cómo funciona...

Crea una plantilla de diseño:

```go
const layout = ` 
<!doctype html> 
<html lang="en"> 
<head> 
<title>{{template "pageTitle" .}}</title> 
</head> 
<body> 
{{template "pageHeader" .}} 
{{template "pageBody" .}} 
{{template "pageFooter" .}} 
</body> 
</html> 
{{define "pageTitle"}}{{end}} 
{{define "pageHeader"}}{{end}} 
{{define "pageBody"}}{{end}} 
{{define "pageFooter"}}{{end}}`
```

Esta plantilla de diseño define cuatro plantillas nombradas sin contenido. Para cada nueva página, podemos recrear estos componentes:

```go
const mainPage = ` 
{{define "pageTitle"}}Main Page{{end}} 
{{define "pageHeader"}} 
<h1>Main page</h1> 
{{end}} 
{{define "pageBody"}} 
This is the page body. 
{{end}} 
{{define "pageFooter"}} 
This is the page footer. 
{{end}}`
```

Podemos definir una segunda página, similar a la primera:

```go
const secondPage = ` 
{{define "pageTitle"}}Second page{{end}} 
{{define "pageHeader"}} 
<h1>Second page</h1> 
{{end}} 
{{define "pageBody"}} 
This is the page body for the second page. 
{{end}}`
```

Ahora, componemos `layout` con `mainPage` para obtener la plantilla de la página principal, y luego `layout` con `secondPage` para obtener la plantilla de la segunda página:

```go
import (
	"html/template"
	"os"
)

func main() {
	mainPageTmpl := template.Must(template.New("body").Parse(layout))
	template.Must(mainPageTmpl.Parse(mainPage))
	secondPageTmpl := template.Must(template.New("body").Parse(layout))
	template.Must(secondPageTmpl.Parse(secondPage))
	mainPageTmpl.Execute(os.Stdout, nil)
	secondPageTmpl.Execute(os.Stdout, nil)
}
```

Puedes extender este patrón para crear una aplicación web sofisticada utilizando plantillas de diseño, junto con un archivo de configuración que defina todas las composiciones válidas de plantillas para cada página. Dicho archivo YAML se ve de la siguiente manera:

```yaml
mainPage: 
  - layouts/main.html 
  - mainPage.html 
  - fragments/status.html 
detailPage: 
  - layouts/2col.html 
  - detailPage.html 
  - fragments/status.html 
...
```

Cuando se inicia la aplicación, creas cada plantilla para `mainPage` y `detailPage` analizando sus plantillas constituyentes en el orden indicado, colocando cada plantilla en un mapa. Luego, puedes buscar el nombre de la plantilla que deseas generar y utilizar la plantilla analizada.

---

### Sección 17: Y Mucho Más...

La documentación de la biblioteca estándar de Go es siempre tu mejor fuente para obtener información actualizada y excelentes ejemplos, como los siguientes:
- [https://pkg.go.dev/strings](https://pkg.go.dev/strings)
- [https://pkg.go.dev/text/template](https://pkg.go.dev/text/template)
- [https://pkg.go.dev/html/template](https://pkg.go.dev/html/template)
- [https://pkg.go.dev/fmt](https://pkg.go.dev/fmt)
- [https://pkg.go.dev/bufio](https://pkg.go.dev/bufio)

Los siguientes enlaces también son útiles:
- Character Model for the World Wide Web: String Matching: [https://www.w3.org/TR/charmod-norm/](https://www.w3.org/TR/charmod-norm/)
- Character Properties, Case Mappings & Names FAQ: [https://unicode.org/faq/casemap_charprop.html](https://unicode.org/faq/casemap_charprop.html)
- RFC7564: PRECIS [https://www.rfc-editor.org/rfc/rfc7564](https://www.rfc-editor.org/rfc/rfc7564)
- Excelente artículo de blog sobre el proceso de normalización en Unicode: [https://go.dev/blog/normalization](https://go.dev/blog/normalization)
- Para todos los problemas de codificación, internacionalización y Unicode que no maneja la biblioteca estándar, echa un vistazo a los paquetes aquí antes de buscar cualquier otra cosa: [https://pkg.go.dev/golang.org/x/text](https://pkg.go.dev/golang.org/x/text)

