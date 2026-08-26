# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 10: Trabajo con Grandes Volúmenes de Datos

Existen varias formas de utilizar las primitivas de concurrencia de Go para procesar grandes cantidades de datos de manera eficiente. A diferencia de los hilos del sistema operativo (*threads*), las *goroutines* se pueden crear sin una sobrecarga significativa de recursos. Es común tener miles de *goroutines* en un programa. Con esto en mente, veremos algunos patrones comunes para manejar grandes cantidades de datos concurrentemente.

Este capítulo incluye las siguientes recetas:
- *Pools* de *workers*
- *Pools* de conexiones
- *Pipelines*
- Trabajo con conjuntos de resultados grandes

---

### Sección 1: *Pools* de *Workers*

Supongamos que tienes grandes cantidades de elementos de datos (por ejemplo, archivos de imagen) y deseas aplicar la misma lógica a cada uno de ellos. Puedes escribir una función que procese una instancia de la entrada y luego llamar a esta función en un bucle `for`. Dicho programa procesará los elementos de entrada secuencialmente, y si cada elemento tarda $t$ segundos en procesarse, todas las entradas se completarán al cabo de $n \cdot t$ segundos, siendo $n$ el número de entradas.

Si deseas aumentar el rendimiento (*throughput*) mediante programación concurrente, puedes crear un *pool* de *goroutines* trabajadoras (*worker goroutines*). Puedes enviar la siguiente entrada a un miembro inactivo del *pool* de *workers* y, mientras se procesa, asignar la entrada subsiguiente a otro miembro. Si tienes $p$ procesadores lógicos (que pueden ser núcleos de procesadores físicos) ejecutándose en paralelo, el resultado puede estar disponible en un tiempo tan rápido como $n \cdot t / p$ segundos (este es un límite superior teórico porque la distribución de la carga entre procesos paralelos no siempre es perfecta, y también existe una sobrecarga de sincronización y comunicación).

A continuación, veremos dos formas diferentes de implementar *pools* de *workers*.

#### *Pools* de *Workers* con Límite Máximo (*Capped Worker Pools*)

Si no hay una inicialización costosa (por ejemplo, cargar un archivo o establecer una conexión de red puede ser costoso) para cada *worker*, lo mejor es crear *workers* según sea necesario con un límite determinado en el número de *workers*.

##### Cómo hacerlo...

Crea una nueva *goroutine* para cada entrada. Usa un canal como contador sincronizado para limitar el número máximo de *workers* (aquí, el canal se usa como un semáforo). Usa un canal de salida para recopilar los resultados, si los hay:

```go
// Establish a maximum pool size
const maxPoolSize = 100
func main() {
    // 1. Initialization
    // Receive outputs from the pool via outputCh
    outputCh := make(chan Output)
    // A semaphore to limit the pool size
    sem := make(chan struct{}, maxPoolSize)
    // 2. Read outputs
    // Reader goroutine reads results until outputCh is closed
    readerWg := sync.WaitGroup{}
    readerWg.Add(1)
    go func() {
        defer readerWg.Done()
        for result := range outputCh {
            // process result
            fmt.Println(result)
        }
    }()
    // 3. Processing loop
    // Create the workers as needed, but the number of active workers
    // are limited by the capacity of sem
    wg := sync.WaitGroup{}
    // This loop sends the inputs to workers, creating them as 
    // necessary
    for {
        nextInput, done := getNextInput()
        if done {
            break
        }
        wg.Add(1)
        // This will block if there are too many goroutines
        sem <- struct{}{}
        go func(inp Input) {
            defer wg.Done()
            defer func() {
                <-sem
            }()
            outputCh <- doWork(inp)
        }(nextInput)
    }
    // 4. Wait until processing is complete
    // This goroutine waits until all worker pool goroutines are done, 
    // then closes the output channel
    go func() {
        // Wait until processing is complete
        wg.Wait()
        // Close the output channel so the reader goroutine can 
        // terminate
        close(outputCh)
    }()
    // Wait until the output channel is closed
    readerWg.Wait()
    // If we are here, all goroutines are done
}
```

##### Cómo funciona...

1. **Inicialización**: Creamos dos canales:
   - `outputCh`: La salida del *pool* de *workers*. Cada *worker* escribirá el resultado en este canal.
   - `sem`: El canal de semáforo que se utilizará para limitar el número de *workers* activos. Se crea con una capacidad `maxPoolSize`. Cuando iniciamos una nueva *goroutine worker*, enviamos un elemento a este canal. Las operaciones de envío no se bloquearán mientras el canal `sem` tenga menos de `maxPoolSize` elementos. Cuando finaliza una *goroutine worker*, recibe un elemento del canal, liberando capacidad. Dado que este canal tiene capacidad `maxPoolSize`, una operación de envío se bloqueará si se están ejecutando `maxPoolSize` *workers* hasta que una *goroutine* finalice y reciba del canal.
2. **Lectura de salidas**: Iniciamos una *goroutine* para leer de `outputCh` antes de iniciar el procesamiento, de modo que los resultados se puedan leer antes de que se envíen todas las entradas a los *workers*. Dado que el número de *workers* es limitado, los *workers* se bloquearían después de crear `maxPoolSize` de ellos, por lo que tenemos que empezar a escuchar las salidas antes de crear el *pool* de *workers*.
3. **Bucle de procesamiento**: Leemos la siguiente entrada y creamos un nuevo *worker* para trabajar en ella. Los *workers* activos se rastrean con el `sync.WaitGroup` `wg`, que luego se utilizará para esperar a que los *workers* finalicen. Antes de crear un nuevo *worker*, enviamos un elemento al canal del semáforo. Si ya se están ejecutando `maxPoolSize` *workers*, esto se bloqueará hasta que uno de ellos termine. El *worker* procesa la entrada, escribe la salida en `outputCh` y termina, recibiendo un elemento del semáforo.
4. **Espera a que se complete el procesamiento**: Esta *goroutine* espera al `WaitGroup` que realiza el seguimiento de los *workers*. Cuando todos los *workers* terminan, se cierra el canal de salida. Eso también señala al `readerWg` creado en el paso 2.
5. **Espera hasta que se complete el procesamiento de la salida**: El programa tiene que esperar hasta que se generen todas las salidas. Esto solo sucede después del cierre de `outputCh` (que ocurre en el paso 4) y luego de la liberación de `readerWg`.

#### *Pools* de *Workers* de Tamaño Fijo (*Fixed-Size Worker Pools*)

Un *pool* de *workers* de tamaño fijo tiene sentido si crear un *worker* es una operación costosa. Simplemente crea el número máximo de *workers* que leen de un canal de entrada común. Este canal de entrada se encarga de distribuir el trabajo entre los *workers* disponibles.

##### Cómo hacerlo...

Hay varias formas de lograr esto. Veremos dos de ellas.

En la siguiente función, se crea un *pool* de *workers* de tamaño fijo con `poolSize` *workers*. Todos los *workers* leen del mismo canal de entrada y escriben la salida en el mismo canal de salida. Este programa utiliza una *goroutine* lectora para recopilar los resultados del *pool* de *workers* mientras proporciona las entradas en la misma *goroutine* que el llamador:

```go
const poolSize = 50
func workerPoolWithConcurrentReader() {
    // 1. Initialization
    // Send inputs to the pool via inputCh
    inputCh := make(chan Input)
    // Receive outputs from the pool via outputCh
    outputCh := make(chan Output)
    // 2. Create the pool of workers
    wg := sync.WaitGroup{}
    for i := 0; i < poolSize; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for work := range inputCh {
                outputCh <- doWork(work)
            }
        }()
    }
    // 3.a Reader goroutine
    // Reader goroutine reads results until outputCh is closed
    readerWg := sync.WaitGroup{}
    readerWg.Add(1)
    go func() {
        defer readerWg.Done()
        for result := range outputCh {
            // process result
            fmt.Println(result)
        }
    }()
    // 4. Wait workers
    // This goroutine waits until all worker pool goroutines are done, 
    // then closes the output channel
    go func() {
        // Wait until processing is complete
        wg.Wait()
        // Close the output channel so the reader goroutine can 
        // terminate
        close(outputCh)
    }()
    // 5.a Send inputs
    // This loop sends the inputs to the worker pool
    for {
        nextInput, done := getNextInput()
        if done {
            break
        }
        inputCh <- nextInput
    }
    // Close the input channel, so worker pool goroutines terminate
    close(inputCh)
    // Wait until the output channel is closed
    readerWg.Wait()
    // If we are here, all goroutines are done
}
```

La siguiente versión utiliza una *goroutine* para enviar el trabajo al *pool* de *workers*, mientras lee los resultados en la misma *goroutine* que el llamador:

```go
func workerPoolWithConcurrentWriter() {
    // 1. Initialization
    // Send inputs to the pool via inputCh
    inputCh := make(chan Input)
    // Receive outputs from the pool via outputCh
    outputCh := make(chan Output)
    // 2. Create the pool of workers
    wg := sync.WaitGroup{}
    for i := 0; i < poolSize; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for work := range inputCh {
                outputCh <- doWork(work)
            }
        }()
    }
    // 3.b Writer goroutine
    // Writer goroutine submits work to the worker pool
    go func() {
        for {
            nextInput, done := getNextInput()
            if done {
                break
            }
            inputCh <- nextInput
        }
        // Close the input channel, so worker pool goroutines 
        // terminate
        close(inputCh)
    }()
    // 4. Wait workers
    // This goroutine waits until all worker pool goroutines are done, 
    // then closes the output channel
    go func() {
        // Wait until processing is complete
        wg.Wait()
        // Close the output channel so the reader goroutine can 
        // terminate
        close(outputCh)
    }()
    // 5.b Read results
    // Read results until outputCh is closed
    for result := range outputCh {
        // process result
        fmt.Println(result)
    }
}
```

##### Cómo funciona...

1. **Inicialización**: Creamos dos canales:
   - `inputCh`: Esta es la entrada al *pool* de *workers*. Cada *worker* en el *pool* lee del mismo `inputCh` en un bucle `for-range`, por lo que cuando un *worker* recibe una entrada, deja de escuchar en el canal, permitiendo que otro *worker* tome la siguiente entrada.
   - `outputCh`: Esta es la salida del *pool* de *workers*. Todos los *workers* escriben la salida en este canal cuando terminan.
2. **Creación del *pool* de *workers***: Dado que este es un *pool* de tamaño fijo, podemos crear los *workers* en un simple bucle `for`. Es necesario un `WaitGroup` para que podamos esperar a que se complete el procesamiento. Cada *worker* lee de `inputCh` hasta que se cierra, procesa la entrada y escribe en `outputCh`.

El resto del algoritmo es diferente para los dos ejemplos:

- En el primer caso:
  - **Goroutine lectora**: La salida del *pool* de *workers* se lee en esta *goroutine* separada hasta que se cierra `outputCh`. Cuando se cierra `outputCh`, se señala `readerWg`.
  - **Espera de workers**: Esta es una *goroutine* separada que espera la finalización de todos los *workers*. Cuando todos los *workers* terminan (lo que ocurre porque se cierra `inputCh`), cierra `outputCh`.
  - El bucle `for` envía entradas a `inputCh` y luego cierra `inputCh`. Esto hace que todos los *workers* terminen cuando completan su trabajo. Cuando todos los *workers* terminan, la *goroutine* creada en el paso 4 cierra `outputCh`. Cuando se completa el procesamiento de la salida, se señala `readerWg`, terminando el cómputo.
- En el segundo caso:
  - **Goroutine escritora**: Las entradas al *pool* de *workers* son generadas por esta *goroutine*. Envía todas las entradas a `inputCh` una por una, y cuando se envían todas las entradas, cierra `inputCh`, lo que hace que termine el *pool* de *workers*.
  - **Espera de workers**: Funciona igual que en el caso anterior.
  - **Lectura de resultados**: Este bucle `for` lee los resultados de `outputCh` hasta que se cierra. `outputCh` se cerrará cuando se hayan completado todos los *workers*.

---

### Sección 2: *Pools* de Conexiones

Un *pool* de conexiones es útil cuando se trabaja con múltiples usuarios de un recurso escaso donde establecer una instancia de ese recurso puede ser costoso, como una conexión de red o una conexión a una base de datos. Utilizando un par de canales, puedes implementar un *pool* de conexiones eficiente y seguro para subprocesos (*thread-safe*).

#### Cómo hacerlo...

Crea un tipo de *pool* de conexiones con dos canales con capacidad `PoolSize`:
- `available` mantiene las conexiones que ya están establecidas pero devueltas al *pool*.
- `total` mantiene el número total de conexiones, es decir, el número de conexiones disponibles más el número de conexiones que están activamente en uso.

Para obtener una conexión del *pool*, consulta el canal `available`. Si hay una disponible, devuélvela. De lo contrario, verifica el *pool* de conexiones `total` y crea una nueva si no se excede el límite.

Los usuarios de este *pool* deben devolver las conexiones al *pool* después de que hayan terminado enviando la conexión al canal `available`.

El siguiente fragmento de código ilustra dicho *pool* de conexiones:

```go
type ConnectionPool struct {
    // This channel keeps connections returned to the pool
    available chan net.Conn
    // This channel counts the total number of connection active
    total     chan struct{}
}
func NewConnectionPool(poolSize int) *ConnectionPool {
  return &ConnectionPool {
    available: make(chan net.Conn,poolSize),
    total: make(chan struct{}, poolSize),
 }
}
func (pool *ConnectionPool) GetConnection() (net.Conn, error) {
    select {
    // If there are connections available in the pool, return one
    case conn := <-pool.available:
        fmt.Printf("Returning an idle connection.\n")
        return conn, nil
    default:
        // No connections are available
        select {
        case conn := <-pool.available:
            fmt.Printf("Returning an idle connection.\n")
            return conn, nil
        case pool.total <- struct{}{}: // Wait until pool is not full
            fmt.Println("Creating a new connection")
            // Create a new connection
            conn, err := net.Dial("tcp", "localhost:2000")
            if err != nil {
                return nil, err
            }
            return conn, nil
        }
    }
}
func (pool *ConnectionPool) Release(conn net.Conn) {
    pool.available <- conn
    fmt.Printf("Releasing a connection. \n")
}
func (pool *ConnectionPool) Close(conn net.Conn) {
    fmt.Println("Closing connection")
    conn.Close()
    <-pool.total
}
```

#### Cómo funciona...

1. Inicializa el *pool* de conexiones con un `PoolSize`:

```go
pool := NewConnectionPool(PoolSize)
```

Esto creará dos canales, ambos con capacidad `PoolSize`. El canal `available` contendrá todas las conexiones devueltas al *pool*, mientras que `total` mantendrá el número de conexiones establecidas.

2. Para obtener una nueva conexión, usa lo siguiente:

```go
conn, err := pool.GetConnection()
```

Esta implementación de `GetConnection` ilustra cómo se pueden establecer prioridades de canal. `GetConnection` devolverá una conexión inactiva si hay una disponible en el canal `available`. De lo contrario, entrará en el caso `default` donde creará una nueva conexión o utilizará una devuelta al canal `available`.

Observa el patrón de sentencias `select` anidadas en `GetConnection`. Este es un patrón común para implementar prioridad entre canales. Si hay una conexión disponible, se elegirá `case conn := <-pool.available` y la conexión se eliminará del canal de conexiones disponibles. Sin embargo, si no hay conexiones disponibles cuando se ejecuta la primera sentencia `select`, se ejecutará el caso `default`, que ejecutará un `select` entre los casos `conn:=<-pool.available` y `pool.total<-struct{}{}`. Si el primer caso pasa a estar disponible (lo que sucede cuando alguna otra *goroutine* devuelve una conexión al *pool*), esa conexión se devolverá al llamador. Si el segundo caso pasa a estar disponible (lo que sucede cuando se cierra una conexión, eliminando así un elemento de `pool.total`), se crea una nueva conexión y se devuelve al llamador.

3. Cuando el cliente del *pool* haya terminado con la conexión, debe llamar a lo siguiente:

```go
pool.Release(conn)
```

Esto agregará la conexión al canal `available`.

4. Si una conexión deja de responder, el cliente puede cerrarla. Cuando esto sucede, se debe notificar al *pool* y decrementar `total`, pero la conexión no debe agregarse a `available`. Esto se hace mediante:

```go
pool.Close(conn)
```

---

### Sección 3: *Pipelines*

Siempre que tengas varias etapas de operaciones realizadas sobre una entrada, puedes construir una canalización (*pipeline*). Las *goroutines* y los canales se pueden utilizar para construir *pipelines* de procesamiento de alto rendimiento con diferentes estructuras.

#### *Pipeline* Simple sin *Fan-Out*/*Fan-In*

Se puede construir un *pipeline* simple conectando cada etapa ejecutándose en su propia *goroutine* mediante canales. La estructura del *pipeline* se muestra a continuación:

#### Figura 10.1: Pipeline Asíncrono Simple

##### Cómo hacerlo...

Este *pipeline* utiliza un canal de errores separado para informar sobre errores de procesamiento. Usamos un tipo de error personalizado para capturar información de diagnóstico:

```go
type PipelineError struct {
    // The stage in which error happened
    Stage   int
    // The payload
    Payload any
    // The actual error
    Err     error
}
```

Cada etapa se implementa como una función que crea una nueva *goroutine*. La *goroutine* lee datos de entrada de un canal de entrada y escribe la salida en un canal de salida:

```go
func Stage1(input <-chan InputPayload, errCh chan<- error) <-chan Stage2Payload {
    // 1. Create the output channel for this stage.
    // This will be the input for the next stage
    output := make(chan Stage2Payload)
    // 2. Create processing goroutine
    go func() {
        // 3. Close the output channel when done
        defer close(output)
        // 4. Process all inputs until input channel is closed
        for in := range input {
            // 5. Process data
            err := processData(in.Id)
            // 6. Send errors to the error channel
            if err != nil {
                errCh <- PipelineError{
                    Stage:   1,
                    Payload: in,
                    Err:     err,
                }
                continue
            }
            // 7. Send the output to the next stage
            output <- Stage2Payload{
                Id: in.Id,
            }
        }
    }()
    return output
}
```

Las etapas 2 y 3 se implementan utilizando el mismo patrón.

El *pipeline* se ensambla de la siguiente manera:

```go
func main() {
    // 1. Create the input and error channels
    errCh := make(chan error)
    inputCh := make(chan InputPayload)
    // 2. Prepare the pipeline by attaching stages
    outputCh := Stage3(Stage2(Stage1(inputCh, errCh), errCh), errCh)
    // 3. Feed input asynchronously
    go func() {
        defer close(inputCh)
        for i := 0; i < 1000; i++ {
            inputCh <- InputPayload{
                Id: i,
            }
        }
    }()
    // 4. Listen to the error channel asynchronously
    go func() {
        for err := range errCh {
            fmt.Println(err)
        }
    }()
    // 5. Read outputs
    for out := range outputCh {
        fmt.Println(out)
    }
    // 6. Close the error channel
    close(errCh)
}
```

Para cada etapa, sigue estos pasos:
1. Crea el canal de salida para la etapa. Este se pasará a la siguiente etapa como canal de entrada.
2. La *goroutine* de procesamiento continúa ejecutándose después de que retorna la función de la etapa.
3. Asegúrate de que el canal de salida de esta etapa se cierre cuando termine la *goroutine* de procesamiento.
4. Lee las entradas de la etapa anterior hasta que se cierre el canal de entrada.
5. Procesa la entrada.
6. Si hay un error, envía el error al canal de errores. No se generará ninguna salida.
7. Envía la salida a la siguiente etapa.

> **Advertencia**  
> Cada etapa se ejecuta en su propia *goroutine*. Eso significa que una vez que pasas la carga útil (*payload*) a la siguiente etapa, no debes acceder a ella en la etapa actual. Si el *payload* contiene punteros, o si el *payload* en sí es un puntero, pueden ocurrir carreras de datos.

La configuración del *pipeline* se realiza de la siguiente manera:
1. Crea el canal de entrada y el canal de errores.
2. Conecta las etapas para formar el *pipeline*. La salida de la etapa $n$ se convierte en la entrada de la etapa $n+1$. La salida de la última etapa se convierte en el canal de salida.
3. Envía las entradas al canal de entrada de forma asíncrona. Cuando se envíen todas las entradas, cierra el canal de entrada. Esto terminará la primera etapa, cerrando su canal de salida, que también es el canal de entrada para la etapa 2. Esto continúa hasta que salen todas las etapas.
4. Inicia una *goroutine* para escuchar y registrar errores.
5. Recopila las salidas.
6. Cierra el canal de errores para que termine la *goroutine* recopiladora de errores.

#### *Pipeline* con *Pools* de *Workers* como Etapas

El ejemplo anterior utilizó un único *worker* para cada etapa. Puedes aumentar el rendimiento de una canalización reemplazando cada etapa con *pools* de *workers*.

#### Figura 10.2: Pipeline con Pools de Workers como Etapas

##### Cómo hacerlo...

Cada etapa ahora crea múltiples *goroutines*, todas leyendo del mismo canal de entrada (*fan-out*). La salida de cada *worker* se escribe en un canal de salida común (*fan-in*), que se convierte en la entrada para la siguiente etapa. Ya no podemos cerrar el canal de salida de la etapa cada vez que se cierra el canal de entrada porque ahora hay múltiples *goroutines* escribiendo en ese canal de salida. En su lugar, utilizamos un `WaitGroup` y una segunda *goroutine* para cerrar la salida cuando terminen todas las *goroutines* de procesamiento:

```go
func Stage1(input <-chan InputPayload, errCh chan<- error, nInstances int) <-chan Stage2Payload {
    // 1. Create the common output channel
    output := make(chan Stage2Payload)
    // 2. Close the output channel when all the processing is done
    wg := sync.WaitGroup{}
    // 3. Create nInstances goroutines
    for i := 0; i < nInstances; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            // Process all inputs
            for in := range input {
                // Process data
                err := processData(in.Id)
                if err != nil {
                    errCh <- PipelineError{
                        Stage:   1,
                        Payload: in,
                        Err:     err,
                    }
                    continue
                }
                //Send output to the common output channel
                output <- Stage2Payload{
                    Id: in.Id,
                }
            }
        }()
    }
    // 4. Another goroutine waits until all workers are done, and 
    //closes the output channel
    go func() {
        wg.Wait()
        close(output)
    }()
    return output
}
```

El *pipeline* se construye como en el caso anterior:

```go
func main() {
    errCh := make(chan error)
    inputCh := make(chan InputPayload)
    nInstances := 5
    // Prepare the pipeline by attaching stages
    outputCh := Stage3(Stage2(Stage1(inputCh, errCh, nInstances), 
    errCh, nInstances), errCh, nInstances)
    // Feed input asynchronously
    go func() {
        defer close(inputCh)
        for i := 0; i < 1000; i++ {
            inputCh <- InputPayload{
                Id: i,
            }
        }
    }()
    // Listen to the error channel asynchronously
    go func() {
        for err := range errCh {
            fmt.Println(err)
        }
    }()
    // Read outputs
    for out := range outputCh {
        fmt.Println(out)
    }
    // Close the error channel
    close(errCh)
}
```

##### Cómo funciona...

Para cada etapa, sigue estos pasos:
1. Crea el canal de salida, que se convertirá en el canal de entrada para la siguiente etapa.
2. Hay múltiples *goroutines* leyendo del mismo canal de entrada en un bucle `for-range`, por lo que cuando el canal de entrada se cierre, todas esas *goroutines* terminarán. Sin embargo, no podemos hacer `defer close` del canal de salida, porque eso resultaría en cerrar el canal de salida varias veces (lo que generaría un pánico). Por lo tanto, usamos un `WaitGroup` para realizar un seguimiento de las *goroutines worker*. Una *goroutine* separada espera en ese grupo de espera y, cuando todas las *goroutines* terminan, cierra el canal de salida.
3. Crea `nInstances` *goroutines* que leen todas del mismo canal de entrada y escriben en el canal de salida. En caso de error, los *workers* envían el error al canal de errores.
4. Esta es la *goroutine* que espera a que terminen las *goroutines worker*. Cuando lo hacen, cierra el canal de salida.

La configuración del *pipeline* es idéntica a la sección anterior, excepto que la inicialización también envía el tamaño del *pool* de *workers* a las funciones de etapa.

#### *Pipeline* con *Fan-Out* y *Fan-In*

En esta configuración, las etapas se conectan una tras otra utilizando canales dedicados:

#### Figura 10.3: Pipeline con Fan-Out y Fan-In

##### Cómo hacerlo...

Cada etapa de la canalización lee de un canal de entrada determinado y escribe en un canal de salida:

```go
func Stage1(input <-chan InputPayload, errCh chan<- error) <-chan Stage2Payload {
    output := make(chan Stage2Payload)
    go func() {
        defer close(output)
        // Process all inputs
        for in := range input {
            // Process data
            err := processData(in.Id)
            if err != nil {
                errCh <- PipelineError{
                    Stage:   1,
                    Payload: in,
                    Err:     err,
                }
                continue
            }
            output <- Stage2Payload{
                Id: in.Id,
            }
        }
    }()
    return output
}
```

Una función `fanIn` independiente toma una lista de canales de salida y los combina usando una *goroutine* que escucha en cada canal:

```go
func fanIn(inputs []<-chan OutputPayload) <-chan OutputPayload {
    result := make(chan OutputPayload)
    // Listen to input channels in separate goroutines
    inputWg := sync.WaitGroup{}
    for inputIndex := range inputs {
        inputWg.Add(1)
        go func(index int) {
            defer inputWg.Done()
            for data := range inputs[index] {
                // Send the data to the output
                result <- data
            }
        }(inputIndex)
    }
    // When all input channels are closed, close the fan in ch
    go func() {
        inputWg.Wait()
        close(result)
    }()
    return result
}
```

El *pipeline* se configura en un bucle `for` combinando la salida de cada etapa con la entrada de la siguiente etapa. Los canales de salida resultantes se dirigen todos a la función `fanIn`:

```go
func main() {
    errCh := make(chan error)
    inputCh := make(chan InputPayload)
    poolSize := 5
    outputs := make([]<-chan OutputPayload, 0)
    // All Stage1 goroutines listen to a single input channel
    for i := 0; i < poolSize; i++ {
        outputCh1 := Stage1(inputCh, errCh)
        outputCh2 := Stage2(outputCh1, errCh)
        outputCh3 := Stage3(outputCh2, errCh)
        outputs = append(outputs, outputCh3)
    }
    outputCh := fanIn(outputs)
    // Feed input asynchronously
    go func() {
        defer close(inputCh)
        for i := 0; i < 1000; i++ {
            inputCh <- InputPayload{
                Id: i,
            }
        }
    }()
    // Listen to the error channel asynchronously
    go func() {
        for err := range errCh {
            fmt.Println(err)
        }
    }()
    // Read outputs
    for out := range outputCh {
        fmt.Println(out)
    }
    // Close the error channel
    close(errCh)
}
```

##### Cómo funciona...

Las etapas de los *workers* son idénticas al caso del *pipeline* simple. La etapa *fan-in* funciona de la siguiente manera:

Para cada canal de salida, la función `fanIn` crea una *goroutine* que lee datos de ese canal de salida y escribe en un canal común. Este canal común se convierte en el canal de salida combinado del *pipeline*. La función `fanIn` crea otra *goroutine* que espera en un grupo de espera que realiza el seguimiento de todas las *goroutines*. Cuando todas se completan, esta *goroutine* cierra el canal de salida.

La función `main` construye el *pipeline* conectando la salida de cada etapa a la entrada de la siguiente. Los canales de salida de la última etapa se almacenan en un *slice* y se pasan a la función `fanIn`. El canal de salida de la función `fanIn` se convierte en la salida combinada del *pipeline*.

Ten en cuenta que todas estas variaciones de *pipeline* utilizan un canal de error independiente. Un enfoque alternativo es almacenar cualquier error en la carga útil y pasarlo a la siguiente etapa. Si el *payload* entrante tiene un error no nulo, todas las etapas lo pasan a la siguiente, de modo que el *payload* se puede registrar como un error al final del *pipeline*:

```go
type Stage2Paylaod struct {
   // Payload data
   Err error
}
func Stage2(input <-chan Stage2Payload) <-chan Stage3Payload {
    output := make(chan Stage2Payload)
    go func() {
        defer close(output)
        // Process all inputs
        for in := range input {
            // If there is error, pass it
            if in.Err!=nil {
               output <- StagerPayload {
                  Err: in.Err,
               }
               continue
             }
             ...
```

Ten en cuenta también que, excepto en el caso del *pipeline* simple, las etapas devuelven resultados fuera de orden porque múltiples entradas pasan por el *pipeline* en cualquier momento dado, y no hay garantía sobre el orden en que llegan a la salida.

---

### Sección 4: Trabajo con Conjuntos de Resultados Grandes

Cuando se trabaja con conjuntos de resultados potencialmente grandes, es posible que no siempre sea factible cargar todos los datos en la memoria y trabajar con ellos. Es posible que debas transmitir elementos de datos en *streaming* de manera controlada. Esta sección muestra cómo lidiar con tales situaciones utilizando primitivas de concurrencia.

#### Transmisión en *Streaming* de Resultados Usando una *Goroutine*

En este caso de uso, una *goroutine* envía los resultados de una consulta a través de un canal. Se puede utilizar un contexto para cancelar la *goroutine* de transmisión.

##### Cómo hacerlo...

1. Crea una estructura de datos que contenga los elementos de datos y la información del error:

```go
type Result struct {
  Err error
  // Other data elements
}
```

2. La función `StreamResults` ejecuta la consulta a la base de datos y crea una *goroutine* que itera los resultados de la consulta. La *goroutine* envía cada resultado a través de un canal:

```go
func StreamResults(
    ctx context.Context,
    db *sql.DB,
    query string,
    args ...any,
) (<-chan Result, error) {
    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    output := make(chan Result)
    go func() {
        defer rows.Close()
        defer close(output)
        var result Result
        for rows.Next() {
            // Check context cancellation
            if result.Err = ctx.Err(); result.Err != nil {
                // Context canceled. return
                output <- result
                return
            }
            // Set result fields
            result.Err = buildResult(rows, &result)
            output <- result
        }
        // If there was an error, return it
        if result.Err = rows.Err(); result.Err != nil {
            output <- result
        }
    }()
    return output, nil
}
```

3. Usa los resultados en *streaming* de la siguiente manera:

```go
// Setup a cancelable context
cancelableCtx, cancel := context.WithCancel(ctx)
defer cancel()
// Call the streaming API
results, err := StreamResults(cancelableCtx,db,"SELECT EMAIL FROM USERS")
if err!=nil {
  return err
}
// Collect and process results
for result:=range results {
   if result.Err!=nil {
      // Handle error in the result
      continue
    }
    // Process the result
    if err:=ProcessResult(result); err!=nil {
      // Processing error. Cancel streaming results
      cancel()
      // Expect to receive at least one more message from the channel,
      // because the streaming gorutine sends the error
      for range results {}
    }
}
```

##### Cómo funciona...

Aunque analizamos un ejemplo de consulta de base de datos, este patrón es útil siempre que trabaje con una función que genere cantidades de datos potencialmente grandes. En lugar de cargar todos los datos en la memoria, este patrón carga y procesa elementos de datos uno por uno.

La función generadora `StreamResults` inicia una clausura de *goroutine* que captura el contexto y la información adicional necesaria para producir resultados (en este caso, una instancia de `sql.Rows`). La función generadora crea un canal y retorna inmediatamente. La *goroutine* recopila resultados y los envía al canal. Cuando se procesan todos los resultados o se detecta un error, el canal se cierra.

Ahora le corresponde al llamador comunicarse con la *goroutine*. El llamador recopila los resultados del canal hasta que se cierra el canal y los procesa uno por uno. El llamador también comprueba el campo de error en el mensaje recibido para manejar cualquier error detectado por la *goroutine*.

Este esquema utiliza un contexto cancelable. Cuando se cancela el contexto, la *goroutine* envía otro mensaje a través del canal antes de cerrarlo, por lo que el llamador debe vaciar (*drain*) el canal si se produce la cancelación del contexto.

