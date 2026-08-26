# Parte 3: Persistencia, Diagnóstico y Calidad de Código

## Capítulo 17: Pruebas, Rendimiento y Perfilado (*Testing, Benchmarking, and Profiling*)

Tener pruebas (*tests*) y evaluaciones comparativas (*benchmarks*) para tu código te ayudará de varias maneras. Durante el desarrollo, las pruebas garantizan que lo que estás desarrollando funciona y que no rompes la funcionalidad existente. Los *benchmarks* aseguran que tu programa se mantenga dentro de ciertas restricciones de recursos y tiempo. Una vez completado el desarrollo, las mismas pruebas y *benchmarks* garantizarán que cualquier trabajo de mantenimiento (corrección de errores, mejoras de funciones, etc.) no introduzca errores ni regresiones. Por lo tanto, debes considerar la escritura de pruebas y *benchmarks* como una actividad central de desarrollo, y desarrollar tanto tu programa como sus pruebas de forma conjunta.

Las pruebas deben centrarse en verificar el comportamiento esperado cuando todo funciona ("camino feliz" o *happy path*), así como en el manejo adecuado de fallos.

Este capítulo contiene las siguientes recetas:
- Trabajo con pruebas unitarias
  - Escritura de una prueba unitaria
  - Ejecución de pruebas unitarias
  - Registro de *logs* en pruebas
  - Omitir pruebas (*skipping tests*)
  - Pruebas de servidores HTTP
  - Pruebas de *handlers* HTTP
  - Comprobación de la cobertura de pruebas (*test coverage*)
- *Benchmarking* (Pruebas de rendimiento)
  - Escritura de *benchmarks*
  - Escritura de múltiples *benchmarks* con diferentes tamaños de entrada
  - Ejecución de *benchmarks*
- *Profiling* (Perfilado de rendimiento)
  - Perfilado de CPU y memoria
  - Análisis de perfiles con `go tool pprof`

---

### Sección 1: Trabajo con Pruebas Unitarias

Las pruebas en Go se escriben en archivos que terminan en `_test.go` dentro del mismo paquete (o un paquete de pruebas separado) y utilizan el paquete `testing`.

#### Escritura de una Prueba Unitaria Simple

##### Cómo hacerlo...

```go
func TestAdd(t *testing.T) {
   got := Add(2, 3)
   want := 5
   if got != want {
      t.Errorf("Add(2, 3) = %d; want %d", got, want)
   }
}
```

#### Pruebas Basadas en Tablas (*Table-Driven Tests*)

Las pruebas basadas en tablas son el patrón idiomático en Go para probar múltiples entradas y casos extremos de forma concisa:

```go
func TestAddTableDriven(t *testing.T) {
   tests := []struct {
      name string
      a, b int
      want int
   }{
      {"positive numbers", 2, 3, 5},
      {"negative numbers", -2, -3, -5},
      {"zero", 0, 5, 5},
   }

   for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
         got := Add(tt.a, tt.b)
         if got != tt.want {
            t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, got, tt.want)
         }
      })
   }
}
```

#### Ejecución de Pruebas Unitarias

##### Cómo hacerlo...

```bash
go test ./...
go test -v ./...
go test -run TestAdd ./...
```

#### Registro de *Logs* en Pruebas y Omisión (*Skipping*)

Usa `t.Log` o `t.Logf` para registrar información que solo se mostrará si la prueba falla o si se pasa el flag `-v`. Usa `t.Skip` junto con `testing.Short()` para omitir pruebas lentas durante ejecuciones rápidas:

```go
func TestExpensive(t *testing.T) {
   if testing.Short() {
      t.Skip("skipping test in short mode.")
   }
   t.Logf("Running expensive calculation...")
}
```

Ejecución en modo corto:

```bash
go test -short ./...
```

#### Pruebas de Servidores y *Handlers* HTTP

El paquete `net/http/httptest` proporciona utilidades para probar *handlers* sin necesidad de abrir puertos de red reales, así como servidores de prueba completos.

##### Pruebas de *Handlers* con `httptest.ResponseRecorder`

```go
func TestHealthHandler(t *testing.T) {
   req := httptest.NewRequest(http.MethodGet, "/health", nil)
   w := httptest.NewRecorder()

   HealthHandler(w, req)

   res := w.Result()
   defer res.Body.Close()

   if res.StatusCode != http.StatusOK {
      t.Errorf("expected status OK; got %v", res.Status)
   }
   body, _ := io.ReadAll(res.Body)
   if string(body) != "OK" {
      t.Errorf("expected body 'OK'; got %s", string(body))
   }
}
```

##### Pruebas de Integración con `httptest.Server`

```go
func TestServerIntegration(t *testing.T) {
   ts := httptest.NewServer(http.HandlerFunc(HealthHandler))
   defer ts.Close()

   res, err := http.Get(ts.URL)
   if err != nil {
      t.Fatal(err)
   }
   defer res.Body.Close()

   if res.StatusCode != http.StatusOK {
      t.Errorf("expected status 200; got %d", res.StatusCode)
   }
}
```

#### Comprobación de la Cobertura de Pruebas (*Test Coverage*)

##### Cómo hacerlo...

```bash
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

---

### Sección 2: *Benchmarking* (Pruebas de Rendimiento)

Las funciones de *benchmark* tienen el prefijo `Benchmark` y reciben un puntero `*testing.B`. El bucle debe ejecutarse `b.N` veces.

#### Escritura de *Benchmarks*

##### Cómo hacerlo...

```go
func BenchmarkAdd(b *testing.B) {
   for i := 0; i < b.N; i++ {
      Add(2, 3)
   }
}
```

#### *Benchmarks* con Diferentes Tamaños de Entrada

##### Cómo hacerlo...

```go
func BenchmarkProcessData(b *testing.B) {
   sizes := []int{10, 100, 1000}
   for _, size := range sizes {
      data := make([]byte, size)
      b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
         b.ResetTimer()
         b.ReportAllocs()
         for i := 0; i < b.N; i++ {
            ProcessData(data)
         }
      })
   }
}
```

#### Ejecución de *Benchmarks*

##### Cómo hacerlo...

```bash
go test -bench=. ./...
go test -bench=BenchmarkAdd -benchmem ./...
```

---

### Sección 3: *Profiling* (Perfilado de Rendimiento)

Go cuenta con soporte integrado de bajo coste para el perfilado de CPU, asignaciones de memoria y contención de bloqueos (*mutex/blocking*).

#### Generación de Perfiles de CPU y Memoria

##### Cómo hacerlo...

```bash
go test -cpuprofile=cpu.prof -memprofile=mem.prof -bench=.
```

#### Análisis de Perfiles con `go tool pprof`

##### Cómo hacerlo...

```bash
go tool pprof cpu.prof
```

Comandos interactivos comunes dentro de `pprof`:
- `top`: Muestra las funciones que más CPU o memoria consumen.
- `list <NombreDeFuncion>`: Muestra el código fuente anotado línea por línea.
- `web`: Genera y abre un gráfico visual de llamadas en el navegador web (requiere Graphviz).
