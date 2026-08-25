# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 4: Trabajo con *Arrays*, *Slices* y *Maps*

Los *arrays*, *slices* y *maps* son los tipos de contenedores integrados definidos por el lenguaje Go. Son partes esenciales de casi cualquier programa y, por lo general, los bloques de construcción de otras estructuras de datos. Esta sección describe algunos de los patrones comunes de trabajo con estas estructuras de datos básicas, ya que presentan matices que pueden no ser obvios para un recién llegado.

En este capítulo, hablaremos de lo siguiente:
- Trabajo con *arrays*
- Trabajo con *slices*
- Implementación de una pila (*stack*) usando *slices*
- Trabajo con *maps*
- Implementación de conjuntos (*sets*)
- Uso de *maps* para caché segura en entornos concurrentes (*thread-safe*)

---

### Sección 1: Trabajo con *Arrays*

Los *arrays* son estructuras de datos de tamaño fijo. No hay forma de cambiar el tamaño de un *array* ni de crear un *array* usando una variable como su tamaño (en otras palabras, `[n]int` es válido solo si `n` es un entero constante). Debido a esto, los *arrays* son útiles para representar un objeto con un número fijo de elementos, como un hash SHA256, que tiene 32 bytes.

El valor cero (*zero-value*) para un *array* tiene valores cero para cada elemento del *array*. Por ejemplo, `[5]int` se inicializa con cinco enteros, todos `0`. Un *array* de cadenas tendrá cadenas vacías.

#### Creación de *Arrays* y Paso como Argumentos

Esta receta muestra cómo puedes crear *arrays* y pasar valores de *arrays* a funciones y métodos. También hablaremos de los efectos de pasar *arrays* por valor.

##### Cómo hacerlo...

Crea *arrays* usando un tamaño fijo:

```go
var arr [2]int // Array of 2 ints
```

También puedes declarar un *array* utilizando un literal de *array* sin especificar su tamaño:

```go
x := [...]int{1,2} // Array of 2 ints
```

Puedes especificar índices de *array* de forma similar a como se define un mapa:

```go
y := [...]int{1, 4: 10} // Array of 5 ints,
// [0]1, y[4]=10, all other elements are 0
// [1 0 0 0 10]
```

Usa *arrays* para definir nuevos tipos de datos de tamaño fijo:

```go
// SHA256 hash is 256 bits - 32 bytes
type SHA256 [32]byte
```

Los *arrays* se pasan por valor:

```go
func main() {
  var h SHA256
  h = getHash()
  // f will get a 32-byte array that is a copy of h
  f(h)
...
}
func f(hash SHA256) {
  hash[0]=0 // This changes the copy of `hash` passed to `f`.
            // It does not affect the `h` value declared in main
  ...
}
```

> **Advertencia**  
> Pasar un *array* por valor significa que cada vez que usas un *array* como argumento para una función, el *array* se copiará. Si pasas un *array* `[1000]int64` a una función, el entorno de ejecución asignará y copiará 8,000 bytes (`int64` tiene 64 bits, es decir, 8 bytes, y 1,000 valores `int64` son 8,000 bytes). La copia será una copia superficial (*shallow copy*), es decir, si pasas un *array* que contiene punteros, o si pasas un *array* que contiene estructuras que contienen punteros, se copiarán los punteros, no el contenido al que apuntan.

Observa el siguiente ejemplo:

```go
func f(m [2]map[string]int) {
   m[0]["x"]=1
}
func main() {
  array := [2]map[string]int{}
  // A copy of array is passed to f
  // but array[0] and array[1] are maps
  // Contents of those maps are not copied.
  f(array)
  fmt.Println(array[0])
  // This will print [x:1]
}
```

---

### Sección 2: Trabajo con *Slices*

Un *slice* es una vista sobre un *array*. Puedes estar manejando múltiples *slices* que trabajan con los mismos datos subyacentes.

El valor cero (*zero-value*) para un *slice* es `nil`. Leer o escribir en un *slice* `nil` provocará un pánico (*panic*); sin embargo, puedes agregar elementos (*append*) a un *slice* `nil`, lo que creará un nuevo *slice*.

#### Creación de *Slices*

Hay varias formas de crear un *slice*.

##### Cómo hacerlo...

Usa `make(sliceType, length[, capacity])`:

```go
slice1 := make([]int,0)
// len(slice1)=0, cap(slice1)=0
slice2 := make([]int,0,10)
// len(slice2)=0, cap(slice2)=10
slice3 := make([]int,10)
// len(slice3)=10, cap(slice3)=10
```

En el fragmento de código anterior, ves tres usos diferentes de `make` para crear un *slice*:
- `slice1 := make([]int, 0)` crea un *slice* vacío, siendo `0` la longitud del *slice*. La variable `slice1` se inicializa como un *slice* no nulo de longitud 0.
- `slice2 := make([]int, 0, 10)` crea un *slice* vacío con capacidad 10. Esto es lo que debes preferir si conoces el tamaño máximo probable para este *slice*. Esta asignación de *slice* evita una operación de asignación/copia hasta que se agregue el elemento número 11.
- `slice3 := make([]int, 10)` crea un *slice* con tamaño 10 y capacidad 10. Los elementos del *slice* se inicializan a `0`. En general, con esta forma, el *slice* asignado se inicializará con el valor cero de su tipo de elemento.

> **Consejo**  
> Ten cuidado al asignar un *slice* con una longitud distinta de cero. Personalmente, tuve que lidiar con errores muy oscuros porque escribí por error `make([]int, 10)` en lugar de `make([]int, 0, 10)`, y continué agregando los 10 elementos al *slice* asignado, terminando con 20 elementos.

Observa el siguiente ejemplo:

```go
values:=make([]string,10)
for _,s:=range results {
  if someFunc(s) {
    values=append(values,s)
  }
}
```

El fragmento de código anterior crea un *slice* de cadenas que tiene 10 cadenas vacías, y luego las cadenas se agregan mediante el bucle `for`.

También puedes inicializar un *slice* mediante un literal:

```go
slice := []int{1,2,3,4,5}
// len(slice)=5 cap(slice)=5
```

Alternativamente, puedes dejar una variable de *slice* en `nil` y agregarle elementos. La función integrada `append` aceptará un *slice* `nil` y creará uno:

```go
// values slice is nil after declaration
var values []string
for _,x:=range results {
  if someFunc(s) {
    values=appennd(values, s)
  }
}
```

---

### Sección 3: Creación de un *Slice* a partir de un *Array*

Muchas funciones aceptarán *slices* y no *arrays*. Si tienes un *array* de valores y necesitas pasarlo a una función que requiere un *slice*, debes crear un *slice* a partir de un *array*. Esto es fácil y eficiente. Crear un *slice* a partir de un *array* es una operación de tiempo constante ($O(1)$).

#### Cómo hacerlo...

Usa la notación `[:]` para crear un *slice* a partir del *array*. El *slice* tendrá el *array* como su almacenamiento subyacente:

```go
arr := [...]int{0, 1, 2, 3, 4, 5}
slice := arr[:] // slice has all elements of arr
slice[2]=10
// Here, arr = [...]int{0,1,10,3, 4,5}
// len(slice) = 6
// cap(slice) = 6
```

Puedes crear un *slice* que apunte a una sección del *array*:

```go
slice2 := arr[1:3]
// Here, slice2 = {1,10}
// len(slice2) = 2
// cap(slice2) = 5
```

Puedes rebanar (*slice*) un *slice* existente. Los límites de la operación de corte están determinados por la capacidad del *slice* original:

```go
slice3 := slice2[0:4]
// len(slice3)=4
// cap(slice3)=5
// slice3 = {1,10,3,4}
```

#### Cómo funciona...

Un *slice* es una estructura de datos que contiene tres valores: la longitud del *slice*, la capacidad y un puntero al *array* subyacente. Rebanar un *array* simplemente crea esta estructura de datos con un puntero inicializado al *array*. Es una operación de tiempo constante.

#### Figura 4.1 – Diferencia entre un array `arr` y un slice `arr[:]`

---

### Sección 4: Adición, Inserción y Eliminación de Elementos en *Slices*

Los *slices* utilizan *arrays* como su almacenamiento subyacente, pero no es posible hacer crecer los *arrays* cuando te quedas sin espacio. Debido a esto, si una operación `append` excede la capacidad del *slice*, se asigna un nuevo *array* más grande y el contenido del *slice* se copia a este nuevo *array*.

#### Cómo hacerlo...

Para agregar nuevos valores al final del *slice*, usa la función integrada `append`:

```go
// Create an empty integer slice
islice := make([]int, 0)
// Append values 1, 2, 3 to islice, assign it to newSlice
newSlice := append(islice, 1, 2, 3)
// islice:  []
// newSlice: [1 2 3]
// Create an empty integer slice
islice = make([]int, 0)
// Another integer slice with 3 elements
otherSlice := []int{1, 2, 3}
// Append 'otherSlice' to 'islice'
newSlice = append(islice, otherSlice...)
newSlice = append(newSlice, otherSlice...)
// islice: []
// otherSlice: [1 2 3]
// newSlice: [1 2 3 1 2 3]
```

Para eliminar elementos del principio o del final de un *slice*, usa rebanado (*slicing*):

```go
slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
// Slice elements starting from index 1
suffix := slice[1:]
// suffix: [1 2 3 4 5 6 7 8 9]
// Slice elements starting from index 3
suffix2 := slice[3:]
// suffix2: [3 4 5 6 7 8 9]
// Slice elements up to index 5 (excluding 5)
prefix := slice[:5]
// prefix: [0 1 2 3 4]
// Slice elements from 3 up to index 6 (excluding 6)
mid := slice[3:6]
// [3 4 5]
```

Usa el paquete `slices` para insertar o eliminar elementos de ubicaciones arbitrarias en un *slice*:
- `slices.Delete(slice, i, j)` elimina los elementos `slice[i:j]` del *slice* y devuelve el *slice* modificado.
- `slices.Insert(slice, i, value...)` inserta los valores comenzando en el índice `i`, desplazando todos los elementos a partir de `i` para hacer espacio.

```go
slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
// Remove the section slice[3:7]
edges := slices.Delete(slice, 3, 7)
// edges: [0 1 2 7 8 9]
// slice: [0 1 2 7 8 9 0 0 0 0]
inserted := slices.Insert(slice, 3, 3, 4)
// inserted: [0 1 2 3 4 7 8 9 0 0 0 0]
// edges: [0 1 2 7 8 9]
// slices: [0 1 2 7 8 9 0 0 0 0]
```

Alternativamente, puedes eliminar elementos de un *slice* y truncarlo usando un bucle `for`, como en lo siguiente:

```go
slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
// Keep an index to write to
write:=0
for _, elem := range slice {
  if elem %2 == 0 { // Copy only even numbers
    slice[write]=elem
    write++
  }
}
// Truncate the slice
slice=slice[:write]
```

#### Cómo funciona...

Un *slice* es una vista sobre un *array*. Contiene tres piezas de información:
- **`ptr`**: Un puntero a un elemento de un *array*, que es la ubicación de inicio del *slice*.
- **`len`**: El número de elementos en el *slice*.
- **`cap`**: La capacidad restante en el *array* subyacente para este *slice*.

Si agregas elementos a un *slice* más allá de su capacidad, el entorno de ejecución asigna un *array* más grande y el contenido del *slice* se copia allí. Después de esto, el nuevo *slice* apunta a un nuevo *array*.

Esta es una fuente de confusión para muchos. Un *slice* puede compartir sus elementos con otros *slices*. Por lo tanto, modificar un *slice* puede modificar otros también.

La Figura 4.2 ilustra un caso donde se utiliza el mismo *array* subyacente para cuatro *slices* diferentes:

#### Figura 4.2 – Slices compartiendo el mismo array subyacente

Observa el siguiente ejemplo:

```go
// Appends 1 to a slice, and returns the new slice
func Append1(input []int) []int {
  return append(input,1)
}
func main() {
   slice:= []int{0,1,2,3,4,5,6,7,8,9}
   shortSlice := slice[:4]
   // shortSlice: []int{0,1,2,3}
   newSlice:=Append1(slice[:4])
   // newSlice:= []int{0,1,2,3,1}
   // slice: []int{0,1,2,3,1,5,6,7,8,9}
}
```

Ten en cuenta que agregar a `newSlice` también modificó un elemento de `slice`, porque `newSlice` tiene suficiente capacidad para acomodar un elemento más, lo que sobrescribe `slice[4]`.

Truncar un *slice* es simplemente crear un nuevo *slice* que es más corto que el original. El *array* subyacente no cambia. Observa lo siguiente:

```go
slice:= []int{0,1,2,3,4,5,6,7,8,9}
newSlice:=slice[:5]
// newSlice: []int{0,1,2,3,4}
```

Recuerda, `newSlice` es simplemente una estructura de datos que contiene el mismo `ptr` y `cap` que `slice`, con un `len` más corto. Debido a esto, crear un nuevo *slice* a partir de un *slice* existente o de un *array* es una operación de tiempo constante ($O(1)$).

---

### Sección 5: Implementación de una Pila (*Stack*) Usando un *Slice*

Un uso sorprendentemente común de un *slice* es implementar una pila (*stack*). Así es como se hace.

#### Cómo hacerlo...

Una operación *push* en la pila es simplemente `append`:

```go
// A generic stack of type T
type Stack[T any] []T
func (s *Stack[T]) Push(val T) {
     *s = append(*s, val)
}
```

Para implementar *pop*, trunca el *slice*:

```go
func (s *Stack[T]) Pop() (val T) {
     val = (*s)[len(*s)-1]
     *s = (*s)[:len(*s)-1]
     return
}
```

Nuevamente, observa el uso de paréntesis e indirecciones. No podemos escribir `*s[len(*s)-1]`, porque eso se interpreta como `*(s[len(*s)-1])`. Para evitar eso, usamos `(*s)`.

---

### Sección 6: Trabajo con *Maps*

Accedes a los elementos de un *array* o un *slice* usando índices enteros. Los *maps* proporcionan una sintaxis similar para usar claves de índice que no son solo enteros, sino también cualquier tipo que sea "comparable" (lo que significa que se puede comparar usando `==` o `!=`). Un *map* es un tipo de datos asociativo, es decir, almacena pares clave-valor. Cada clave aparece una sola vez en un mapa. Un mapa en Go proporciona acceso en tiempo constante amortizado a sus elementos (es decir, cuando se mide a lo largo del tiempo, el acceso a elementos del mapa parece una operación de tiempo constante).

El tipo *map* de Go proporciona un acceso conveniente a una estructura de datos subyacente compleja. Es uno de los tipos por "referencia", es decir, asignar una variable de mapa a otro mapa simplemente asigna un puntero a la estructura subyacente y no copia los elementos del mapa.

> **Advertencia**  
> Un mapa es una colección no ordenada. No confíes en el orden de los elementos en un mapa. El mismo orden de inserción puede dar lugar a diferentes órdenes de iteración en el mismo programa en diferentes momentos.

#### Definición, Inicialización y Uso de *Maps*

De manera similar a un *slice*, el valor cero para un mapa es `nil`. Leer de un mapa `nil` tendrá el mismo resultado que leer de un mapa no nulo que no tiene elementos. Escribir en un mapa `nil` provocará un pánico. Esta sección muestra diferentes formas en que se puede inicializar y usar un mapa.

##### Cómo hacerlo...

Usa `make` para crear un nuevo mapa, o usa un literal. No puedes escribir en un mapa `nil` (¡pero sí puedes leer de él!), por lo que debes inicializar todos los mapas ya sea con `make` o usando un literal:

```go
func main() {
   // Make a new empty map
   m1 := make(map[int]string)
   // Initilize a map using empty map literal
   m2 := map[int]string{}
   // Initialize a map using a map literal
   m3 := map[int]string {
      1: "a",
      2: "b",
  }
 ...
```

A diferencia de un *slice*, los valores de un mapa no son direccionables (*addressable*):

```go
type User struct {
  Name string
}
func main() {
   usersByID := make(map[int]User)
   usersByID[1]=User{Name:"John Doe"}
   fmt.Println(usersByID[1].Name)
   // Prints: John Doe
   // The following will give a compile error
   usersByID[1].Name="James"
...
}
```

En el ejemplo anterior, no puedes establecer una variable miembro de un *struct* almacenado en un mapa. Cuando accedes a ese elemento del mapa con `usersByID[1]`, lo que obtienes es una copia de `User` almacenada en el mapa, y el efecto de cambiar su `Name` se perderá, ya que esa copia no se almacena en ningún lugar.

Por lo tanto, en su lugar, puedes leer y asignar el valor del mapa a una variable direccionable, modificarla y volver a guardarla:

```go
  user := usersByID[1]
  user.Name="James"
  usersByID[1]=user
```

Alternativamente, puedes almacenar punteros en el mapa:

```go
  userPtrsByID := make(map[int]*User)
  userPtrsByID[1]=&User {
    Name: "John Doe"
  }
  userPtrsByID[1].Name = "James" // This works.
```

Si el mapa no tiene un elemento para la clave dada, devolverá el valor cero para el tipo de valor del mapa:

```go
  user := usersByID[2]  // user is set to User{}
  userPtr := userPtrsByID[2] // userPtr is set to nil
```

Para distinguir si el valor cero se devuelve porque el mapa no contiene el elemento o porque el valor cero está almacenado explícitamente en el mapa, usa la versión de dos valores de retorno en la búsqueda del mapa:

```go
  user, exists := usersByID[1] // exists = true
  userPtr, exists := userPtrsByID[2] // exists = false
```

Usa `delete` para eliminar un elemento de un mapa:

```go
delete(usersByID, 1)
```

---

### Sección 7: Implementación de un Conjunto (*Set*) Usando un *Map*

Un conjunto (*set*) es útil para eliminar duplicados de una colección de valores. Los mapas se pueden usar como conjuntos de manera eficiente utilizando una estructura de valor de tamaño cero.

#### Cómo hacerlo...

Usa un mapa cuyo tipo de clave sea el tipo de elemento del conjunto y cuyo tipo de valor sea `struct{}`:

```go
stringSet := make(map[string]struct{})
```

Agrega valores al conjunto con el valor `struct{}{}`:

```go
stringSet[value]=struct{}{}
```

Verifica la existencia de un valor usando la versión de dos valores en la búsqueda del mapa:

```go
if _,exists:=stringSet[str]; exists {
  // String str exists in the set
}
```

Un mapa no está ordenado. Si el orden de los elementos es importante, mantén un *slice* junto con el mapa:

```go
// Remove duplicate inputs from the input, preserving order
func DedupOrdered(input []string) []string {
   set:=make(map[string]struct{})
   output:=make([]string,0,len(input))
   for _,in:=range input {
     if _,exists:=set[in]; exists {
       continue
     }
     output=append(output,in)
     set[in]=struct{}{}
   }
   return output
}
```

#### Cómo funciona...

La estructura `struct{}` es un objeto de tamaño cero. Dichos objetos son manejados de forma especial por el compilador y el entorno de ejecución. Cuando se usa como valor en un mapa, el mapa solo asignará almacenamiento para sus claves. Por lo tanto, es una forma eficiente de implementar conjuntos.

> **Advertencia**  
> Nunca confíes en la equivalencia de punteros para estructuras de tamaño cero. El compilador puede optar por ubicar dos variables separadas que tienen tamaño cero en la misma ubicación de memoria.

El resultado de la siguiente comparación no está definido:

```go
x:=&struct{}{}

y:=&struct{}{}

if x==y {

  // Do something

}
```

El resultado de `x==y` puede retornar `true` o `false`.

---

### Sección 8: Claves Compuestas

Necesitas claves compuestas cuando tienes múltiples valores que identifican a un objeto en particular. Por ejemplo, supongamos que estás manejando un sistema donde los usuarios pueden tener múltiples sesiones. Puedes almacenar esta información en un mapa de mapas, o puedes crear una clave compuesta que contenga el ID de usuario y el ID de sesión.

#### Cómo hacerlo...

Usa un *struct* comparable o un *array* como clave del mapa. Un *struct* comparable es, en general, un *struct* que no contiene lo siguiente:
- *Slices*
- Canales (*channels*)
- Funciones
- *Maps*
- Otros *structs* no comparables

Por lo tanto, para usar claves compuestas, sigue los siguientes pasos:

1. Define un *struct* comparable:

```go
type Key struct {
  UserID string
  SessionID string
}
type User struct {
  Name string
  ...
}
var compositeKeyMap = map[Key]User{}
```

2. Usa una instancia de la clave de mapa para acceder a los elementos:

```go
compositeKeyMap[Key{
  UserID: "123",
  SessionID: "1",
   }] = User {
    Name: "John Doe",
  }
```

3. Puedes usar un mapa literal para inicializarlo:

```go
var compositeKeyMap = map[Key]User {
   Key {
     UserID: "123",
     SessionID: "1",
   }: User {
      Name: "John Doe",
  },
}
```

#### Cómo funciona...

La implementación del mapa genera valores hash a partir de sus claves y luego utiliza operadores de comparación para verificar la equivalencia. Debido a esto, cualquier estructura de datos que sea comparable se puede utilizar como valor de clave.

Ten cuidado con las comparaciones de punteros. Un *struct* que contiene un campo de puntero verificará la equivalencia del puntero. Considera la siguiente clave:

```go
type KeyWithPointer struct {
  UserID string
  SessionID *int
}
var sessionMap = map[KeyWithPointer]{}
func main() {
  session := 1
  key := KeyWithPointer{
     UserID: "John",
     SessionID: &session,
  }
  sessionMap[key]=User{ Name: "John Doe"}
```

En el fragmento de código anterior, la clave del mapa compuesta contiene un puntero a `session`, un entero. Después de agregar un elemento al mapa, cambiar el valor de `session` no afectará a las claves del mapa que apuntan a esa variable. La clave del mapa seguirá apuntando a la misma variable. Se puede utilizar otra instancia de `KeyWithPointer` para localizar el objeto `User` solo si también apunta a la misma variable `session`, según lo siguiente:

```go
fmt.Println( sessionMap[KeyWithPointer{
   UserID: "John",
   SessionID: &session,
   }].Name) // "John Doe"
```

Pero:

```go
i:=1
fmt.Println( sessionMap[KeyWithPointer{
   UserID: "John",
   SessionID: &i,
   }].Name) // ""
```

---

### Sección 9: Caché Segura para Hilos (*Thread-Safe*) con *Maps*

El almacenamiento en caché a veces es necesario para lograr un rendimiento aceptable. La idea es reutilizar valores que han sido calculados o recuperados previamente. Un mapa es una opción natural para almacenar en caché dichos valores pero, debido a su naturaleza, las cachés suelen compartirse entre múltiples *goroutines* y debes tener cuidado al utilizarlas.

#### Caché Simple

Esta es una caché simple con métodos `Get`/`Put` para recuperar objetos de la caché y colocar elementos en ella.

##### Cómo hacerlo...

Para almacenar en caché valores que son accesibles mediante una clave, usa una estructura con un mapa y un mutex:

```go
type ObjectCache struct {
   mutex sync.RWMutex
   values map[string]*Object
}
// Initialize and return a new instance of the cache
func NewObjectCache() *ObjectCache {
    return &ObjectCache{
        values: make(map[string]*Object),
    }
}
```

Se debe evitar el acceso directo a las partes internas de la caché para garantizar que se respete el protocolo adecuado siempre que se use la caché:

```go
// Get an object from the cache
func (cache *ObjectCache) Get(key string) (*Object, bool) {
    cache.mutex.RLock()
    obj, exists := cache.values[key]
    cache.mutex.RUnlock()
    return obj, exists
}
// Put an object into the cache with the given key
func (cache *ObjectCache) Put(key string, value *Object) {
    cache.mutex.Lock()
    cache.values[key] = value
    cache.mutex.Unlock()
}
```

---

### Sección 10: Caché con Comportamiento Bloqueante

Si múltiples *goroutines* solicitan la misma clave de la caché simple del ejemplo anterior, todas pueden decidir recuperar el objeto y volver a colocarlo en la caché. Eso es ineficiente. Por lo general, querrás que una de esas *goroutines* recupere el objeto mientras las demás esperan. Esto se puede hacer usando `sync.Once`.

#### Cómo hacerlo...

Los elementos de la caché son estructuras que contienen `sync.Once` para garantizar que una *goroutine* obtenga el objeto mientras otras esperan por él. Además, la caché contiene un método `Get` que utiliza una devolución de llamada (*callback*) `getObjectFunc` para recuperar un objeto si no está en la caché:

```go
type cacheItem struct {
   sync.Once
   object *Object
}
type ObjectCache struct {
   mutex sync.RWMutex
   values map[string]*cacheItem
   getObjectFunc func(string) (*Object, error)
}
func NewObjectCache(getObjectFunc func(string) (*Object,error)) *ObjectCache {
  return &ObjectCache{
     values: make(map[string]*cacheItem),
     getObjectFunc: getObjectFunc,
  }
}
func (item *cacheItem) get(key string, cache *ObjectCache) (err error) {
  // Calling item.Once.Do
  item.Do(func() {
     item.object, err=cache.getObjectFunc(key)
  })
  return
}
func (cache *ObjectCache) Get(key string) (*Object, error) {
  cache.mutex.RLock()
  object, exists := cache.values[key]
  cache.mutex.RUnlock()
  if exists {
    return object.object, nil
  }
  cache.mutex.Lock()
  object, exists = cache.values[key]
  if !exists {
    object = &cacheItem{}
    cache.values[key] = object
  }
  cache.mutex.Unlock()
  err := object.get(key, cache)
  return object.object, err
}
```

#### Cómo funciona...

El método `Get` comienza bloqueando la caché para lectura. Luego verifica si la clave existe en la caché y la desbloquea. Si el valor está en caché, se retorna.

Si el valor no está en la caché, entonces la caché se bloquea para escritura, porque esto será una modificación concurrente del mapa `values`. El mapa `values` se verifica nuevamente para asegurarse de que otra *goroutine* no haya colocado un valor allí ya. Si no, esta *goroutine* coloca un `cacheItem` sin inicializar en la caché y la desbloquea.

El `cacheItem` contiene un `sync.Once`, que permitirá que solo una *goroutine* llame a `Once.Do` mientras otras se bloquean esperando que se complete la llamada ganadora. Aquí es cuando se invoca la devolución de llamada `getObjectFunc` desde el método `cacheItem.get`. En este punto, no hay posibilidad de una condición de carrera de memoria (*memory race*), porque solo una *goroutine* puede estar ejecutando la función `item.Do`. El resultado de la función se almacenará en el `cacheItem`, por lo que no causará ningún problema con los usuarios del mapa `values`. De hecho, ten en cuenta que mientras `getObjectFunc` se está ejecutando, la caché no está bloqueada. Puede haber muchas otras *goroutines* leyendo y/o escribiendo en la caché.
