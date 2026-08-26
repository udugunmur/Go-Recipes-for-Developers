# Parte 2: Redes, Flujos de Datos y Servicios

## Capítulo 14: Entrada/Salida en *Streaming* (I/O)

Hay flexibilidad y elegancia en la simplicidad. A diferencia de varios lenguajes que decidieron implementar un *framework* de transmisión (*streaming*) repleto de funciones complejas, Go eligió un enfoque simple basado en capacidades: un lector (*reader*) es algo de lo que se leen bytes, y un escritor (*writer*) es algo en lo que se escriben bytes. Los búferes en memoria, archivos, conexiones de red, etc., son todos lectores y escritores, definidos por las interfaces `io.Reader` e `io.Writer`. Un archivo también es un `io.Seeker`, ya que puedes cambiar aleatoriamente la ubicación de lectura/escritura, pero una conexión de red no lo es. Un archivo y una conexión de red se pueden cerrar, por lo que ambos son `io.Closer`, pero un búfer de memoria no lo es. Estas abstracciones simples y elegantes son la clave para escribir algoritmos que se pueden utilizar en diferentes contextos.

En este capítulo, veremos algunas recetas que muestran cómo se puede utilizar este marco de transmisión basado en capacidades de forma idiomática. También veremos cómo trabajar con archivos y el sistema de archivos.

Las recetas cubiertas en este capítulo se encuentran en las siguientes secciones principales:
- *Readers* y *writers*
- Trabajo con archivos
- Trabajo con datos binarios
- Copia de datos
- Trabajo con el sistema de archivos
- Trabajo con tuberías (*pipes*)

---

### Sección 1: *Readers* y *Writers*

Las interfaces fundamentales de E/S en Go son:
- `io.Reader`: `Read(p []byte) (n int, err error)`
- `io.Writer`: `Write(p []byte) (n int, err error)`

#### Creación de un *Reader*/*Writer* a Partir de *Slices* de Bytes y Cadenas

##### Cómo hacerlo...

```go
// Create a reader from a string
reader := strings.NewReader("some string")
// Create a reader from a byte slice
reader2 := bytes.NewReader([]byte("some other string"))
// Create a writer using a byte slice
var writer bytes.Buffer
// Write string to a writer
io.WriteString(writer, "yet another string")
// Create a writer from a strings.Builder
var writer2 strings.Builder
io.WriteString(writer2, "more string")
```

---

### Sección 2: Trabajo con Archivos

Los archivos en Go se gestionan principalmente a través del paquete `os`. Un `*os.File` implementa `io.Reader`, `io.Writer`, `io.Seeker`, `io.ReaderAt`, `io.WriterAt` e `io.Closer`.

#### Creación y Apertura de Archivos

##### Cómo hacerlo...

```go
// Open a file for reading
file, err := os.Open("somefile.txt")
if err != nil {
   return err
}
defer file.Close()
```

```go
// Create a new file, or truncate if it already exists
file, err := os.Create("somefile.txt")
if err != nil {
   return err
}
defer file.Close()
```

```go
// Open a file for appending
file, err := os.OpenFile("somefile.txt", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0600)
if err != nil {
   return err
}
defer file.Close()
```

```go
// Open a file for reading and writing, do not create if it does not exist
file, err := os.OpenFile("somefile.txt", os.O_RDWR, 0600)
if err != nil {
   return err
}
defer file.Close()
```

```go
// Open a file for writing, create if it does not exist, fail if it already exists
file, err := os.OpenFile("somefile.txt", os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0600)
if err != nil {
   return err
}
defer file.Close()
```

Banderas principales de `os.OpenFile`:

| Bandera | Descripción |
| --- | --- |
| `O_RDONLY` | Abre el archivo solo para lectura. |
| `O_WRONLY` | Abre el archivo solo para escritura. |
| `O_RDWR` | Abre el archivo para lectura y escritura. |
| `O_APPEND` | Añade datos al archivo al escribir. |
| `O_CREATE` | Crea un nuevo archivo si no existe. |
| `O_EXCL` | Usado con `O_CREATE`, el archivo no debe existir. |
| `O_SYNC` | Abre para E/S síncrona. |
| `O_TRUNC` | Trunca el archivo regular existente al abrirlo. |

#### Cierre Seguro con `defer` y Clausuras

Al escribir archivos, `Close()` puede devolver un error si los datos del búfer no se pudieron descargar al disco. Por lo tanto, debes capturar el error de `Close()`:

```go
func processFile(filename string) (err error) {
   file, err := os.Open(filename)
   if err != nil {
      return err
   }
   defer func() {
      closeErr := file.Close()
      if err == nil {
         err = closeErr
      }
   }()
   // Process file...
   return nil
}
```

#### Lectura y Escritura en una Ubicación Específica

##### Cómo hacerlo...

Uso de `Seek`:

```go
// Seek to offset from beginning, current position, or end
file.Seek(offset, io.SeekStart) // io.SeekCurrent, io.SeekEnd
```

Uso de `ReadAt` y `WriteAt` (acceso posicional sin alterar el puntero de archivo global):

```go
// ReadAt reads without moving file pointer
n, err := file.ReadAt(buf, offset)
// WriteAt writes without moving file pointer
n, err := file.WriteAt(data, offset)
```

#### Truncado y Redimensionamiento de Archivos

##### Cómo hacerlo...

```go
err := file.Truncate(size)
// or
err := os.Truncate("somefile.txt", size)
```

---

### Sección 3: Trabajo con Datos Binarios

Para la codificación binaria en Go se utiliza el paquete `encoding/binary`.

#### Codificación Binaria (*Endianness*)

##### Cómo hacerlo...

```go
// Little Endian
binary.LittleEndian.PutUint32(buf, val)
val := binary.LittleEndian.Uint32(buf)
// Big Endian
binary.BigEndian.PutUint32(buf, val)
val := binary.BigEndian.Uint32(buf)
```

#### Codificación de Longitud-Valor (LV) y Tipo-Longitud-Valor (TLV)

##### Cómo hacerlo...

```go
// Writing LV
err := binary.Write(writer, binary.BigEndian, uint32(len(data)))
if err != nil { return err }
_, err = writer.Write(data)
```

---

### Sección 4: Copia de Datos

El paquete `io` proporciona utilidades eficientes para copiar datos entre cualquier combinación de `io.Reader` e `io.Writer`.

#### Copia con `io.Copy`, `io.CopyN` e `io.CopyBuffer`

##### Cómo hacerlo...

```go
written, err := io.Copy(dst, src)
written, err := io.CopyN(dst, src, n)
written, err := io.CopyBuffer(dst, src, buf)
```

---

### Sección 5: Trabajo con el Sistema de Archivos

Go proporciona paquetes como `path/filepath` y funciones dentro de `os` para manipular rutas y directorios de forma independiente de la plataforma.

#### Rutas y Archivos Temporales

##### Cómo hacerlo...

```go
dir, file := filepath.Split(path)
joined := filepath.Join("a", "b", "c")
tempDir, err := os.MkdirTemp("", "prefix")
tempFile, err := os.CreateTemp("", "prefix")
```

#### Lectura y Recorrido de Directorios

##### Cómo hacerlo...

Lectura plana de un directorio:

```go
entries, err := os.ReadDir(dir)
for _, entry := range entries {
   info, _ := entry.Info()
   fmt.Println(entry.Name(), entry.IsDir(), info.Size())
}
```

Recorrido recursivo con `filepath.WalkDir`:

```go
err := filepath.WalkDir(root, func(path string, d os.DirEntry, err error) error {
   if err != nil { return err }
   fmt.Println(path, d.IsDir())
   return nil
})
```

---

### Sección 6: Trabajo con Tuberías (*Pipes*)

Las tuberías en memoria y los adaptadores de E/S permiten conectar lectores y escritores de forma síncrona o duplicar flujos de datos.

#### Tuberías en Memoria (`io.Pipe`)

##### Cómo hacerlo...

```go
pr, pw := io.Pipe()
go func() {
   // Write to pw
   pw.Write([]byte("data"))
   pw.Close()
}()
// Read from pr
io.Copy(os.Stdout, pr)
```

#### Divisores y Combinadores de Flujos

##### Cómo hacerlo...

- `io.TeeReader`: Devuelve un `Reader` que escribe en un `Writer` todo lo que lee de la fuente (similar al comando Unix `tee`).
- `io.MultiReader`: Concatena múltiples lectores secuencialmente.
- `io.MultiWriter`: Duplica cada escritura en múltiples destinos concurrentemente.

```go
tee := io.TeeReader(src, dstWriter)
multiR := io.MultiReader(r1, r2, r3)
multiW := io.MultiWriter(w1, w2, w3)
```

