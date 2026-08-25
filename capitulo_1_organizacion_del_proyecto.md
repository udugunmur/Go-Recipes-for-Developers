# Parte 1: Fundamentos y Estructura del Proyecto

## Capítulo 1: Organización del Proyecto

Este capítulo trata sobre cómo iniciar un nuevo proyecto, organizar un árbol de fuentes y gestionar los paquetes necesarios para desarrollar tus programas. Una estructura de proyecto bien diseñada es fundamental, ya que cuando otros desarrolladores trabajen en tu proyecto o intenten utilizar sus componentes, podrán encontrar rápida y fácilmente lo que buscan. 

Este capítulo responderá primero a algunas de las preguntas que puedas tener al iniciar un nuevo proyecto. A continuación, veremos cómo utilizar el sistema de paquetes de Go, trabajar con la biblioteca estándar y paquetes de terceros, y facilitar que otros desarrolladores utilicen tus paquetes.

Este capítulo incluye las siguientes recetas:
- Creación de un módulo
- Creación de un árbol de fuentes
- Compilación y ejecución de programas
- Importación de paquetes de terceros
- Importación de versiones específicas de paquetes
- Uso de paquetes internos para reducir la superficie de la API
- Uso de una copia local de un módulo
- Espacios de trabajo (*Workspaces*)
- Gestión de las versiones de tu módulo

---

### Sección 1: Módulos y Paquetes

En primer lugar, unas breves palabras sobre módulos y paquetes resultarán de gran utilidad. Un **paquete** (*package*) es una unidad cohesiva de tipos de datos, constantes, variables y funciones. Compilas y pruebas paquetes, no archivos individuales ni módulos. Cuando compilas un paquete, el sistema de compilación recopila y también compila todos los paquetes dependientes. Si el nombre del paquete es `main`, compilarlo generará un ejecutable. Puedes ejecutar el paquete `main` sin generar un binario permanente (más específicamente, el sistema de compilación de Go primero compila el paquete, produce el binario en una ubicación temporal y lo ejecuta). Para utilizar otro paquete, lo importas. 

Los **módulos** ayudan a organizar múltiples paquetes y a la resolución de referencias de paquetes dentro de un proyecto. Un módulo es simplemente una colección de paquetes. Si importas un paquete en tu programa, el módulo que contiene ese paquete se agregará al archivo `go.mod`, y una suma de verificación (*checksum*) del contenido de ese módulo se agregará al archivo `go.sum`. Los módulos también te ayudan a gestionar las versiones de tus programas.

Todos los archivos de un paquete se almacenan en un único directorio en el sistema de archivos. Cada paquete tiene un nombre declarado mediante la directiva `package`, compartido por todos los archivos fuente que contiene. El nombre del paquete generalmente coincide con el nombre del directorio que contiene los archivos, pero esto no es estrictamente obligatorio. Por ejemplo, el paquete `main` generalmente no se encuentra en un directorio llamado `main/`. El directorio del paquete determina la "ruta de importación" (*import path*) del paquete. Importas otro paquete en tu paquete actual mediante la sentencia `import <importPath>`. Una vez importado un paquete, utilizas los nombres declarados en ese paquete a través de su nombre de paquete (que no es necesariamente el nombre del directorio).

El nombre de un módulo apunta a la ubicación donde se almacena el contenido del módulo en un sistema de control de versiones en Internet. Al momento de escribir esto, no es un requisito estricto, por lo que puedes crear nombres de módulos que no sigan esta convención; sin embargo, esto debe evitarse para prevenir posibles incompatibilidades futuras con el sistema de compilación. Los nombres de tus módulos deben formar parte de las rutas de importación para los paquetes de dichos módulos. En particular, los nombres de módulos cuyo primer componente (la parte antes de la primera `/`) no contenga un punto (`.`) están reservados para la biblioteca estándar.

Estos conceptos se ilustran en la Figura 1.1:

#### Figura 1.1 – Módulos y paquetes
- El nombre del módulo declarado en `go.mod` es la ruta del repositorio donde se puede encontrar el módulo.
- La ruta de importación en `main.go` define dónde se puede encontrar el paquete importado. El sistema de compilación de Go localizará el paquete utilizando esta ruta de importación y luego ubicará el módulo que contiene el paquete escaneando los directorios padre de la ruta del paquete. Una vez encontrado el módulo, se descargará en la caché de módulos.
- El nombre del paquete definido en el módulo importado es el nombre de paquete que utilizas para acceder a los símbolos de ese paquete. Puede ser diferente del último componente de la ruta de importación. En nuestro ejemplo, el nombre del paquete es `example`, pero la ruta de importación para este paquete es `github.com/bserdar/go-recipes-module`.
- La función `Example` se encuentra en el paquete `example`.
- El paquete `example` también importa otro paquete contenido en el mismo módulo. El sistema de compilación identificará este paquete como parte del mismo módulo y resolverá las referencias utilizando la versión descargada del módulo.

---

### Sección 2: Requisitos Técnicos

Necesitarás una versión reciente de Go en tu computadora para compilar y ejecutar los ejemplos de este capítulo. Los ejemplos de este libro fueron probados utilizando **Go versión 1.22**. 

El código de este capítulo se puede encontrar en:
[https://github.com/PacktPublishing/Go-Recipes-for-Developers/tree/main/src/chp1](https://github.com/PacktPublishing/Go-Recipes-for-Developers/tree/main/src/chp1)

---

### Sección 3: Creación de un Módulo

Cuando comienzas a trabajar en un nuevo proyecto, lo primero que debes hacer es crear un módulo para él. Un módulo es la forma en que Go gestiona las dependencias.

#### Cómo hacerlo...

1. Crea un directorio para almacenar un nuevo módulo.
2. Dentro de ese directorio, usa `go mod init <moduleName>` para crear el nuevo módulo. El archivo `go.mod` marca el directorio raíz de un módulo. Cualquier paquete bajo este directorio formará parte de este módulo a menos que ese directorio también contenga un archivo `go.mod`. Aunque tales módulos anidados son compatibles con el sistema de compilación, no se obtiene gran beneficio de ellos.
3. Para importar un paquete en el mismo módulo, usa `moduleName/packagePath`. Cuando `moduleName` coincide con la ubicación del módulo en Internet, no hay ambigüedades sobre a qué te estás refiriendo.
4. Para los paquetes bajo un módulo, la raíz del módulo es el directorio padre más cercano que contiene un archivo `go.mod`. Todas las referencias a otros paquetes dentro de un módulo se buscarán en el árbol de directorios bajo la raíz del módulo.

Comienza creando un directorio para almacenar los archivos del proyecto. Tu directorio actual puede ser cualquier lugar del sistema de archivos. Muchas personas utilizan un directorio común para almacenar su trabajo, como `$HOME/projects` (o `\user\myUser\projects` en Windows). Puedes optar por utilizar una estructura de directorios similar al nombre del módulo, como `$HOME/github.com/mycompany/mymodule` (o `\user\myUser\github.com\mycompany\mymodule` en Windows). Según tu sistema operativo, puedes encontrar una ubicación más conveniente.

> **Advertencia**  
> No trabajes dentro del directorio `src/` de tu instalación de Go. Ese directorio contiene el código fuente de la biblioteca estándar de Go.

> **Consejo**  
> No deberías tener una variable de entorno `GOPATH`; si tienes que mantenerla, no trabajes dentro de ella. Esta variable se utilizaba en un modo de operación anterior (versiones de Go <1.13) que ahora está en desuso en favor del sistema de módulos de Go.

A lo largo de este capítulo, utilizaremos un programa simple que muestra un formulario en un navegador web y almacena la información ingresada en una base de datos.

Después de crear el directorio del módulo, utiliza `go mod init`. Los siguientes comandos crearán un directorio `webform` dentro de `projects` e inicializarán un módulo de Go allí:

```bash
$ cd projects
$ mkdir webform
$ go mod init github.com/examplecompany/webform
```

Esto creará un archivo `go.mod` en este directorio con el siguiente aspecto:

```text
module github.com/PacktPublishing/Go-Recipes-for-Developers/chapter1/webform

go 1.21.0
```

Utiliza un nombre que describa dónde se puede encontrar tu módulo. Utiliza siempre una estructura de URL como el formato `<host>.<domain>/location/to/module` (por ejemplo, `github.com/bserdar/jsonom`). En particular, el primer componente del nombre del módulo debe tener un punto (`.`) (el sistema de compilación de Go verifica esto).

Por lo tanto, aunque puedas nombrar el módulo algo como `webform` o `mywork/webform`, no lo hagas. Sin embargo, puedes usar algo como `workspace.local/webform`. En caso de duda, utiliza el nombre del repositorio de código.

---

### Sección 4: Creación de un Árbol de Fuentes

Una vez que tienes un nuevo módulo, es el momento de decidir cómo vas a organizar los archivos fuente.

#### Cómo hacerlo...

Existen varias convenciones establecidas, dependiendo del proyecto:
- Utilizar una estructura estándar, como [https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout).
- Una biblioteca con un propósito específico puede colocar todos los nombres exportados en la raíz del módulo, almacenando opcionalmente los detalles de implementación dentro de paquetes internos (`internal`). Un módulo que produce un único ejecutable con relativamente pocos o ningún componente reutilizable también puede utilizar una estructura de directorios plana.

Para un proyecto como el nuestro, que genera un ejecutable, la estructura establecida en [https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout) encaja a la perfección. Sigamos esa plantilla:

```text
webform/
  go.mod
  cmd/
    webform/
      main.go
  web/
    static/
  pkg/
    ...
  internal/
    ...
  build/
    ci/
    package/
  configs/
```

Aquí, el directorio `cmd/webform` contendrá el paquete `main`. Como puedes ver, este es un caso donde el nombre del paquete no coincide con el directorio en el que se encuentra. El sistema de compilación de Go creará ejecutables utilizando el nombre del directorio, por lo que cuando compilas el paquete `main` bajo `cmd/webform`, obtienes un ejecutable llamado `webform`. Si tienes múltiples ejecutables compilados dentro de un solo módulo, puedes organizarlos creando un paquete `main` separado bajo un directorio que coincida con el nombre del programa, dentro del directorio `cmd/`.

El directorio `pkg/` contendrá los paquetes exportados del programa. Estos son paquetes que se pueden importar y reutilizar en otros proyectos.

Si tienes paquetes que no son utilizables fuera de este proyecto, debes colocarlos bajo el directorio `internal/`. El sistema de compilación de Go reconoce este directorio y no permite importar paquetes bajo `internal/` desde otros paquetes que se encuentren fuera del directorio que contiene dicho directorio `internal/`. Con esta configuración, todos los paquetes de nuestro programa `webform` tendrán acceso a los paquetes bajo `internal/`, pero serán inaccesibles para paquetes externos que importen este módulo.

El directorio `web/` contendrá cualquier recurso relacionado con la web. En este ejemplo, tendremos un directorio `web/static` con páginas web estáticas. También puedes agregar `web/templates` para almacenar plantillas del lado del servidor si las necesitas.

El directorio `build/package` debe contener scripts de empaquetado y configuración para la nube, contenedores y sistemas de empaquetado (`dep`, `rpm`, `pkg`, etc.).

El directorio `build/ci` debe contener scripts y configuraciones de herramientas de integración continua. Si la herramienta de integración continua que utilizas requiere que sus archivos estén en un directorio específico diferente a este, puedes crear enlaces simbólicos o simplemente colocar esos archivos donde la herramienta los requiera en lugar de `/build/ci`.

El directorio `configs/` debe contener plantillas de archivos de configuración y configuraciones predeterminadas.

También puedes encontrar proyectos que tienen el paquete `main` en la raíz del módulo, eliminando el directorio `cmd/`. Esta es una estructura común cuando el módulo contiene solo un ejecutable:

```text
webform/
  go.mod
  go.sum
  main.go
  internal/
    ...
  pkg/
    ...
```

Luego existen módulos sin ningún paquete `main`. Generalmente son bibliotecas que puedes importar en tus proyectos. Por ejemplo, [https://github.com/google/uuid](https://github.com/google/uuid) contiene la popular implementación de UUID utilizando una estructura de directorios plana.

---

### Sección 5: Compilación y Ejecución de Programas

Ahora que tienes un módulo y un árbol de fuentes con algunos archivos Go, puedes compilar o ejecutar tu programa.

#### Cómo hacerlo...

- Usa `go build` para compilar el paquete actual.
- Usa `go build ./path/to/package` para compilar el paquete en el directorio indicado.
- Usa `go build <moduleName>` para compilar un módulo.
- Usa `go run` para ejecutar el paquete `main` actual.
- Usa `go run ./path/to/main/package` para compilar y ejecutar el paquete `main` en el directorio indicado.
- Usa `go run <moduleName/mainpkg>` para compilar y ejecutar el `main` del módulo bajo el directorio indicado.

Escribamos la función `main` que inicia un servidor HTTP. El siguiente fragmento corresponde a `cmd/webform/main.go`:

```go
package main

import (
	"net/http"
)

func main() {
	server := http.Server{
		Addr:    ":8181",
		Handler: http.FileServer(http.Dir("web/static")),
	}
	server.ListenAndServe()
}
```

Actualmente, `main` solo importa el paquete `net/http` de la biblioteca estándar. Inicia un servidor que sirve los archivos bajo el directorio `web/static`. Ten en cuenta que para que esto funcione, debes ejecutar el programa desde la raíz del módulo:

```bash
$ go run ./cmd/webform
```

Ejecuta siempre el paquete `main`; evita hacer `go run main.go`. Esto último solo ejecutaría `main.go`, excluyendo cualquier otro archivo dentro del paquete `main`, y fallará si tienes otros archivos `.go` que contengan funciones auxiliares en el paquete `main`.

Si ejecutas este programa desde otro directorio, no podrá encontrar el directorio `web/static`; debido a que es una ruta relativa, se resuelve en relación con el directorio actual.

Cuando ejecutas un programa mediante `go run`, el ejecutable del programa se coloca en un directorio temporal. Para compilar el ejecutable de forma permanente, utiliza lo siguiente:

```bash
$ go build ./cmd/webform
```

Esto creará un binario en el directorio actual. El nombre del binario vendrá determinado por el último segmento del paquete principal; en este caso, `webform`. Para compilar un binario con un nombre diferente, utiliza lo siguiente:

```bash
$ go build -o wform ./cmd/webform
```

Esto compilará un binario llamado `wform`.

---

### Sección 6: Importación de Paquetes de Terceros

La mayoría de los proyectos dependerán de bibliotecas de terceros que deben ser importadas. El sistema de módulos de Go gestiona estas dependencias.

#### Cómo hacerlo...

1. Encuentra la ruta de importación del paquete que necesitas utilizar en tu proyecto.
2. Agrega los `import` necesarios a los archivos fuente donde utilices el paquete externo.
3. Utiliza el comando `go get` o `go mod tidy` para agregar el módulo a `go.mod` y `go.sum`. Si el módulo no se había descargado antes, este paso también descargará el módulo.

> **Consejo**  
> Puedes usar [https://pkg.go.dev](https://pkg.go.dev/) para descubrir paquetes. También es el lugar donde se publica la documentación de los proyectos Go que publiques.

Agreguemos una base de datos a nuestro programa de la sección anterior para poder almacenar los datos enviados mediante el formulario web. Para este ejercicio, utilizaremos la base de datos SQLite.

Modifica el archivo `cmd/webform/main.go` para importar el paquete de base de datos y agregar el código de inicialización necesario:

```go
package main

import (
	"database/sql"
	"net/http"

	_ "modernc.org/sqlite"

	"github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp1/webform/pkg/commentdb"
)

func main() {
	db, err := sql.Open("sqlite", "webform.db")
	if err != nil {
		panic(err)
	}
	commentdb.InitDB(db)
	server := http.Server{
		Addr:    ":8181",
		Handler: http.FileServer(http.Dir("web/static")),
	}
	server.ListenAndServe()
}
```

La línea `_ "modernc.org/sqlite"` importa el controlador de SQLite en el proyecto. El guion bajo es el identificador en blanco (*blank identifier*), lo que significa que el paquete `sqlite` no es utilizado directamente por este archivo y solo se incluye por sus efectos secundarios. Sin el identificador en blanco, el compilador se quejaría de que la importación no se está utilizando. En este caso, el paquete `modernc.org/sqlite` es un controlador de base de datos, y al importarlo, sus funciones `init()` registrarán el controlador requerido en la biblioteca estándar.

La siguiente declaración importa el paquete `commentdb` desde nuestro módulo. Observa que se utiliza el nombre completo del módulo para importar el paquete. El sistema de compilación reconocerá el prefijo de esta declaración de importación como el nombre del módulo actual y lo traducirá a una referencia local del sistema de archivos, que en este caso es `webform/pkg/commentdb`.

En la línea `db, err := sql.Open("sqlite", "webform.db")`, usamos la función `Open` del paquete `database/sql` para iniciar una instancia de base de datos SQLite. `sqlite` nombra el controlador de la base de datos que fue registrado por la importación `_ "modernc.org/sqlite"`.

La sentencia `commentdb.InitDB(db)` llamará a una función del paquete `commentdb`.

Ahora veamos cómo es `commentdb.InitDB`. Este es el archivo `webform/pkg/commentdb/initdb.go`:

```go
package commentdb

import (
	"context"
	"database/sql"
)

const createStmt = `create table if not exists comments (
email TEXT,
comment TEXT)`

func InitDB(conn *sql.DB) {
	_, err := conn.ExecContext(context.Background(), createStmt)
	if err != nil {
		panic(err)
	}
}
```

Como puedes ver, esta función crea las tablas de la base de datos si aún no han sido creadas.

Observa las mayúsculas en `InitDB`. Si la primera letra del nombre de un símbolo declarado en un paquete es mayúscula, ese símbolo es accesible desde otros paquetes (es decir, está exportado). Si no, el símbolo solo se puede utilizar dentro del paquete donde está declarado (es decir, no está exportado). La constante `createStmt` no está exportada y será invisible para otros paquetes.

Compilemos el programa:

```bash
$ go build ./cmd/webform
cmd/webform/main.go:7:2: no required module provides package modernc.org/sqlite; to add it:
	go get modernc.org/sqlite
```

Puedes ejecutar `go get modernc.org/sqlite` para agregar un módulo a tu proyecto. Alternativamente, puedes ejecutar:

```bash
$ go get
```

Eso obtendrá todos los módulos faltantes. O también puedes ejecutar:

```bash
$ go mod tidy
```

`go mod tidy` descargará todos los paquetes faltantes, actualizará `go.mod` y `go.sum` con las dependencias actualizadas y eliminará las referencias a cualquier módulo no utilizado. `go get` solo descargará los módulos faltantes.

---

### Sección 7: Importación de Versiones Específicas de Paquetes

A veces necesitas una versión específica de un paquete de terceros debido a incompatibilidades de API o a un comportamiento particular del que dependes.

#### Cómo hacerlo...

Para obtener una versión específica de un paquete, especifica la etiqueta de versión:

```bash
$ go get modernc.org/sqlite@v1.26.0
```

Para obtener la última versión de una versión mayor específica de un paquete, usa:

```bash
$ go get gopkg.in/yaml.v3
```

O también:

```bash
$ go get github.com/ory/dockertest/v3
```

Para importar la última versión disponible, usa:

```bash
$ go get modernc.org/sqlite
```

También puedes especificar una rama diferente. Lo siguiente obtendrá un módulo desde la rama `devel`, si existe:

```bash
$ go get modernc.org/sqlite@devel
```

Alternativamente, puedes obtener un commit específico:

```bash
$ go get modernc.org/sqlite@a8c3eea199bc8fdc39391d5d261eaa3577566050
```

Como puedes ver, puedes obtener una revisión específica de un módulo mediante la convención `@revision`:

```bash
$ go get modernc.org/sqlite@v1.26.0
```

La parte de la revisión de la URL es evaluada por el sistema de control de versiones, que en este caso es `git`, por lo que se puede utilizar cualquier sintaxis de revisión válida de Git.

> **Consejo**  
> Puedes comprobar qué sistemas de control de versiones son compatibles revisando el archivo `src/cmd/go/alldocs.go` bajo tu instalación de Go.

Eso también significa que puedes usar ramas:

```bash
$ go get modernc.org/sqlite@master
```

> **Consejo**  
> El servicio [https://gopkg.in](https://gopkg.in/) traduce números de versión a URLs compatibles con el sistema de compilación de Go. Consulta las instrucciones en ese sitio web para saber cómo utilizarlo.

---

### Sección 8: Trabajo con la Caché de Módulos

La caché de módulos es un directorio donde el sistema de compilación de Go almacena los archivos de módulos descargados. Esta sección describe cómo trabajar con la caché de módulos.

#### Cómo hacerlo...

La caché de módulos se encuentra, de forma predeterminada, en `$GOPATH/pkg/mod`, que corresponde a `$HOME/go/pkg/mod` cuando `GOPATH` no está configurada.

- De forma predeterminada, el sistema de compilación de Go crea archivos de solo lectura dentro de la caché de módulos para evitar modificaciones accidentales.
- Para verificar que la caché de módulos no haya sido modificada y refleje las versiones originales de los módulos, usa:

```bash
go mod verify
```

- Para limpiar la caché de módulos, usa:

```bash
go clean -modcache
```

La fuente autorizada de información sobre la caché de módulos es la Referencia de Módulos de Go ([https://go.dev/ref/mod](https://go.dev/ref/mod)).

---

### Sección 9: Uso de Paquetes Internos para Reducir la Superficie de la API

No todo fragmento de código es reutilizable. Tener una superficie de API más pequeña facilita que otros adapten y utilicen tu código. Por lo tanto, no debes exportar APIs que sean específicas de tu programa.

#### Cómo hacerlo...

Crea paquetes internos para ocultar los detalles de implementación a otros paquetes. Cualquier elemento bajo un paquete interno solo se puede importar desde los paquetes que se encuentren bajo el paquete que contiene dicho paquete interno; es decir, cualquier elemento bajo `myproject/internal` solo se puede importar desde los paquetes bajo `myproject`.

En nuestro ejemplo, colocamos el código de acceso a la base de datos en un paquete donde otros programas pueden acceder a él. Sin embargo, no tiene sentido exponer las rutas HTTP a terceros, ya que son específicas de este programa. Por lo tanto, las colocaremos bajo el paquete `webform/internal`.

Este es el archivo `internal/routes/routes.go`:

```go
package routes

import (
	"database/sql"
	"net/http"

	"github.com/gorilla/mux"
)

func Build(router *mux.Router, conn *sql.DB) {
	router.Path("/form").Methods("GET").HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.ServeFile(w, r, "web/static/form.html")
	})
	router.Path("/form").Methods("POST").HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		handlePost(conn, w, r)
	})
}

func handlePost(conn *sql.DB, w http.ResponseWriter, r *http.Request) {
	email := r.PostFormValue("email")
	comment := r.PostFormValue("comment")
	_, err := conn.ExecContext(r.Context(), "insert into comments (email,comment) values (?,?)", email, comment)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	http.Redirect(w, r, "/form", http.StatusFound)
}
```

Luego, modificamos el archivo `main.go` para utilizar el paquete interno:

```go
package main

import (
	"database/sql"
	"net/http"

	"github.com/gorilla/mux"
	_ "modernc.org/sqlite"

	"github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp1/webform/internal/routes"
	"github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp1/webform/pkg/commentdb"
)

func main() {
	db, err := sql.Open("sqlite", "webform.db")
	if err != nil {
		panic(err)
	}
	commentdb.InitDB(db)
	r := mux.NewRouter()
	routes.Build(r, db)
	server := http.Server{
		Addr:    ":8181",
		Handler: r,
	}
	server.ListenAndServe()
}
```

---

### Sección 10: Uso de una Copia Local de un Módulo

A veces trabajarás en múltiples módulos, o descargarás un módulo de un repositorio, realizarás algunos cambios en él y luego querrás utilizar la versión modificada en lugar de la versión disponible en el repositorio.

#### Cómo hacerlo...

Usa la directiva `replace` en `go.mod` para apuntar al directorio local que contiene un módulo.

Volvamos a nuestro ejemplo: supongamos que deseas realizar algunos cambios en el paquete `sqlite`:

1. Clónalo:

```bash
$ ls
webform
$ git clone git@gitlab.com:cznic/sqlite.git
$ ls
sqlite webform
```

2. Modifica el archivo `go.mod` de tu proyecto para que apunte a la copia local del módulo. `go.mod` quedará de la siguiente manera:

```text
module github.com/PacktPublishing/Go-Recipes-for-Developers/chapter1/webform

go 1.22.1

replace modernc.org/sqlite => ../sqlite

require (
	github.com/gorilla/mux v1.8.1
	modernc.org/sqlite v1.27.0
)
...
```

Ahora puedes hacer cambios en el módulo `sqlite` en tu sistema, y esos cambios se compilarán en tu aplicación.

---

### Sección 11: Trabajo con Múltiples Módulos – Espacios de Trabajo (*Workspaces*)

A veces necesitas trabajar con múltiples módulos interdependientes. Una forma conveniente de hacerlo es definiendo un espacio de trabajo (*workspace*). Un espacio de trabajo es simplemente un conjunto de módulos. Si uno de los módulos dentro de un espacio de trabajo hace referencia a un paquete de otro módulo en el mismo espacio de trabajo, se resuelve localmente en lugar de descargarse a través de la red.

#### Cómo hacerlo...

1. Para crear un espacio de trabajo, debes tener un directorio padre que contenga todos tus módulos de trabajo:

```bash
$ cd ~/projects
$ mkdir ws
$ cd ws
```

2. Luego, inicia un espacio de trabajo usando:

```bash
$ go work init
```

Esto creará un archivo `go.work` en este directorio.

3. Coloca los módulos en los que estás trabajando dentro de este directorio. Demostrémoslo con nuestro ejemplo. Supongamos que tenemos la siguiente estructura de directorios:

```text
$HOME/
  projects/
    ws/
      go.work
      webform
      sqlite
```

4. Ahora queremos agregar los dos módulos, `webform` y `sqlite`, al espacio de trabajo. Para hacerlo, usa:

```bash
$ go work use ./webform
$ go work use ./sqlite
```

Estos comandos agregarán los dos módulos a tu espacio de trabajo. Cualquier referencia a `sqlite` desde el módulo `webform` se resolverá ahora utilizando la copia local del módulo.

---

### Sección 12: Gestión de las Versiones de tu Módulo

Las herramientas de Go utilizan el sistema de versionado semántico (*Semantic Versioning*). Esto significa que los números de versión tienen la forma `X.Y.z`, desglosados de la siguiente manera:
- **X** se incrementa para versiones mayores (*major releases*) que no son necesariamente compatibles hacia atrás.
- **Y** se incrementa para versiones menores (*minor releases*) que son incrementales pero compatibles hacia atrás.
- **z** se incrementa para parches (*patches*) compatibles hacia atrás.

Puedes aprender más sobre el versionado semántico en [https://semver.org](https://semver.org/).

#### Cómo hacerlo...

Para publicar un parche o una versión menor, etiqueta la rama que contiene tus cambios con el nuevo número de versión:

```bash
$ git tag v1.0.0
$ git push origin v1.0.0
```

Si deseas publicar una nueva versión que tenga una API incompatible con las versiones anteriores, debes incrementar la versión mayor de ese módulo. Para lanzar una nueva versión mayor de tu módulo, utiliza una nueva rama:

```bash
$ git checkout -b v2
```

Luego, cambia el nombre de tu módulo en `go.mod` para que termine con `/v2`, y actualiza todas las referencias en el árbol de fuentes para que utilicen la versión `/v2` del módulo.

Por ejemplo, supongamos que lanzaste la primera versión del módulo `webform`, `v1.0.0`. Luego, decidiste que querías agregar nuevos endpoints de API. Esto no representaría un cambio que rompa la compatibilidad (*breaking change*), por lo que simplemente incrementas el número de versión menor a `v1.1.0`. Pero luego resulta que algunas de las APIs añadidas estaban causando problemas, así que las eliminaste. Ahora sí es un cambio incompatible, por lo que deberías publicar `v2.0.0`. ¿Cómo puedes hacer eso?

La respuesta es que utilizas una nueva rama en el sistema de control de versiones. Crea la rama `v2`:

```bash
$ git checkout -b v2
```

Luego, modifica `go.mod` para reflejar la nueva versión:

```text
module github.com/PacktPublishing/Go-Recipes-for-Developers/chapter1/webform/v2

go 1.22.1

require (
	...
)
```

Si hay múltiples paquetes en el módulo, debes actualizar el árbol de fuentes para que cualquier referencia a paquetes dentro de ese módulo también utilice la versión `v2`.

Haz commit y push de la nueva rama:

```bash
$ git add go.mod
$ git commit -m "New version"
$ git push origin v2
```

Para utilizar la nueva versión, ahora debes importar la versión `v2` de los paquetes:

```go
import "github.com/PacktPublishing/Go-Recipes-for-Developers/chapter1/webform/v2/pkg/commentdb"
```

---

### Sección 13: Resumen y Lecturas Adicionales

Este capítulo se centró en los conceptos y la mecánica de configuración y gestión de proyectos en Go. No pretende ser una referencia exhaustiva, pero las recetas presentadas aquí deberían proporcionarte los fundamentos necesarios para utilizar el sistema de compilación de Go de manera eficaz.

- La guía definitiva sobre los módulos de Go es la Referencia de Módulos de Go ([https://go.dev/ref/mod](https://go.dev/ref/mod)).
- Consulta el enlace sobre Gestión de Dependencias ([https://go.dev/doc/modules/managing-dependencies](https://go.dev/doc/modules/managing-dependencies)) para obtener una explicación detallada sobre la gestión de dependencias.

En el próximo capítulo, comenzaremos a trabajar con datos de texto.
