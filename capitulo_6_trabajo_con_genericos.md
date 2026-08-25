# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 6: Trabajo con Genéricos

Ocurre con frecuencia que escribes una función que realiza algún cálculo utilizando valores de un tipo determinado (por ejemplo, enteros), pero a medida que avanza el desarrollo, de repente necesitas hacer lo mismo pero también con otro tipo de datos (por ejemplo, `float64`). Así que copias y pegas la primera función y la modificas para que tenga un nombre y tipos de datos diferentes. Quizás los ejemplos más obvios y conocidos de esta situación son los tipos de datos contenedores, como los mapas y los conjuntos (*sets*). Construyes un tipo contenedor para valores enteros, luego haces lo mismo para cadenas, luego para una estructura, y así sucesivamente.

Los **genéricos** son una forma de realizar este copiado/pegado de código en tiempo de compilación utilizando plantillas de código. Primero, creas una plantilla de función (función genérica) o una plantilla de tipo de datos (tipo genérico). Instancias una función o tipo genérico proporcionando tipos. El compilador se encarga de instanciar la plantilla con los tipos que proporcionaste y comprueba si el tipo o la función genérica instanciada se puede compilar con los tipos proporcionados.

En este capítulo, aprenderás a utilizar funciones y tipos de datos genéricos para escenarios comunes:
- Funciones genéricas
  - Escritura de una función genérica que suma números
  - Declaración de restricciones como interfaces
  - Uso de funciones genéricas como adaptadores y accesores
- Tipos genéricos
  - Escritura de un conjunto (*set*) con seguridad de tipos
  - Un mapa ordenado: uso de múltiples parámetros de tipo

---

### Sección 1: Funciones Genéricas

Una función genérica es una plantilla de función que toma tipos como parámetros. La función genérica debe compilarse para todas las posibles asignaciones de tipos de sus argumentos. Los tipos que una función genérica puede aceptar se describen mediante "restricciones de tipos" (*type constraints*). Aprenderemos sobre estos conceptos en esta sección.

#### Escritura de una Función Genérica que Suma Números

Un buen ejemplo introductorio para ilustrar los genéricos es una función que suma números. Estos números pueden ser varios tipos de enteros o números de coma flotante. Aquí, estudiaremos varias recetas con diferentes capacidades.

##### Cómo hacerlo...

Una función de suma genérica que acepta números `int` y `float64` es la siguiente:

```go
func Sum[T int | float64](values ...T) T {
  var result T
  for _, x := range values {
    result += x
  }
  return result
}
```

La construcción `[T int | float64]` define el parámetro de tipo para la función `Sum`:
- `T` es el nombre del tipo. Por ejemplo, si instancias la función `Sum` para `int`, entonces `T` es `int`.
- La expresión `int | float64` es la restricción de tipo para `T`. En este caso, significa "T es `int` o `float64`". La restricción le dice al compilador que la función `Sum` solo se puede instanciar para valores `int` o `float64`.

Como expliqué antes, una función genérica es solo una plantilla. Por ejemplo, no puedes declarar una variable de función y asignarla a `Sum`, porque `Sum` no es una función real. La siguiente sentencia instancia la función genérica `Sum` para `int`:

```go
fmt.Println(Sum[int](1,2,3))
```

Para muchos casos, el compilador puede inferir el parámetro de tipo, por lo que lo siguiente también es válido. Dado que todos los argumentos son valores `int`, el compilador infiere que lo que se quiere decir aquí es `Sum[int]`:

```go
fmt.Println(Sum(1,2,3))
```

Pero en el siguiente caso, la función instanciada es `Sum[float64]`, y los argumentos se interpretan como valores `float64`:

```go
fmt.Println(Sum[float64](1,2,3))
```

La función genérica debe compilarse correctamente para todos los `T` posibles. En este caso, `T` puede ser un `int` o un `float64`, por lo que el cuerpo de la función debe ser válido tanto para `T` siendo un `int` como para `T` siendo un `float64`. Las restricciones de tipo permiten al compilador producir errores significativos en tiempo de compilación. Por ejemplo, la restricción `[T int | float64 | big.Int]` no compila, porque `result += x` no compila para `big.Int`.

La función `Sum` no funcionará para tipos derivados de `int` o `float64`, por ejemplo:

```go
type ID int
```

Aunque `ID` es un `int`, `Sum[ID]` resultará en un error de compilación, porque `ID` es un nuevo tipo. Para incluir todos los tipos derivados de un `int`, usa `~int` en la restricción; por ejemplo:

```go
func Sum[T ~int | ~float64](values ...T) T{...}
```

Esta declaración manejará todos los tipos derivados de `int` y `float64`.

---

### Sección 2: Declaración de Restricciones como Interfaces

No es práctico seguir repitiendo restricciones cada vez que declaras nuevas funciones. En su lugar, puedes definirlas en una interfaz como una lista de tipos o como una lista de métodos.

#### Cómo hacerlo...

Una interfaz de Go especifica un conjunto de métodos. La implementación de genéricos en Go amplía esta definición para que las interfaces definan conjuntos de tipos cuando se usan como restricciones. Esto requiere algunos cambios para acomodar los tipos básicos porque los tipos básicos (como `int`) no tienen métodos. Por lo tanto, existen dos tipos de sintaxis en lo que respecta a interfaces como restricciones:

1. **Listas de tipos**: Especifican la lista de tipos aceptables en lugar de un parámetro de tipo. Por ejemplo, la siguiente restricción `UnsignedInteger` acepta todos los tipos de enteros sin signo y todos los tipos derivados de enteros sin signo:

```go
type UnsignedInteger interface {
  ~uint8 | ~uint16 | ~uint32 | ~uint64
}
```

2. **Conjuntos de métodos**: Especifican los métodos que deben implementar los tipos que son aceptables. La siguiente restricción `Stringer` acepta todos los tipos que tienen el método `String() string`:

```go
type Stringer interface {
  String() string
}
```

Estas restricciones se pueden combinar. Por ejemplo, la siguiente restricción `UnsignedIntegerString` acepta tipos que se derivan de un tipo de entero sin signo y que tienen el método `String() string`:

```go
type UnsignedIntegerString interface {
  UnsignedInteger
  Stringer
}
```

La interfaz `Stringer` se puede utilizar tanto como una restricción como una interfaz tradicional. Las interfaces `UnsignedInteger` y `UnsignedIntegerString` solo se pueden utilizar como restricciones.

---

### Sección 3: Uso de Funciones Genéricas como Accesores y Adaptadores

Las funciones genéricas ofrecen soluciones prácticas para accesores y adaptadores de tipos con seguridad de tipos. Por ejemplo, inicializar una variable `*int` con un valor constante requiere declarar una variable temporal, lo cual se puede simplificar mediante una función genérica. Esta receta incluye varios de tales accesores y adaptadores.

#### Cómo hacerlo...

Esta función genérica crea un puntero a partir de valores arbitrarios:

```go
func ToPtr[T any](value T) *T {
  return &value
}
```

Esto se puede utilizar para inicializar punteros sin una variable temporal:

```go
type UpdateRequest struct {
  Name *string
  ...
}
...
request:=UpdateRequest {
  Name:ToPtr("test"),
}
```

De manera similar, esta función genérica crea un *slice* a partir de valores arbitrarios:

```go
func ToSlice[T any](value T) []T {
        return []T{value}
}
func main() {
  fmt.Println(ToSlice(1))
  // Prints an int slice: [1]
}
```

La siguiente función genérica devuelve el último elemento de un *slice*:

```go
func Last[T any](slice []T) (T, bool) {
  if len(slice) == 0 {
    var zero T
    return zero, false
  }
  return slice[len(slice)-1], true
}
```

Devuelve `false` si el *slice* está vacío.

La siguiente función genérica se puede utilizar para adaptar funciones que devuelven un valor y un error para usarlas en contextos que aceptan solo el valor. La función genera un pánico (*panic*) si hay un error:

```go
func Must[T any](value T, err error) T {
  if err != nil {
    panic(err)
  }
  return value
}
```

Esto adapta la función `f() (T, error)` en `Must(f()) T`.

---

### Sección 4: Retorno de un Valor Cero desde una Función Genérica

Como dije antes, una función genérica debe compilarse para todos los tipos posibles permitidos por las restricciones de tipo. Esto puede causar problemas al crear un valor cero.

#### Cómo hacerlo...

Para crear un valor cero de un tipo parametrizado, simplemente declara una variable:

```go
func Search[T []E, E comparable](slice T,value E) (E, bool) {
  for _,v:=range slice {
    if v==value {
      return v,true
    }
  }
  // Declare a zero value like this
  var zero E
  return zero, false
}
```

---

### Sección 5: Uso de Aserción de Tipos en Argumentos Genéricos

A veces necesitas hacer algo diferente según el tipo de un valor en una función genérica. Eso requiere una aserción de tipo o una bifurcación por tipo (*type switch*); ambos funcionan para interfaces. Sin embargo, no hay garantía de que la función se instancie para una interfaz. Esta receta muestra cómo puedes lograr esto.

#### Cómo hacerlo...

Supongamos que tienes una función genérica que trata a los enteros de forma diferente:

```go
func Print[T any](value T) {
  // The following does not work because value is not necessarily an 
  // interface{}.
  if intValue, ok:=value.(int); ok {
    ...
  } else {
    ...
  }
}
```

Para que esto funcione, debes asegurarte de que el valor sea una interfaz:

```go
func Print[T any](value T) {
  // Convert value to an interface
  valueIntf := any{value)
  if intValue, ok:=valueIntf.(int); ok {
    // Value is an integer
  } else {
    // Value is not an integer
  }
}
```

La misma idea funciona para un *type switch*:

```go
func Print[T any](value T) {
  switch v:=any(value).(type) {
  case int:
    // Value is an integer
  default;
    // Value is not an integer
  }
}
```

---

### Sección 6: Tipos Genéricos

La sintaxis de funciones genéricas se extiende naturalmente a los tipos genéricos. Un tipo genérico también tiene los mismos parámetros de tipo y restricciones, y cada método de ese tipo también tiene implícitamente los mismos parámetros que el tipo en sí.

#### Escritura de un Conjunto (*Set*) con Seguridad de Tipos

Un conjunto (*set*) con seguridad de tipos se puede implementar usando un `map[T]struct{}`. Una cosa con la que hay que tener cuidado es que `T` no puede ser cualquier tipo. Solo los tipos comparables pueden ser claves de mapa, y existe una restricción predefinida para abordar esta necesidad: `comparable`.

##### Cómo hacerlo...

1. Declara un tipo de conjunto parametrizado usando `map`:

```go
type Set[T comparable] map[T]struct{}
```

2. Declara los métodos del tipo usando los mismos parámetros de tipo. Al declarar métodos, tienes que hacer referencia a los parámetros de tipo solo por su nombre:

```go
// Has returns if the set has the given value
func (s Set[T]) Has(value T) bool {
     _, exists := s[value]
     return exists
}
// Add adds values to s
func (s Set[T]) Add(values ...T) {
     for _, v := range values {
          s[v] = struct{}{}
     }
}
// Remove removes values from s
func (s Set[T]) Remove(values ...T) {
     for _, v := range values {
          delete(s, v)
     }
}
```

3. Si es necesario, crea un constructor genérico para el nuevo tipo:

```go
// NewSet creates a new set
func NewSet[T comparable]() Set[T] {
     return make(Set[T])
}
```

4. Instancia el tipo para usarlo:

```go
stringSet := NewSet[string]()
```

Observa la instanciación explícita de la función `NewSet` con el parámetro de tipo `string`. El compilador no puede inferir qué tipo quieres decir, por lo que tienes que escribir explícitamente `NewSet[string]()`. Luego, el compilador instancia el tipo `Set[string]`, lo que también instancia todos los métodos de ese tipo.

---

### Sección 7: Un Mapa Ordenado: Uso de Múltiples Parámetros de Tipo

Esta implementación de un mapa ordenado te permite mantener el orden de los elementos agregados a un mapa utilizando un *slice* combinado con un mapa.

#### Cómo hacerlo...

1. Define un *struct* con dos parámetros de tipo:

```go
type OrderedMap[Key comparable, Value any] struct {
     m     map[Key]Value
     slice []Key
}
```

Dado que `Key` se utilizará como clave del mapa, tiene que ser `comparable`. No hay restricciones sobre el tipo `Value`.

2. Define los métodos para el tipo. Los métodos ahora se declaran usando tanto `Key` como `Value`:

```go
// Add key:value to the map
func (m *OrderedMap[Key, Value]) Add(key Key, value Value) {
     _, exists := m.m[key]
     if exists {
          m.m[key] = value
     } else {
          m.slice = append(m.slice, key)
          m.m[key] = value
     }
}
// ValueAt returns the value at the given index
func (m *OrderedMap[Key, Value]) ValueAt(index int) Value {
     return m.m[m.slice[index]]
}
// KeyAt returns the key at the given index
func (m *OrderedMap[Key, Value]) KeyAt(index int) Key {
     return m.slice[index]
}
// Get returns the value corresponding to the key, and whether or not
// key exists
func (m *OrderedMap[Key, Value]) Get(key Key) (Value, bool) {
     v, bool := m.m[key]
     return v, bool
}
```

> **Consejo**  
> Los parámetros de tipo para el receptor coinciden por posición, no por nombre. En otras palabras, puedes definir un método de la siguiente manera:
>
> ```go
> func (m *OrderedMap[K, V]) ValueAt(index int) V {
> 
>      return m.m[m.slice[index]]
> 
> }
> ```
>
> Aquí, `K` es para `Key`, y `V` es para `Value`.

3. Define una función genérica constructora si es necesario:

```go
func NewOrderedMap[Key comparable, Value any]() *OrderedMap[Key, Value] {
     return &OrderedMap[Key, Value]{
          m:     make(map[Key]Value),
          slice: make([]Key, 0),
     }
}
```

> **Consejo**  
> Un constructor es necesario en este caso porque queremos inicializar el mapa en la estructura genérica. Es tentador verificar si hay un mapa `nil` cada vez que deseas agregar algo al contenedor. Tienes que elegir entre la conveniencia de tener un tipo contenedor cuyo valor cero esté listo para usar frente a la penalización de rendimiento que pagas al verificar un mapa `nil` cada vez que se agrega algo.
