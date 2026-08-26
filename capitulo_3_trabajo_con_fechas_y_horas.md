# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 3: Trabajo con Fechas y Horas

Trabajar con fechas y horas puede resultar difícil en cualquier lenguaje de programación. La biblioteca estándar de Go ofrece herramientas fáciles de usar para trabajar con constructos de fecha y hora. Estos pueden ser algo diferentes de lo que muchas personas están acostumbradas. Por ejemplo, existen bibliotecas en diferentes lenguajes que hacen una distinción entre un tipo de hora y un tipo de fecha. La biblioteca estándar de Go solo incluye un tipo `time.Time`. Eso podría hacerte sentir desorientado cuando trabajes con el tiempo en Go.

Me gusta pensar que el tratamiento que Go da a la fecha y la hora reduce las posibilidades de generar errores sutiles. Verás, tienes que ser muy cuidadoso y claro sobre a qué te refieres cuando hablas de tiempo: ¿estás hablando de un punto en el tiempo o de un intervalo? Una fecha es en realidad un intervalo (por ejemplo, `08/01/2024` comienza en `08/01/2024T00:00:00` y continúa hasta `08/01/2024T23:59:59`), aunque por lo general esa no sea la intención. Un valor específico de fecha/hora también depende del lugar donde estés midiendo el tiempo. `2023-11-05T08:00` en Denver, Colorado, es diferente de `2023-11-05T08:00` en Berlín, Alemania. El tiempo siempre avanza, pero la fecha/hora puede saltar o retroceder: después de `2023-11-05T02:59` en Denver, Colorado, la hora vuelve a `2023-11-05T02:00` porque es cuando termina el horario de verano en Colorado. Por lo tanto, en realidad hay dos instancias temporales para `2023-11-05T02:10:10`, una en el Horario de Verano de la Montaña (*Mountain Daylight Time*) y otra en el Horario Estándar de la Montaña (*Mountain Standard Time*).

Hoy en día existen muchos errores de software en producción debido al manejo incorrecto del tiempo. Por ejemplo, si estás calculando cuándo finaliza la suscripción de un cliente, debes tener en cuenta la ubicación de ese cliente y la hora del día en que finaliza la suscripción; de lo contrario, sus suscripciones pueden cancelarse antes (o después) en su último día.

Este capítulo contiene las siguientes recetas para trabajar correctamente con fecha y hora:
- Trabajo con tiempo Unix (*Unix time*)
- Componentes de fecha y hora
- Aritmética de fecha y hora
- Formateo y parseo de fecha y hora
- Trabajo con zonas horarias
- Temporizadores (*Timers*)
- *Tickers*
- Almacenamiento de información de tiempo

---

### Sección 1: Trabajo con Tiempo Unix (*Unix Time*)

El tiempo Unix es el número de segundos (o milisegundos, microsegundos o nanosegundos) transcurridos desde el 1 de enero de 1970 UTC (la época o *epoch*). Go utiliza `int64` para representar estos valores, por lo que el tiempo Unix en segundos puede representar miles de millones de años en el pasado o en el futuro. El tiempo Unix en nanosegundos puede representar valores de fecha entre los años 1678 y 2262. El tiempo Unix es una medida absoluta de un instante expresado como la duración transcurrida desde (o hasta) la época. Es independiente de la ubicación, por lo que con dos tiempos Unix, $s$ y $t$, si $s < t$, entonces $s$ ocurrió antes de $t$, sin importar la ubicación. Debido a estas propiedades, el tiempo Unix se utiliza generalmente como una marca de tiempo (*timestamp*) que indica cuándo ocurrió un evento (cuándo se escribe un registro de log, cuándo se inserta un registro, etc.).

#### Cómo hacerlo...

Para obtener el tiempo Unix actual, usa lo siguiente:
- `time.Now().Unix() int64`: Tiempo Unix en segundos
- `time.Now().UnixMilli() int64`: Tiempo Unix en milisegundos
- `time.Now().UnixMicro() int64`: Tiempo Unix en microsegundos
- `time.Now().UnixNano() int64`: Tiempo Unix en nanosegundos

Dado un tiempo Unix, conviértelo a un tipo `time.Time` utilizando lo siguiente:
- `time.Unix(sec, nanosec int64) time.Time`: Traduce el tiempo Unix en segundos y/o nanosegundos a `time.Time`
- `time.UnixMilli(int64) time.Time`: Traduce el tiempo Unix en milisegundos a `time.Time`
- `time.UnixMicro(int64) time.Time`: Traduce el tiempo Unix en microsegundos a `time.Time`

Para traducir un tiempo Unix a la hora local, usa `localTime := time.Unix(unixTimeSeconds, 0).In(location)`, donde `location` es un `*time.Location` para la ubicación en la que se interpretará el tiempo Unix.

---

### Sección 2: Componentes de Fecha y Hora

Al trabajar con valores de fecha, a menudo debes componer una fecha/hora a partir de sus componentes o necesitas acceder a los componentes de un valor de fecha/hora. Esta receta muestra cómo se puede realizar.

#### Cómo hacerlo...

- Para construir un valor de fecha/hora a partir de sus partes, utiliza la función `time.Date`.
- Para obtener las partes de un valor de fecha/hora, utiliza los métodos de `time.Time`:
  - `time.Day() int`
  - `time.Month() time.Month`
  - `time.Year() int`
  - `time.Date() (year, month, day int)`
  - `time.Hour() int`
  - `time.Minute() int`
  - `time.Second() int`
  - `time.Nanosecond() int`
  - `time.Zone() (name string, offset int)`
  - `time.Location() *time.Location`

`time.Date` creará un valor de tiempo a partir de sus componentes:

```go
d := time.Date(2020, 3, 31, 15, 30, 0, 0, time.UTC)
fmt.Println(d)
// 2020-03-31 15:30:00 +0000 UTC
```

La salida se normalizará, de la siguiente manera:

```go
d := time.Date(2020, 3, 0, 15, 30, 0, 0, time.UTC)
fmt.Println(d)
// 2020-02-29 15:30:00 +0000 UTC
```

Dado que el día del mes comienza en 1, crear una fecha con el día 0 dará como resultado el último día del mes anterior.

Una vez que tengas un valor `time.Time`, puedes obtener sus componentes:

```go
d := time.Date(2020, 3, 0, 15, 30, 0, 0, time.UTC)
fmt.Println(d.Day())
// 29
```

Nuevamente, `time.Date` normaliza el valor de la fecha, por lo que `d.Day()` devolverá 29.

---

### Sección 3: Aritmética de Fecha y Hora

La aritmética de fecha/hora es necesaria para responder a preguntas como las siguientes:
- ¿Cuánto tiempo tardó en completarse una operación?
- ¿Qué hora será dentro de 5 minutos?
- ¿Cuántos días faltan para el próximo mes?

Esta receta muestra cómo puedes responder a estas preguntas utilizando el paquete `time`.

#### Cómo hacerlo...

- Para averiguar cuánto tiempo ha transcurrido entre dos instantes en el tiempo, usa el método `Time.Sub` para restarlos.
- Para encontrar la duración desde el momento actual hasta un tiempo posterior, usa `time.Until(laterTime)`.
- Para encontrar cuánto tiempo ha transcurrido desde un momento dado, usa `time.Since(beforeTime)`.
- Para averiguar qué hora será después de una duración determinada, usa el método `Time.Add`. Usa una duración negativa para encontrar el tiempo anterior a una cierta duración.
- Para sumar o restar años, meses o días a/de un tiempo, usa el método `Time.AddDate`.
- Para comparar dos valores `time.Time`, usa lo siguiente:
  - `Time.Equal` para verificar si dos valores de tiempo representan el mismo instante.
  - `Time.Before` o `Time.After` para verificar si un valor de tiempo es anterior o posterior a un valor de tiempo dado.

#### Cómo funciona...

Un tipo `time.Duration` representa el tiempo transcurrido entre dos instantes en nanosegundos como un valor `int64`. En otras palabras, si restas un valor `time.Time` de otro, obtienes un `time.Duration`:

```go
dur := tm1.Sub(tm2)
```

Dado que `Duration` es un `int64` que representa nanosegundos, puedes realizar operaciones aritméticas de duración:

```go
// Add 1 day to duration
dur+=time.Hour*24
```

Ten en cuenta que la última operación anterior también implica una multiplicación, ya que `time.Hour` es de tipo `time.Duration`.

Puedes sumar un valor de duración a un valor `time.Time`:

```go
now := time.Now()
then := now.Add(dur)
```

> **Consejo**  
> El hecho de que `Duration` sea un `int64` significa que un valor `time.Duration` está limitado a alrededor de 290 años. Esto debería ser suficiente para la mayoría de los casos prácticos. Sin embargo, si este no es tu caso, necesitarás construir una solución personalizada o buscar una biblioteca de terceros.

Puedes restar la duración de un valor `time.Time` sumando un valor de duración negativo:

```go
fmt.Println( then.Add(-dur).Equal(now) )
```

Observa el uso del método `Time.Equal`. Este compara dos instancias de tiempo teniendo en cuenta sus zonas horarias, que pueden ser diferentes. Por ejemplo, `Time.Equal` devolverá `true` para `2024-01-09 09:00 MST` y `2024-01-09 08:00 PST`.

Usa `Time.Before` y `Time.After` para comparar valores de tiempo. Por ejemplo, puedes comprobar si un objeto con fecha de vencimiento ha caducado mediante lo siguiente:

```go
if object.Expiration.After(time.Now()) {
   // Object expired
}
```

También puedes sumar años, meses o días a una fecha determinada:

```go
t:=time.Now()
// Subtract 1 year from now to get this moment in last year
lastYear := t.AddDate(-1,0,0)
// Add 1 day to get same time tomorrow
tomorrow := t.AddDate(0,0,1)
// Add 1 day to get the next month
nextMonth := t.AddDate(0,1,0)
```

El resultado de estas operaciones será normalizado. Por ejemplo, si restas un año a `2020-02-29`, obtendrás `2019-03-01`. Esto puede causar problemas cuando trabajas con una fecha al final de un mes y tienes que sumar/restar valores de mes. Sumar un mes a `2020-03-31` dos veces dará `2020-06-01`, pero sumar dos meses directamente dará `2020-05-31`:

```go
d := time.Date(2020, 3, 31, 0, 0, 0, 0, time.UTC)
fmt.Println(d.AddDate(0, 1, 0).AddDate(0, 1, 0))
// 2020-06-01 00:00:00 +0000 UTC
fmt.Println(d.AddDate(0, 2, 0))
// 2020-05-31 00:00:00 +0000 UTC
```

---

### Sección 4: Formateo y Parseo de Fecha y Hora

Go utiliza un esquema de formateo de fecha/hora interesante y un tanto controvertido. El formato de fecha/hora se expresa utilizando un punto de referencia específico en el tiempo, ajustado de modo que cada componente de la fecha/hora sea un número único:

- **1** es el mes: `"Jan"` `"January"` `"01"` `"1"`
- **2** es el día del mes: `"2"` `"_2"` `"02"`
- **3** es la hora del día en formato de 12 horas: `"3"` `"03"`
- **15** es la hora del día en formato de 24 horas: `"15"`
- **4** es el minuto: `"4"` `"04"`
- **5** es el segundo: `"5"` `"05"`
- **6** es el año: `"2006"` `"06"`
- **MST** es la zona horaria: `"-0700"` `"-07:00"` `"-07"` `"-070000"` `"-07:00:00"` `"MST"`
- **0** es el milisegundo rellenado con ceros: `"0"` `"000"`
- **9** es el milisegundo sin ceros a la izquierda: `"9"` `"999"`

#### Cómo hacerlo...

- Usa `time.Parse` con un formato adecuado para parsear fecha/hora. Cualquier parte de la fecha/hora que no se especifique en el formato se inicializará con su valor cero, que es enero para los meses, 1 para el año, 1 para el día del mes y 0 para todo lo demás. Si falta la información de la zona horaria, la fecha/hora parseada estará en UTC.
- Usa `time.ParseInLocation` para parsear fecha/hora en una ubicación determinada. La zona horaria se determinará en función del valor de la fecha y la ubicación.
- Usa el método `Format()` para formatear un valor de fecha/hora.

```go
func main() {
  t := time.Date(2024, 3, 8, 18, 2, 13, 500, time.UTC)
  fmt.Println("Date in yyyy/mm/dd format", t.Format("2006/01/02"))
  // Date in yyyy/mm/dd format 2024/03/08
  fmt.Println("Date in yyyy/m/d format", t.Format("2006/1/2"))
  // Date in yyyy/m/d format 2024/3/8
  fmt.Println("Date in yy/m/d format", t.Format("06/1/2"))
  // Date in yy/m/d format 24/3/8
  fmt.Println("Time in hh:mm format (12 hr)", t.Format("03:04"))
  // Time in hh:mm format (12 hr) 06:02
  fmt.Println("Time in hh:m format (24 hr)", t.Format("15:4"))
  // Time in hh:m format (24 hr) 18:2
  fmt.Println("Date-time with time zone", t.Format("2006-01-02 13:04:05 -07:00"))
  // Date-time with time zone 2024-03-08 36:02:13 +00:00
}
```

Las zonas horarias cambian según la ubicación y la fecha. En el siguiente ejemplo, aunque se utiliza la misma ubicación para analizar la fecha, la zona horaria cambia porque el 9 de julio está en Horario de Verano de la Montaña (*MDT*), pero el 9 de enero está en Horario Estándar de la Montaña (*MST*):

```go
loc, _ := time.LoadLocation("America/Denver")
const format = "Jan 2, 2006 at 3:04pm"
str, _ := time.ParseInLocation(format, "Jul 9, 2012 at 5:02am", loc)
fmt.Println(str)
// 2012-07-09 05:02:00 -0600 MDT
str, _ = time.ParseInLocation(format, "Jan 9, 2012 at 5:02am", loc)
fmt.Println(str)
// 2012-01-09 05:02:00 -0700 MST
```

---

### Sección 5: Trabajo con Zonas Horarias

El valor `time.Time` de Go incluye `time.Location`, que puede ser una de dos cosas:
- Una ubicación real, como `America/Denver`. En este caso, la zona horaria real depende del valor de tiempo. Para Denver, la zona horaria será MDT (*Mountain Daylight Time*) o MST (*Mountain Standard Time*) según el valor de tiempo real.
- Una zona horaria fija que proporciona el desfase (*offset*).

Algunas aplicaciones trabajan con la hora local. Este es el valor de fecha/hora capturado en una ubicación particular e interpretado como el mismo valor en todas partes, en lugar de interpretarse como el mismo punto en el tiempo. Los cumpleaños (y por ende, las edades) se interpretan habitualmente utilizando la hora local. Es decir, si naces el `2005-07-14`, se considerará que tienes 2 años en Nueva York (zona horaria del Este) el `2007-07-14` a las `00:00`, pero seguirás teniendo 1 año en ese mismo instante de tiempo en Los Ángeles, que son las `21:00` del `2007-07-13` (zona horaria del Pacífico).

#### Cómo hacerlo...

- Si estás trabajando con instantes en el tiempo, captura siempre los valores de fecha/hora con la ubicación asociada. Dichos valores de fecha/hora se pueden traducir fácilmente a otras zonas horarias.
- Si estás trabajando con la hora local en múltiples zonas horarias, recrea `time.Time` en una nueva ubicación o zona horaria para traducirla.

#### Cómo funciona...

Cuando creas un `time.Time`, siempre está asociado con una ubicación:

```go
// Create a new time using the local time zone
t := time.Date(2021,12,31,15,0,0,0, time.Local)
// 2021-12-31 15:00:00 -0700 MST
```

Una vez que tienes un `time.Time`, puedes obtener el mismo instante en diferentes zonas horarias:

```go
utcTime := t.In(time.UTC)
fmt.Println(utcTime)
// 2021-12-31 22:00:00 +0000 UTC
ny,err:=time.LoadLocation("America/New_York")
if err!=nil {
  panic(err)
}
nyTime := t.In(ny)
fmt.Println(nyTime)
// 2021-12-31 17:00:00 -0500 EST
```

Estas son representaciones diferentes del mismo instante en diferentes zonas horarias.

También puedes crear una zona horaria personalizada:

```go
zone30 := time.FixedZone("30min", 30)
fmt.Println(t.In(zone30))
// 2021-12-31 22:00:30 +0000 30min
```

Cuando estás manejando la hora local, descartas la información de ubicación y zona horaria:

```go
// Create a local time, UTC zone
t := time.Date(2021,12,31,15,0,0,0, time.UTC)
// 2021-12-31 15:00:00 +0000 UTC
```

Para obtener el mismo valor de hora en Nueva York, utiliza lo siguiente:

```go
ny,err:=time.LoadLocation("America/New_York")
if err!=nil {
  panic(err)
}
nyTime := time.Date(t.Year(), t.Month(), t.Day(), t.Hour(), t.Minute(), t.Second(), t.Nanosecond(), ny)
fmt.Println(nyTime)
// 2021-12-31 15:00:00 -0500 EST
```

---

### Sección 6: Almacenamiento de Información de Tiempo

Un problema común es almacenar información de fecha/hora en bases de datos, archivos, etc., de manera portable para que se pueda interpretar correctamente.

#### Cómo hacerlo...

Primero debes identificar las necesidades exactas: ¿necesitas almacenar un instante de tiempo o una hora del día?

- Para almacenar un instante de tiempo, haz una de las siguientes opciones:
  - Almacena el tiempo Unix en la granularidad requerida (es decir, `time.Unix` para segundos, `time.UnixMilli` para milisegundos, etc.).
  - Almacena el tiempo UTC (`time.UTC()`).
- Para almacenar la hora del día, almacena el valor `time.Duration` que representa el instante dentro del día. La siguiente función calcula el instante dentro de ese día como `time.Duration`:

```go
func GetTimeOfDay(t time.Time) time.Duration {
  beginningOfDay:=time.Date(t.Year(),t.Month(),t.
  Day(),0,0,0,0,t.Location())
  return t.Sub(beginningOfDay)
}
```

- Para almacenar un valor de fecha, puedes limpiar las porciones horarias de `time.Time`:

```go
date:=time.Date(t.Year(), t.Month(), t.Day(), 0,0,0,0,t.Location())
```

Ten en cuenta que comparar fechas almacenadas de esta manera puede ser problemático, ya que cada día se interpretará como un instante diferente en diferentes zonas horarias.

---

### Sección 7: Temporizadores (*Timers*)

Usa `time.Timer` para programar una tarea que se ejecutará en el futuro. Cuando el temporizador expire, recibirás una señal a través de un canal. Puedes usar un temporizador para ejecutar una función más tarde o para cancelar un proceso que tomó demasiado tiempo.

#### Cómo hacerlo...

Puedes crear un temporizador de una de dos maneras:
- Usa `time.NewTimer` o `time.After`. El temporizador enviará una señal a través de un canal cuando expire. Usa una sentencia `select` o lee del canal para recibir la señal de expiración del temporizador.
- Usa `time.AfterFunc` para llamar a una función cuando el temporizador expire.

#### Cómo funciona...

Un temporizador `time.Timer` se crea con `time.Duration`:

```go
// Create a 10-second timer
timer := time.NewTimer(time.Second*10)
```

El temporizador contiene un canal que recibirá la marca de tiempo actual después de que transcurran 10 segundos. Un temporizador se crea con una capacidad de canal de 1, por lo que el entorno de ejecución del temporizador siempre podrá escribir en ese canal y detener el temporizador. En otras palabras, si no lees del temporizador, no provocará fugas de memoria; eventualmente será recolectado por el recolector de basura (*garbage collector*).

El temporizador se puede utilizar para detener un proceso de larga duración:

```go
func longProcess() {
  timer := time.NewTimer(time.Second*10)
  for {
     processData()
     select {
       case <-timer.C:
          // 2 seconds passed
          return
       default:
     }
  }
}
```

El siguiente ejemplo muestra cómo se puede utilizar un temporizador para limitar el tiempo que tarda una función en retornar. Si el cálculo se completa en un segundo, se devuelve la respuesta. Si el cálculo tarda más, la función devuelve un canal que el llamador puede usar para recibir el resultado. Esta función también demuestra cómo puedes detener un temporizador:

```go
func longComputation() (concurrent chan Result, result Result) {
  timer:=time.NewTimer(time.Second)
  concurrent=make(chan Result)
  // Start the concurrent computation. Its result will be received 
  // from the channel
  go func() {
     concurrent <- processData()
  }()
  // Wait until result is available, or timer expires
  select {
     case result:=<-concurrent:
        // Result became available quickly. Stop the timer and return 
        //the result.
        timer.Stop()
        return nil,result
     case <-timer.C:
        // Timer expired before result is computed. Return the channel
        return concurrent,Result{}
  }
}
```

Ten en cuenta que el temporizador puede expirar justo antes de la llamada a `timer.Stop()`. Esto está bien. Los temporizadores eventualmente expirarán y serán recolectados por el recolector de basura. Llamar a `timer.Stop()` simplemente evita que el temporizador permanezca activo más tiempo del necesario.

> **Consejo**  
> No puedes llamar a `Timer.Stop` de forma concurrente mientras otra *goroutine* está escuchando del temporizador. Por lo tanto, si tienes que llamar a `Timer.Stop`, hazlo desde la misma *goroutine* que escucha el canal del temporizador.

Lo mismo se puede lograr con `time.After`:

```go
  concurrent=make(chan Result)
  // Start the concurrent computation. Its result will be received 
  // from the channel
  go func() {
     concurrent <- processData()
  }()
  select {
     case result:=<-concurrent:
        return nil,result
     case <-time.After(time.Second):
        return concurrent,Result{}
  }
```

---

### Sección 8: *Tickers*

Usa `time.Ticker` para realizar una tarea periódicamente. Recibirás periódicamente una señal a través de un canal. A diferencia de `time.Timer`, debes tener cuidado con cómo desechas los *tickers*. Si olvidas detener un *ticker*, no será recolectado por el recolector de basura una vez que salga del ámbito y provocará una fuga de memoria (*leak*).

#### Cómo hacerlo...

1. Usa `time.Ticker` para crear un nuevo *ticker*.
2. Lee del canal del *ticker* para recibir los tics periódicos.
3. Cuando hayas terminado con el *ticker*, deténlo con `Stop()`. No es necesario vaciar el canal del *ticker*.

#### Cómo funciona...

Usa un *ticker* para eventos periódicos. Un patrón común es el siguiente:

```go
func poorMansClock(done chan struct{}) {
  // Create a new ticker with a 1 second period
  ticker:=time.NewTicker(time.Second)
  // Stop the ticker once we're done
  defer ticker.Stop()
  for {
    select {
      case <-done:
         return
      case <-ticker.C:
         fmt.Println(time.Now())
    }
  }
}
```

¿Qué sucede si pierdes tics? Esto es posible si ejecutas un proceso largo que te impida escuchar el canal del *ticker*. ¿Enviará el *ticker* una avalancha de tics cuando comiences a escuchar nuevamente?

De manera similar a `time.Timer`, `time.Ticker` utiliza un canal con una capacidad de 1. Debido a esto, si no lees del canal, puede almacenar, como máximo, un tic. Cuando comiences a escuchar desde el canal nuevamente, recibirás el tic que perdiste de inmediato y el siguiente tic cuando expire su período. Por ejemplo, considera el siguiente programa que llama a una función dada cada segundo:

```go
func everySecond(f func(), done chan struct{}) {
  // Create a new ticker with a 1 second period
  ticker:=time.NewTicker(time.Second)
  start:=time.Now()
  // Stop the ticker once we're done
  defer ticker.Stop()
  for {
    select {
      case <-done:
         return
      case <-ticker.C:
         fmt.Println(time.Since(start).Milliseconds())
         // Call the function
         f()
    }
  }
}
```

Supongamos que la primera llamada a `f()` se ejecuta durante 10 milisegundos, pero la segunda llamada se ejecuta durante 1.5 segundos. Mientras `f()` se está ejecutando, no hay nadie leyendo del canal del *ticker*, por lo que se perderá un tic. Una vez que `f()` retorna, la sentencia `select` leerá inmediatamente este tic perdido y, después de 500 milisegundos, recibirá el siguiente tic. La salida se ve así:

```text
1000
2000
3500
4000
5000
...
```

> **Consejo**  
> A diferencia de `time.Timer`, puedes detener un *ticker* de manera concurrente mientras lees de su canal.

