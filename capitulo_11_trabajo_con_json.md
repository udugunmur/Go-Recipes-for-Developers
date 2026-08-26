# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 11: Trabajo con JSON

JSON es un acrónimo de *JavaScript Object Notation* (Notación de Objetos de JavaScript). Es un formato popular para el intercambio de datos porque los objetos JSON se parecen mucho a los tipos estructurados (*structs* en Go) y es una codificación basada en texto, lo que hace que los datos codificados sean legibles por humanos. Admite *arrays*, objetos (pares nombre-valor) y relativamente pocos tipos básicos (cadenas, números, booleanos y `null`). Estas propiedades hacen que JSON sea un formato bastante fácil de trabajar.

La **codificación** (*encoding*) se refiere al proceso de transformar elementos de datos en una secuencia de bytes. Cuando codificas (o serializas/*marshal*) elementos de datos en JSON, creas una representación textual de esos elementos de datos siguiendo las reglas de sintaxis de JSON. El proceso inverso, la **decodificación** (o deserialización/*unmarshal*), asigna valores JSON a objetos de Go. El proceso de codificación implica pérdida de precisión conceptual: tienes que describir valores de datos como texto, y eso no siempre es trivial para tipos de datos complejos. Cuando decodificas dichos datos, debes saber cómo interpretar la representación textual para poder analizar la representación JSON correctamente.

En este capítulo, examinaremos primero la codificación y decodificación de tipos de datos básicos. Luego, veremos algunas recetas que abordan casos de uso y tipos de datos más complicados. Debes utilizar estas recetas como guía al implementar tus propias soluciones. Estas recetas demuestran soluciones para casos de uso particulares y es posible que debas adaptarlas a tus necesidades específicas.

Este capítulo incluye las siguientes recetas:
- Codificación de estructuras (*structs*)
- Manejo de estructuras embebidas
- Codificación sin definir estructuras
- Decodificación de estructuras
- Decodificación con interfaces, mapas y *slices*
- Otras formas de decodificar números
- *Marshaling* y *unmarshaling* de tipos de datos personalizados
- *Marshaling* y *unmarshaling* personalizado de claves de objetos
- Nombres de campos dinámicos
- Estructuras de datos polimórficas
- Procesamiento en *streaming* de datos JSON

---

### Sección 1: Conceptos Básicos de *Marshaling* y *Unmarshaling*

El paquete `encoding/json` de la biblioteca estándar proporciona funciones y convenciones convenientes para codificar y decodificar datos JSON.

#### Codificación de *Structs*

Los tipos de estructura de Go generalmente se codifican como objetos JSON. Esta sección muestra las herramientas de la biblioteca estándar que se encargan de la codificación de tipos de datos.

##### Cómo hacerlo...

Usa etiquetas JSON (*tags*) para anotar los campos de una estructura con sus claves JSON correspondientes:

```go
type Config struct {
  Version string `json:"ver"` // Encoded as "ver"
  Name string // Encoded as "Name"
  Type string `json:"type,omitempty"` // Encoded as "type", and will be omitted if empty
  Style string `json:"-"` // Not encoded
  value string // Unexported field, not encoded
  kind string `json:"kind"` // Unexported field, not encoded
}
```

Usa la función `json.Marshal` para codificar objetos de datos de Go en JSON. La biblioteca estándar utiliza las siguientes convenciones para los tipos básicos:

| Declaración en Go | Valor | Salida JSON |
| --- | --- | --- |
| `NumberValue int json:"num"` | `0` | `"num": 0` |
| `NumberValue *int json:"num"` | `nil` | `"num": null` |
| `NumberValue *int json:"num,omitempty"` | `nil` | *omitido* |
| `BoolValue bool json:"bvalue"` | `true` | `"bvalue": true` |
| `BoolValue *bool json:"bvalue"` | `nil` | `"bvalue": null` |
| `BoolValue *bool json:"bvalue,omitempty"` | `nil` | *omitido* |
| `StringValue string json:"svalue"` | `"str"` | `"svalue":"str"` |
| `StringValue string json:"svalue"` | `""` | `"svalue":""` |
| `StringValue string json:"svalue,omitempty"` | `"str"` | `"svalue":"str"` |
| `StringValue string json:"svalue,omitempty"` | `""` | *omitido* |
| `StringValue *string json:"svalue"` | `nil` | `"svalue": null` |
| `StringValue *string json:"svalue,omitempty"` | `nil` | *omitido* |

- Los tipos `struct` y `map` se serializan como objetos JSON.
- Los tipos `slice` y `array` se serializan como *arrays* JSON.
- Si un tipo implementa la interfaz `json.Marshaler`, se llama al método `json.Marshaler.MarshalJSON` de la instancia de la variable para codificar los datos.
- Si un tipo implementa la interfaz `encoding.TextMarshaler`, el valor se codifica como una cadena JSON y el valor de la cadena se obtiene del método `encoding.TextMarshaler.MarshalText` del valor.
- Cualquier otra cosa fallará con `UnsupportedValueError`.

> **Consejo**  
> Solo los campos exportados de los tipos `struct` se pueden serializar.

> **Consejo**  
> Si no hay etiquetas JSON para un campo de estructura, su clave de objeto JSON será la misma que el nombre del campo.

Considera el siguiente segmento de código:

```go
type Config struct {
  Version string `json:"ver"` // Encoded as "ver"
  Name string // Encoded as "Name"
  Type string `json:"type,omitempty"` // Encoded as "type", and will be omitted if empty
  Style string `json:"-"` // Not encoded
  value string // Unexported field, not encoded
  kind string `json:"kind"` // Unexported field, not encoded
}
...
cfg := Config{
  Version: "1.1",
  Name: "name",
  Type: "example",
  Style: "json",
  value: "example config value",
  kind: "test",
}
data, err := json.Marshal(cfg)
fmt.Println(string(err))
```

Esto imprime lo siguiente:

```json
{"ver":"1.1","Name":"name","type":"example"}
```

> **Consejo**  
> El orden de los campos en el objeto JSON codificado es el mismo que el orden en que se declaran los campos.

---

### Sección 2: Manejo de *Structs* Embebidos

Los campos de un tipo `struct` se codificarán como objetos JSON. Si hay estructuras embebidas, el codificador tiene dos opciones: codificar la estructura embebida en el mismo nivel que la estructura contenedora o como un nuevo objeto JSON anidado.

#### Cómo hacerlo...

Usa etiquetas JSON para nombrar los campos de la estructura contenedora y los campos de la estructura embebida:

```go
type Enclosing struct {
  Field string `json:"field"`
  Embedded
}
type Embedded struct {
  Field string `json:"embeddedField"`
}
```

Usa `json.Marshal` para codificar la estructura como un objeto JSON:

```go
enc := Enclosing{
  Field: "enclosing",
  Embedded: Embedded{
    Field: "embedded",
  },
}
data, err = json.Marshal(enc) // {"field":"enclosing","embeddedField":"embedded"}
```

Agregar una etiqueta JSON a la estructura embebida creará un objeto JSON anidado:

```go
type Enclosing struct {
  Field string `json:"field"`
  Embedded `json:"embedded"`
}
type Embedded struct {
  Field string `json:"embeddedField"`
}
...
enc := Enclosing{
  Field: "enclosing",
  Embedded: Embedded{
    Field: "embedded",
  },
}
data, err = json.Marshal(enc) // {"field":"enclosing","embedded":{"embeddedField":"embedded"}}
```

---

### Sección 3: Codificación sin Definir *Structs*

Los tipos de datos básicos, *slices* y mapas se pueden utilizar para codificar datos JSON.

#### Cómo hacerlo...

Usa un mapa para representar objetos JSON:

```go
config:=map[string]any{
  "ver": "1.0",
  "Name": "config",
  "type": "example",
}
data, err:=json.Marshal(config) // `{"ver":"1.0","Name":"config","type":"example"}`
```

Usa un *slice* para representar *arrays* JSON:

```go
numbersWithNil:=[]any{
  1,
  2,
  nil,
  3,
}
data, err:=json.Marshal(numbersWithNil) // `[1,2,null,3]`
```

Haz coincidir la estructura JSON deseada con los equivalentes de Go:

```go
configurations:=map[string]map[string]any {
  "cfg1": {
    "ver": "1.0",
    "Name": "config1",
  },
  "cfg2": {
    "ver": "1.1",
    "Name" : "config2",
  },
}
data, err:=json.Marshal(configurations) // {"cfg1":{"Name":"config1","ver":"1.0"}, "cfg2":{"Name":"config2","ver":"1.1"}}`
```

---

### Sección 4: Decodificación de *Structs*

Codificar objetos de datos de Go en JSON es una tarea relativamente fácil: los tipos de datos y la semántica bien definidos se traducen a una representación menos expresiva, lo que generalmente genera cierta pérdida de información. Por ejemplo, una variable entera y una variable `float64` pueden codificarse para dar una salida idéntica. Debido a esto, decodificar datos JSON suele ser más difícil.

#### Cómo hacerlo...

1. Usa etiquetas JSON para asignar claves JSON a campos de estructura.
2. Usa la función `json.Unmarshal` para decodificar datos JSON en objetos de datos de Go. La biblioteca estándar utiliza las siguientes convenciones para tipos básicos:

| Entrada JSON | Tipo Go | Resultado |
| --- | --- | --- |
| `"strValue"` | `string` | `"strValue"` |
| `1` (número) | `int` | `1` |
| `1.2` (número) | `int` | `error` |
| `1.2` (número) | `float64`, `float32` | `1.2` |
| `true` | `bool` | `true` |
| `null` | `string` | Variable sin modificar |
| `null` | `int` | Variable sin modificar |
| `"strValue"` | `*string` | `"strValue"` |
| `null` | `*string` | `nil` |
| `1` | `*int` | `1` |
| `null` | `*int` | `nil` |
| `true` | `*bool` | `true` |
| `null` | `*bool` | `nil` |

Si el tipo de Go es `interface{}`, la biblioteca estándar crea objetos según la siguiente convención:

| Entrada JSON | Resultado |
| --- | --- |
| `"strValue"` | `"strValue"` |
| `1` | `float64(1)` |
| `1.2` | `float64(1.2)` |
| `true` | `true` |
| `null` | `nil` |
| Objeto JSON | `map[string]any` |
| Array JSON | `[]any` |

- Si el tipo Go de destino implementa la interfaz `json.Unmarshaler`, se llama a `json.Unmarshal.UnmarshalJSON` para decodificar los datos. Esta operación puede implicar la creación de una nueva instancia del tipo de destino si es necesario.
- Si el tipo Go de destino implementa la interfaz `encoding.TextUnmarshaler` y la entrada es una cadena JSON entre comillas, se llama a `encoding.TextUnmarshaler.UnmarshalText` para decodificar el valor.
- Cualquier otra cosa fallará con `UnsupportedValueError`.

> **Consejo**  
> Los valores numéricos pueden causar confusión si la entrada JSON incluye valores de varios tipos numéricos. Por ejemplo, si un valor numérico JSON se deserializa en un valor `int`, funcionará si los datos JSON se pueden representar como un entero, pero fallará si los datos JSON tienen un valor de punto flotante.

> **Consejo**  
> El decodificador JSON nunca cambiará los campos no exportados de una estructura. El decodificador utiliza reflexión, y solo los campos exportados son accesibles mediante reflexión.

> **Consejo**  
> Los campos JSON que no tengan campos de Go coincidentes se ignorarán.

---

### Sección 5: Decodificación con Interfaces, *Maps* y *Slices*

Al decodificar valores de Go a JSON, los tipos de valores de Go dictan cómo se realizará la codificación JSON. JSON no tiene un sistema de tipos rico como Go. Los tipos válidos de JSON son cadena, número, booleano, objeto, matriz y `null`. Cuando decodificas datos JSON en una estructura de Go, sigue siendo el sistema de tipos de Go el que dicta cómo deben interpretarse los datos JSON. Pero cuando decodificas JSON en una `interface{}`, las cosas cambian. Ahora son los datos JSON los que dictan cómo deben construirse los valores de Go, y esto a veces produce resultados inesperados.

#### Cómo hacerlo...

Para deserializar datos JSON en una interfaz, usa lo siguiente:

```go
var output interface{}
err:=json.Unmarshal(jsonData,&output)
```

Esto crea un árbol de objetos basado en las siguientes reglas de traducción:

| JSON | Go |
| --- | --- |
| Objeto | `map[string]interface{}` |
| Array | `[]interface{}` |
| Número | `float64` |
| Booleano | `bool` |
| Cadena | `string` |
| Null | `nil` |

---

### Sección 6: Otras Formas de Decodificar Números

Cuando se decodifican en una `interface{}`, los números JSON se convierten a `float64`. Este no es siempre el resultado deseado. Puedes usar `json.Number` en su lugar.

#### Cómo hacerlo...

Usa `json.Decoder` con `UseNumber`:

```go
var output interface{}
decoder:=json.NewDecoder(strings.NewReader(`[1.1,2,3,4.4]`))
// Tell the decoder to use json.Number instead of float64
decoder.UseNumber()
err:=decoder.Decode(&output) // [1.1 2 3 4.4]
```

Cada elemento de `output` en el ejemplo anterior es una instancia de `json.Number`. Puedes traducirlo a un `int`, `float64` o `big.Int` según sea necesario.

---

### Sección 7: Manejo de Valores Faltantes y Opcionales

Por lo general, tienes que lidiar con entradas JSON con campos faltantes y tienes que generar JSON donde se omiten los campos vacíos. Esta sección proporciona recetas que muestran cómo lidiar con estos escenarios.

#### Omisión de Campos Vacíos al Codificar

Omitir campos vacíos de la codificación JSON generalmente ahorra espacio y hace que el JSON sea más fácil de leer. Sin embargo, lo que se entiende por "vacío" debe quedar claro.

##### Cómo hacerlo...

Usa la etiqueta JSON `,omitempty` para omitir valores de cadena vacíos, valores enteros/punto flotante cero, valores `time.Duration` cero y valores de puntero `nil`.

La etiqueta `,omitempty` no funciona para valores `time.Time`. Usa `*time.Time` y establécelo en `nil` para omitir valores de tiempo vacíos:

```go
type Config struct {
  ...
  Type string `json:"type,omitempty"`
  IntValue int `json:"intValue,omitempty"`
  FloatValue float64 `json:"floatValue,omitempty"`
  When *time.Time `json:"when,omitempty"`
  HowLong time.Duration `json:"howLong,omitempty"`
}
```

A veces es importante distinguir entre una cadena vacía y una cadena `null`. En JavaScript y JSON, `null` es un valor válido para cadenas. Si ese es el caso, usa `*string`:

```go
type Config struct {
  Value *string `json:"value,omitempty"`
  ...
}
...
emptyString := ""
emptyValue := Config {
  Value: &emptyString,
}
// JSON output: { "value": "" }
nullValue := Config {
  Value: nil,
}
// JSON output: {}
```

#### Manejo de Campos Faltantes al Decodificar

Existen varios casos de uso donde los desarrolladores tienen que lidiar con datos JSON dispersos que no incluyen todos los campos de datos. Por ejemplo, una llamada a la API de actualización parcial puede aceptar un objeto JSON que contenga solo aquellos campos que deben actualizarse, sin modificar ningún campo de datos no especificado. En tales casos, resulta importante identificar qué campos se proporcionaron. Luego, hay casos de uso en los que es apropiado asumir valores predeterminados para los campos faltantes.

##### Cómo hacerlo...

- Si deseas determinar qué campos se especifican en una entrada JSON, utiliza campos de tipo puntero. Cualquier campo que falte en la entrada permanecerá como `nil`.
- Para proporcionar valores predeterminados para los campos faltantes, inicializa esos campos con sus valores predeterminados antes de deserializar:

```go
type APIRequest struct {
  // If type is not specified, it will be nil
  Type *string `json:"type"`
  // There will be a default value for seq
  Seq int `json:"seq"`
  ...
}
func handler(w http.ResponseWriter,r *http.Request) {
  data, err:=io.ReadAll(r.Body)
  if err!=nil {
     http.Error(w, "Bad request",http.StatusBadRequest)
     return
  }
  req:=APIRequest{
     Seq: 1, // Set the default value
  }
  if err:=json.Unmarshal(data, &req); err!=nil {
     http.Error(w, "Bad request", http.StatusBadRequest)
     return
  }
  // Check which fields are provided
  if req.Type!=nil {
     ...
  }
  // If seq is provided in the input, req.Seq will be set to that
  // value. Otherwise, it will be 1.
  if req.Seq==1 {
     ...
  }
}
```

---

### Sección 8: *Marshaling* y *Unmarshaling* de Tipos de Datos Personalizados

Usa estas recetas cuando tengas elementos de datos cuya representación JSON deba generarse mediante programación.

#### Cómo hacerlo...

Para controlar cómo se codifica un objeto de datos en JSON, implementa la interfaz `json.Marshaler`:

```go
// TypeAndID is encoded to JSON as type:id
type TypeAndID struct {
   Type string
   ID int
}
// Implementation of json.Marshaler
func (t TypeAndID) MarshalJSON() (out []byte, err error) {
   s := fmt.Sprintf(`"%s:%d"`,t.Type,t.ID)
   out=[]byte(s)
   return
}
```

Para controlar cómo se decodifica un objeto de datos desde JSON, implementa la interfaz `json.Unmarshaler`:

> **Consejo**  
> Un decodificador (*unmarshaler*) debe tener un receptor de puntero (*pointer receiver*).

```go
// Implementation of json.Unmarshaler. Note the pointer receiver
func (t *TypeAndID) UnmarshalJSON(in []byte) (err error) {
   if len(in)<2 || in[0] != '"' || in[len(in)-1] != '"' {
      err = ErrInvalidTypeAndID
      return
   }
   in = in[1 : len(in)-1]
   parts := strings.Split(string(in), ":")
   if len(parts) != 2 {
      err = ErrInvalidTypeAndID
      return
   }
   // The second part must be a valid integer
   t.ID, err = strconv.Atoi(parts[1])
   if err != nil {
      return
   }
   t.Type = parts[0]
   return
}
```

---

### Sección 9: *Marshaling* y *Unmarshaling* Personalizado de Claves de Objetos

Los mapas se serializan/deserializan como objetos JSON. Pero si tienes un mapa que tiene claves que no son de tipo cadena, ¿cómo puedes serializarlo/deserializarlo a JSON?

#### Cómo hacerlo...

La solución depende del tipo exacto de la clave:

1. Los mapas con tipos de clave derivados de tipos de cadena o enteros se pueden serializar/deserializar utilizando los métodos de la biblioteca estándar:

```go
type Key int64
func main() {
  var m map[Key]int
  err := json.Unmarshal([]byte(`{"123":123}`), &m)
  if err!=nil {
     panic(err)
  }
  fmt.Println(m[123]) // Prints 123
}
```

2. Si las claves del mapa requieren un procesamiento adicional para la serialización/deserialización, implementa las interfaces `encoding.TextMarshaler` y `encoding.TextUnmarshaler`:

```go
// Key is an uint that is encoded as an hex strings for JSON key
type Key uint
func (k *Key) UnmarshalText(data []byte) error {
   v, err := strconv.ParseInt(string(data), 16, 64)
   if err != nil {
      return err
   }
   *k = Key(v)
   return nil
}
func (k Key) MarshalText() ([]byte, error) {
   s := strconv.FormatUint(uint64(k), 16)
   return []byte(s), nil
}
func main() {
   input := `{
      "13AD": "5037",
      "3E22": "15906",
      "90A3": "37027"
   }`
   var data map[Key]string
   if err := json.Unmarshal([]byte(input), &data); err != nil {
      panic(err)
   }
   fmt.Println(data)
   d, err := json.Marshal(map[Key]any{
      Key(123): "123",
      Key(255): "255",
   })
   if err != nil {
      panic(err)
   }
   fmt.Println(string(d))
}
```

---

### Sección 10: Nombres de Campos Dinámicos

Hay casos en los que los nombres de los campos (claves de objetos) no son constantes. Por ejemplo, una API puede preferir devolver una lista de objetos como un objeto JSON donde los identificadores únicos de cada objeto son la clave. En tales casos, no es posible utilizar etiquetas `json` en un `struct`.

#### Cómo hacerlo...

Usa un `map[string]ValueType` para representar un objeto con nombres de campo dinámicos:

```go
type User struct {
  Name string `json:"name"`
  Type string `json:"type"`
}
type Users struct {
  Users map[string]User `json:"users"`
}
func main() {
   input := `{
      "users": {
         "abb64dfe-d4a8-47a5-b7b0-7613fe3fd11f": {
            "name": "John",
            "type": "admin"
         },
         "b158161c-0588-4c67-8e4b-c07a8978f711": {
            "name": "Amy",
            "type": "editor"
         }
      }
   }`
   var users Users
   if err := json.Unmarshal([]byte(input), &users); err != nil {
      panic(err)
   }
}
```

---

### Sección 11: Estructuras de Datos Polimórficas

Una estructura de datos polimórfica puede ser uno de varios tipos diferentes que comparten una interfaz común. El tipo real se determina en tiempo de ejecución. Para los objetos en tiempo de ejecución, el sistema de tipos de Go garantiza operaciones con seguridad de tipos utilizando dichos campos. Con el uso de interfaces, los objetos polimórficos se pueden serializar como JSON fácilmente. Surge un problema cuando necesitas deserializar un objeto JSON polimórfico. En esta receta, veremos una forma de lograrlo.

#### *Unmarshaling* Personalizado en Dos Pasos

El primer paso deserializa los campos discriminadores mientras deja el resto de la entrada sin procesar. Según el discriminador, se construye y deserializa la instancia concreta del objeto.

##### Cómo hacerlo...

Trabajaremos con un ejemplo de estructura `Key` en esta sección. La estructura `Key` contiene diferentes tipos de claves públicas criptográficas, cuyo tipo se indica en un campo `Type`:

```go
type KeyType string
const (
   KeyTypeRSA = "rsa"
   KeyTypeED25519 = "ed25519"
)
type Key struct {
   Type KeyType `json:"type"`
   Key crypto.PublicKey `json:"key"`
}
```

1. Define las etiquetas JSON para la estructura de datos como de costumbre. La mayoría de las estructuras polimórficas se pueden serializar sin un serializador personalizado porque el tipo de objetos en tiempo de ejecución se conoce durante la serialización.
2. Define otra estructura que sea una copia de la original, con las partes de tipo dinámico reemplazadas con un campo de tipo `json.RawMessage`:

```go
type keyUnmarshal struct {
   Type KeyType `json:"type"`
   Key json.RawMessage `json:"key"`
}
```

3. Crea un decodificador para la estructura original. En este decodificador, primero deserializa la entrada a una instancia de la estructura creada en el paso 2:

```go
func (k *Key) UnmarshalJSON(in []byte) error {
   var key keyUnmarshal
   err := json.Unmarshal(in, &key)
   if err != nil {
      return err
   }
```

4. Usando los campos del discriminador de tipos, decide cómo decodificar la parte dinámica. El siguiente ejemplo utiliza una fábrica (*factory*) para obtener un decodificador específico del tipo:

```go
   k.Type = key.Type
   unmarshaler := KeyUnmarshalers[key.Type]
   if unmarshaler == nil {
      return ErrInvalidKeyType
   }
```

5. Deserializa la parte de tipo dinámico (que es un `json.RawMessage`) en una instancia de la variable correctamente tipada:

```go
   k.Key, err = unmarshaler(key.Key)
   if err != nil {
      return err
   }
   return nil
}
```

La fábrica es un mapa simple que conoce los decodificadores para diferentes tipos de claves:

```go
var (
   KeyUnmarshalers = map[KeyType]func(json.RawMessage) (crypto.PublicKey, error){}
)
func RegisterKeyUnmarshaler(keyType KeyType, unmarshaler func(json.RawMessage) (crypto.PublicKey, error)) {
   KeyUnmarshalers[keyType] = unmarshaler
}
...
RegisterKeyUnmarshaler(KeyTypeRSA, func(in json.RawMessage) (crypto.PublicKey, error) {
   var key rsa.PublicKey
   if err := json.Unmarshal(in, &key); err != nil {
      return nil, err
   }
   return &key, nil
})
RegisterKeyUnmarshaler(KeyTypeED25519, func(in json.RawMessage) (crypto.PublicKey, error) {
   var key ed25519.PublicKey
   if err := json.Unmarshal(in, &key); err != nil {
      return nil, err
   }
   return &key, nil
})
```

Este es un marco de fábrica extensible que se puede inicializar con decodificadores adicionales determinados en el momento de la compilación. Simplemente crea una función decodificadora para un tipo de objeto y regístrala usando la función anterior `RegisterKeyUnmarshaler` para admitir nuevos tipos de claves.

> **Consejo**  
> Una forma común de registrar tales características es utilizar la función `init()` de los paquetes. Cuando importas ese paquete, se registrarán los tipos de decodificadores admitidos por el paquete.

---

### Sección 12: Procesamiento en *Streaming* de Datos JSON

Cuando tienes que lidiar con grandes cantidades de datos de manera eficiente, debes considerar la transmisión de datos en *streaming* en lugar de trabajar en todo el conjunto de datos a la vez. Esta sección describe algunos métodos para transmitir datos JSON.

#### Procesamiento en *Streaming* de un *Array* de Objetos

Esta receta es útil si tienes un generador (una *goroutine*, un cursor de base de datos, etc.) que produce elementos de datos y deseas transmitirlos como un *array* JSON en lugar de almacenar todo antes de serializarlo.

##### Cómo hacerlo...

1. Crea un generador. Este puede ser:
   - una *goroutine* que envía elementos de datos a través de un canal,
   - un objeto similar a un cursor que contiene un método `Next()`,
   - o algún otro generador de datos.
2. Crea una instancia de `json.Encoder` con `io.Writer` que represente el destino. El destino puede ser un archivo, salida estándar, un búfer, una conexión de red, etc.
3. Escribe el delimitador de inicio de *array*, es decir, `[`.
4. Codifica cada elemento de datos, precedido por una coma si es necesario.
5. Escribe el delimitador de cierre de *array*, es decir, `]`.

El siguiente ejemplo asume que hay una *goroutine* generadora que escribe instancias de `Data` en el canal de entrada. El generador cierra el canal cuando no hay más instancias de `Data`. Aquí, asumimos que `Data` es serializable en JSON:

```go
func stream(out io.Writer, input <-chan Data) error {
   enc := json.NewEncoder(out)
   if _, err := out.Write([]byte{'['}); err != nil {
      return err
   }
   first := true
   for obj := range input {
      if first {
         first = false
      } else {
         if _, err := out.Write([]byte{','}); err != nil {
            return err
         }
      }
      if err := enc.Encode(obj); err != nil {
         return err
      }
   }
   if _, err := out.Write([]byte{']'}); err != nil {
      return err
   }
   return nil
}
```

#### Parseo de un *Array* de Objetos

Si tienes una fuente de datos JSON que proporciona un *array* de objetos, puedes analizar estos elementos y procesarlos usando `json.Decoder`.

##### Cómo hacerlo...

1. Crea un `json.Decoder` que lea del flujo de entrada.
2. Analiza el delimitador de inicio de matriz (`[`) usando `json.Decoder.Token()`.
3. Decodifica cada elemento de la matriz hasta que falle la decodificación.
4. Cuando falla la decodificación, debes determinar si la secuencia terminó o si realmente hay un error. Para verificar eso, lee el siguiente token usando `json.Decoder.Token()`. Si el siguiente token se lee correctamente y si es un delimitador de fin de matriz, `]`, entonces el análisis del flujo finalizó con éxito. De lo contrario, hay un error en los datos de entrada.

El siguiente ejemplo supone que `json.Decoder` ya está construido para leer desde un flujo de entrada. La salida se almacena en un *slice*. Alternativamente, la salida se puede procesar a medida que se analizan los elementos, o cada elemento se puede enviar a una *goroutine* de procesamiento a través de un canal:

```go
func parse(input *json.Decoder) (output []Data, err error) {
   // Parse the array beginning delimiter
   var tok json.Token
   tok, err = input.Token()
   if err != nil {
      return
   }
   if tok != json.Delim('[') {
      err = fmt.Errorf("Array begin delimiter expected")
      return
   }
   // Parse array elements using Decode
   for {
      var data Data
      err = input.Decode(&data)
      if err != nil {
         // Decode failed. Either there is an input error, or
         // we are at the end of the stream
         tok, err = input.Token()
         if err != nil {
            // Data error
            return
         }
         // Are we at the end?
         if tok == json.Delim(']') {
            // Yes, there is no error
            err = nil
            break
         }
      }
      output = append(output, data)
   }
   return
}
```

#### Otras Formas de Realizar *Streaming* de JSON

Hay otras formas de transmitir JSON:
- **JSON concatenado**: Simplemente escribe objetos JSON uno tras otro.
- **JSON delimitado por nuevas líneas (NDJSON/JSON Lines)**: Escribe cada objeto JSON como una línea separada.
- **JSON delimitado por separador de registros**: Utiliza un carácter especial de separador de registros, `0x1E`, y opcionalmente una nueva línea entre cada objeto JSON.
- **JSON con prefijo de longitud**: Prefija la longitud de la cadena de cada objeto JSON como un número decimal.

Todo esto se puede leer y escribir utilizando `json.Decoder` y `json.Encoder`.

---

### Sección 13: Consideraciones de Seguridad

Siempre que aceptes datos del exterior de tu aplicación (datos ingresados por el usuario, llamadas a la API, lectura de un archivo, etc.), debes preocuparte por las entradas maliciosas. La entrada JSON es relativamente segura porque los analizadores JSON no realizan expansiones de datos como lo hacen los analizadores YAML o XML. Sin embargo, todavía hay cosas que debes considerar al tratar con datos JSON.

#### Cómo hacerlo...

Limita la cantidad de datos al aceptar entradas JSON de terceros. No uses a ciegas `io.ReadAll` o `json.Decode`:

```go
const MessageSizeLimit = 10240
func handler(w http.ResponseWriter, r *http.Request) {
  reader:=http.MaxBytesReader(w,r.Body,MessageSizeLimit)
  data, err := io.ReadAll(reader)
  if errors.Is(err,&http.MaxBytesError{}) {
     // If this happens, error is already sent.
     return
  }
  ...
}
```

Proporciona siempre un límite superior para las asignaciones de recursos según los datos que leas de una entrada de terceros. Por ejemplo, si estás leyendo un flujo JSON con prefijo de longitud donde cada objeto JSON tiene como prefijo su longitud, no asignes un `[]byte` para almacenar el siguiente objeto sin verificar. Rechaza la entrada si la longitud es demasiado grande.

