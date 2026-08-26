# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 13: Programación de Redes

La programación de redes es una habilidad crucial para los desarrolladores de aplicaciones. Un tratado exhaustivo sobre el tema sería una tarea formidable, por lo que veremos algunos de los ejemplos selectos que podrías encontrar en tu trabajo. Un punto importante a tener en cuenta es que la programación de redes es el principal medio por el que se crean vulnerabilidades en una aplicación. Los programas de red también son inherentemente concurrentes, lo que hace que la programación de redes correcta y segura sea especialmente difícil. Por lo tanto, esta sección incluirá ejemplos escritos teniendo en cuenta la seguridad y la escalabilidad.

Este capítulo contiene las siguientes recetas:
- Creación de servidores TCP
- Creación de clientes TCP
- Creación de un servidor TCP basado en líneas
- Envío y recepción de archivos mediante una conexión TCP
- Creación de un cliente/servidor TLS
- Un *proxy* TCP para terminación TLS y balanceo de carga
- Configuración de límites de tiempo (*deadlines*) de lectura/escritura
- Desbloqueo de una operación de lectura o escritura bloqueada
- Creación de clientes/servidores UDP
- Realización de peticiones HTTP
- Ejecución de un servidor HTTP
- HTTPS: configuración de un servidor TLS
- Creación de *handlers* HTTP
- Servir archivos estáticos en el sistema de archivos
- Manejo de formularios HTML
- Creación de un *handler* para descarga de archivos grandes
- Manejo de subidas de archivos y formularios HTTP como *stream*

---

### Sección 1: Redes TCP

El Protocolo de Control de Transmisión (*Transmission Control Protocol*, TCP) es un protocolo orientado a conexión que proporciona las siguientes garantías:
- **Fiabilidad**: El emisor sabrá si el destinatario previsto recibió los datos.
- **Orden**: Los mensajes se recibirán en el orden en que fueron enviados.
- **Comprobación de errores**: Los mensajes estarán protegidos contra corrupción durante el tránsito.

Gracias a estas garantías, TCP es relativamente fácil de trabajar. Es la base de muchos protocolos de nivel superior como HTTP y WebSockets. En esta sección, veremos algunas recetas que muestran cómo escribir servidores y clientes TCP.

#### Creación de Servidores TCP

Un servidor TCP es un programa que escucha solicitudes de conexión en un puerto de red. Una vez que se establece una conexión con un cliente, la comunicación entre el cliente y el servidor se realiza a través de un objeto `net.Conn`. El servidor puede continuar escuchando nuevas conexiones. De esta manera, un solo servidor puede comunicarse con muchos clientes.

##### Cómo hacerlo...

1. Selecciona un puerto que se conectará a los servidores de los clientes. Esto suele ser una cuestión de configuración de la aplicación. Los primeros 1.024 puertos (0 a 1023) normalmente requieren que un programa servidor tenga privilegios de *root*. La mayoría de estos puertos están reservados para programas de servidor conocidos, como el puerto 22 para SSH o el puerto 80 para HTTP. Los puertos 1024 y superiores son puertos efímeros. Tu programa de servidor puede utilizar cualquier número de puerto de 1024 en adelante sin privilegios adicionales siempre que ningún otro programa esté escuchando en él.
   - Usa el número de puerto `0` para que el *kernel* elija un puerto no utilizado aleatorio. Puedes crear un *listener* para el puerto 0 y luego consultar el *listener* para averiguar qué número de puerto se seleccionó.
2. Crea un *listener*. Un *listener* es un mecanismo que vincula la dirección:puerto. Una vez que creas un *listener* usando un número de puerto, ningún otro proceso en el mismo host, o dentro del mismo contenedor, puede usar ese número de puerto para escuchar el tráfico de red:

```go
// The address:port to listen. If none given, use :0 to select // port randomly
addr:=":8080"
// Create a TCP listener
listener, err := net.Listen("tcp", addr)
if err != nil {
   panic(err)
}
// Print out the address we are listening
fmt.Println("Listening on ", listener.Addr())
defer listener.Close()
```

El programa primero determina la dirección de red a escuchar. El formato exacto de la dirección depende del protocolo elegido, que en este caso es TCP. Si no se proporciona ningún nombre de host o dirección IP, el *listener* escuchará todas las direcciones IP de unidifusión disponibles del sistema local. Si proporcionas un nombre de host o una dirección IP, el *listener* solo escuchará el tráfico proveniente de la dirección IP proporcionada. Eso significa que si proporcionas `localhost:1234`, el *listener* escuchará el tráfico proveniente únicamente de `localhost`. No escuchará el tráfico externo.

El ejemplo anterior imprime `listener.Addr()`. Esto es útil si proporcionas `:0` como dirección de escucha, o si no proporcionas ninguna. En este caso, el *listener* escuchará en un puerto aleatorio y `listener.Addr()` devolverá la dirección a la que los clientes pueden conectarse.

3. Escucha y acepta conexiones. Acepta conexiones entrantes usando `Listener.Accept()`. Esto generalmente se hace en un bucle como se muestra a continuación:

```go
// Listen to incoming TCP connections
for {
   // Accept a connection
   conn, err := listener.Accept()
   if err != nil {
      fmt.Println(err)
      return
   }
   // Handle the connection in its own goroutine
   go handleConnection(conn)
}
```

En este ejemplo, la llamada `listener.Accept` fallará con un error si el *listener* está cerrado.

4. Maneja la conexión en su propia *goroutine*. De esta manera, el *listener* continuará aceptando conexiones mientras el servidor se comunica con los clientes conectados en sus propias *goroutines*, utilizando las conexiones creadas específicamente para esos clientes:

```go
func handleConnection(conn net.Conn) {
   io.Copy(conn,conn)
}
```

Aquí está el programa completo del servidor:

```go
var address = flag.String("a", ":8008", "Address to listen")
func main() {
   flag.Parse()
   // Create a TCP listener
   listener, err := net.Listen("tcp", *address)
   if err != nil {
      panic(err)
   }
   fmt.Println("Listening on ", listener.Addr())
   defer listener.Close()
   // Listen to incoming TCP connections
   for {
      conn, err := listener.Accept()
      if err != nil {
         fmt.Println(err)
         return
      }
      go handleConnection(conn)
   }
}
func handleConnection(conn net.Conn) {
   io.Copy(conn, conn)
}
```

Este programa escribirá todo lo que lee de la conexión de vuelta a la conexión, formando un servicio de eco (*echo service*). Cuando el cliente finaliza la conexión, la operación de lectura devolverá `io.EOF`, terminando la operación de copia.

##### Cómo funciona...

La interfaz `net.Conn` tiene tanto el método `Read([]byte) (int, error)` (que la convierte en un `io.Reader`), como `Write([]byte) (int, error)` (que también la convierte en un `io.Writer`). Debido a esto, lo que se lee de la conexión se vuelve a escribir en ella.

Habrás notado que debido a `io.Copy`, cada byte leído se escribirá de vuelta en la conexión, por lo que este no es un protocolo basado en líneas.

#### Creación de Clientes TCP

Un cliente TCP se conecta a un servidor TCP que está escuchando en un puerto de algún host. Una vez establecida la conexión, la comunicación es bidireccional. En otras palabras, la distinción entre un servidor y un cliente se basa en cómo se establece la conexión. Cuando decimos "servidor", nos referimos al programa que espera escuchando en un puerto, y cuando decimos "cliente", nos referimos al programa que se conecta (*dials*) a un puerto en un host que está siendo escuchado por un servidor. Una vez establecida la conexión, ambas partes envían y reciben datos de forma asíncrona. TCP garantiza que los mensajes se recibirán en el orden en que se enviaron y que los mensajes no se perderán, pero no hay garantías sobre cuándo la otra parte recibirá un mensaje.

##### Cómo hacerlo...

1. El lado del cliente debe conocer la dirección y el puerto del servidor. Esto debe ser proporcionado por el entorno (línea de comandos, configuración, etc.).
2. Usa `net.Dial` para crear una conexión con el servidor:

```go
conn, err := net.Dial("tcp", addr)
if err != nil {
   // Handle error
}
```

3. Usa el objeto `net.Conn` devuelto para enviar datos al servidor o para recibir datos del servidor:

```go
// Send a line of text
text := []byte("Hello echo server!")
conn.Write(text)
// Read the response
response := make([]byte, len(text))
conn.Read(response)
fmt.Println(string(response))
```

4. Cierra la conexión cuando termines:

```go
conn.Close()
```

Aquí está el programa completo:

```go
var address = flag.String("a", ":8008", "Server address")
func main() {
   flag.Parse()
   conn, err := net.Dial("tcp", *address)
   if err != nil {
      panic(err)
   }
   // Send a line of text
   text := []byte("Hello echo server!")
   conn.Write(text)
   // Read the response
   response := make([]byte, len(text))
   conn.Read(response)
   fmt.Println(string(response))
   conn.Close()
}
```

Este ejemplo demuestra un tipo de interacción de solicitud-respuesta con el servidor. Este no es necesariamente siempre el caso. Una conexión de red proporciona interfaces tanto `io.Writer` como `io.Reader`, y se pueden usar de forma concurrente.

#### Creación de un Servidor TCP Basado en Líneas

En esta receta, veremos un servidor TCP que trabaja con líneas en lugar de bytes. Hay algunos puntos con los que debes tener cuidado al leer líneas de una conexión de red, especialmente relacionados con la seguridad del servidor. El hecho de que esperes leer líneas no significa que el cliente enviará líneas bien formadas.

##### Cómo hacerlo...

1. Usa la misma estructura para configurar el servidor indicada en la sección anterior.
2. En el manejador de conexiones, usa un `bufio.Reader` o `bufio.Scanner` para leer líneas.
3. Envuelve la conexión con un `io.LimitedReader` para limitar la longitud de la línea.

Veamos cómo puede funcionar esto:

```go
// Limit line length to 1KiB.
const MaxLineLength = 1024
func handleConnection(conn net.Conn) error {
   defer conn.Close()
   // Wrap the connection with a limited reader
   // to prevent the client from sending unbounded
   // amount of data
   limiter := &io.LimitedReader {
      R: conn,
      N: MaxLineLength+1, // Read one extra byte to detect long lines
   }
   reader := bufio.NewReader(limiter)
   for {
      bytes, err := reader.ReadBytes(byte('\n'))
      if err != nil {
         if err != io.EOF {
            // Some error other than end-of-stream
            return err
         }
         // End of stream. It could be because the line is too long
         if limiter.N==0 {
            // Line was too long
            return fmt.Errorf("Received a line that is too long")
         }
         // End of stream
         return nil
      }
      // Reset the limiter, so the next line can be read with
      // newlimit
      limiter.N=MaxLineLength+1
      // Process the line: send it back to client
      if _, err := conn.Write(bytes); err != nil {
         return err
      }
   }
}
```

La rutina de manejo de conexiones comienza envolviendo la conexión en un `io.LimitedReader`. Esto es necesario para evitar que `reader.ReadBytes` lea una cantidad ilimitada de datos hasta que vea el carácter de nueva línea. Sin esto, un cliente malicioso puede enviar grandes cantidades de datos sin ningún carácter de nueva línea, consumiendo toda la memoria del servidor. Poner un límite estricto a la longitud de la línea previene este vector de ataque. Después de leer cada línea, restablecemos `limiter.N` a su valor original para que la siguiente línea se pueda leer utilizando los mismos límites. Ten en cuenta que el limitador está configurado para leer un byte adicional. Esto se debe a que `io.LimitedReader` devuelve `io.EOF` tanto para un EOF legítimo (lo que significa que el cliente se desconectó) como para una lectura que excede el límite. Si el lector excede el límite, significa que la última línea leída está al menos un byte por encima del límite, lo que nos permite decidir que se trata de una línea no válida.

#### Envío y Recepción de Archivos Mediante una Conexión TCP

El envío y la recepción de archivos a través de una conexión TCP demuestra varios puntos importantes sobre la programación de redes, a saber, el diseño del protocolo (quién envía qué y cuándo) y la codificación (cómo se representan los elementos de datos en el cable). Este ejemplo mostrará cómo transferir metadatos y un flujo de octetos a través de una conexión TCP.

##### Cómo hacerlo...

1. Usa la misma estructura para configurar el servidor que en la sección anterior.
2. En el extremo emisor (cliente), haz lo siguiente:
   - Codifica los metadatos del archivo que contienen el nombre del archivo, el tamaño y el modo, y envíalos.
   - Envía el contenido del archivo.
   - Cierra la conexión.
3. En el extremo receptor (servidor), haz lo siguiente:
   - Decodifica los metadatos del archivo. Crea un archivo para almacenar el contenido del archivo recibido con el modo dado.
   - Recibe el contenido del archivo y escribe el archivo.
   - Una vez recibido todo el contenido del archivo, cierra el archivo.

La primera parte es la transferencia de metadatos sobre el archivo. Hay varias formas de hacerlo: puedes trabajar con un esquema de codificación basado en texto como pares clave-valor o JSON, pero el problema con tales esquemas es que no tienen una longitud fija. Un esquema de codificación simple, efectivo y portable es la codificación binaria usando el paquete `encoding/binary`. Eso no resuelve la codificación del nombre del archivo, que no es una cadena de tamaño fijo. Por lo tanto, incluimos la longitud del nombre del archivo en los metadatos del archivo y codificamos el nombre del archivo utilizando exactamente la cantidad de bytes necesarios.

La estructura de tamaño fijo `fileMetadata` es la siguiente:

```go
type fileMetadata struct {
   Size uint64
   Mode uint32
   NameLen uint16
}
```

Esta estructura tiene 14 bytes en todas las plataformas (ocho bytes de `Size`, cuatro bytes de `Mode` y dos bytes de `NameLen`). Utilizando `binary.Write`, puedes codificar esta estructura de tamaño fijo en la conexión utilizando la codificación `binary.BigEndian` o `binary.LittleEndian`, y el extremo receptor la decodificará con éxito.

El resto del cliente es el siguiente:

```go
var address = flag.String("a", ":8008", "Server address")
var file = flag.String("file", "", "File to send")
func main() {
   flag.Parse()
   // Open the file
   file, err := os.Open(*file)
   if err != nil {
      panic(err)
   }
   // Connect the receiver
   conn, err := net.Dial("tcp", *address)
   if err != nil {
      panic(err)
   }
   // Encode file metadata
   fileInfo, err := file.Stat()
   if err != nil {
      panic(err)
   }
   md := fileMetadata{
      Size: uint64(fileInfo.Size()),
      Mode: uint32(fileInfo.Mode()),
      NameLen: uint16(len(fileInfo.Name())),
   }
   if err := binary.Write(conn, binary.LittleEndian, md); err != nil {
      panic(err)
   }
   // The file name
   if _, err := conn.Write([]byte(fileInfo.Name())); err != nil {
      panic(err)
   }
   // The file contents
   if _, err := io.Copy(conn, file); err != nil {
      panic(err)
   }
   conn.Close()
}
```

Observa el uso de `io.Copy` para transferir el contenido real del archivo. Con `io.Copy`, puedes transferir archivos de tamaño arbitrario al receptor sin consumir cantidades significativas de memoria.

Ahora veamos el servidor (receptor):

```go
func handleConnection(conn net.Conn) {
   defer conn.Close()
   // Read the file metadata
   var meta fileMetadata
   err := binary.Read(conn, binary.LittleEndian, &meta)
   if err != nil {
      fmt.Println(err)
      return
   }
   // Do not allow file names that are too long
   if meta.NameLen > 255 {
      fmt.Println("File name too long")
      return
   }
   // Read the file name
   name := make([]byte, meta.NameLen)
   _, err = io.ReadFull(conn, name)
   if err != nil {
      fmt.Println(err)
      return
   }
   path:=filepath.Join("downloads",string(name))
   // Create the file
   file, err := os.OpenFile(
      path,
      os.O_CREATE|os.O_WRONLY,
      os.FileMode(meta.Mode),
   )
   if err != nil {
      fmt.Println(err)
      return
   }
   defer file.Close()
   // Copy the file contents
   _, err = io.CopyN(file, conn, int64(meta.Size))
   if err != nil {
      // Remove file in case of error
      os.Remove(path)
      fmt.Println(err)
      return
   }
   fmt.Printf("Received file %s: %d bytes\n", string(name), meta.Size)
}
```

La primera operación es una lectura de tamaño fijo de los metadatos del archivo. Luego leemos el nombre del archivo. Observa la verificación de longitud del nombre de archivo antes de leerlo. Es un enfoque defensivo importante para validar y limitar todas las asignaciones de memoria que involucran tamaños leídos de un sistema o usuario externo. Aquí rechazamos nombres de archivo que tengan más de 255 bytes. Luego creamos el archivo usando el modo dado y usamos `io.CopyN` para leer exactamente los bytes del tamaño del archivo desde la entrada. En caso de error, eliminamos el archivo parcialmente descargado.

#### Creación de un Cliente/Servidor TLS

Transport Layer Security (TLS) proporciona cifrado de extremo a extremo sin revelar la clave de cifrado para evitar ataques de intermediario (*man-in-the-middle*). También proporciona autenticación de pares y garantías de integridad de mensajes.

Un par de claves criptográficas contiene una clave privada y una clave pública. La clave privada se mantiene en secreto y la clave pública se publica.
- **Cifrado**: Dado que la clave pública se publica, cualquiera puede crear un mensaje, cifrarlo con la clave pública y enviarlo a la parte que tiene la clave privada. Solo el propietario de la clave privada puede descifrar ese mensaje.
- **Integridad**: El propietario de una clave privada puede crear una firma (*hash*) de un mensaje usando su clave privada. Cualquiera que tenga la clave pública puede verificar la integridad del mensaje.

Las claves públicas se distribuyen en forma de certificados digitales emitidos por una Autoridad de Certificación (CA).

##### Cómo hacerlo...

Para el servidor:
1. Carga el certificado usando `crypto/tls.LoadX509KeyPair`.
2. Crea un `crypto/tls.Config` usando el certificado.
3. Crea un *listener* usando `crypto/tls.Listen`.

```go
var (
   address = flag.String( "a", ":4433", "Address to listen")
   certificate = flag.String( "c", "../server.crt", "Certificate file")
   key = flag.String( "k", "../privatekey.pem", "Private key")
)
func main() {
   flag.Parse()
   // 2.1 Load the key pair
   cer, err := tls.LoadX509KeyPair(*certificate, *key)
   if err != nil {
      panic(err)
   }
   // 2.2 Create TLS configuration for the listener
   config := &tls.Config{
      Certificates: []tls.Certificate{cer},
   }
   // 2.3 Create the listener
   listener, err := tls.Listen("tcp", *address, config)
   if err != nil {
      panic(err)
      return
   }
   defer listener.Close()
   fmt.Println("Listening TLS on ", listener.Addr())
   // 2.4 Listen to incoming TCP connections
   for {
      conn, err := listener.Accept()
      if err != nil {
         fmt.Println(err)
         return
      }
      go handleConnection(conn)
   }
}
```

Para el cliente:
1. Si utilizas un certificado de una CA conocida, usa `crypto/x509.SystemCertPool`. Si tienes un certificado autofirmado, crea un grupo de certificados vacío usando `crypto/x509.NewCertPool`.
2. Carga el certificado del servidor y agrégalo al grupo de certificados.
3. Usa `crypto/tls.Dial` con una configuración TLS inicializada utilizando el grupo de certificados.

```go
var (
   addr = flag.String( "addr", "", "Server address")
   certFile = flag.String( "cert", "../server.crt", "TLS certificate file")
)
func main() {
   flag.Parse()
   // 3.1 Create new certificate pool
   roots := x509.NewCertPool()
   // 3.2 Load server certificate
   certData, err := os.ReadFile(*certFile)
   if err != nil {
      panic(err)
   }
   ok := roots.AppendCertsFromPEM(certData)
   if !ok {
      panic("failed to parse root certificate")
   }
   // 3.3 Connect the server
   conn, err := tls.Dial("tcp", *addr, &tls.Config{
      RootCAs: roots,
   })
   if err != nil {
      panic(err)
   }
   // 3.4 Send a line of text
   text := []byte("Hello echo server!")
   conn.Write(text)
   // Read the response
   response := make([]byte, len(text))
   conn.Read(response)
   fmt.Println(string(response))
   conn.Close()
}
```

#### Un *Proxy* TCP para Terminación TLS y Balanceo de Carga

La mayoría de las aplicaciones orientadas a Internet utilizan un *proxy* inverso (*ingress*) para separar los recursos internos del mundo exterior. El *proxy* inverso suele recibir conexiones de clientes externos mediante conexiones cifradas (TLS) y reenvía las solicitudes a los servicios de *backend* a través de canales no cifrados o recifrando la conexión mediante la CA interna.

#### Figura 13.1 – Proxy TLS con Balanceo de Carga Round-Robin y Terminación TLS

##### Cómo hacerlo...

```go
var (
   tlsAddress = flag.String( "a", ":4433", "TLS address to listen")
   serverAddresses = flag.String( "s", ":8080", "Server addresses, comma separated")
   certificate = flag.String( "c", "../server.crt", "Certificate file")
   key = flag.String( "k", "../privatekey.pem", "Private key")
)
func main() {
   flag.Parse()
   // 1. Create external facing TLS receiver
   // Load the key pair
   cer, err := tls.LoadX509KeyPair(*certificate, *key)
   if err != nil {
      panic(err)
   }
   // Create TLS configuration for the listener
   config := &tls.Config{
      Certificates: []tls.Certificate{cer},
   }
   // Create the tls listener
   tlsListener, err := tls.Listen("tcp", *tlsAddress, config)
   if err != nil {
      panic(err)
   }
   defer tlsListener.Close()
   fmt.Println("Listening TLS on ", tlsListener.Addr())
   // Listen to incoming TLS connections
   servers := strings.Split(*serverAddresses, ",")
   fmt.Println("Forwarding to servers: ", servers)
   nextServer := 0
   for {
      // 2. Listen to incoming TLS connections
      conn, err := tlsListener.Accept()
      if err != nil {
         fmt.Println(err)
         return
      }
      retries := 0
      for {
         // 3. Select the next server
         server := servers[nextServer]
         nextServer++
         if nextServer >= len(servers) {
            nextServer = 0
         }
         // Start a connection to this server
         targetConn, err := net.Dial("tcp", server)
         if err != nil {
            retries++
            fmt.Errorf("Cannot connect to %s", server)
            if retries > len(servers) {
               panic("None of the servers are available")
            }
            continue
         }
         // 4. Start the proxy
         go handleProxy(conn, targetConn)
      }
   }
}
```

El manejador del *proxy*:

```go
func handleProxy(conn, targetConn net.Conn) {
   defer conn.Close()
   defer targetConn.Close()
   // Copy data from the client to the server
   go io.Copy(targetConn, conn)
   // Copy data from the server to the client
   io.Copy(conn, targetConn)
}
```

#### Configuración de Límites de Tiempo (*Deadlines*) de Lectura/Escritura

Si no deseas esperar indefinidamente a que un host conectado envíe datos o reciba los datos que enviaste, debes establecer un límite de tiempo (*deadline*).

##### Cómo hacerlo...

Establece el límite de tiempo antes de la operación:

```go
conn.SetDeadline(time.Now().Add(timeoutSeconds * time.Second))
if n, err:=conn.Read(data); err!=nil {
   if errors.Is(err, os.ErrDeadlineExceeded) {
      // Deadline exceeded.
   } else {
      // Some other error
   }
}
```

Si continuarás usando la conexión después de que se exceda un plazo, debes restablecerlo:

```go
conn.SetDeadline(time.Time{})
```

#### Desbloqueo de una Operación de Lectura o Escritura Bloqueada

##### Cómo hacerlo...

Si deseas desbloquear una operación de E/S sin intención de reutilizar la conexión, cierra la conexión de forma asíncrona:

```go
cancel:=make(chan struct{})
done:=make(chan struct{})
// Close the connection if a message is sent to cancel channel
go func() {
   select {
   case <-cancel:
      conn.Close()
   case <-done:
   }
}()
go handleConnection(conn)
```

Si deseas desbloquear una operación de E/S pero no terminar la conexión, establece el plazo para ahora:

```go
unblock:=make(chan struct{})
// Unblock the connection if a message is sent to unblock channel
go func() {
   <-unblock
   conn.SetDeadline(time.Now())
}()
timedout:=false
if n, err:=conn.Read(data); err!=nil {
   if errors.Is(err,os.ErrDeadlineExceeded) {
      // Reset connection deadline
      conn.SetDeadline(time.Time{})
      timedout=true // continue using the connection
   } else {
      // Handle error
   }
}
if timedout {
   // Read timedout
} else {
   // Read did not timeout
}
```

#### Creación de Clientes/Servidores UDP

A diferencia de TCP, UDP no está orientado a conexión (*connectionless*). Eso significa que en lugar de establecer una conexión con otro par y enviar datos de ida y vuelta, simplemente envías y recibes paquetes de datos. No existen garantías de entrega u orden.

##### Cómo hacerlo...

Servidor UDP:

```go
addr, err := net.ResolveUDPAddr("udp4", *address)
if err != nil {
   panic(err)
}
// Create a UDP connection
conn, err := net.ListenUDP("udp4", addr)
if err != nil {
   panic(err)
}
defer conn.Close()
// Listen to incoming UDP connections
buf := make([]byte, 1024)
n, remoteAddr, err := conn.ReadFromUDP(buf)
if err != nil {
   // Handle the error
}
fmt.Printf("Received %d bytes from %s\n", n, remoteAddr)
if n > 0 {
   _, err := conn.WriteToUDP(buf[:n], remoteAddr)
   if err != nil {
      // Handle the error
   }
}
```

Cliente UDP:

```go
addr, err := net.ResolveUDPAddr("udp4", *serverAddress)
if err != nil {
   panic(err)
}
// Create a UDP connection, local address chosen randomly
conn, err := net.DialUDP("udp4", nil, addr)
if err != nil {
   panic(err)
}
fmt.Printf("UDP server %s\n", conn.RemoteAddr())
defer conn.Close()
// Send a line of text
text := []byte("Hello echo server!")
n, err := conn.Write(text)
if err != nil {
   panic(err)
}
fmt.Printf("Written %d bytes\n", n)
// Read the response
response := make([]byte, 1024)
conn.ReadFromUDP(response)
```

---

### Sección 2: Trabajo con HTTP

HTTP es un protocolo cliente-servidor donde el cliente (un agente de usuario o *proxy*) envía solicitudes a un servidor y el servidor devuelve una respuesta.

#### Realización de Peticiones HTTP

##### Cómo hacerlo...

Usando el cliente predeterminado:

```go
response, err := http.Get("http://example.com")
if err!=nil {
   // Handle error
}
// Always close response body
defer response.Body.Close()
if response.StatusCode/100==2 {
   // HTTP 2xx, call was successful.
   // Work with response.Body
}
```

Configurando un `http.Client`:

```go
client:=http.Client{
   // Set a timeout for all outgoing calls.
   // If the call does not complete within 30 seconds, timeout.
   Timeout: 30*time.Second,
}
response, err:=client.Get("http://example.com")
if err!=nil {
   // handle error
}
// Always close response body
defer response.Body.Close()
```

Petición HTTPS pública:

```go
response, err := http.Get("https://example.com")
```

Con CA personalizada:

```go
roots := x509.NewCertPool()
certData, err := os.ReadFile(*certFile)
if err != nil {
   panic(err)
}
ok := roots.AppendCertsFromPEM(certData)
if !ok {
   panic("failed to parse root certificate")
}
config:=tls.Config{
   RootCAs: roots,
}
transport := &http.Transport {
   TLSClientConfig: &config,
}
client:= &http.Client{
   Transport: transport,
}
resp, err:=client.Get(url)
if err!=nil {
   // Handle error
}
defer resp.Body.Close()
```

> **Consejo**  
> Cierra siempre el cuerpo de la respuesta (`response.Body.Close()`) cuando hayas terminado de trabajar con él e intenta leer todos los datos disponibles en el cuerpo.

#### Ejecución de un Servidor HTTP

##### Cómo hacerlo...

```go
func myHandler(w http.ResponseWriter, req *http.Request) {
   if req.Method == http.MethodGet {
      // Handle an HTTP GET request
   }
   ...
}
```

```go
err:=http.ListenAndServe(":8080",http.HandlerFunc(myHandler))
log.Fatal(err)
```

O utilizando una estructura `http.Server`:

```go
server := http.Server {
   // The address to listen
   Addr: ":8080",
   // The handler function
   Handler: http.HandlerFunc(myHandler),
   // The handlers must read the request within 10 seconds
   ReadTimeout: 10*time.Second,
   // The headers of a request must be read within 5 seconds
   ReadHeaderTimeout: 5*time.Second,
}
err:=server.ListenAndServe()
log.Fatal(err)
```

#### HTTPS: Configuración de un Servidor TLS

##### Cómo hacerlo...

```go
server := http.Server {
   Addr: ":4443",
   Handler: handler,
}
server.ListenAndServeTLS("cert.pem", "key.pem")
```

O con `http.HandleFunc`:

```go
http.HandleFunc("/",func(w http.ResponseWriter, req *http.Request) {
   // Handle request
})
http.ListenAndServeTLS("cert.pem", "key.pem")
```

O configurando `tls.Config`:

```go
cert, err := tls.LoadX509KeyPair("cert.pem", "key.pem")
if err!=nil {
   panic(err)
}
tlsConfig := &tls.Config{
   Certificates: []tls.Certificate{cert},
}
server := http.Server{
   Addr: ":4443",
   Handler: handler,
   TLSConfig: tlsConfig,
}
server.ListenAndServeTLS("","")
```

#### Creación de *Handlers* HTTP

##### Cómo hacerlo...

Función anónima:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /health",func(w http.ResponseWriter, req *http.Request) {
   w.Write([]byte("Ok"))
})
...
server := http.Server {
   Handler: mux,
   Addr: ":8080",
   ...
}
server.ListenAndServe()
```

Estructura que implementa `http.Handler`:

```go
// The RandomService reads random data from a source, and
// returns random numbers
type RandomService struct {
   rndSource io.Reader
}
func (svc RandomService) ServeHTTP(w http.ResponseWriter, req *http.Request) {
   // Read 4 bytes from the random number source, convert it to string
   data:= make([]byte,4)
   _,err:=svc.rndSource.Read(data)
   if err!=nil {
      // This will return an HTTP 500 error with the error message
      // as the message body
      http.Error(w,err.Error(),http.StatusInternalServerError)
      return
   }
   // Decode random data using binary little endian encoding
   value:=binary.LittleEndian.Uint32(data)
   // Write the data to the output
   w.Write([]byte(strconv.Itoa(int(value))))
}
```

```go
file, err:=os.Open("/dev/random")
if err!=nil {
   panic(err)
}
defer file.Close()
svc:=RandomService {
   rndSource: file,
}
mux:=http.NewServeMux()
mux.Handle("GET /rnd", svc)
server := http.Server {
   Handler: mux,
   Addr: ":8080",
   ...
}
server.ListenAndServe()
```

Múltiples métodos *handler* en una estructura:

```go
type UserHandler struct {
   DB *sql.DB
}
func (hnd UserHandler) GetUser(w http.ResponseWriter, req *http.Request) {
   // User req.PathValue("userId") to get userId portion of /users/
   // {userId}
   // That is, if this API is invoked with GET /users/123, then after
   // the following line `userId` is assigned to "123"
   userId:=req.PathValue("userId")
   // Get user data from the DB
   user, err:=GetUserInformation(hnd.DB,userId)
   if err!=nil {
      http.Error(w,err.Error(),http.StatusNotFound)
      return
   }
   // Marshal user data to JSON
   data, err:=json.Marshal(user)
   if err!=nil {
      http.Error(w, err.Error(),http.StatusInternalServerError)
      return
   }
   // Set the content type header. You **must** set all headers before
   // writing the body. Once the body is placed on the write, there is
   // no way to change a header that is already written.
   w.Header().Set("Content-Type","application/json")
   w.Write(data)
}
```

```go
userDb, err:=sql.Open(driver, UserDBUrl)
if err!=nil {
   panic(err)
}
userHandler := UserHandler {
   DB: userDb,
}
mux := http.NewServeMux()
mux.Handle("GET /users/{userId}",userHandler.GetUser)
mux.Handle("POST /users", userHandler.NewUser)
mux.Handle("DELETE /users/{userId}", userHandler.DeleteUser)
server := http.Server{
   Addr: serverAddr,
   Handler: mux,
}
server.ListenAndServe()
```

#### Servir Archivos Estáticos en el Sistema de Archivos

##### Cómo hacerlo...

```go
fileHandler := http.FileServer(http.Dir("/var/www"))
server:=http.Server{
   Addr: addr,
   Handler: fileHandler,
}
http.ListenAndServe()
```

Con prefijo en la URL:

```go
fileHandler := http.StripPrefix("/static/", http.FileHandler(http.Dir("/var/www"))
```

Personalizado con `http.FileSystem`:

```go
// Serve only HTML files in the given directory
type htmlFS struct {
   fs *http.FileSystem
}
// Filter file names by their extension before opening them
func (h htmlFS) Open(name string) (http.File, error) {
   if strings.ToLower(filepath.Ext(name))==".html" {
      return h.fs.Open(name)
   }
   return nil, os.ErrNotFound
}
...
htmlHandler := http.FileHandler(htmlFS{fs:http.Dir("/var/www")})
// htmlHandler serves all HTML files under /var/www
```

#### Manejo de Formularios HTML

##### Cómo hacerlo...

Formulario del lado del cliente:

```html
<form method="POST" action="/auth/login">
   <input type="text" name="userName">
   <input type="password" name="password">
   <button type="submit">Submit</button>
</form>
```

Manejador del lado del servidor:

```go
type UserHandler struct {
   Auth Authenticator
}
func (h UserHandler) HandleLogin(w http.ResponseWriter, req *http.Request) {
   // Parse the submitted form. This fills up req.PostForm
   // with the submitted information
   if err:=req.ParseForm(); err!=nil {
      http.Error(w, err.Error(), http.StatusBadRequest)
      return
   }
   // Get the submitted fields
   userName := req.PostForm.Get("userName")
   password := req.PostForm.Get("password")
   // Handle the login request, and get a cookie
   cookie,err:=h.Auth.Authenticate(userName,password);
   if err!=nil {
      // Send the user back to login page, setting an error
      // cookie containing an error message
      http.SetCookie(w,h.NewErrorCookie("Username or password invalid"))
      http.Redirect(w, req, "/login.html", http.StatusFound)
      return
   }
   // Set the cookie representing user session
   http.SetCookie(w,cookie)
   // Redirect the user to the main page
   http.Redirect(w,req,"/dashboard.html",http.StatusFound)
}
```

```go
userHandler := UserHandler {
   Auth: authenticator,
}
mux := http.NewServeMux()
mux.HandleFunc("POST /auth/login", userHandler.HandleLogin)
mux.HandleFunc("GET /login.html", userHandler.ShowLoginPage)
```

Manejo de errores con cookies en la página de inicio de sesión:

```go
func (h UserHandler) ShowLoginPage(w http.ResponseWriter, req *http.Request) {
   loginFormData:=map[string]any{}
   cookie, err:= req.Cookie("error_cookie")
   if err==nil {
      loginFormData["error"] = cookie.Value
      // Unset the cookie
      http.SetCookie(w, &http.Cookie {
         Name: "error_cookie",
         MaxAge: 0,
      })
   }
   w.Header().Set("Content-Type","text/html")
   loginFormTemplate.Execute(w,loginFormData)
}
func (h UserHandler) NewErrorCookie(msg string) *http.Cookie {
   return &http.Cookie {
      Name: "error_cookie",
      Value: msg,
      MaxAge: 60, // Cookie lives for 60 seconds
      Path: "/",
   }
}
```

#### Creación de un *Handler* para Descarga de Archivos Grandes

##### Cómo hacerlo...

```go
func DownloadHandler(w http.ResponseWriter, req *http.Request) {
   fileName := req.PathValue("fileName")
   f, err:= os.Open(filepath.Join("/data",fileName))
   if err!=nil {
      http.Error(w,err.Error(),http.StatusNotFound)
      return
   }
   defer f.Close()
   w.Header().Set("Content-Type","application/octet-stream")
   w.Header().Set("Content-Length", strconv.Itoa(f.Length()))
   io.Copy(w,f)
}
```

#### Manejo de Subidas de Archivos y Formularios HTTP como *Stream*

##### Cómo hacerlo...

Formulario del lado del cliente:

```html
<form action="/upload" method="post" enctype="multipart/form-data">
   <input type="text" name="textField">
   <input type="file" name="fileField">
   <button type="submit">submit</button>
</form>
```

Lado del servidor:

```go
reader, err:=request.MultipartReader()
if err!=nil {
   http.Error(w,"Not a multipart request",http.StatusBadRequest)
   return
}
for {
   part, err:= reader.NextPart()
   if errors.Is(err,io.EOF) {
      break
   }
   if err!=nil {
      http.Error(w,err.Error(),http.StatusBadRequest)
      return
   }
}
```

```go
formValues:=make(url.Values)
if fileName:=part.FileName(); fileName!="" {
   // This part contains a file
   output, err:=os.Create(fileName)
   if err!=nil {
      // Handle error
   }
   defer output.Close()
   if err:=io.Copy(output,part); err!=nil {
      // Handle error
   }
} else if fieldName := part.FormName(); fieldName!="" {
   // This part contains form data for an input field
   data, err := io.ReadAll(part)
   if err!=nil {
      // Handle error
   }
   formValues[fieldName]=append(formValues[fieldName], string(data))
}
```

