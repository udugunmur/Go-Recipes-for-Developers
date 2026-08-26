# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 12: Procesos

Este capítulo presenta recetas que muestran cómo ejecutar programas externos, cómo interactuar con ellos y cómo terminar un proceso de forma limpia (*gracefully*). Hay algunos puntos clave a tener en cuenta al tratar con procesos externos:
- Cuando inicias un proceso externo, se ejecuta de forma concurrente con tu programa.
- Si necesitas comunicarte con un proceso hijo, debes utilizar un mecanismo de comunicación entre procesos (IPC), como tuberías (*pipes*).
- Cuando ejecutas un proceso hijo, sus flujos de entrada estándar (*stdin*) y salida estándar (*stdout*) aparecen ante el proceso padre como flujos concurrentes independientes. No puedes confiar en el orden de los datos que recibes de estos flujos.

Esta sección cubre las siguientes recetas principales:
- Ejecución de programas externos
- Paso de argumentos a un proceso
- Procesamiento de la salida de un proceso hijo mediante una tubería (*pipe*)
- Provisión de entrada a un proceso hijo
- Modificación de variables de entorno de un proceso hijo
- Terminación limpia (*graceful*) mediante señales

---

### Sección 1: Ejecución de Programas Externos

Existen muchos casos de uso en los que deseas ejecutar un programa externo para realizar una tarea. Por lo general, esto se debe a que realizar la misma tarea dentro de tu propio programa no es posible o no es fácil. Por ejemplo, puedes optar por ejecutar varias instancias de un programa externo de procesamiento de imágenes para modificar un grupo de imágenes. Otro caso de uso es cuando deseas configurar algún dispositivo mediante programas proporcionados por su fabricante. Esta receta incluye varias formas de ejecutar programas externos.

#### Cómo hacerlo...

Usa `exec.Command` o `exec.CommandContext` para ejecutar otro programa desde tu programa. `exec.Command` es apropiado si no necesitas cancelar (*kill*) el proceso hijo o imponer un tiempo de espera. De lo contrario, usa `exec.CommandContext` y cancela o establece un tiempo de espera en el contexto para terminar el proceso hijo:

1. Crea el objeto `exec.Command` (o `exec.CommandContext`) utilizando el nombre del programa y sus argumentos:
   - Si necesitas buscar el programa en la ruta de comandos ejecutables de la plataforma (`PATH`), no incluyas ningún separador de ruta.
   - Si utilizas separadores de ruta en el nombre del programa, debe ser una ruta relativa a `exec.Command.Dir`, o si `exec.Command.Dir` está vacío, debe ser una ruta relativa al directorio de trabajo actual.
   - Utiliza una ruta absoluta si sabes dónde está el ejecutable.
2. Prepara los flujos de entrada y salida para capturar la salida del programa o para enviar entradas a través del flujo de entrada estándar.
3. Inicia el programa.
4. Espera a que el programa termine.

El siguiente ejemplo compila un programa Go usando el comando `go` bajo el directorio `sub/`:

```go
// Run "go build" to build the subprocess in the "sub" directory
func buildProgram() {
    // Create a Command with the executable and its arguments
     cmd := exec.Command(
       "go", "build", "-o", "subprocess", ".")
    // Set the working directory
     cmd.Dir = "sub"
    // Collect the stdout and stderr as a combined output from the 
    // process
    // This will run the process, and wait for it to end
     output, err := cmd.CombinedOutput()
     if err != nil {
          panic(err)
     }
     // The build command will not print anything if successful. So if
     // there is any output, it is a failure.
     if len(output) > 0 {
          panic(string(output))
     }
}
```

El ejemplo anterior recopilará la salida del proceso como una cadena combinada. La salida estándar y el error estándar del programa se devolverán como una sola cadena, por lo que no tendrás forma de identificar qué partes de la cadena de salida provinieron de la salida estándar y qué partes del error estándar. Asegúrate de poder analizar la salida correctamente.

> **Advertencia**  
> Los flujos de salida estándar y error estándar de un proceso son flujos concurrentes independientes. En general, no existe una forma portable de determinar qué flujo produjo la salida primero. Esto puede tener implicaciones graves. Por ejemplo, supón que ejecutaste un programa que produce un flujo de líneas en `stdout`, pero cada vez que detecta un error, imprime un mensaje en `stderr` que dice algo como "la última línea impresa tiene problemas". Pero cuando lees el error en tu programa, es posible que la última línea impresa aún no haya llegado a tu programa.

El siguiente programa demuestra el uso de `exec.CommandContext` y tuberías (*pipes*):

```go
// Run the program built by buildProgram function for 10ms, reading 
// from the output
// and error pipes concurrently
func runSubProcessStreamingOutputs() {
    // Create a context with timeout
     ctx, cancel := context.WithTimeout(context.Background(), 10*time.
     Millisecond)
     defer cancel()
    // Create the command that will timeout in 10ms
     cmd := exec.CommandContext(ctx, "sub/subprocess")
    // Pipe the output and error streams
     stdout, err := cmd.StdoutPipe()
     if err != nil {
          panic(err)
     }
     stderr, err := cmd.StderrPipe()
     if err != nil {
          panic(err)
     }
     // Read from stderr from a separate goroutine
     go func() {
          io.Copy(os.Stderr, stderr)
     }()
    // Start running the program
     err = cmd.Start()
     if err != nil {
          panic(err)
     }
    // Copy the stdout of the child program to our stdout
     io.Copy(os.Stdout, stdout)
    // Wait for the program to end
     err = cmd.Wait()
     if err != nil {
          fmt.Println(err)
     }
}
```

El ejemplo anterior se conecta a las salidas de salida estándar y error estándar del proceso hijo. Ten en cuenta que el programa comienza a leer del flujo `stderr` antes de que se inicie el programa. Esa *goroutine* se bloqueará hasta que el proceso hijo emita un error o hasta que el proceso hijo termine, momento en el cual la tubería de `stderr` se cerrará y la *goroutine* terminará. La parte que lee de la salida estándar se ejecuta en la *goroutine* principal, antes de `cmd.Wait`. Este orden es importante. Si el proceso hijo comienza a producir salida en `stdout` pero el programa padre no está escuchando, el proceso hijo se bloqueará. Llamar a `cmd.Wait` en este punto crearía un bloqueo mutuo (*deadlock*), pero el entorno de ejecución no puede detectarlo como tal porque el programa padre depende del comportamiento del hijo.

Puedes asignar el mismo flujo a `stdout` y `stderr` del proceso hijo, como se muestra aquí:

```go
// Run the build subprocess for 10 ms with combined output
func runSubProcessCombinedOutput() {
    // Create a context with timeout
     ctx, cancel := context.WithTimeout(context.Background(), 10*time.
     Millisecond)
     defer cancel()
    // Define the command with the context
     cmd := exec.CommandContext(ctx, "sub/subprocess")
    // Assign both stdout and stderr to the same stream. This is 
    // equivalent to calling CombinedOutput
     cmd.Stdout = os.Stdout
     cmd.Stderr = os.Stdout
    // Start the process
     err := cmd.Start()
     if err != nil {
          panic(err)
     }
    // Wait until it ends. The output will be printed to our stdout
     err = cmd.Wait()
     if err != nil {
          fmt.Println(err)
     }
}
```

El enfoque anterior es similar a ejecutar el proceso hijo con `CombinedOutput`. Asignar `cmd.Stdout` y `cmd.Stderr` al mismo flujo tiene el mismo efecto que combinar ambas salidas del proceso hijo.

---

### Sección 2: Paso de Argumentos a un Proceso

La mecánica de pasar argumentos a un proceso hijo puede resultar confusa. Los entornos de *shell* analizan y expanden los argumentos del proceso. Por ejemplo, un argumento `*.txt` se reemplaza por una lista de nombres de archivos que coinciden con ese patrón, y cada uno de esos nombres de archivos se convierte en un argumento independiente. Esta receta explica cómo pasar dichos argumentos a los procesos hijos correctamente.

Hay dos opciones para pasar argumentos a un proceso hijo.

#### Expansión de Argumentos

La primera opción es realizar el procesamiento de argumentos de la *shell* manualmente.

##### Cómo hacerlo...

Para realizar manualmente el procesamiento de la *shell*, sigue estos pasos:

1. Elimina las comillas específicas de la *shell* de los argumentos:
   - El comando de shell `./prog "test directory"` se convierte en `cmd := exec.Command("./prog", "test directory")`.
   - El comando de Bash `./prog dir1 "long dir name" '"quoted name"'` se convierte en `cmd := exec.Command("./prog", "dir1", "long dir name", "'\"quoted name\"'")`. Observa el tratamiento de comillas específico de Bash.
2. Expande los patrones. `./prog *.txt` se convierte en `cmd := exec.Command("./prog", listFiles("*.txt")...)`, donde `listFiles` es una función que devuelve un *slice* de nombres de archivo.

> **Consejo**  
> Pasar una lista de archivos separados por un espacio los pasará como un único argumento. Es decir, `cmd := exec.Command("./prog", "file1.txt file2.txt")` pasará un solo argumento al proceso, que es `"file1.txt file2.txt"`.

3. Sustituye las variables de entorno. `./prog $HOME` se convierte en `cmd := exec.Command("./prog", os.Getenv("HOME"))`. Ejecutar `cmd := exec.Command("./prog", "$HOME")` pasará la cadena literal `"$HOME"` al programa, no su valor del entorno.
4. Finalmente, tienes que procesar manualmente las tuberías y redirecciones. Es decir, para un comando de shell `./prog >output.txt`, debes ejecutar `cmd := exec.Command("./prog")`, crear un archivo `output.txt` y establecer `cmd.Stdout = outputFile`.

#### Ejecución del Comando a través de la *Shell*

La segunda opción es ejecutar el programa a través de una *shell*.

##### Cómo hacerlo...

Usa la *shell* específica de la plataforma y su sintaxis para ejecutar un comando:

```go
var cmd *exec.Cmd
switch runtime.GOOS {
case "windows":
     cmd = exec.Command("cmd", "/C", "echo test>test.txt")
case "darwin": // Mac OS
     cmd = exec.Command("/bin/sh", "-c", "echo test>test.txt")
case "linux": // Linux system, assuming there is bash
     cmd = exec.Command("/bin/bash", "-c", "echo test>test.txt")
default: // Some other OS. Assume it has `sh`
     cmd = exec.Command("/bin/sh", "-c", "echo test>test.txt")
}
out, err := cmd.Output()
```

Este ejemplo selecciona `cmd` para plataformas Windows, `/bin/sh` para Darwin (Mac), `/bin/bash` para Linux y `/bin/sh` para cualquier otra cosa. El comando pasado a la *shell* contiene una redirección, que es manejada por la *shell*. La salida del comando se escribirá en `test.txt`.

---

### Sección 3: Procesamiento de la Salida de un Proceso Hijo Mediante una Tubería (*Pipe*)

Recuerda que los flujos de salida estándar y error estándar de un proceso son flujos concurrentes. Si la salida generada por el proceso hijo es potencialmente ilimitada, puedes trabajar con ella en una *goroutine* separada. Esta receta muestra cómo.

#### Cómo hacerlo...

Unas palabras sobre las tuberías (*pipes*): un *pipe* es un análogo basado en flujos de un canal de Go. Es un mecanismo de comunicación FIFO (primero en entrar, primero en salir) con dos extremos: un escritor y un lector. El lado del lector se bloquea hasta que el escritor escribe algo, y el lado del escritor se bloquea hasta que el lector lee de él. Cuando terminas con una tubería, cierras el lado del escritor, lo que también cierra el lado del lector de la tubería. Esto sucede cuando finaliza un proceso hijo. Si cierras el lado del lector de una tubería y luego escribes en ella, el programa recibirá una señal y posiblemente terminará. Esto sucede si el programa padre termina antes que el hijo.

1. Crea el comando y obtén su `StdoutPipe`:

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
defer cancel()
cmd := exec.CommandContext(ctx, "sub/subprocess")
pipe, err := cmd.StdoutPipe()
if err != nil {
  panic(err)
}
```

2. Crea una nueva *goroutine* y lee de la salida estándar del proceso hijo. Trabaja con la salida del proceso hijo en esta *goroutine*:

```go
// Read from the pipe in a separate goroutine
go func() {
  // Filter lines that contain "0"
  scanner := bufio.NewScanner(pipe)
  for scanner.Scan() {
    line := scanner.Text()
    if strings.Contains(line, "0")  {
      fmt.Printf("Filtered line: %s\n", line)
    }
  }
  if err := scanner.Err(); err != nil {
    fmt.Println("Scanner error: %v", err)
  }
}()
```

3. Inicia el proceso:

```go
err = cmd.Start()
if err != nil {
  panic(err)
}
```

4. Espera a que termine el proceso:

```go
err = cmd.Wait()
if err != nil {
  fmt.Println(err)
}
```

---

### Sección 4: Provisión de Entrada a un Proceso Hijo

Hay dos métodos que puedes utilizar para proporcionar entrada a un proceso hijo: establecer `cmd.Stdin` en un flujo o utilizar `cmd.StdinPipe` para obtener un escritor para enviar la entrada al proceso hijo.

#### Cómo hacerlo...

1. Crea el comando:

```go
// Run grep and search for a word
cmd := exec.Command("grep", word)
```

2. Proporciona la entrada al proceso configurando el flujo `Stdin`:

```go
// Open a file
input, err := os.Open("input.txt")
if err != nil {
  panic(err)
}
cmd.Stdin = input
```

3. Ejecuta el programa y espera a que termine:

```go
if err = cmd.Start(); err != nil {
  panic(err)
}
if err = cmd.Wait(); err != nil {
  panic(err)
}
```

Alternativamente, puedes proporcionar una entrada en flujo (*streaming*) utilizando una tubería.

1. Crea el comando:

```go
// Run grep and search for a word
cmd := exec.Command("grep", word)
```

2. Obtén la tubería de entrada:

```go
input, err:=cmd.StdinPipe()
if err!=nil {
  panic(err)
}
```

3. Envía la entrada al programa a través de la tubería. Cuando termines, cierra la tubería:

```go
go func() {
  // Defer close the pipe
  defer input.Close()
  // Open a file
  file, err := os.Open("input.txt")
  if err != nil {
    panic(err)
  }
  defer file.Close()
  io.Copy(input,file)
}()
```

4. Ejecuta el programa y espera a que termine:

```go
if err = cmd.Start(); err != nil {
  panic(err)
}
if err = cmd.Wait(); err != nil {
  panic(err)
}
```

---

### Sección 5: Modificación de Variables de Entorno de un Proceso Hijo

Las variables de entorno son pares clave-valor asociados con un proceso. Son útiles para pasar información específica del entorno, como el directorio de inicio del usuario actual, la ruta de búsqueda de ejecutables, las opciones de configuración y más. En implementaciones en contenedores, las variables de entorno son una forma conveniente de pasar las credenciales que necesita un programa.

Las variables de entorno de un proceso las proporciona su proceso padre, pero una vez que se inicia el proceso, se asigna una copia de esas variables de entorno al proceso hijo. Debido a esto, un proceso padre no puede cambiar las variables de entorno de su proceso hijo una vez que este comienza a ejecutarse.

#### Cómo hacerlo...

- Para usar las mismas variables de entorno que el proceso actual al iniciar un proceso hijo, establece `Command.Env` en `nil`. Eso copiará las variables de entorno del proceso actual al hijo.
- Para iniciar el proceso hijo utilizando variables de entorno adicionales, agrega esas nuevas variables a las variables del proceso actual:

```go
// Run the server
cmd:=exec.Command("./server")
// Copy current process environment variables
cmd.Env=os.Environ()
// Append new environment variables
// Set the authentication key as an environment variable
// of the current process
cmd.Env=append(cmd.Env,fmt.Sprintf("AUTH_KEY=%s", authkey))
// Start the server process. Parent process environment is copied to
cmd.Start()
```

---

### Sección 6: Terminación Limpia (*Graceful*) Mediante Señales

Para terminar limpiamente un programa, debes hacer lo siguiente:
1. No aceptar nuevas solicitudes.
2. Finalizar las solicitudes que se hayan aceptado pero que no se hayan completado.
3. Permitir una cierta cantidad de tiempo para que finalicen los procesos de larga duración y terminarlos si no se pueden completar en el tiempo dado.

La terminación limpia es especialmente importante en el desarrollo de servicios basados en la nube porque la mayoría de los servicios en la nube son efímeros y se reemplazan por nuevas instancias con frecuencia. Esta receta muestra cómo se puede hacer.

#### Cómo hacerlo...

1. Maneja señales de interrupción y terminación. Una señal de interrupción (`SIGINT`) generalmente la inicia el usuario (por ejemplo, presionando Ctrl + C) y una señal de terminación (`SIGTERM`) generalmente la inicia el sistema operativo host o, en un entorno en contenedores, el sistema de orquestación de contenedores.
2. Deshabilita la aceptación de nuevas solicitudes.
3. Espera a que se completen las solicitudes existentes con un tiempo de espera (*timeout*).
4. Termina el proceso.

A continuación se muestra un ejemplo. Este es un servidor de eco HTTP simple. Cuando se inicia el programa, crea una *goroutine* que escucha en un canal que responde a las señales `SIGINT` y `SIGTERM`. Cuando se recibe cualquiera de estas señales, apaga el servidor (que primero deshabilita la aceptación de nuevas solicitudes y luego espera a que se completen las solicitudes existentes hasta un tiempo límite), lo que luego termina el programa:

```go
func main() {
  // Create a simple HTTP echo service
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    io.Copy(w, r.Body)
  })
  server := &http.Server{Addr: ":8080"}
  // Listen for SIGINT and SIGTERM signals
  // Terminate the server with the signal
  sigTerm := make(chan os.Signal, 1)
  signal.Notify(sigTerm, syscall.SIGINT, syscall.SIGTERM)
  go func() {
    <-sigTerm
    // 5 second timeout for the server to shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.
    Second)
    defer cancel()
    server.Shutdown(ctx)
  }()
  // Start the server. When the server shuts down, program will end
  server.ListenAndServe()
}
```

