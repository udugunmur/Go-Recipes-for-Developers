# Parte 3: Persistencia, Diagnóstico y Calidad de Código

## Capítulo 16: Registro de Eventos (*Logging*)

Imprimir mensajes de registro (*logs*) desde un programa puede ser una herramienta importante para la resolución de problemas. Los mensajes de registro indican qué está sucediendo en cualquier momento dado y proporcionan información contextual muy necesaria cuando algo sale mal. La biblioteca estándar de Go proporciona paquetes convenientes para generar y administrar mensajes de registro desde programas. Aquí, veremos el uso del paquete `log`, que se puede usar para generar mensajes de texto, y el paquete `log/slog`, que se puede usar para generar mensajes de registro estructurados.

Este capítulo contiene las siguientes recetas:
- Uso del *logger* estándar
  - Escritura de mensajes de *log*
  - Control de formato
  - Cambio de destino de los *logs*
- Uso del *logger* estructurado
  - Registro mediante el *logger* global
  - Escritura de *logs* estructurados usando diferentes niveles
  - Cambio de nivel de *log* en tiempo de ejecución
  - Uso de *loggers* con atributos adicionales
  - Adición de información de *logging* desde el contexto

---

### Sección 1: Uso del *Logger* Estándar

El registrador de la biblioteca estándar se define en el paquete `log`. Es una biblioteca de registro simple que escribe mensajes formateados en `os.Stderr`.

#### Escritura de Mensajes de *Log*

##### Cómo hacerlo...

```go
log.Println("This is a simple log message")
log.Printf("User %s logged in from %s", username, ip)
// Fatal prints and calls os.Exit(1)
// Panic prints and panics
```

#### Control de Formato

##### Cómo hacerlo...

```go
// Setting flags
log.SetFlags(log.Ldate | log.Ltime | log.Lmicroseconds | log.Lshortfile)
log.SetPrefix("[AUTH] ")
```

Banderas de formato de `log`:

| Bandera | Descripción |
| --- | --- |
| `log.Ldate` | Fecha en la zona horaria local: `2009/01/23` |
| `log.Ltime` | Hora en la zona horaria local: `01:23:23` |
| `log.Lmicroseconds` | Resolución de microsegundos: `01:23:23.123123` |
| `log.Llongfile` | Ruta de archivo completa y número de línea: `/a/b/c/d.go:23` |
| `log.Lshortfile` | Nombre de archivo final y número de línea: `d.go:23` |
| `log.LUTC` | Usar UTC en lugar de la zona horaria local |
| `log.Lmsgprefix` | Mover el prefijo antes del mensaje en lugar de al inicio de la línea |
| `log.LstdFlags` | Banderas estándar (`Ldate | Ltime`) |

#### Cambio de Destino de los *Logs*

##### Cómo hacerlo...

```go
logFile, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
if err != nil {
   log.Fatal(err)
}
defer logFile.Close()
log.SetOutput(logFile)
```

---

### Sección 2: Uso del *Logger* Estructurado (`log/slog`)

A partir de Go 1.21, el paquete `log/slog` proporciona registro estructurado de alto rendimiento con pares clave-valor y formato de salida en texto legible o JSON.

#### Registro Mediante el *Logger* Global

##### Cómo hacerlo...

```go
slog.Info("User logged in", "user_id", 123, "ip", "192.168.1.1")
slog.Warn("Disk space low", "free_mb", 256)
slog.Error("Database connection failed", "error", err)
```

#### Configuración de *Handlers* (JSON y Texto)

##### Cómo hacerlo...

```go
// JSON Handler
jsonHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})
logger := slog.New(jsonHandler)
logger.Debug("Debug info", "details", "foo")
```

#### Cambio de Nivel de *Log* en Tiempo de Ejecución

##### Cómo hacerlo...

```go
var programLevel = new(slog.LevelVar) // Info by default
h := slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{Level: programLevel})
slog.SetDefault(slog.New(h))

// Later at runtime:
programLevel.Set(slog.LevelDebug)
```

#### Uso de *Loggers* con Atributos Adicionales

##### Cómo hacerlo...

```go
reqLogger := logger.With("request_id", reqID, "service", "payment")
reqLogger.Info("Processing transaction", "amount", 100)
```

#### Adición de Información de *Logging* Desde el Contexto

##### Cómo hacerlo...

```go
type ContextHandler struct {
   slog.Handler
}

func (h ContextHandler) Handle(ctx context.Context, r slog.Record) error {
   if traceID, ok := ctx.Value("trace_id").(string); ok {
      r.AddAttrs(slog.String("trace_id", traceID))
   }
   return h.Handler.Handle(ctx, r)
}
```
