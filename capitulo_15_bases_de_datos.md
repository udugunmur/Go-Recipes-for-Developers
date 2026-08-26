# Parte 3: Persistencia, Diagnóstico y Calidad de Código

## Capítulo 15: Bases de Datos

La mayoría de las aplicaciones tienen que trabajar con al menos un tipo de base de datos. Las bases de datos SQL son lo suficientemente comunes como para que la biblioteca estándar de Go ofrezca una forma unificada de conectarse y utilizarlas a través del paquete `database/sql`. Este capítulo muestra algunos de los patrones que puedes utilizar para trabajar con la implementación del paquete SQL de la biblioteca estándar.

Muchas bases de datos ofrecen extensiones no estándar, tanto en términos de funcionalidad como de dialecto de consultas. Incluso si utilizas la biblioteca estándar para interactuar con una base de datos, siempre debes consultar el controlador (*driver*) específico de la base de datos para comprender las posibles limitaciones, diferencias de implementación y el dialecto SQL admitido.

Aquí puede ser útil mencionar las bases de datos NoSQL. La biblioteca estándar de Go no ofrece un paquete de base de datos NoSQL genérico. Esto se debe a que, a diferencia de SQL, la mayoría de las bases de datos NoSQL tienen lenguajes de consulta no estándar creados específicamente para cada motor.

Este capítulo contiene las siguientes recetas:
- Conexión a una base de datos
- Ejecución de sentencias SQL
- Ejecución de sentencias preparadas (*prepared statements*) dentro de una transacción
- Obtención de valores a partir de una consulta
- Construcción dinámica de sentencias SQL
- Construcción de sentencias UPDATE
- Construcción de cláusulas WHERE

---

### Sección 1: Conexión a una Base de Datos

El paquete `database/sql` gestiona un grupo de conexiones (*connection pool*) automáticamente. La función `sql.Open` no establece una conexión de inmediato; simplemente valida los argumentos de configuración.

#### Cómo hacerlo...

```go
// Open a database connection
db, err := sql.Open("driver-name", "connection-string")
if err != nil {
   log.Fatal(err)
}
defer db.Close()
// Verify the connection
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()
if err := db.PingContext(ctx); err != nil {
   log.Fatal(err)
}
```

Configuración de límites del *pool* de conexiones:

```go
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

---

### Sección 2: Ejecución de Sentencias SQL

Las sentencias que modifican datos o esquemas (`INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`) se ejecutan mediante `ExecContext`.

#### Ejecución sin Transacción

##### Cómo hacerlo...

```go
result, err := db.ExecContext(ctx, "INSERT INTO users (name, email) VALUES ($1, $2)", "John Doe", "john@example.com")
if err != nil {
   return err
}
rowsAffected, err := result.RowsAffected()
lastInsertId, err := result.LastInsertId()
```

#### Ejecución Dentro de una Transacción

##### Cómo hacerlo...

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
if err != nil {
   return err
}
defer tx.Rollback() // Rollback is ignored if tx has already been committed

_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - 100 WHERE id = $1", fromID)
if err != nil {
   return err
}
_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + 100 WHERE id = $1", toID)
if err != nil {
   return err
}

return tx.Commit()
```

---

### Sección 3: Ejecución de Sentencias Preparadas (*Prepared Statements*) Dentro de una Transacción

Las sentencias preparadas son útiles cuando se ejecuta la misma consulta repetidamente con diferentes parámetros, mejorando el rendimiento y la seguridad contra inyecciones SQL.

#### Cómo hacerlo...

```go
stmt, err := tx.PrepareContext(ctx, "INSERT INTO logs (user_id, action) VALUES ($1, $2)")
if err != nil {
   return err
}
defer stmt.Close()

for _, entry := range logEntries {
   if _, err := stmt.ExecContext(ctx, entry.UserID, entry.Action); err != nil {
      return err
   }
}
```

---

### Sección 4: Obtención de Valores a Partir de una Consulta

Para consultas de lectura (`SELECT`), se utilizan `QueryRowContext` (para un único registro) o `QueryContext` (para múltiples registros).

#### Fila Única con `QueryRowContext`

##### Cómo hacerlo...

```go
var user User
err := db.QueryRowContext(ctx, "SELECT id, name, email, bio FROM users WHERE id = $1", userID).Scan(
   &user.ID,
   &user.Name,
   &user.Email,
   &user.Bio, // Note: Use *string or sql.NullString if bio is nullable
)
if errors.Is(err, sql.ErrNoRows) {
   // User not found
} else if err != nil {
   // Other error
}
```

#### Múltiples Filas con `QueryContext`

##### Cómo hacerlo...

```go
rows, err := db.QueryContext(ctx, "SELECT id, name, email FROM users WHERE active = $1", true)
if err != nil {
   return nil, err
}
defer rows.Close()

var users []User
for rows.Next() {
   var u User
   if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
      return nil, err
   }
   users = append(users, u)
}
if err := rows.Err(); err != nil {
   return nil, err
}
```

---

### Sección 5: Construcción Dinámica de Sentencias SQL

A menudo es necesario construir consultas dinámicamente en función de los filtros o campos proporcionados por el usuario.

#### Construcción de Sentencias UPDATE Dinámicas

##### Cómo hacerlo...

```go
func buildUpdateQuery(table string, updates map[string]any, whereClause string, whereArgs ...any) (string, []any) {
   query := fmt.Sprintf("UPDATE %s SET ", table)
   var setClauses []string
   var args []any
   argID := 1

   for col, val := range updates {
      setClauses = append(setClauses, fmt.Sprintf("%s = $%d", col, argID))
      args = append(args, val)
      argID++
   }
   query += strings.Join(setClauses, ", ")

   if whereClause != "" {
      query += fmt.Sprintf(" WHERE %s", whereClause)
      args = append(args, whereArgs...)
   }
   return query, args
}
```

#### Construcción de Cláusulas WHERE con `IN`

##### Cómo hacerlo...

```go
func buildInClause(col string, startArgIndex int, count int) string {
   var placeholders []string
   for i := 0; i < count; i++ {
      placeholders = append(placeholders, fmt.Sprintf("$%d", startArgIndex+i))
   }
   clause := fmt.Sprintf("%s IN (%s)", col, strings.Join(placeholders, ", "))
   return clause
}
```
