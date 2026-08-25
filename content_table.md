# Tabla de Contenidos

## Capítulo 1: Organización del Proyecto

- Módulos y paquetes
- Requisitos técnicos
- Creación de un módulo
- Creación de un árbol de fuentes
- Compilación y ejecución de programas
- Importación de paquetes de terceros
- Importación de versiones específicas de paquetes
- Trabajo con la caché de módulos
- Uso de paquetes internos para reducir la superficie de la API
- Uso de una copia local de un módulo
- Trabajo con múltiples módulos: espacios de trabajo (*workspaces*)
- Gestión de las versiones de tu módulo
- Resumen y lecturas adicionales

---

## Capítulo 2: Trabajo con Cadenas de Texto (*Strings*)

- Creación de *strings*
- Formateo de *strings*
- Combinación de *strings*
- Trabajo con mayúsculas y minúsculas
- Trabajo con codificaciones (*encodings*)
- Iteración de bytes y *runes* en *strings*
- División (*splitting*)
- Lectura de *strings* línea por línea o palabra por palabra
- Recorte de los extremos de un *string* (*trimming*)
- Expresiones regulares
- Extracción de datos de *strings*
- Reemplazo de partes de un *string*
- Plantillas (*templates*)
- Manejo de líneas vacías
- Composición de plantillas
- Composición de plantillas: plantillas de diseño (*layout*)
- Y mucho más...

---

## Capítulo 3: Trabajo con Fecha y Hora

- Trabajo con tiempo Unix (*Unix time*)
- Componentes de fecha y hora
- Aritmética de fecha y hora
- Formateo y parseo de fecha y hora
- Trabajo con zonas horarias
- Almacenamiento de información de tiempo
- Temporizadores (*timers*)
- *Tickers*

---

## Capítulo 4: Trabajo con *Arrays*, *Slices* y *Maps*

- Trabajo con *arrays*
- Trabajo con *slices*
- Creación de un *slice* a partir de un *array*
- Adición, inserción y eliminación de elementos en *slices*
- Implementación de una pila (*stack*) usando un *slice*
- Trabajo con *maps*
- Implementación de un conjunto (*set*) usando un *map*
- Claves compuestas
- Caché segura para hilos (*thread-safe*) con *maps*
- Caché con comportamiento bloqueante

---

## Capítulo 5: Trabajo con Tipos, *Structs* e Interfaces

- Creación de nuevos tipos
- Creación de un nuevo tipo basado en un tipo existente
- Creación de enumeraciones con seguridad de tipos (*type-safe*)
- Creación de tipos *struct*
- Extensión de tipos
- Inicialización de *structs*
- Definición de interfaces
- Fábricas (*factories*)
- Definición de interfaces donde se utilizan
- Uso de una función como interfaz
- Descubrimiento de capacidades de tipos de datos en tiempo de ejecución: comprobación de la relación "implementa" (*implements*)
- Comprobación de si un valor de interfaz es uno de los tipos conocidos
- Garantizar que un tipo implementa una interfaz durante el desarrollo
- Decidir si usar un receptor de puntero (*pointer receiver*) o de valor (*value receiver*) para métodos
- Contenedores polimórficos
- Acceso a partes de un objeto no expuestas directamente a través de la interfaz
- Acceso al *struct* contenedor desde el *struct* embebido
- Comprobación de si una interfaz es `nil`

---

## Capítulo 6: Trabajo con Genéricos

- Funciones genéricas
- Tipos genéricos

---

## Capítulo 7: Concurrencia

- Realización de tareas concurrentes mediante *goroutines*
- Comunicación entre *goroutines* mediante canales (*channels*)
- Trabajo con múltiples canales usando la sentencia `select`
- Memoria compartida

---

## Capítulo 8: Errores y Pánicos (*Panics*)

- Retorno y manejo de errores
- Envoltura de errores (*wrapping*) para añadir información contextual
- Comparación de errores
- Errores estructurados
- Envoltura de errores estructurados
- Comparación de errores estructurados por tipo
- Extracción de un error específico del árbol de errores
- Manejo de pánicos (*panics*)
- Provocar un pánico cuando sea necesario
- Recuperación de pánicos (*recover*)
- Modificación del valor de retorno en *recover*
- Captura de la traza de la pila (*stack trace*) de un pánico

---

## Capítulo 9: El Paquete *Context*

- Uso de *context* para pasar datos de ámbito de petición (*request-scoped*)
- Uso de *context* para cancelaciones
- Uso de *context* para tiempos de espera (*timeouts*)
- Uso de cancelaciones y *timeouts* en servidores

---

## Capítulo 10: Trabajo con Grandes Volúmenes de Datos

- *Pools* de *workers*
- *Pipelines*
- Trabajo con conjuntos de resultados grandes

---

## Capítulo 11: Trabajo con JSON

- Conceptos básicos de *marshaling* y *unmarshaling*
- Codificación de *structs*
- Manejo de *structs* embebidos
- Codificación sin definir *structs*
- Decodificación de *structs*
- Decodificación con interfaces, *maps* y *slices*
- Otras formas de decodificar números
- Manejo de valores faltantes y opcionales
- Omisión de campos vacíos al codificar
- Manejo de campos faltantes al decodificar
- *Marshaling* y *unmarshaling* de tipos de datos personalizados
- *Marshaling* y *unmarshaling* personalizado de claves de objetos
- *Unmarshaling* personalizado en dos pasos
- Procesamiento en *streaming* de un *array* de objetos
- Parseo de un *array* de objetos
- Otras formas de realizar *streaming* de JSON

---

## Capítulo 12: Procesos

- Ejecución de programas externos
- Paso de argumentos a un proceso
- Procesamiento de la salida de un proceso hijo mediante una tubería (*pipe*)
- Provisión de entrada a un proceso hijo
- Modificación de variables de entorno de un proceso hijo
- Terminación limpia (*graceful*) mediante señales

---

## Capítulo 13: Programación de Red

- Redes TCP
- Creación de servidores TCP
- Creación de clientes TCP
- Creación de un servidor TCP basado en líneas
- Envío y recepción de archivos mediante una conexión TCP
- Creación de un cliente/servidor TLS
- Un *proxy* TCP para terminación TLS y balanceo de carga
- Configuración de límites de tiempo (*deadlines*) de lectura/escritura
- Desbloqueo de una operación de lectura o escritura bloqueada
- Creación de clientes/servidores UDP
- Trabajo con HTTP
- Realización de peticiones HTTP
- Ejecución de un servidor HTTP
- HTTPS: configuración de un servidor TLS
- Creación de *handlers* HTTP
- Servir archivos estáticos en el sistema de archivos
- Manejo de formularios HTML
- Creación de un *handler* para descarga de archivos grandes
- Manejo de subidas de archivos y formularios HTTP como *stream*

---

## Capítulo 14: Entrada/Salida en *Streaming* (I/O)

- *Readers* y *writers*
- Trabajo con archivos
- Trabajo con datos binarios
- Copia de datos
- Trabajo con el sistema de archivos
- Trabajo con tuberías (*pipes*)

---

## Capítulo 15: Bases de Datos

- Conexión a una base de datos
- Ejecución de sentencias SQL
- Ejecución de sentencias preparadas (*prepared statements*) dentro de una transacción
- Obtención de valores a partir de una consulta
- Construcción dinámica de sentencias SQL
- Construcción de sentencias UPDATE
- Construcción de cláusulas WHERE

---

## Capítulo 16: Registro de Eventos (*Logging*)

- Uso del *logger* estándar
- Uso del *logger* estructurado

---

## Capítulo 17: Pruebas, *Benchmarking* y *Profiling*

- Trabajo con pruebas unitarias
- Escritura de una prueba unitaria
- Ejecución de pruebas unitarias
- Registro de *logs* en pruebas
- Omitir pruebas (*skipping tests*)
- Pruebas de servidores HTTP
- Pruebas de *handlers* HTTP
- Comprobación de la cobertura de pruebas (*test coverage*)
- *Benchmarking*
- Escritura de *benchmarks*
- Escritura de múltiples *benchmarks* con diferentes tamaños de entrada
- Ejecución de *benchmarks*
- *Profiling* (perfilado de rendimiento)

---

## Secciones Adicionales

- Índice
- ¿Por qué suscribirse?
- Otros libros que te pueden interesar
- Packt busca autores como tú
- Comparte tus comentarios
- Descarga una copia gratuita en PDF de este libro
