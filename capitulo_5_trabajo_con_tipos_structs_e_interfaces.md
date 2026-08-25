# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 5: Trabajo con Tipos, *Structs* e Interfaces

Go es un lenguaje fuertemente tipado. Esto significa que cada valor en un programa debe definirse utilizando un conjunto de tipos básicos predefinidos. Las reglas del sistema de tipos determinan qué se puede hacer con esos valores y cómo interactúan los valores de diferentes tipos. El sistema de tipos de Go adopta un enfoque simplista: solo permite conversiones explícitas entre valores de diferentes tipos compatibles.

Go también es un lenguaje de tipado estático, lo que significa que los tipos de valores se declaran y verifican explícitamente en tiempo de compilación, a diferencia de lo que ocurre en tiempo de ejecución en lenguajes interpretados como Python o JavaScript.

En este capítulo, examinaremos algunas de las propiedades del sistema de tipos de Go, definiendo nuevos tipos, estructuras e interfaces, y considerando cómo hacer un uso eficaz del mismo para implementar algunos patrones comunes.

Este capítulo contiene las siguientes recetas:
- Creación de nuevos tipos
- Uso de composición para extender tipos
- Inicialización de estructuras
- Trabajo con interfaces
- Patrón de fábrica (*Factory pattern*)
- Contenedores polimórficos

---

### Sección 1: Creación de Nuevos Tipos

Existen varias razones por las cuales querrás definir nuevos tipos. Una muy importante es garantizar la seguridad de tipos (*type safety*). La seguridad de tipos asegura que las operaciones reciban el tipo correcto de datos. Un programa con seguridad de tipos está libre de errores de tipos, limitando los posibles errores en el programa únicamente a errores lógicos.

Otras razones para crear nuevos tipos incluyen las siguientes:
- Puedes compartir los métodos y campos de datos de un tipo en múltiples tipos diferentes mediante la incrustación o composición (*embedding*).
- Más adelante en este capítulo, veremos las interfaces. Puedes definir un conjunto de métodos para un nuevo tipo para implementar una interfaz determinada, lo que te permite usar ese tipo en diferentes contextos.

---

### Sección 2: Creación de un Nuevo Tipo Basado en un Tipo Existente

Crear un nuevo tipo te permite aplicar reglas de seguridad de tipos y agregar métodos específicos del tipo.

#### Cómo hacerlo...

Crea un nuevo tipo basado en un tipo existente usando la siguiente sintaxis:

```text
type <NewTypeName> <ExistingTypeName>
```

Por ejemplo, la siguiente declaración define un nuevo tipo de datos, `Duration`, como un entero sin signo de 64 bits:

```go
type Duration uint64
```

Así es como la biblioteca estándar de Go define `time.Duration`. Para llamar a la función `time.Sleep(d Duration)`, ahora tienes que usar un valor `time.Duration`, o convertir explícitamente un valor numérico a un valor `time.Duration`.

> **Advertencia**  
> Cuando creas un nuevo tipo a partir de un tipo existente, el nuevo tipo se crea sin ningún método, incluso si el tipo existente tiene métodos definidos.

---

### Sección 3: Creación de Enumeraciones con Seguridad de Tipos (*Type-Safe*)

En esta receta, definiremos un conjunto de constantes (una enumeración) con un nuevo tipo.

#### Cómo hacerlo...

1. Define un nuevo tipo:

```go
type Direction int
```

2. Crea una secuencia de constantes que representen los valores de la enumeración usando el nuevo tipo. Puedes usar `iota` para constantes numéricas a fin de generar números incrementales:

```go
const (
  DirectionLeft Direction = iota
  DirectionRight
)
```

3. Usa el nuevo tipo en funciones o elementos de datos que esperen este nuevo tipo:

```go
func SetDirection(dir Direction) {...}
func main() {
  SetDirection(DirectionLeft)
  SetDirection(Direction(0))
  ...
}
```

> **Consejo**  
> Esto no evita que alguien llame a `SetDirection(Direction(3))`, que es un valor no válido. Por lo general, esto solo representa un problema para los valores enumerados leídos de la entrada del usuario o de fuentes de terceros. Debes validar la entrada en ese punto.

---

### Sección 4: Creación de Tipos *Struct*

Un *struct* de Go es una colección de campos. Define *structs* para agrupar campos de datos interrelacionados para formar un registro. Esta receta muestra cómo crear nuevos tipos de estructura en tu programa.

#### Cómo hacerlo...

Crea un tipo de estructura usando la siguiente sintaxis:

```go
type NewTypeName struct {
   // List of fields
}
```

Por ejemplo:

```go
type User struct {
  Username string
  Password string
}
```

---

### Sección 5: Extensión de Tipos

Go utiliza la composición de tipos mediante incrustación (*embedding*) y el tipado estructural a través del uso de interfaces. Comencemos examinando qué significan estos conceptos.

Cuando incrustas un tipo existente en otro, los métodos y campos de datos definidos para el tipo incrustado se convierten en los métodos y campos de datos del tipo contenedor. Si has trabajado con lenguajes orientados a objetos, esto puede parecer similar a la herencia de clases, pero existe una diferencia crucial: si una clase A se deriva de una clase B, entonces A "es un" B, lo que significa que donde sea que se necesite B, puedes sustituir una instancia de A. Con la composición, si A incrusta a B, A y B son tipos distintos, y no puedes usar A donde se necesita B.

> **Consejo**  
> No hay herencia de tipos en Go. Go prefiere la composición sobre la herencia. La razón principal de esto es la simplicidad de combinar componentes para construir otros más complejos. La mayoría de los casos de uso de la herencia en lenguajes orientados a objetos se pueden rediseñar utilizando composición, interfaces y tipado estructural. Utilicé la palabra "rediseñar" intencionalmente aquí: no intentes migrar programas orientados a objetos existentes a Go emulando la herencia. En su lugar, rediséñalos y refactorízalos para que sean programas idiomáticos de Go utilizando composición e interfaces.

Las siguientes recetas analizarán cómo se puede hacer esto.

#### Extensión de un Tipo Base

Primero, veremos cómo podemos extender un tipo base para compartir sus elementos de datos y métodos en nuevos tipos.

##### Cómo hacerlo...

Supongamos que tienes algunos campos de datos y funcionalidades compartidas entre múltiples tipos de datos. Entonces puedes crear un tipo de datos base e incrustarlo en otros tipos de datos para compartir las partes comunes:

```go
type Common struct {
  commonField int
}
func (a Common) CommonMethod() {}
type A struct {
  Common
  aField int
}
func (a A) AMethod() {}
type B struct {
  Common
  bField int
}
func (b B) BMethod() {}
```

En el fragmento de código anterior, los campos y métodos de cada *struct* son los siguientes:

| Tipo | Campos | Métodos |
| --- | --- | --- |
| `Common` | `commonField` | `CommonMethod` |
| `A` | `commonField`, `aField` | `CommonMethod`, `AMethod` |
| `B` | `commonField`, `bField` | `CommonMethod`, `BMethod` |

##### Cómo funciona...

Hemos utilizado la incrustación de estructuras para compartir elementos de datos y funcionalidades comunes en la sección anterior. El siguiente ejemplo muestra dos estructuras, `Customer` y `Product`, que comparten la misma estructura `Metadata`. `Metadata` contiene el identificador único, la fecha de creación y la fecha de modificación de un registro:

```go
type Metadata struct {
  ID string
  CreatedAt time.Time
  ModifiedAt time.Time
}
// New initializes metadata fields
func (m *Metadata) New() {
  m.ID=uuid.New().String()
  m.CreatedAt=time.Now()
  m.ModifiedAt=m.CreatedAt
}
// Customer.New() uses the promoted Metadata.New() method.
// Calling Customer.New() will initialize Customer.Metadata, but
// will not modify Customer specific fields.
type Customer struct {
  Metadata
  Name string
}
// Product.New(string) shadows `Metadata.New() method. You cannot
// call `Product.New()`, but call `Product.New(string)` or
// `Product.Metadata.New()`
type Product struct {
  Metadata
  SKU string
}
func (p *Product) New(sku string) {
  // Initialize the metadata part of product
  p.Metadata.New()
  p.SKU=sku
}
func main() {
   c:=Customer{}
   c.New() // Initialize customer metadata
   p:=Product{}
   p.New("sku") // Initialize product metadata and sku
   // p.New() // Compile error: p.New() takes a string argument
}
```

La incrustación no es herencia. El receptor de un método de estructura incrustada no es una copia de la estructura contenedora. En el fragmento anterior, cuando llamamos a `c.New()`, el método `Metadata.New()` obtiene un receptor que es una instancia de `*Metadata`, no una instancia de `*Customer`.

---

### Sección 6: Inicialización de *Structs*

Esta receta muestra cómo puedes usar literales de estructura para inicializar estructuras de datos complejas que contienen estructuras incrustadas.

#### Cómo hacerlo...

Go garantiza que todas las variables declaradas se inicialicen con sus valores cero. Esto no es muy útil si tienes una estructura de datos complicada que debe inicializarse con valores predeterminados o componentes de puntero no nulos. Para tales casos, utiliza funciones similares a constructores para crear una nueva instancia de una estructura. La convención establecida es escribir una función `NewX` para un tipo `X` que inicializa una instancia de `X` o `*X` y la retorna.

Aquí, `NewIndex` crea una nueva instancia inicializada del tipo `Index`:

```go
type Index struct {
   index map[string]any
   name string
}
func NewIndex(name string) *Index {
  return &Index{
    index:make(map[string]any),
    name:name,
  }
}
func (index *Index) Name() string {return index.name}
func (index *Index) Add(key string, value any) {
  index.index[key]=value
}
```

Además, observa que los campos `Index.name` e `Index.index` no están exportados. Por lo tanto, solo se puede acceder a ellos mediante los métodos exportados de `Index`. Este patrón es útil para evitar la modificación no intencionada de los campos de datos.

---

### Sección 7: Definición de Interfaces

Go utiliza el "tipado estructural" (*structural typing*). Si un tipo `T` define todos los métodos de una interfaz `I`, entonces `T` implementa `I`. Esto causa cierta confusión entre desarrolladores familiarizados con lenguajes que usan tipado nominativo, como Java, donde tienes que nombrar explícitamente los tipos constituyentes.

Las interfaces de Go son simplemente conjuntos de métodos. Cuando un tipo de datos define un conjunto de métodos, también implementa automáticamente todas las interfaces que contienen un subconjunto de sus métodos. Por ejemplo, si el tipo de datos `A` define un método `func (A) F()`, entonces `A` también implementa las interfaces `interface { func F() }` e `interface{}`. Si la interfaz `A` es un subconjunto de la interfaz `B`, entonces un tipo de datos que implemente la interfaz `B` se puede utilizar donde sea que se necesite `A`.

#### Interfaces como Contratos

Una interfaz se puede utilizar como una "especificación" o como un "contrato" que define ciertas funciones que una implementación debe satisfacer.

##### Cómo hacerlo...

Define una interfaz o un conjunto de interfaces para especificar el comportamiento esperado de un objeto. Esto es adecuado cuando se esperan múltiples implementaciones diferentes de la misma interfaz. Por ejemplo, el paquete de controladores SQL `database/driver` de la biblioteca estándar define un conjunto de interfaces que deben implementar diferentes controladores de bases de datos.

Por ejemplo, el siguiente fragmento de código define un backend de almacenamiento para guardar archivos:

```go
type Storage interface {
   Create(name string, reader io.Reader) error
   Read(name string) (io.ReadCloser,error)
   Update(name string, reader io.Reader) error
   Delete(name string) error
}
```

Puedes utilizar las instancias de objetos que implementan la interfaz `Storage` para almacenar datos en diferentes backends, como un sistema de archivos o un sistema de almacenamiento en red.

En muchos casos, los tipos de datos utilizados para declarar los métodos de dicha interfaz dependen ellos mismos de la implementación real. En ese caso, es necesario un sistema de interfaces. El paquete `database/driver` de la biblioteca estándar utiliza este enfoque. Como ejemplo, considera la siguiente interfaz de proveedor de autenticación:

```go
// Authenticator uses implementation-specific credentials to create an
// implementation-specific session
type Authenticator interface {
  Login(credentials Credentials) (Session,error)
}
// Credentials contains the credentials to authenticate a user to the 
// backend
type Credentials interface {
  Serialize() []byte
  Type() string
}
// CredentialParse implementation parses backend-specific credentials 
// from []byte input
type CredentialParser interface {
  Parse([]byte) (Credentials, error)
}
// A backend-specific session identifies the user and provides a way 
// to close the session
type Session interface {
  UserID() string
  Close()
}
```

---

### Sección 8: Fábricas (*Factories*)

Esta sección muestra una receta que se utiliza a menudo para admitir estructuras extensibles, como controladores de bases de datos, donde la importación de un paquete de controlador de base de datos específico "registra" automáticamente el controlador en una fábrica.

#### Cómo hacerlo...

1. Define una interfaz o un conjunto de interfaces que especifiquen cómo debe comportarse una implementación.
2. Crea un registro (un mapa) y una función para registrar implementaciones.
3. Cada implementación diferente se registra a sí misma en el registro usando `init()`.
4. Importa las implementaciones que se incluirán en el programa utilizando el paquete `main`.

Implementemos un marco de autenticación utilizando el ejemplo `Authenticator` de la sección anterior. Permitiremos diferentes implementaciones del marco `Authenticator`.

Primero, define una interfaz de fábrica y un mapa para mantener todas las implementaciones registradas:

```go
package auth
type AuthenticatorFactory interface {
   NewInstance() Authenticator
}
var registry = map[string]AuthenticatorFactory{}
```

Luego, declara una función `Register` exportada:

```go
func RegisterAuthenticator(name string, factory AuthenticatorFactory) {
   registry[name]=factory
}
```

Para crear dinámicamente instancias de autenticador, necesitaremos una función como la siguiente:

```go
func NewInstance(authType string) Authenticator {
   // Create a new instance using the selected factory.
   // If the given authType has not been registered, this will panic
   return registry[authType].NewInstance()
}
```

Las implementaciones pueden registrar sus propias fábricas utilizando la función `init()`:

```go
type factory struct{}
func (factory) NewInstance() auth.Authenticator {
  // Create and return a new instance of db authenticator
}
func init() {
  auth.RegisterAuthenticator("dbauthenticator",factory{})
}
```

Finalmente, tienes que unir todo esto. El sistema de compilación de Go solo incluirá paquetes que hayan sido utilizados directa o indirectamente por el código accesible desde `main()`, y las implementaciones no se referencian directamente. Tenemos que asegurarnos de que esos paquetes se importen y, por lo tanto, las implementaciones se registren. Así que impórtalos en `main`:

```go
package main
import (
  _ "import/path/of/the/implementation"
  ...
)
```

La importación anterior incluirá el paquete de implementación en el programa. Dado que el paquete está incluido en el programa, su función `init()` se llamará durante la inicialización del programa y el tipo de autenticador que proporciona quedará registrado.

---

### Sección 9: Definición de Interfaces Donde se Utilizan

El tipado estructural te permite definir una interfaz en el momento en que necesitas usarla, en lugar de predefinir una interfaz exportada. A veces, esto se confunde con el "duck-typing" (*si camina como un pato y grazna como un pato, es un pato*). La diferencia es que el *duck-typing* se refiere a determinar la compatibilidad de tipos de datos examinando el subconjunto de la estructura de un tipo en tiempo de ejecución, mientras que el tipado estructural se refiere a examinar la estructura de un tipo en tiempo de compilación. Esta receta muestra cómo puedes definir interfaces a medida que las necesitas.

#### Cómo hacerlo...

Supongamos que tienes un código como el siguiente:

```go
type A struct {
  ...
  options  map[string]any
}
func (a A) GetOptions() map[string]any {return a.options}
type B struct {
  ...
  options map[string]any
}
func (b B) GetOptions() map[string]any {return b.options}
```

Si deseas escribir una función que opere sobre las opciones de una variable de tipo `A` o `B` (o cualquier tipo que tenga opciones), simplemente puedes definir una interfaz en ese mismo lugar:

```go
type withOptions interface {
  GetOptions() map[string]any
}
func ProcessOptions(item withOptions) {
  for key, value:=range item.GetOptions() {
    ...
  }
}
```

#### Cómo funciona...

Recuerda, Go utiliza tipado estructural. Por lo tanto, puedes crear una interfaz especificando un conjunto de métodos, y cualquier tipo de datos que declare esos métodos implementará automáticamente esa interfaz. De este modo, puedes crear tales interfaces *ad hoc* y escribir funciones que tomen instancias de esas interfaces para trabajar con un número potencialmente grande de tipos de datos.

Si utilizaras un lenguaje nominativo, habrías tenido que especificar explícitamente que esos tipos implementan tu interfaz. No ocurre así en Go.

Eso también significa que si tienes una interfaz `A` y otra interfaz `B` tales que `A` declara los mismos métodos que `B`, entonces cualquier tipo que implemente `A` también implementa `B`. En otras palabras, si no puedes importar una interfaz porque está en un paquete que causaría una dependencia circular si se importa, o si esa interfaz no está exportada por ese paquete, simplemente puedes definir una interfaz equivalente en tu paquete actual.

---

### Sección 10: Uso de una Función como Interfaz

A veces, puedes encontrarte con una situación en la que tienes una función cuando se necesita una interfaz. Esta receta muestra cómo puedes definir un nuevo tipo de datos de función que también implemente una interfaz.

#### Cómo hacerlo...

Si necesitas implementar una interfaz de un solo método sin ningún elemento de datos, puedes definir un nuevo tipo basado en un *struct* vacío y declarar un método para ese tipo para implementar esa interfaz. Alternativamente, simplemente puedes usar la función en sí como una implementación de esa interfaz. El siguiente extracto es del paquete `net/http` de la biblioteca estándar:

```go
// An interface with a single function
type Handler interface {
     ServeHTTP(ResponseWriter, *Request)
}
// Define a new function type matching the interface method signature
type HandlerFunc func(ResponseWriter, *Request)
// Implement the method for the function type
func (h HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
   h(w.r) // Call the underlying function
}
```

Aquí, puedes usar funciones del tipo `HandlerFunc` siempre que se necesite una implementación de la interfaz `Handler`.

#### Cómo funciona...

El sistema de tipos de Go trata los tipos de función como cualquier otro tipo definido. Por lo tanto, puedes declarar métodos para un tipo de función. Cuando declaras métodos para un tipo de función, el tipo de función implementa automáticamente todas las interfaces que definen todos o algunos de esos métodos.

Examinemos esta afirmación con un ejemplo. Podemos declarar un nuevo tipo vacío como una implementación de la interfaz `Handler`:

```go
type MyHandler struct{}
func (MyHandler) ServeHTTP(w ResponseWriter, r *Request) {...}
```

Con esta declaración, puedes usar instancias de `MyHandler` dondequiera que se requiera un `Handler`. Sin embargo, observa que `MyHandler` no tiene elementos de datos y solo tiene un método. Por lo tanto, en su lugar, definimos un tipo de función:

```go
type MyHandler func(ResponseWriter,*Request)
```

Ahora `MyHandler` es un nuevo tipo con nombre. Esto no es muy diferente de declarar `MyHandler` como una estructura, pero en este caso, `MyHandler` es una función con una firma fija.

Dado que `MyHandler` es un tipo con nombre, podemos definir métodos para él:

```go
func (h MyHandler) ServeHTTP(w ResponseWriter, r *Request) {
  h(w,r)
}
```

Dado que `MyHandler` ahora define el método `ServeHTTP`, implementa la interfaz `Handler`. Sin embargo, `MyHandler` es un tipo de función, por lo que `h` es en realidad una función que tiene la misma firma que `ServeHTTP`. Debido a eso, la llamada `h(w, r)` funciona y `MyHandler` se puede usar en lugares donde se requiere un `Handler`.

---

### Sección 11: Descubrimiento de Capacidades de Tipos de Datos en Tiempo de Ejecución – Comprobación de la Relación "Implementa" (*Implements*)

Una interfaz proporciona una forma de llamar a los métodos de un objeto de datos subyacente. Si la misma interfaz es implementada por muchos tipos diferentes, puedes usar una función para manipular diversos tipos de datos simplemente usando su interfaz común. Sin embargo, muchas veces necesitas acceder al objeto subyacente almacenado en una interfaz. Go proporciona varios mecanismos para lograrlo. Veremos la aserción de tipos (*type assertion*) y la bifurcación por tipo (*type switch*).

#### Cómo hacerlo...

Usa interfaces y aserciones de tipo para descubrir diferentes métodos que proporciona un tipo. Recuerda que una interfaz es un conjunto de métodos. Un tipo que implementa los métodos indicados en una interfaz implementa automáticamente esa interfaz.

Usa los siguientes patrones para determinar si un tipo de datos tiene un método:

```go
func f(rd io.Reader) {
  // Is rd also an io.Writer?
  if wr, ok:= rd.(io.Writer); ok {
     // Yes, rd is an io.Writer, and wr is that writer.
     ...
  }
  // Does rd have a function ReadLine() (string,error)?
  // Define an interface here
  type hasReadLine interface {
     ReadLine() (string,error)
  }
  // And see if rd implements it:
  if readLine, ok:=rd.(hasReadLine); ok {
    // Yes, you can use readLine:
    line, err:=readLine.ReadLine()
    ...
  }
  // You can even define anonymous interfaces inline:
  if readLine, ok:=rd.(interface{ReadLine()(string,error)}); ok {
     line, err:=readLine.ReadLine()
  }
}
```

#### Cómo funciona...

Las aserciones de tipo tienen dos formas. La siguiente forma comprueba si una variable de interfaz `intf` contiene un valor concreto del tipo `concreteValue`:

```go
value, ok:=intf.(concreteValue)
```

Si la interfaz contiene un valor de ese tipo, entonces `value` tiene ahora ese valor y `ok` se convierte en `true`.

La segunda forma comprueba si el valor concreto contenido dentro de la interfaz `intf` también implementa la interfaz `otherIntf`:

```go
value, ok:=intf.(otherIntf)
```

Si el valor contenido en `intf` también tiene los métodos declarados por `otherIntf`, entonces `value` es ahora un valor de interfaz del tipo `otherIntf` que contiene el mismo valor concreto que `intf`, y `ok` se establece en `true`.

Usando esta segunda forma, puedes comprobar si una variable de interfaz implementa los métodos que necesitas.

Podrías pensar que puedes hacer lo mismo usando reflexión. La reflexión es un método para descubrir los nombres de campos y métodos de tipos en tiempo de ejecución. No es un método eficiente ni sencillo para verificar tales equivalencias de tipos.

---

### Sección 12: Comprobación de si un Valor de Interfaz es Uno de los Tipos Conocidos

Se utiliza un *type-switch* para comprobar si un valor de interfaz es un tipo concreto conocido o si implementa una determinada interfaz. Esta receta muestra cómo se puede utilizar.

#### Cómo hacerlo...

Usa un *type-switch* en lugar de una secuencia de aserciones de tipos si necesitas comparar una interfaz con múltiples tipos.

El siguiente ejemplo utiliza una `interface{}` para sumar dos valores. Los valores pueden ser ambos `int` o ambos `float64`. La función también proporciona una forma de anular el comportamiento de la suma: si el valor tiene un método `Add` compatible, llama a ese método en su lugar:

```go
// a and b must have the same types. They can be int, float64, or 
// another type
// that has Add method.
func Add(a, b interface{}) interface{} {
  // type switch:
  // In this form, a matching case block will declare aValue
  // with the correct type
  switch aValue:=a.(type) {
    case int:
      // Here, aValue is an int
      // b must be an int!
      bValue:=b.(int)
      return aValue+bValue
    case float64:
      // Here, aValue is a float64
      // b must be a float64!
      bValue:=b.(float64)
      return aValue+bValue
    case interface { Add(interface{}) interface{} }:
      // Here, aValue is an interface {Add{interface{}) interface{}}
      return aValue.Add(b)
    default:
      // Here, aValue is not defined
      // This is an unhandled case
      return nil
  }
}
```

Observa la forma en que se utiliza el *type-switch* para extraer el valor contenido en la interfaz si el caso coincide. Esto solo funciona si el caso enumera un único tipo y si no es el caso `default`. Para esos casos, la variable simplemente no está definida y trabajas con la interfaz.

---

### Sección 13: Garantizar que un Tipo Implementa una Interfaz Durante el Desarrollo

Durante las etapas de desarrollo de un proyecto, los tipos de interfaz pueden cambiar rápidamente al agregar nuevos métodos o modificar firmas de métodos existentes cambiando los tipos de argumentos o tipos de retorno. ¿Cómo pueden los desarrolladores asegurarse de que ciertas implementaciones de esas interfaces no se rompan con esos cambios?

#### Cómo hacerlo...

Supongamos que tu equipo definió la siguiente interfaz:

```go
type Car interface {
   Move(int,int)
}
```

También diremos que implementaste esa interfaz con la siguiente estructura:

```go
type RaceCar struct {
   X, Y int
}
func (r *RaceCar) Move(dx, dy int) {
  r.X+=dx
  r.Y+=dy
}
```

Sin embargo, más adelante en el desarrollo, resulta que no todos los coches pueden moverse con éxito, por lo que la firma de la interfaz cambia a la siguiente:

```go
type Car interface {
   Move(int,int) error
}
```

Con este cambio, `RaceCar` ya no implementa `Car`. Muchas veces este error se detectará en tiempo de compilación, pero no siempre. Por ejemplo, si se pasan instancias de `*RaceCar` a funciones que requieren `any`, la compilación tendrá éxito, pero se generará un pánico en tiempo de ejecución si ese argumento se convierte a `Car` o `*RaceCar` mediante una aserción de tipo:

```go
rc := item.(Car)
```

Supongamos que declaras lo siguiente:

```go
var _ Car = &RaceCar{}
```

Cualquier modificación en la interfaz `Car` que haga que `*RaceCar` ya no implemente la interfaz `Car` será un error de compilación.

Por lo tanto, en general: declara una variable en blanco con el tipo de interfaz y asígnala al tipo concreto:

```go
type I interface {...}
type Implem struct { ... }
// If something changes in Implem or I that causes Implem
// to no longer implement interface I, this will give a
// compile-time error
var _ I = Implem{}
// Same as above, but this ensures *Implem implements I
var _ I = &Implem{}
```

Si hay cambios que hacen que el tipo ya no implemente esa interfaz, se generará un error de compilación.

---

### Sección 14: Decidir si Usar un Receptor de Puntero (*Pointer Receiver*) o de Valor (*Value Receiver*) para Métodos

En esta receta, exploraremos cómo elegir entre un receptor de puntero y un receptor de valor para los métodos.

#### Cómo hacerlo...

En general, usa un tipo, no ambos. Hay dos razones para esto:
1. Coherencia en todo el código.
2. Mezclar receptores de valor y de puntero puede dar lugar a condiciones de carrera de datos (*data races*).

- Si un método modifica el objeto receptor, usa un receptor de puntero.
- Si un método no modifica el objeto receptor, o si el método depende de obtener una copia del objeto receptor, puedes usar un receptor de valor.
- Si estás implementando un tipo inmutable, en la mayoría de los casos, debes usar un receptor de valor.
- Si tus estructuras son grandes, usar un receptor de puntero reducirá la sobrecarga de copiado. Puedes encontrar diferentes pautas sobre si una estructura puede considerarse grande o no. En caso de duda, escribe un *benchmark* y mide.

#### Cómo funciona...

Para un tipo `T`, si declaras un método usando un receptor de valor, ese método se declara tanto para `T` como para `*T`. El método obtiene una copia del receptor, no un puntero a él, por lo que cualquier modificación realizada en el receptor no se reflejará en el objeto utilizado para llamar al método.

Por ejemplo, el siguiente método devuelve una copia del objeto original mientras modifica un campo:

```go
type Action struct {
   Option string
}
// Returns a copy of a with the given option. The original a is not 
// modified.
func (a Action) WithOption(option string) Action {
   a.Option=option
   return a
}
func main() {
   x:=Action{
      Option:"a",
   }
   y:=x.WithOption("b")
   fmt.Println(x.Option, y.Option) // Outputs: a b
}
```

Un receptor de valor crea una copia superficial del original. Si la estructura receptora tiene mapas, *slices* o punteros a otros objetos, solo se copiarán los encabezados de mapa, encabezados de *slice* o punteros, no el contenido del objeto apuntado. Eso significa que aunque el método obtenga un receptor de valor en el siguiente ejemplo, los cambios en el mapa se reflejan tanto en el original como en la copia:

```go
type T struct {
  m map[string]int
}
func (t T) add(k string, v int) {
   t.m[k]=v
}
func main() {
  t:=T{
     m:make(map[string]int,
  }
  t.add("a",1)
  fmt.Println(t) // [a:1]
}
```

Ten cuidado con cómo afecta esto a las operaciones con *slices*. Un *slice* es una tupla triple (`pointer`, `len`, `cap`), y eso es lo que se copia cuando pasas un receptor de valor:

```go
type T struct {
  s []string
}
func (t T) set(i int, s string) {
  t.s[i]=s
}
func (t T) add(s string) {
  t.s=append(t.s,s)
}
func main() {
  t:=T{
    s: []string{"a","b"},
  }
  fmt.Println(t.s) // [a, b]
  // Setting a slice element contained in the value receiver will be 
  // visible here
  t.set(0,"x")
  fmt.Println(t.s) // [x, b]
  // Appending to the slice contained in the value receiver will not 
  // be visible here
  // The appended slice header is set in the copy of t, the original 
  // never sees that update
  t.add("y")
  fmt.Println(t.s) // [x, b]
}
```

Un receptor de puntero es más sencillo de utilizar. El método siempre obtiene un puntero al objeto con el que se le llama. En el ejemplo anterior, declarar el método `add` con un receptor de puntero se comporta como se espera:

```go
func (t *T) add(s string) {
  t.s=append(t.s,s)
}
...
 t.add("y")
 fmt.Println(t.s) // [x, b, y]
```

Al comienzo de esta sección, también mencioné que mezclar receptores de puntero y de valor provoca una carrera de datos (*data race*). Así es como sucede.

Recuerda que ocurre una carrera de datos cuando una *goroutine* lee de una variable que está siendo modificada concurrentemente por otra. Considera el siguiente ejemplo donde el método `Version` utiliza un receptor de valor que provoca la creación de una copia de `T`:

```go
type T struct {
  X int
}
func (t T) Version() int  {return 1}
func (t *T) SetValue(x int) {t.X=x}
func main() {
  t:=T{}
  go func () {
     t.SetValue(1) // Writes to t.X
  }()
  ver := t.Version() // Makes a copy of t, which reads t.X
  ...
}
```

El acto de llamar a `t.Version()` crea una copia de la variable `t`, leyendo `t.X` concurrentemente mientras se modifica, provocando así una carrera. Esta carrera es más obvia si `t.Version` lee de `t.X` explícitamente. No hay garantía de que esa operación de lectura vea los efectos de la operación de escritura en la *goroutine*.

---

### Sección 15: Contenedores Polimórficos

En este contexto, un contenedor es una estructura de datos que almacena muchos objetos. Los principios de esta sección se pueden aplicar también a objetos individuales. En otras palabras, puedes usar la misma idea cuando tienes una sola variable polimórfica o un campo de estructura.

#### Cómo hacerlo...

1. Define una interfaz que contenga los métodos comunes a todos los tipos de datos que se almacenarán en el contenedor.
2. Declara el tipo de contenedor utilizando esa interfaz.
3. Coloca instancias de objetos reales dentro del contenedor.
4. Cuando recuperes objetos del contenedor, puedes trabajar con el objeto a través de la interfaz o realizar una aserción de tipo, obtener el tipo real u otra interfaz, y trabajar con eso.

#### Cómo funciona...

He aquí un ejemplo sencillo que trabaja con objetos `Shape`. Un objeto `Shape` es algo que se puede dibujar en una imagen y mover:

```go
type Shape interface {
  Draw(image.Image)
  Move(dx, dy int)
}
```

`Shape` tiene varias implementaciones:

```go
type Rectangle struct {
   rect image.Rectangle
   color color.Color
}
func (r *Rectangle) Draw(target image.Image) {...}
func (r *Rectangle) Move(dx, dy int) {...}
type Circle struct {
   center image.Point
   color color.Color
}
func (c *Circle) Draw(target image.Image) {...}
func (c *Circle) Move(dx, dy int) {...}
```

Tanto `*Rectangle` como `*Circle` implementan la interfaz `Shape` (observa que `Rectangle` y `Circle` sin puntero no lo hacen). Ahora podemos trabajar con un *slice* de `Shapes`:

```go
func Draw(target image.Image, shapes []Shape) {
  for _,shape:=range shapes {
    shape.Draw(targeT)
  }
}
```

Así es como se ve el *slice* `shapes`:

#### Figura 5.1 – Slice de variables de interfaz

Dado que cada interfaz contiene un puntero a la forma real, es posible utilizar la interfaz para llamar a métodos que modifican el objeto también:

```go
func Move(dx, dy int, shapes []Shape) {
  for _,shape:=range shapes {
    shape.Move(dx, dy)
  }
}
```

---

### Sección 16: Acceso a Partes de un Objeto no Expuestas Directamente a Través de la Interfaz

Al trabajar con interfaces, hay muchas ocasiones en las que necesitas acceder al objeto subyacente. Esto se logra mediante una aserción de tipo, es decir, comprobando si el valor de una interfaz satisface un tipo determinado y, de ser así, recuperándolo.

#### Cómo hacerlo...

Usa una aserción de tipo o un *type switch* para comprobar el tipo del objeto contenido en una interfaz:

```go
func f(shape Shape) {
   if rect, ok := shape.(*Rectangle); ok {
      // shape contains a *Rectangle, and rect now points to it
   }
   switch actualShape := shape.(type) {
      case *Circle :
         // shape is a *Circle, and actualShape is a *Circle variable
      case *Rectangle:
         // shape is a *Rectangle, and actualShape is a *Rectangle 
         // variable
      default:
         // shape is not a circle or rectangle. actualShape is not 
         // defined here
   }
}
```

---

### Sección 17: Acceso al *Struct* Contenedor desde el *Struct* Embebido

En lenguajes orientados a objetos como Java o C++, existe el concepto de método abstracto o método virtual, junto con la herencia de tipos. Un efecto de esta característica es que si llamas a un método `M` de una clase base `base`, entonces el método que se ejecuta en tiempo de ejecución es la implementación de `M` declarada para el objeto real en tiempo de ejecución. En otras palabras, puedes invocar un método que será anulado (*overridden*) por otras declaraciones, y simplemente no sabes qué método estás llamando realmente.

Hay formas de hacer lo mismo en Go. Esta receta muestra cómo.

#### Cómo hacerlo...

Supongamos que necesitas escribir una estructura de datos de lista enlazada circular cuyos elementos serán estructuras que incrustan una estructura base:

```go
type ListNodeHeader struct {
  next Node
  prev Node
  list *List
}
```

La lista en sí es la siguiente:

```go
type List struct {
  first Node
}
```

Por lo tanto, la lista apunta al primer nodo, que es un nodo arbitrario en la lista, y cada nodo apunta al siguiente, con el último nodo apuntando de vuelta al primero.

Necesitamos una interfaz `Node` que defina la mecánica de mantenimiento de una lista. Por supuesto, la interfaz `Node` será implementada por `ListNodeHeader` y, por tanto, por todos los nodos de la lista:

```go
type Node interface {
  ...
}
```

Se supone que los usuarios de la lista deben incrustar `ListHeader` para implementar un nodo de la lista:

```go
type ByteSliceElement struct {
  ListNodeHeader
  Payload []byte
}
type StringElement struct {
  ListNodeHeader
  Payload string
}
```

Ahora la parte difícil es implementar la interfaz `Node`. Supongamos que deseas insertar un `ByteSliceElement` en esta lista. Dado que `ByteSliceElement` incrusta `ListNodeHeader`, tiene todos sus métodos y, por lo tanto, implementa `Node`. Sin embargo, no podemos escribir, por ejemplo, un método `Insert` para `ListNodeHeader` sin conocer el objeto real que se está insertando.

Una forma de hacer esto es utilizando el siguiente patrón:

```go
type Node interface {
   Insert(list *List, this Node)
   getHeader() *ListNodeHeader
}
func (header *ListNodeHeader) getHeader() *ListNodeHeader {return header}
func (header *ListNodeHeader) Insert(list *List,this Node) {
   // If list is empty, this is the only node
   if list.first == nil {
      list.first = this
      header.next = this
      header.prev = this
      return
   }
   header.next=list.first
   header.prev=list.first.getHeader().prev
   header.prev.getHeader().next=this
   header.next.getHeader().prev=this
}
```

Hay varias cosas sucediendo aquí:
1. El método `Insert` obtiene dos vistas del nodo que se está insertando: si el nodo que se inserta es un `*ByteSliceElement`, obtiene una versión `Node` de `this`, y también obtiene el `*ListNodeHeader` incrustado en `ByteSliceElement` como receptor. Usando esto, puede ajustar los miembros del `ByteSliceElement` para que apunten a los nodos anterior y siguiente.
2. Sin embargo, no puede acceder a los miembros `prev` y `next` de un `Node`.

Una opción es lo que se muestra: declarar un método no exportado en la interfaz `Node` que devolverá el `ListNodeHeader` de un nodo dado. Otra opción es agregar métodos `getNext`/`setNext` y `getPrev`/`setPrev` a la interfaz.

Ahora has logrado dos cosas:
- Primero, cualquier usuario de esta estructura de lista fuera de este paquete debe incrustar `ListNodeHeader` para implementar un nodo de lista. Hay un método no exportado en la interfaz. No hay forma de implementar dicha interfaz en un paquete diferente; la única forma es incrustar una estructura que ya la implemente.
- Segundo, tienes una estructura de datos de contenedor polimórfico cuya mecánica es administrada por una estructura base.

---

### Sección 18: Comprobación de si una Interfaz es `nil`

Quizás te preguntes por qué esto es siquiera un problema. Después de todo, ¿no basta con comparar con `nil`? No siempre.

Una interfaz contiene dos valores: el **tipo** del valor contenido en la interfaz y un **puntero** a ese valor. Una interfaz es `nil` solo si ambos son `nil`. Hay casos en los que una interfaz puede apuntar a un valor `nil` de un tipo distinto de `nil`, lo que hace que la interfaz **no** sea `nil`.

No puedes comprobar este caso fácilmente. Tienes que evitar crear interfaces con valores `nil`.

#### Cómo hacerlo...

Evita convertir un puntero a una variable que puede ser `nil` a una interfaz:

```go
type myerror struct{}
func (myerror) Error() string { return "" }
func main() {
   var x *myerror
   var y error
   y = x // Avoid this
   if y!=nil {
      // y is not nil!
   }
}
```

En su lugar, comprueba explícitamente los valores de interfaz `nil`, como en lo siguiente:

```go
var y error
if x!=nil {
   y=x
}
```

Alternativamente, utiliza errores de valor en lugar de punteros. El siguiente código evita este problema por completo:

```go
var x myerror
```

No hay posibilidad de que `x` sea `nil`.

#### Cómo funciona...

Como expliqué anteriormente, una interfaz contiene dos valores: tipo y valor. Lo que estás intentando evitar es crear una interfaz que contenga un valor `nil` con un tipo que no es `nil`.

Después de la siguiente declaración, la interfaz `y` es `nil` porque tanto su tipo como su valor son `nil`:

```go
var y error
```

Después de la siguiente asignación, el tipo almacenado en `y` es ahora el tipo de `x`, y el valor es `nil`. Por lo tanto, `y` ya no es `nil`:

```go
y=x
```

Esto también se aplica al retorno de una función:

```go
func f() error {
     var x *myerror
     return x
}
```

La función `f` nunca devuelve `nil`.
