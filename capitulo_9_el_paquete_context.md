# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 9: El Paquete *Context*

El contexto son las circunstancias en las que ocurre algo. Cuando hablamos de un programa, el contexto es el entorno del programa, la configuración, etc. Para un programa servidor (un servidor HTTP que responde a la solicitud de un cliente, un servidor RPC que responde a llamadas a funciones, etc.) o un programa que responde a solicitudes de usuarios (un programa interactivo, una aplicación de línea de comandos, etc.), se puede hablar de un contexto específico de la solicitud (*request-specific context*). Un contexto específico de la solicitud se crea cuando el servidor o programa comienza a procesar una solicitud particular y termina cuando finaliza dicho procesamiento. El contexto de la solicitud contiene información como un identificador de solicitud que te ayuda a identificar los mensajes de registro (*logs*) generados mientras se procesa una solicitud, o la identidad del llamador para que puedas determinar sus derechos de acceso. Uno de los usos del paquete `context` es proporcionar una abstracción de dicho contexto de solicitud, es decir, un objeto que mantiene datos específicos de la solicitud.

También puedes tener dudas sobre el tiempo de ejecución de una solicitud. Por lo general, deseas limitar la cantidad de tiempo que se procesa una solicitud, o tal vez desees detectar que el cliente ya no está interesado en los resultados de la solicitud (como cuando un par de WebSocket se desconecta). El paquete `context` está diseñado para manejar estos casos de uso también.

El paquete `context` define la interfaz `context.Context`. Tiene dos usos principales:
- Agregar un tiempo de espera (*timeout*) y/o cancelación al procesamiento de solicitudes.
- Pasar metadatos específicos de la solicitud hacia abajo en la pila de llamadas (*stack*).

El uso de `context.Context` no se limita a programas de servidor. El término "procesamiento de solicitudes" debe entenderse en un sentido amplio: la solicitud puede ser una petición de red a través de una conexión TCP, una petición HTTP, un comando leído desde una línea de comandos, la ejecución de un programa con un determinado flag, y así sucesivamente. Por lo tanto, los usos de `context.Context` son mucho más diversos.

En este capítulo, aprenderás sobre lo siguiente:
- Uso de `context` para pasar datos de ámbito de petición (*request-scoped*)
- Uso de `context` para cancelaciones y tiempos de espera (*timeouts*)

---

### Sección 1: Uso de *Context* para Pasar Datos de Ámbito de Petición (*Request-Scoped*)

Los objetos de ámbito de petición (*request-scoped*) son aquellos que se crean cuando comienza el procesamiento de la solicitud y se descartan cuando finaliza dicho procesamiento. Por lo general, se trata de objetos ligeros, como un identificador de solicitud, información de autenticación que identifica al llamador o *loggers*. En esta sección, verás cómo se pueden transmitir estos objetos utilizando un contexto.

#### Cómo hacerlo...

La forma idiomática de agregar valores de datos a un contexto es la siguiente:

1. Define un tipo de clave de contexto. Esto evita colisiones de nombres accidentales. El uso de un nombre de tipo no exportado como el siguiente es común. Este patrón limita la capacidad de colocar u obtener valores de contexto de este tipo particular al paquete actual:

```go
type requestIDKeyType int
```

> **Advertencia**  
> Podrías tener la tentación de utilizar `struct{}` en lugar de `int` aquí. Después de todo, `struct{}` no consume memoria adicional. Tienes que tener mucho cuidado al trabajar con estructuras de tamaño 0, ya que la especificación del lenguaje Go no ofrece garantías sobre la equivalencia de dos estructuras de tamaño 0. Es decir, si creas múltiples variables de un tipo de tamaño 0, a veces pueden ser iguales y a veces no. En resumen, no uses `struct{}` para esto.

2. Define el valor o los valores de clave utilizando el tipo de clave. En la siguiente línea de código, `requestIDKey` se define como de tipo `requestIDKeyType` con el valor 0 (`requestIDKey` se inicializa con su valor 0 al declararse):

```go
var requestIDKey requestIDKeyType
```

3. Usa `context.WithValue` para agregar el nuevo valor al contexto. Puedes definir un par de funciones auxiliares para establecer y obtener valores hacia y desde el contexto:

```go
func WithRequestID(ctx context.Context,requestID string) context.Context {
  return context.WithValue(ctx,requestIDKey,requestID)
}
func GetRequestID(ctx context.Context) string {
  id,_:=ctx.Value(requestIDKey).(string)
  return id
}
```

4. Pasa el nuevo contexto a las funciones llamadas desde la función actual:

```go
newCtx:=WithRequestID(ctx,requestID)
handleRequest(newCtx)
```

#### Cómo funciona...

Habrás notado que `context.Context` no se parece exactamente a un mapa de clave-valor (no existe un método `SetValue`; de hecho, `context.Context` es inmutable), aunque puedes usarlo para almacenar pares clave-valor. De hecho, no puedes agregar un par clave-valor a un contexto existente, pero puedes obtener un nuevo contexto que contenga ese par clave-valor mientras conservas el contexto anterior. Los contextos tienen capas como una cebolla; cada adición a un contexto crea un nuevo contexto que está vinculado al anterior, pero con más características:

```go
// ctx: An empty context
ctx := context.Background()
// ctx1: ctx + {key1:value1}
ctx1 := context.WithValue(ctx, "key1", "value1")
// ctx2: ctx1 + {key2:value2}
ctx2 := context.WithValue(ctx, "key2", "value2")
```

En el código anterior, `ctx`, `ctx1` y `ctx2` son tres contextos diferentes. El contexto `ctx` está vacío. `ctx1` contiene `ctx` y el par clave-valor `key1: value1`. `ctx2` contiene `ctx1` y el par clave-valor `key2: value2`. Por lo tanto, si haces lo siguiente:

```go
val1,_ := ctx2.Value("key1")
val2,_ := ctx2.Value("key2")
fmt.Println(val1, val2)
```

Esto imprimirá:

```text
value1 value2
```

Si haces lo mismo con `ctx1`:

```go
val1,_ = ctx1.Value("key1")
val2,_ = ctx1.Value("key2")
fmt.Println(val1, val2)
```

Esto imprimirá:

```text
value1 <nil>
```

Lo siguiente se usa para `ctx`:

```go
val1,_ = ctx.Value("key1")
val2,_ = ctx.Value("key2")
fmt.Println(val1, val2)
```

Esto imprimirá:

```text
<nil> <nil>
```

> **Consejo**  
> Aunque no puedes establecer los valores dentro de un contexto directamente (es decir, un contexto es inmutable), puedes almacenar un puntero a una estructura y modificar los valores dentro de esa estructura:
>
> ```go
> type ctxData struct {
>   value int
> }
> ...
> ctx:=context.WithValue(context.Background(),dataKey, &ctxData{})
> ...
> if data,exists:=ctx.Value(dataKey); exists {
>   data.(*ctxData).value=1
> }
> ```

La biblioteca estándar proporciona un par de valores de contexto predefinidos:
- `context.Background()` devuelve un contexto que no tiene valores y que no se puede cancelar. Este suele ser el contexto base para la mayoría de las operaciones.
- `context.TODO()` es similar a `context.Background()` con un nombre que indica que el lugar donde se use debe refactorizarse eventualmente para aceptar un contexto real.

#### Y hay más...

Un contexto suele compartirse entre varias *goroutines*. Debes tener cuidado con los problemas de concurrencia, especialmente si colocas punteros a objetos en un contexto. Echa un vistazo al siguiente ejemplo, que muestra un *middleware* de autenticación para un servicio HTTP:

```go
type AuthInfo struct {
  // Set when AuthInfo is created
  UserID string
  // Lazy-initialized
  privileges map[string]Privilege
}
type authInfoKeyType int
var authInfoKey authInfoKeyType
// Set the privileges if is it not initialized.
// Do not do this!!
func (auth *AuthInfo) GetPrivileges() map[string]Privilege {
   if auth.privileges==nil {
      auth.privileges=GetPrivileges(auth.UserID)
   }
   return auth.privileges
}
// Authentication middleware
func AuthMiddleware(next http.Handler) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.
        Request) {
            // Authenticate the caller
            var authInfo *AuthInfo
            var err error
            authInfo, err = authenticate(r)
            if err != nil {
                http.Error(w, err.Error(), http.StatusUnauthorized)
                return
            }
            // Create a new context with the authentication info
            newCtx := context.WithValue(r.Context(), authInfoKey, 
            authInfo)
            // Pass the new context to the next handler
            next.ServeHTTP(w, r.WithContext(newCtx))
        })
    }
}
```

El *middleware* de autenticación crea una instancia de `*AuthInfo` y llama al siguiente *handler* en la cadena usando un contexto con la información de autenticación. El problema en este código es que `*AuthInfo` contiene un campo `privileges` que se inicializa cuando se llama a `AuthInfo.GetPrivileges`. Dado que los *handlers* pueden pasar el contexto a múltiples *goroutines*, este esquema de inicialización perezosa (*lazy initialization*) es propenso a carreras de datos; varias *goroutines* que llamen a `AuthInfo.GetPrivileges` pueden intentar inicializar el mapa varias veces, sobrescribiéndose mutuamente.

Esto se puede corregir usando un mutex:

```go
type AuthInfo struct {
  sync.Mutex
  UserID string
  privileges map[string]Privilege
}
func (auth *AuthInfo) GetPrivileges() map[string]Privilege {
   // Use mutex to initialize the privileges in a thread-safe way
   auth.Lock()
   defer auth.Unlock()
   if auth.privileges==nil {
      auth.privileges=GetPrivileges(auth.UserID)
   }
   return auth.privileges
}
```

También se puede corregir inicializando los privilegios una sola vez en el *middleware*:

```go
     authInfo, err=authenticate(r)
     if err!=nil {
       http.Error(w,err.Error(),http.StatusUnauthorized)
       return
     }
     // Initialize the privileges here when the structure is created
     authInfo.GetPrivileges()
```

---

### Sección 2: Uso de *Context* para Cancelaciones

Hay varias razones por las que podrías querer cancelar un cálculo: el cliente puede haberse desconectado, o puedes tener múltiples *goroutines* trabajando en un cálculo y una de ellas falló, por lo que ya no deseas que las demás continúen. Puedes utilizar otros métodos, como un canal `done` que cierras para señalar la cancelación, pero un contexto puede ser más conveniente según el caso de uso. Un contexto se puede cancelar muchas veces (solo la primera llamada realmente cancelará; las restantes serán ignoradas), mientras que no puedes cerrar un canal ya cerrado ya que generará un pánico. Además, puedes crear un árbol de contextos donde cancelar un contexto solo cancela las *goroutines* controladas por él, sin afectar a las demás.

#### Cómo hacerlo...

Estos son los pasos para crear un contexto cancelable y detectar una cancelación:

1. Usa `context.WithCancel` para crear un nuevo contexto cancelable basado en un contexto existente y una función de cancelación:

```go
ctx:=context.Background()
cancelable, cancel:=context.WithCancel(ctx)
defer cancel()
```

Asegúrate de que la función `cancel` se llame eventualmente. La cancelación libera los recursos asociados con el contexto.

2. Pasa el contexto cancelable a cálculos o *goroutines* que se puedan cancelar:

```go
go cancelableGoroutine1(cancelable)
go cancelableGoroutine2(cancelable)
cancelableFunc(cancelable)
```

3. En la función cancelable, comprueba si el contexto se cancela utilizando el canal `ctx.Done()` o `ctx.Err()`:

```go
func cancelableFunc(ctx context.Context) {
  // Process some data
  // Check context cancelation
  select {
     case <-ctx.Done():
        // Context canceled
        return
     default:
  }
  // Continue computation
}
```

O utiliza lo siguiente:

```go
 func cancelableFunc(ctx context.Context) {
   // Process some data
   // Check context cancelation
   if ctx.Err()!=nil {
       // Context canceled
       return
   }
   // Continue computation
}
```

4. Para cancelar una función manualmente, llama a la función de cancelación:

```go
ctx:=context.Background()
cancelable, cancel:=context.WithCancel(ctx)
defer cancel()
wg:=sync.WaitGroup{}
wg.Add(1)
go cancelableGoroutine1(cancelable,&wg)
if err:=process(ctx); err!=nil {
   // Cancel the context
   cancel()
   // Do other things
}
wg.Wait()
```

Asegúrate de que la función `cancel` se llame eventualmente (usa `defer cancel()`):

```go
cancelable, cancel := context.WithCancel(ctx)
defer cancel()
...
```

> **Advertencia**  
> Asegurarse de que se llame a `cancel` es importante. Si no cancelas un contexto cancelable, las *goroutines* asociadas con ese contexto se fugarán (*leak*) (es decir, no habrá forma de terminar las *goroutines* y consumirán memoria).

> **Consejo**  
> La función `cancel` se puede llamar varias veces. Las llamadas posteriores serán ignoradas.

#### Cómo funciona...

`context.WithCancel` devuelve un nuevo contexto y la clausura `cancel`. El contexto devuelto es un contexto cancelable basado en el contexto original:

```go
// Empty context, no cancelation
originalContext := context.Background()
// Cancelable context based on originalContext
cancelableContext1, cancel1 := context.WithCancel(originalContext)
```

Puedes usar este contexto para controlar varias *goroutines*:

```go
go f1(cancelableContext1)
go f2(cancelableContext1)
```

También puedes crear otros contextos cancelables basados en un contexto cancelable:

```go
cancelableContext2, cancel2 := context.WithCancel(cancelableContext)
go g1(cancelableContext2)
go g2(cancelableContext2)
```

Ahora tenemos dos contextos cancelables. Llamar a `cancel2` solo cancelará `cancelableContext2`:

```go
cancal2() // canceling g1 and g2 only
```

Llamar a `cancel1` cancelará tanto `cancelableContext1` como `cancelableContext2`:

```go
cancel1() // canceling f1, f2, g1, g2
```

La cancelación de contexto no es una forma automatizada de cancelar *goroutines*. Tienes que comprobar la cancelación del contexto y limpiar en consecuencia:

```go
func f1(cancelableContext context.Context) {
   for {
      if cancelableContext.Err()!=nil {
         // Context is canceled
         // Cleanup and return
         return
      }
      // Process
   }
}
```

---

### Sección 3: Uso de *Context* para Tiempos de Espera (*Timeouts*)

Un tiempo de espera (*timeout*) es simplemente una cancelación automatizada. El contexto se cancelará una vez transcurrido un temporizador. Esto es útil para limitar el consumo de recursos en cálculos que es poco probable que finalicen.

#### Cómo hacerlo...

Estos son los pasos para crear un contexto con tiempo de espera y detectar cuándo ocurre un evento de tiempo de espera:

1. Usa `context.WithTimeout` para crear un nuevo contexto cancelable que se cancelará automáticamente después de una duración determinada según un contexto existente y una función de cancelación:

```go
ctx:=context.Background()
timeoutable, cancel:=context.WithTimeout(ctx,5*time.Second)
defer cancel()
```

Alternativamente, puedes usar `WithDeadline` para cancelar el contexto en un momento dado.

2. Asegúrate de que la función `cancel` se llame eventualmente.

3. Pasa el contexto de tiempo de espera a cálculos o *goroutines* que pueden agotar el tiempo de espera:

```go
go longRunningGoroutine1(timeoutable)
go longRunningGoroutine2(timeoutable)
```

4. En la *goroutine*, comprueba si el contexto se cancela utilizando el canal `ctx.Done()` o `ctx.Err()`:

```go
func longRunningGoroutine(ctx context.Context) {
  // Process some data
  // Check context cancelation
  select {
     case <-ctx.Done():
        // Context canceled
        return
     default:
  }
  // Continue computation
}
```

Alternativamente, utiliza lo siguiente:

```go
 func cancelableFunc(ctx context.Context) {
   // Process some data
   // Check context cancelation
   if ctx.Err()!=nil {
       // Context canceled
       return
   }
   // Continue computation
}
```

5. Para cancelar una función manualmente, llama a la función de cancelación:

```go
ctx:=context.Background()
timeoutable, cancel:=context.WithTimeout(ctx, 5*time.Second)
defer cancel()
wg:=sync.WaitGroup{}
wg.Add(1)
go longRunningGoroutine(timeoutable,&wg)
if err:=process(ctx); err!=nil {
   // Cancel the context
   cancel()
   // Do other things
}
wg.Wait()
```

Asegúrate de que la función `cancel` se llame eventualmente (usa `defer cancel()`):

```go
timeoutable, cancel := context.WithTimeout(ctx,5*time.Second)
defer cancel()
...
```

#### Cómo funciona...

La función de tiempo de espera es simplemente una cancelación con un temporizador adjunto. Cuando el temporizador expira, el contexto se cancela.

#### Y hay más...

Puede haber situaciones en las que una *goroutine* se bloquee sin ninguna forma obvia de cancelarla. Por ejemplo, puedes bloquearte esperando leer de una conexión de red:

```go
func readData(conn net.Conn) {
  // Read a block of data from the connection
  msg:=make([]byte,1024)
  n, err:=conn.Read(msg)
  ...
}
```

Esta operación no se puede cancelar porque `Read` no acepta `Context`. Si deseas cancelar dicha operación, puedes cerrar la conexión subyacente (o archivo) de forma asíncrona. El siguiente fragmento de código demuestra un caso de uso donde todos los datos de una conexión deben leerse en un segundo, o una *goroutine* cerrará la conexión de forma asíncrona:

```go
timeout, cancel := context.WithTimeout(context.Background(),1*time.Second)
defer cancel()
// Close the connection when context times out
go func() {
   // Wait for cancelation signal
   <-cancelable.Done()
   // Close the connection
   conn.Close()
}()
wg:=sync.WaitGroup()
wg.Add(1)
// This goroutine must complete within a second, or the connection 
// will be closed
go func() {
   defer wg.Done()
    // Read a block of data from the connetion
   msg:=make([]byte,1024)
   // This call may block
   n, err:=conn.Read(msg)
   if err!=nil {
      return
   }
   // Process data
}()
wg.Wait() // Wait for the processing of connection to complete
...
```

---

### Sección 4: Uso de Cancelaciones y *Timeouts* en Servidores

Los servidores de red generalmente inician un nuevo contexto cuando se recibe una nueva solicitud. Normalmente, el servidor cancela el contexto cuando el solicitante cierra la conexión. La mayoría de los *frameworks* HTTP, incluida la biblioteca estándar, siguen este patrón básico. Si estás escribiendo tu propio servidor TCP, debes implementarlo tú mismo.

#### Cómo hacerlo...

Estos son los pasos para manejar conexiones de red con un tiempo de espera o cancelación:
1. Cuando aceptes una conexión de red, crea un nuevo contexto con una cancelación o tiempo de espera.
2. Asegúrate de que el contexto se cancele eventualmente.
3. Pasa el contexto al *handler*:

```go
ln, err:=net.Listen("tcp",":8080")
if err!=nil {
  return err
}
for {
  conn, err:=ln.Accept()
  if err!=nil {
    return err
  }
  go func(c net.Conn) {
     // Step 1:
     // Request times out after duration: RequestTimeout
     ctx, cancel:=context.WithTimeout(context.
     Background(),RequestTimeout)
     // Step 2:
     // Make sure cancel is called
     defer cancel()
     // Step 3:
     // Pass the context to handler
     handleRequest(ctx,c)
  }(conn)
}
```

