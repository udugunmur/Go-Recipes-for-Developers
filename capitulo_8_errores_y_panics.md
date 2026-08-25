# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 8: Errores y *Panics*

El manejo de errores en Go no ha dejado indiferente a nadie y suele ser polarizante. Quienes provienen de lenguajes con manejo de excepciones (como Java) tienden a detestarlo, mientras que aquellos que vienen de lenguajes donde los errores son valores devueltos por funciones (como C) se sienten cómodos con él.

Habiendo trabajado con ambos paradigmas, opino que la naturaleza explícita del manejo de errores te obliga a pensar en situaciones excepcionales en cada paso del desarrollo. La generación, propagación y manejo de errores requieren el mismo tipo de disciplina y escrutinio que el "camino feliz" (*happy path*, cuando no ocurre ningún error).

Como habrás notado, hago una distinción entre tres fases al lidiar con errores:
- **Detección y generación de errores**: Se ocupa de detectar una situación excepcional y capturar información de diagnóstico.
- **Propagación de errores**: Se ocupa de permitir que los errores se propaguen hacia arriba en la pila de llamadas, decorándolos opcionalmente con información contextual.
- **Manejo de errores**: Se ocupa de resolver realmente el error, lo que puede incluir terminar el programa.

En este capítulo, aprenderás sobre lo siguiente:
- Cómo generar errores
- Cómo propagarlos anotándolos con información contextual
- Cómo manejar errores
- Organización de errores en un proyecto
- Gestión de *panics*

---

### Sección 1: Retorno y Manejo de Errores

Esta receta muestra cómo detectar errores y cómo envolverlos con información contextual adicional.

#### Cómo hacerlo...

Usa el último valor de retorno de una función o método para los errores:

```go
func DoesNotReturnError() {...}
func MayReturnError() error {...}
func MayReturnStringAndError() (string,error) {...}
```

Si la función o método tiene éxito, devolverá un error `nil`. Si se detecta una condición de error dentro de la función o método, devuelve ese error literalmente o envuelve el error con otro que contenga información contextual:

```go
func LoadConfig(f string) (*Config, error) {
   file, err:=os.Open(f)
   if err!=nil {
      return nil, fmt.Errorf("file %s: %w", f,err)
   }
   defer file.Close()
   var cfg Config
   err = json.NewDecoder(file).Decode(&cfg)
   if err!=nil {
     return nil, fmt.Errorf("While unmarshaling %s: %w",f,err)
   }
   return &cfg, nil
}
```

> **Consejo**  
> No uses `panic` como sustituto de `error`. `panic` debe usarse para señalar un error potencial en el código (*bug*) o una situación irrecuperable. Un `error` se utiliza para señalar una situación dependiente del contexto, como un archivo faltante o una entrada no válida.

#### Cómo funciona...

Go utiliza detección y manejo explícitos de errores. Esto significa que no hay una ruta de ejecución implícita u oculta para los errores (como lanzar una excepción). Los errores de Go son simplemente valores de interfaz y un error que sea `nil` se interpreta como la ausencia de un error. La función anterior llama a algunas funciones de administración de archivos que pueden devolver un error. Cuando eso sucede (es decir, cuando la función devuelve un error no nulo), esta función simplemente envuelve ese error con información adicional y lo devuelve. La información adicional permite al llamador, y a veces al usuario del programa, determinar el curso de acción correcto.

---

### Sección 2: Envoltura de Errores para Agregar Información Contextual

Utilizando el paquete `errors` de la biblioteca estándar, puedes envolver un error con otro error que contenga información contextual adicional. Este paquete también proporciona utilidades y convenciones que te permitirán comprobar si un árbol de errores contiene un error en particular o extraer un error específico de un árbol de errores.

#### Cómo hacerlo...

Agrega información contextual a un error usando `fmt.Errorf`. En el siguiente ejemplo, el error devuelto contendrá el error devuelto por `os.Open` y también incluirá el nombre del archivo:

```go
file, err := os.Open(fileName)
if err!=nil {
   return fmt.Errorf("%w: While opening %s",err,fileName)
}
```

Observa el uso del verbo `%w` en `fmt.Errorf` arriba. El verbo `%w` se utiliza para crear un error que envuelve al que se le pasa como argumento. Si hubiéramos usado `%v` o `%s`, el error devuelto contendría el texto del error original, pero no lo envolvería.

---

### Sección 3: Comparación de Errores

Cuando envuelves un error con información adicional, el nuevo valor del error no es del mismo tipo ni valor que el error original. Por ejemplo, `os.Open` puede devolver `os.ErrNotExist` si no se encuentra el archivo, y si envuelves este error con información adicional, como el nombre del archivo, el llamador de esta función necesitará una forma de llegar al error original para manejarlo adecuadamente. Esta receta muestra cómo lidiar con tales valores de error envueltos.

#### Cómo hacerlo...

Comprobar si hay un error o no es simple: verifica si el valor del error es `nil` o no:

```go
file, err := os.Open(fileName)
if err!=nil {
  // File could not be opened
}
```

Comprobar si un error es el esperado debe hacerse usando `errors.Is`:

```go
file, err := os.Open(fileName)
if errors.Is(err,os.ErrNotExist) {
  // File does not exist
}
```

#### Cómo funciona...

`errors.Is(err, target error)` compara si `err` es igual a `target` haciendo lo siguiente:
1. Comprueba si `err == target`.
2. Si eso falla, comprueba si `err` tiene un método `Is(error) bool` llamando a `err.Is(target)`.
3. Si eso falla, comprueba si `err` tiene un método `Unwrap() error` y `err.Unwrap()` no es `nil` comprobando si `err.Unwrap()` es igual a `target`.
4. Si eso falla, comprueba si `err` tiene un método `Unwrap() []error`, y si `target` es igual a cualquiera de los elementos de ese *slice*.

El significado de esto es que si envuelves un error, el llamador aún puede verificar si ocurrió el error envuelto y comportarse en consecuencia.

Si defines un error usando `errors.New()` o `fmt.Errorf()`, la interfaz de error devuelta contiene un puntero a un objeto. En este caso, el hecho de que dos errores tengan la misma representación de cadena no significa que sean iguales. El siguiente programa muestra esta situación:

```go
var e1 = errors.New("test")
var e2 = errors.New("test")
if e1 != e2 {
   fmt.Println("Errors are different!")
}
```

Arriba, aunque las cadenas de error son iguales, `e1` y `e2` son punteros que apuntan a objetos diferentes. El programa imprimirá `Errors are different!`. Por lo tanto, declarar errores como el siguiente funciona:

```go
var (
  ErrNotFound = errors.New("Not found")
)
```

Una comparación con `ErrNotFound` comparará si un valor de error es un puntero al mismo objeto que `ErrNotFound`.

---

### Sección 4: Errores Estructurados

Un error estructurado proporciona información contextual que puede ser crucial para manejar los errores antes de que lleguen al usuario de un programa. Esta receta muestra cómo se pueden utilizar dichos errores.

#### Cómo hacerlo...

1. Define un *struct* que contenga metadatos que capturen la situación del error.
2. Implementa el método `Error() string` para convertirlo en un error.
3. Si el error puede envolver otros errores, incluye un `error` o `[]error` para almacenarlos.
4. Opcionalmente, implementa el método `Is(error) bool` para controlar cómo comparar este error.
5. Opcionalmente, implementa `Unwrap() error` o `Unwrap() []error` para devolver errores envueltos.

#### Cómo funciona...

Cualquier tipo de datos que implemente la interfaz `error` (que contiene un solo método, `Error() string`) se puede utilizar como un error. Esto significa que puedes crear estructuras de datos que contengan información detallada del error sobre la cual se pueda actuar más tarde. Por lo tanto, si necesitas varios campos de datos para describir un error, en lugar de construir una cadena elaborada y devolverla mediante `fmt.Errorf`, puedes usar un *struct*.

Como ejemplo, supongamos que estás analizando una entrada de texto formateada de varias líneas. Devolver información precisa y útil a tus usuarios es importante; nadie disfrutará recibir un mensaje de error de sintaxis (*Syntax error*) sin mostrar dónde está el error. Por lo tanto, declaras esta estructura de error:

```go
type ErrSyntax struct {
   Line int
   Col int
   Diag string
}
func (err ErrSyntax) Error() string {
  return fmt.Sprintf("Syntax error line: %d col: %d, %s", err.Line, 
  err.Col, err.Diag)
}
```

Ahora puedes generar información de error útil:

```go
func ParseInput(input string) error {
  ...
  if nextRune != ',' {
     return ErrSyntax {
        Line: line,
        Col: col,
        Diag: "Expected comma",
    }
  }
  ...
}
```

Puedes utilizar esta información de error para mostrar mensajes útiles a tus usuarios o controlar una respuesta interactiva, como posicionar el cursor donde está el error o resaltar el texto cerca de la ubicación del error.

---

### Sección 5: Envoltura de Errores Estructurados

Un error estructurado se puede utilizar para decorar otro error con información adicional envolviéndolo. Esta receta muestra cómo hacerlo.

#### Cómo hacerlo...

1. Mantén una variable miembro de tipo `error` (o un *slice* de errores) para almacenar la causa raíz en la estructura.
2. Implementa el método `Unwrap() error` (o `Unwrap() []error`).

#### Cómo funciona...

Puedes envolver el error de la causa raíz en un error estructurado. Esto te permite agregar información contextual estructurada sobre el error:

```go
type ErrFile struct {
   Name string
   When string
   Err error
}
func (err ErrFile) Error() string {
   return fmt.Sprintf("%s: file %s, when %s", err.Err, err.Name, err.
   When)
}
func (err ErrFile) Unwrap() error { return err.Err }
func ReadConfigFile(name string) error {
  f, err:=os.Open(name)
  if err!=nil {
     return ErrFile {
        Name: name,
        Err:err,
        When: "opening configuration file",
     }
  }
  ...
}
```

Observa que `Unwrap` es necesario. Sin eso, el siguiente código no detectará que el error se deriva de `os.ErrNotFound`:

```go
err:=ReadConfig("config.json")
if errors.Is(err,os.ErrNotFound) {
   // file not found
}
```

Con el método `Unwrap`, la función `errors.Is` puede descender por los errores contenidos y determinar si al menos uno de ellos es `os.ErrNotFound`.

---

### Sección 6: Comparación de Errores Estructurados por Tipo

En los lenguajes que admiten bloques `try-catch`, normalmente capturas errores según su tipo. Puedes emular la misma funcionalidad confiando en `errors.Is`.

#### Cómo hacerlo...

Implementa el método `Is(error) bool` en tu tipo de error para definir qué tipo de equivalencia te interesa.

#### Cómo funciona...

Tal vez recuerdes que la función `errors.Is(err, target)` primero comprueba si `err == target`, y si eso falla, comprueba si `err.Is(target)`, siempre que `err` implemente el método `Is(error) bool`. Por lo tanto, puedes usar el método `Is(error) bool` para ajustar cómo comparar tus tipos de error personalizados. Sin el método `Is(error) bool`, `errors.Is` comparará usando `==`, lo que fallará si el contenido de dos errores es diferente incluso si son del mismo tipo. El siguiente ejemplo te permite verificar si el error dado contiene `ErrSyntax` en algún lugar del árbol de errores:

```go
type ErrSyntax struct {
   Line int
   Col int
   Err error
}
func (err ErrSyntax) Error() string {...}
func (err ErrSyntax) Is(e error) bool {
  _,ok:=e.(ErrSyntax)
  return ok
}
```

Ahora, puedes probar si un error es un error de sintaxis:

```go
err:=Parse(input)
if errors.Is(err,ErrSyntax{}) {
   // err is a syntax error
}
```

---

### Sección 7: Extracción de un Error Específico del Árbol de Errores

#### Cómo hacerlo...

Usa la función `errors.As` para descender por un árbol de errores, encontrar un error en particular y extraerlo.

#### Cómo funciona...

Similar a la función `errors.Is`, `errors.As(err error, target any) bool` desciende por el árbol de errores de `err` hasta que se encuentra un error que se puede asignar a `target`. Eso se hace mediante lo siguiente:
1. Comprueba si el valor apuntado por `target` es asignable al valor apuntado por `err`.
2. Si eso falla, comprueba si `err` tiene un método `As(error) bool` llamando a `err.As(target)`. Si devuelve `true`, entonces se encuentra un error.
3. Si no, comprueba si `err` tiene un método `Unwrap() error` y `err.Unwrap()` no es `nil`, descendiendo por el árbol.
4. De lo contrario, comprueba si `err` tiene un método `Unwrap() []error`, y si devuelve un *slice* no vacío, desciende por el árbol para cada uno de ellos hasta que se encuentra una coincidencia.

En otras palabras, `errors.As` copia en `target` el error que se puede asignar a `target`.

El siguiente ejemplo se puede utilizar para extraer una instancia de `ErrSyntax` de un árbol de errores:

```go
func (err ErrSyntax) As(target any) bool {
   if tgt, ok:=target.(*ErrSyntax); ok {
      *tgt=err
      return true
   }
   return false
}
func main() {
  ...
  err:=Parse(in)
  var syntaxError ErrSyntax
  if errors.As(err,&syntaxError) {
    // syntaxError has a copy of the ErrSyntax
  }
  ...
}
```

Observa el uso de punteros aquí. La estructura de error se utiliza como un valor, y deseas una copia de esa estructura de error, por lo que pasas un puntero a ella: una instancia de `ErrSyntax` se puede copiar en una instancia de `*ErrSyntax`. Si tu programa usara `*ErrSyntax` como valor de error, necesitarías enviar `**ErrSyntax` declarando `var syntaxError *ErrSyntax` y pasando `&syntaxError` para copiar el puntero en la ubicación de memoria apuntada por el doble puntero.

---

### Sección 8: Gestión de *Panics*

En general, un *panic* es una situación irrecuperable, como el agotamiento de recursos o la violación de una invariante (es decir, un *bug* en el código). Algunos *panics*, como la falta de memoria o la división por cero, serán generados por el entorno de ejecución (o generados por el hardware y transferidos al programa como un *panic*). Debes generar un *panic* en tu programa cuando detectes un error de programación. Pero, ¿cómo decides si una situación es un error de programación y debes entrar en *panic* o si es un error normal?

En general, una entrada externa (entrada del usuario, datos enviados por una API o datos leídos de un archivo) no debería causar un *panic*. Tales situaciones deben detectarse y devolverse como errores significativos al usuario. Un *panic* en esta situación sería, por ejemplo, una compilación fallida de una expresión regular que se declara como una cadena constante en tu programa. La entrada no es algo que se pueda solucionar volviendo a ejecutar el programa con diferentes entradas; es simplemente un fallo de código.

Si un *panic* no se maneja con `recover`, el programa terminará imprimiendo la salida de diagnóstico, incluido el motivo del *panic* y las pilas de llamadas de las *goroutines* activas.

#### Cuándo Generar un *Panic*

La mayoría de las veces, decidir si entrar en *panic* o devolver un error no es una decisión fácil. Esta receta ofrece algunas pautas para facilitar esa decisión.

##### Cómo hacerlo...

Hay dos situaciones en las que puedes generar un *panic*. Entra en *panic* si se da alguno de los siguientes casos:
1. Se viola una invariante.
2. El programa no puede continuar en el estado actual.

Una **invariante** es una condición que no puede violarse en un programa. Por lo tanto, si detectas que se ha violado, en lugar de devolver un error, entra en *panic*.

El siguiente ejemplo es de una biblioteca de grafos que escribí. Un grafo contiene nodos y aristas, administrados por una estructura `*Graph`. El método `Graph.NewEdge` crea una nueva arista entre dos nodos. Esos dos nodos deben pertenecer al mismo grafo que el receptor del método `NewEdge`, por lo que es apropiado entrar en *panic* si ese no es el caso, como se muestra a continuación:

```go
func (g *Graph) NewEdge(from,to *Node) *Edge {
  if from.graph!=g {
     panic("from node is not in graph")
  }
  if to.graph!=g {
     panic("to node is not in graph")
  }
   ...
}
```

Arriba, no se puede ganar nada devolviendo un error desde este método. Esto es claramente un error de código que el llamador no advirtió, y si se permite que el programa continúe, se violará la integridad del objeto `Graph`, creando errores difíciles de encontrar. El mejor curso de acción es generar un *panic*.

La segunda situación es un caso amplio en el que no es posible continuar. Como ejemplo, considera que estás escribiendo una aplicación web y cargas plantillas HTML desde el sistema de archivos. Si falla la compilación de dicha plantilla, el programa no puede continuar. Debes generar un *panic*.

#### Recuperación de *Panics*

Un *panic* no controlado terminará el programa. A menudo, este es el único curso de acción correcto. Sin embargo, hay casos en los que deseas fallar aquello que causó el error, registrarlo y continuar. Por ejemplo, un servidor que maneja muchas solicitudes simultáneamente no se detiene solo porque una de las solicitudes haya entrado en *panic*. Esta receta muestra cómo puedes recuperarte de un *panic*.

##### Cómo hacerlo...

Usa una sentencia `recover` en una función diferida (`defer`):

```go
func main() {
  defer func() {
     if r:=recover(); r != nil {
        // deal with the panic
     }
  }()
  ...
}
```

##### Cómo funciona...

Cuando un programa entra en *panic*, la función en *panic* retornará después de que se ejecuten todos los bloques diferidos. La pila de llamadas de esa *goroutine* se desenrollará una función tras otra, limpiando mediante la ejecución de sus sentencias diferidas, hasta que se alcance el inicio de la *goroutine* o una de las funciones diferidas invoque a `recover`. Si no se recupera el *panic*, el programa fallará imprimiendo información de diagnóstico y de la pila. Si se recupera el *panic*, la función `recover()` devolverá cualquier parámetro que se le haya pasado a `panic`, que puede ser cualquier valor.

Por lo tanto, si te recuperas de un *panic*, debes comprobar si el valor recuperado es un error que puedas utilizar para proporcionar información más útil.

#### Modificación del Valor de Retorno en `recover`

Cuando te recuperas de un *panic*, normalmente deseas devolver algún tipo de error que describa lo sucedido. Esta receta te muestra cómo hacerlo.

##### Cómo hacerlo...

Para cambiar el valor de retorno de una función cuando se recupera de un *panic*, utiliza valores de retorno con nombre.

##### Cómo funciona...

Un valor de retorno con nombre te permite acceder y establecer los valores de retorno de una función. Como se muestra a continuación, puedes cambiar el valor de retorno de una función usando valores de retorno con nombre:

```go
func process() (err error) {
  defer func() {
     r:=recover()
     if e, ok:=r.(error); ok {
         err = e
     }
```

#### Captura de la Traza de la Pila (*Stack Trace*) de un *Panic*

Imprimir o registrar una traza de la pila cuando se detecta un *panic* es una herramienta fundamental para identificar problemas en tiempo de ejecución. Esta receta muestra cómo puedes agregar una traza de la pila a tus mensajes de registro (*logs*).

##### Cómo hacerlo...

Usa la función `debug.Stack` con `recover`:

```go
import "runtime/debug"
import "fmt"
func main() {
    defer func() {
        if r := recover(); r != nil {
            stackTrace := string(debug.Stack())
            // Work with stackTrace
            fmt.Println(stackTrace)
        }
    }()
    f()
}
func f() {
   var i *int
   *i=0
}
```

Cuando estás dentro de la función de recuperación, la función `debug.Stack` devolverá la pila del *panic* que se está recuperando, no la pila desde donde se llama. Por lo tanto, si puedes registrar esta información o imprimirla, te mostrará la ubicación exacta del origen del *panic*.

> **Advertencia**  
> Obtener la pila de esta manera es una operación costosa. Úsala con cuidado y solo cuando sea necesario.

El programa anterior imprimirá lo siguiente:

```text
goroutine 1 [running]:
runtime/debug.Stack()
     /usr/local/go-faketime/src/runtime/debug/stack.go:24 +0x5e
main.main.func1()
     /tmp/sandbox381445105/prog.go:13 +0x25
panic({0x48bbc0?, 0x5287c0?})
     /usr/local/go-faketime/src/runtime/panic.go:770 +0x132
main.f(...)
     /tmp/sandbox381445105/prog.go:23
main.main()
     /tmp/sandbox381445105/prog.go:18 +0x2e
```

Aquí:
- `prog.go:13` es donde se llama a `debug.Stack()`
- `prog.go:23` es donde se ejecuta `*i=0`
- `prog.go:18` es donde se llama a `f()`

Como puedes ver, la pila señala la ubicación exacta del error (`prog.go:23`).
