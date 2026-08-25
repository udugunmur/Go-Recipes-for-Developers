# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 7: Concurrencia

La concurrencia es una parte fundamental del lenguaje Go. A diferencia de muchos otros lenguajes que admiten la concurrencia a través de ricas bibliotecas de subprocesos múltiples (*multithreading*), Go proporciona relativamente pocas primitivas de lenguaje para escribir programas concurrentes.

Comencemos enfatizando que la concurrencia no es paralelismo. La concurrencia tiene que ver con cómo escribes los programas; el paralelismo tiene que ver con cómo se ejecutan los programas. Un programa concurrente especifica qué partes del programa pueden ejecutarse en paralelo. Dependiendo de la ejecución real, las partes concurrentes de un programa pueden ejecutarse secuencialmente, concurrentemente mediante tiempo compartido (*time-sharing*) o en paralelo. Un programa concurrente correcto produce el mismo resultado independientemente de cómo se ejecute.

Este capítulo introduce algunas de las primitivas de concurrencia de Go mediante recetas. En este capítulo, aprenderás sobre lo siguiente:
- Creación de *goroutines*
- Ejecución concurrente de múltiples funciones independientes y espera hasta su finalización
- Envío y recepción de datos mediante canales (*channels*)
- Envío de datos a un canal desde múltiples *goroutines*
- Recopilación de los resultados de cálculos concurrentes mediante canales
- Trabajo con múltiples canales usando la sentencia `select`
- Cancelación de una *goroutine*
- Detección de cancelación usando `select` no bloqueante
- Actualización de variables compartidas de forma concurrente

---

### Sección 1: Realización de Tareas Concurrentes Mediante *Goroutines*

Una *goroutine* es una función que se ejecuta de forma concurrente con otras *goroutines*. Cuando se inicia un programa, el entorno de ejecución de Go (*Go runtime*) crea varias *goroutines*. Una de estas *goroutines* ejecuta el recolector de basura (*garbage collector*). Otra *goroutine* ejecuta la función `main`. A medida que el programa se ejecuta, crea más *goroutines* según sea necesario. Un programa típico de Go puede tener miles de *goroutines* ejecutándose todas simultáneamente. El entorno de ejecución de Go programa estas *goroutines* en hilos del sistema operativo. A cada hilo del sistema operativo se le asigna una serie de *goroutines* que ejecuta mediante tiempo compartido. En cualquier momento dado, puede haber tantas *goroutines* activas como el número de procesadores lógicos:

```text
Number of threads per core * Number of cores per CPU * Number of CPUs
```

#### Creación de *Goroutines*

Las *goroutines* son una parte integral del lenguaje Go. Creas *goroutines* usando la palabra clave `go`.

##### Cómo hacerlo...

Crea *goroutines* usando la palabra clave `go` seguida de una llamada a una función:

```go
func f() {
  // Do some work
}
func main() {
  go f()
  ...
}
```

Cuando se evalúa `go f()`, el entorno de ejecución crea una nueva *goroutine* y llama a la función `f`. La *goroutine* que ejecuta `main` también continúa ejecutándose. En otras palabras, cuando se evalúa la palabra clave `go`, la ejecución del programa se divide en dos flujos de ejecución concurrentes: uno es el flujo de ejecución original (en el ejemplo anterior, el flujo que está ejecutando `main`) y el otro ejecuta la función que viene después de la palabra clave `go`.

La función puede tomar argumentos si es necesario:

```go
func f(i int) {
  // Do some work
}
func main() {
  var x int
  go f(x)
  ...
}
```

Los argumentos de la función se evalúan antes de que comience la *goroutine*. Es decir, la *goroutine* principal primero evalúa el argumento de `f` (que es, en este caso, el valor `x`) y luego crea una nueva *goroutine* y ejecuta `f`.

Es una práctica común utilizar una clausura (*closure*) para ejecutar *goroutines*. Proporcionan el contexto necesario para comprender el código y también evitan pasar muchas variables como argumentos a las *goroutines*:

```go
func main() {
  var x int
  var y int
  ...
  go func(i int) {
    if y > 0 {
      // Do some work
    }
  }(x)
  ...
}
```

Aquí, `x` se pasa como argumento a la *goroutine*, pero `y` se captura del entorno.

Cuando la función ejecutada por la palabra clave `go` finaliza, la *goroutine* termina.

#### Ejecución Concurrente de Múltiples Funciones Independientes y Espera hasta su Finalización

Cuando tienes múltiples funciones independientes que no comparten datos, puedes usar esta receta para ejecutarlas concurrentemente. También utilizaremos `sync.WaitGroup` para esperar a que terminen las *goroutines*.

##### Cómo hacerlo...

1. Crea una instancia de `sync.WaitGroup` para esperar a las *goroutines*:

```go
wg := sync.WaitGroup{}
```

Un `sync.WaitGroup` es simplemente un contador seguro para subprocesos (*thread-safe*). Usaremos `wg.Add(1)` para cada *goroutine* que creemos, y usaremos `wg.Done()` para restar 1 cada vez que termine una *goroutine*. Luego podemos esperar a que el grupo de espera llegue a cero, lo que indica la terminación de todas las *goroutines*.

2. Para cada función que se ejecutará concurrentemente, haz lo siguiente:
   - Agrega 1 al grupo de espera
   - Inicia una nueva *goroutine*
   - Llama a `defer wg.Done()` para asegurarte de señalar la terminación de la *goroutine*

```go
wg.Add(1)
go func() {
  defer wg.Done()
  // Do work
}()
```

> **Consejo**  
> En lugar de agregar 1 al grupo de espera para cada *goroutine*, simplemente puedes agregar el número total de *goroutines*. Por ejemplo, si sabes que crearás 5 *goroutines*, simplemente puedes hacer `wg.Add(5)` antes de crear la primera *goroutine*.

3. Espera a que terminen las *goroutines*:

```go
wg.Wait()
```

Esta llamada se bloqueará hasta que `wg` llegue a cero, es decir, hasta que todas las *goroutines* llamen a `wg.Done()`.

4. Ahora puedes utilizar los resultados de todas las *goroutines*.

El detalle crucial de esta receta es que todas las *goroutines* son independientes, lo que significa lo siguiente:
- Todas las variables escritas por cada *goroutine* son utilizadas exclusivamente por esa *goroutine* hasta `wg.Done()`. Las *goroutines* pueden leer variables compartidas, pero no pueden escribir en ellas. Después de `wg.Done()`, todas las *goroutines* han terminado y las variables que escribieron se pueden utilizar de forma segura.
- Ninguna *goroutine* depende del resultado de otra *goroutine*.
- No debes intentar leer los resultados de una *goroutine* antes de `wg.Wait`. Eso constituiría una condición de carrera de memoria con comportamiento indefinido.

Una condición de carrera de memoria (*memory race*) ocurre cuando escribes en una variable compartida de forma concurrente con otras escrituras o lecturas. El resultado de un programa que contiene una carrera de memoria es indefinido.

---

### Sección 2: Comunicación entre *Goroutines* Mediante Canales (*Channels*)

La mayoría de las veces, múltiples *goroutines* tienen que comunicarse y coordinarse para distribuir el trabajo, gestionar el estado y cotejar los resultados de los cálculos. Los canales son el mecanismo preferido para esto. Un canal es un mecanismo de sincronización con un búfer opcional de tamaño fijo.

> **Consejo**  
> Las siguientes recetas muestran canales que se cierran. Cerrar un canal es un método para comunicar el fin de los datos. Si no cierras un canal, será recolectado por el recolector de basura cuando ya no esté referenciado. En otras palabras, no necesitas cerrar un canal si no necesitas señalar el fin de los datos a los receptores.

#### Envío y Recepción de Datos Mediante Canales

Una *goroutine* puede enviar a un canal si hay otra *goroutine* esperando recibir de él o, en el caso de un canal con búfer (*buffered channel*), si hay espacio disponible en el búfer del canal. De lo contrario, la *goroutine* se bloquea hasta que pueda enviar.

Una *goroutine* puede recibir de un canal si hay otra *goroutine* esperando enviar a él o, en el caso de un canal con búfer, si hay datos en el búfer del canal. De lo contrario, el receptor se bloquea hasta que pueda recibir.

##### Cómo hacerlo...

1. Crea un canal con el tipo de datos que transmitirá. El siguiente ejemplo crea un canal que puede transmitir cadenas:

```go
ch := make(chan string)
```

2. En una *goroutine*, envía elementos de datos al canal. Cuando se envíen todos los elementos de datos, cierra el canal:

```go
go func() {
  for _, str := range stringData {
     // Send the string to the channel. This will block until
     // another goroutine can receive from the channel.
     ch <- str
  }
  // Close the channel when done. This is the way to signal the
  // receiver goroutine that there is no more data available.
  close(ch)
}()
```

3. Recibe datos del canal en otra *goroutine*. En el siguiente ejemplo, la *goroutine* principal recibe cadenas del canal y las imprime. El bucle `for` finaliza cuando se cierra el canal:

```go
for str := range ch {
  fmt.Println(str)
}
```

#### Envío de Datos a un Canal desde Múltiples *Goroutines*

Hay casos en los que tienes muchas *goroutines* trabajando en una parte de un problema y, cuando terminan, envían el resultado usando un canal. Un problema en esta situación es decidir cuándo cerrar el canal. Esta receta muestra cómo se hace.

##### Cómo hacerlo...

1. Crea el canal de resultados con el tipo de datos que transmitirá:

```go
ch := make(chan string)
```

2. Crea la *goroutine* de escucha y un grupo de espera para esperar su finalización más tarde. Esta *goroutine* se bloqueará hasta que las otras *goroutines* comiencen a enviar datos:

```go
// Allocate results
results := make([]string,0)
// WaitGroup will be used later to wait for the listener 
// goroutine to end
listenerWg := sync.WaitGroup{}
listenerWg.Add(1)
go func() {
  defer listenerWg.Done()
  // Collect results and store in a slice
  for str:=range ch {
    results=append(results,str)
  }
}()
```

3. Crea un grupo de espera para realizar un seguimiento de las *goroutines* que escribirán en el canal de resultados. Luego, crea las *goroutines* que envían al canal:

```go
wg := sync.WaitGroup{}
for _,input := range inputs {
  wg.Add(1)
  go func(data string) {
    defer wg.Done()
    ch <- processInput(data)
  }(input)
}
```

4. Espera a que terminen las *goroutines* de procesamiento y cierra el canal de resultados:

```go
// Wait for all goroutines to end
wg.Wait()
// Close the channel to signal end of data
// This will signal the listener goroutine that no more data 
// will be arriving via the channel
close(ch)
```

5. Espera a que termine la *goroutine* de escucha:

```go
listenerWg.Wait()
```

6. Ahora puedes usar el *slice* `results`.

#### Recopilación de los Resultados de Cálculos Concurrentes Mediante Canales

A menudo, tienes múltiples *goroutines* trabajando en partes de un problema y tienes que recopilar el resultado de cada *goroutine* para compilar un único objeto de resultado. Los canales son el mecanismo perfecto para esto.

##### Cómo hacerlo...

1. Crea un canal para recopilar los resultados del cálculo:

```go
resultCh := make(chan int)
```

En este ejemplo, el canal `resultCh` es un canal de valores `int`. Es decir, los resultados de los cálculos serán números enteros.

2. Crea una instancia de `sync.WaitGroup` para esperar a las *goroutines*:

```go
wg := sync.WaitGroup{}
```

3. Distribuye el trabajo entre las *goroutines*. Cada *goroutine* debe tener acceso a `resultCh`. Agrega cada *goroutine* al grupo de espera y asegúrate de llamar a `defer wg.Done()` en la *goroutine*. Realiza el cálculo en la *goroutine* y envía el resultado a `resultCh`:

```go
var inputs [][]int=[]int{...}
...
for i:=range inputs {
  wg.Add(1)
  go func(data []int) {
     defer wg.Done()
     // Perform the computation
     // computeResult takes a []int, and returns int
     // Send the result to resultCh
     resultCh <- computeResult(data)
  }(inputs[i])
}
```

4. Aquí tienes que hacer dos cosas: esperar a que se completen todas las *goroutines* y recopilar los resultados de `resultCh`. Hay dos formas de hacer esto:

- **Opción A**: Recopilar los resultados mientras se espera concurrentemente a que terminen las *goroutines*. Es decir, crea una *goroutine* y espera a que terminen las *goroutines*. Cuando todas las *goroutines* hayan terminado, cierra el canal:

```go
go func() {
  // Wait for the goroutines to end
  wg.Wait()
  // When all goroutines are done, close the channel
  close(resultCh)
}()
// Create a slice to contain results of the computations
results:=make([]int,0)
// Collect the results from the `resultCh`
// The for-loop will terminate when resultCh is closed
for result:=range resultCh {
  results=append(results,result)
}
```

- **Opción B**: Recopilar los resultados de forma asíncrona mientras se espera a que terminen las *goroutines*. Cuando se completen todas las *goroutines*, cierra el canal. Sin embargo, al cerrar el canal, es posible que la *goroutine* que recopila los resultados todavía se esté ejecutando. Tenemos que esperar a que esa *goroutine* termine también. Podemos usar otro grupo de espera para ese propósito:

```go
results:=make([]int,0)
// Create a new wait group just for the result collection 
// goroutine
collectWg := sync.WaitGroup{}
// Add the collection goroutine to the waitgroup
collectWg.Add(1)
go func() {
  // Announce the completion of this goroutine
  defer collectWg.Done()
  // Collect results. The for-loop will terminate when resultCh 
  // is closed.
  for result:= range resultCh {
    results=append(results,result)
  }
}()
// Wait for the goroutines to end.
wg.Wait()
// Close the channel so the result collection goroutine can 
// finish
close(resultCh)
// Now wait for the result collection goroutine to finish
collectWg.Wait()
// results slice is ready
```

---

### Sección 3: Trabajo con Múltiples Canales Usando la Sentencia `select`

Solo puedes enviar datos o recibir datos de un canal en un momento determinado. Si estás interactuando con múltiples *goroutines* (y por tanto, con múltiples eventos concurrentes), necesitas una construcción de lenguaje que te permita interactuar con múltiples canales a la vez. Esa construcción es la sentencia `select`.

Esta sección muestra cómo se utiliza `select`.

#### Cómo hacerlo...

Una sentencia `select` bloqueante elige un caso activo de entre cero o más casos. Cada caso es un evento de envío o de recepción de un canal. Si no hay casos activos (es decir, no se puede enviar a ninguno de los canales ni recibir de ellos), `select` se bloquea.

En el siguiente ejemplo, la sentencia `select` espera recibir de uno de dos canales. El programa recibe de solo uno de los canales. Si ambos canales están listos, uno de ellos se seleccionará aleatoriamente. El otro canal quedará sin leer:

```go
ch1:=make(chan int)
ch2:=make(chan int)
go func() {
  ch1<-1
}()
go func() {
  ch2<-2
}()
select {
case data1:= <- ch1:
  fmt.Println("Read from channel 1: %v", data1)
case data2:= <- ch2:
  fmt.Println("Read from channel 2: %v", data2)
}
```

#### Cancelación de *Goroutines*

Crear *goroutines* es fácil y eficiente en Go, pero también debes asegurarte de que tus *goroutines* terminen eventualmente. Si una *goroutine* se deja ejecutando de forma no intencionada, se denomina *goroutine* "fugada" (*leaked goroutine*). Si un programa sigue fugando *goroutines*, eventualmente fallará con un error de memoria agotada (*out-of-memory*).

Algunas *goroutines* realizan un número limitado de operaciones y terminan de forma natural, pero otras se ejecutan indefinidamente hasta que se recibe un estímulo externo. Un patrón común para que las *goroutines* de larga duración reciban tal estímulo es usar un canal `done`.

##### Cómo hacerlo...

1. Crea un canal `done` con un tipo de datos vacío:

```go
done:=make(chan struct{})
```

2. Crea un canal para proporcionar entrada a las *goroutines*:

```go
input := make(chan int)
```

3. Crea *goroutines* estructuradas de la siguiente manera:

```go
go func() {
  for {
    select {
      case data:= <- input:
        // Process data
      case <-done:
        // Done signal. Terminate
        return
     }
  }
}()
```

4. Para cancelar la(s) *goroutine(s)*, simplemente cierra el canal `done`:

```go
close(done)
```

Esto habilitará la rama `case <-done` en todas las *goroutines* que estén escuchando el canal `done`, y terminarán.

#### Detección de Cancelación Usando `select` No Bloqueante

Un `select` no bloqueante tiene un caso `default`. Cuando se ejecuta la sentencia `select`, comprueba todos los casos disponibles y, si ninguno de ellos está disponible, se selecciona el caso `default`. Esto permite que un `select` continúe sin bloquearse.

##### Cómo hacerlo...

1. Crea un canal `done` con un tipo de datos vacío:

```go
done:=make(chan struct{})
```

2. Crea *goroutines* estructuradas de la siguiente manera:

```go
go func() {
  for {
    select {
      case <-done:
        // Done signal. Terminate
        return
       default:
         // Done signal is not sent. Continue
     }
     // Do work
  }
}()
```

3. Para cancelar la(s) *goroutine(s)*, simplemente cierra el canal `done`:

```go
close(done)
```

---

### Sección 4: Memoria Compartida

Uno de los modismos más famosos de Go es: *"No te comuniques compartiendo memoria, comparte memoria comunicándote"*. Los canales sirven para compartir memoria comunicándose. La comunicación compartiendo memoria se realiza utilizando variables compartidas en múltiples *goroutines*. Aunque está desaconsejado, existen muchos casos de uso donde la memoria compartida tiene más sentido que un canal. Si al menos una de las *goroutines* actualiza una variable compartida que es leída por otras *goroutines*, debes asegurarte de que no existan carreras de memoria (*memory races*).

Ocurre una carrera de memoria cuando una *goroutine* actualiza una variable de forma concurrente mientras otra *goroutine* lee de ella o escribe en ella. Cuando esto sucede, no hay garantía de que la actualización de esa variable sea visible para otras *goroutines*. Un ejemplo famoso de esta situación es el bucle de espera activa (*busy-wait loop*):

```go
func main() {
  done:=false
  go func() {
    // Wait while done==false
    for !done {}
    fmt.Println("Done is true now")
  }()
  done=true
  // Wait indefinitely
  select{}
}
```

Este programa tiene una carrera de memoria. La asignación `done=true` es concurrente con el bucle `for !done`. Eso significa que, aunque la *goroutine* principal ejecute `done=true`, es posible que la *goroutine* que lee `done` nunca vea esa actualización, permaneciendo en el bucle `for` indefinidamente.

#### Actualización de Variables Compartidas de Forma Concurrente

El modelo de memoria de Go garantiza que el efecto de la escritura de una variable sea visible para las instrucciones posteriores a esa escritura únicamente dentro de esa misma *goroutine*. Es decir, si actualizas una variable compartida, debes usar herramientas especiales para hacer que esa actualización sea visible para otras *goroutines*. Una forma sencilla de asegurar esto es utilizar un mutex. Mutex significa "exclusión mutua" (*mutual exclusion*). Un mutex es una herramienta que puedes utilizar para garantizar lo siguiente:
- Solo una *goroutine* actualiza una variable en cualquier momento dado.
- Una vez realizada esa actualización y liberado el mutex, todas las *goroutines* pueden ver esa actualización.

En esta receta, mostramos cómo se hace esto.

##### Cómo hacerlo...

La sección de un programa que actualiza variables compartidas es una "sección crítica". Utilizas un mutex para asegurarte de que solo una única *goroutine* pueda entrar en su sección crítica.

1. Declara un mutex para proteger una sección crítica:

```go
// cacheMutex will be used to protect access to cache
var cacheMutex sync.Mutex
var cache map[string]any = map[string]any{}
```

Un mutex protege un conjunto de variables compartidas. Por ejemplo, si tienes *goroutines* que actualizan un único entero, declaras un mutex para las secciones críticas que actualizan ese entero. Debes usar el mismo mutex cada vez que leas o escribas ese valor entero.

2. Al actualizar la(s) variable(s) compartida(s), primero bloquea el mutex. Luego realiza la actualización y desbloquea el mutex:

```go
cacheMutex.Lock()
cache[key]=value
cacheMutex.Unlock()
```

Con este patrón, si múltiples *goroutines* intentan actualizar `cache`, harán cola en `cacheMutex.Lock()` y solo se permitirá una a la vez. Cuando esa *goroutine* realice la actualización, llamará a `cacheMutex.Unlock()`, lo que permitirá a una de las *goroutines* en espera adquirir el bloqueo y actualizar la caché nuevamente.

3. Al leer la variable compartida, primero bloquea el mutex. Luego realiza la lectura y posteriormente desbloquea el mutex:

```go
cacheMutex.Lock()
cachedValue, cached := cache[key]
cacheMutex.Unlock()
if cached {
  // Value found in cache
}
```
