# SQL dinámico seguro

## En una frase

El SQL dinámico arma el texto de una consulta en tiempo de ejecución y lo ejecuta, lo que permite resolver lo que el SQL estático no puede (nombres de tablas o columnas variables, filtros opcionales, columnas de un pivot); la regla de seguridad es una sola, y no tiene excepciones: los **valores** viajan siempre como **parámetros** de `sp_executesql`, y los **identificadores** se validan contra el catálogo y se escapan con `QUOTENAME`.

-----

## Antes de empezar

Conviene que ya sepas:

* Consultas "para todo" y `OPTION (RECOMPILE)`, de [Consultas SARGables y antipatrones](../04-indices-y-rendimiento/06-Consultas%20SARGables%20y%20antipatrones.md).
* Procedimientos almacenados y el encadenamiento de propiedad, de [Procedimientos almacenados](02-Procedimientos%20almacenados.md).
* La caché de planes, de [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **SQL dinámico:** consulta que se construye como texto y se ejecuta con `EXEC` o `sp_executesql`.
* **Inyección de SQL (SQL injection):** ataque en el que un texto ingresado por el usuario se ejecuta como parte del código SQL.
* **`sp_executesql`:** procedimiento del sistema que ejecuta SQL dinámico **con parámetros**.
* **`QUOTENAME`:** función que encierra un identificador entre corchetes y escapa los corchetes que contenga.
* **Lista blanca (allowlist):** conjunto cerrado de valores permitidos, contra el que se valida una entrada.
* **Identificador:** nombre de un objeto: tabla, columna, esquema.

-----

## El problema

Una pantalla de búsqueda de clientes tiene este procedimiento:

```sql
CREATE PROCEDURE dbo.BuscarClientes @Nombre NVARCHAR(100)
AS
BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT ClienteId, Nombre, Email FROM dbo.Cliente WHERE Nombre LIKE ''' + @Nombre + N'%''';
    EXEC (@sql);
END;
```

Con `@Nombre = N'Ana'`, el texto que se ejecuta es:

```sql
SELECT ClienteId, Nombre, Email FROM dbo.Cliente WHERE Nombre LIKE 'Ana%'
```

Con `@Nombre = N'x'' OR 1=1; DROP TABLE dbo.Pedido; --'`, el texto es:

```sql
SELECT ClienteId, Nombre, Email FROM dbo.Cliente WHERE Nombre LIKE 'x' OR 1=1; DROP TABLE dbo.Pedido; --%'
```

Devuelve **todos** los clientes y, si el usuario de la aplicación tiene permisos, **borra la tabla de pedidos**. El parámetro dejó de ser un dato y se convirtió en código. Es la **inyección de SQL**, una de las vulnerabilidades más antiguas y todavía de las más explotadas.

Y lo peor: el problema no era necesario. Esta búsqueda no necesitaba SQL dinámico.

-----

## Cómo funciona

### ¿Hace falta SQL dinámico?

| Lo que varía | ¿SQL dinámico? |
| --- | --- |
| Un **valor** (un nombre, una fecha, un ID) | **No.** Un parámetro normal: `WHERE Nombre LIKE @Nombre + N'%'` |
| Qué **filtros** se aplican (búsqueda con filtros opcionales) | Opcional: `OPTION (RECOMPILE)` con SQL estático, o dinámico parametrizado para planes por combinación |
| El **orden** (`ORDER BY` elegido por el usuario) | A veces: un `CASE` en el `ORDER BY`, o dinámico con lista blanca |
| Un **nombre de tabla o columna** | **Sí.** Los identificadores no se pueden parametrizar |
| Las **columnas** de un pivot (una por mes, por categoría) | **Sí** |
| **DDL** sobre objetos variables (mantenimiento de índices de todas las tablas) | **Sí** |

La primera fila resuelve el problema del principio: los valores **nunca** necesitan concatenarse.

### `EXEC` frente a `sp_executesql`

```sql
-- EXEC: solo texto, sin parámetros
EXEC (N'SELECT ... WHERE ClienteId = ' + CAST(@Id AS NVARCHAR(10)));

-- sp_executesql: texto + definición de parámetros + valores
EXEC sp_executesql
     N'SELECT ClienteId, Nombre FROM dbo.Cliente WHERE ClienteId = @pId',
     N'@pId INT',
     @pId = @Id;
```

| | `EXEC (@sql)` | `sp_executesql` |
| --- | --- | --- |
| Parámetros | No: todo se concatena | Sí, de entrada y de salida |
| Inyección de valores | Vulnerable si se concatena una entrada | Inmune para los valores pasados como parámetros |
| Reutilización del plan | Un plan por cada texto distinto (cada valor distinto) | Un plan por texto, reutilizado con cualquier valor |
| Uso | Casi nunca | Siempre que haya SQL dinámico |

Con `EXEC`, buscar 1.000 clientes distintos genera 1.000 textos distintos y 1.000 planes en la caché. Con `sp_executesql`, un solo texto y un solo plan.

### Los valores: siempre parámetros

```sql
DECLARE @sql NVARCHAR(MAX) = N'
    SELECT ClienteId, Nombre, Email
    FROM dbo.Cliente
    WHERE Nombre LIKE @pNombre + N''%''';

EXEC sp_executesql @sql, N'@pNombre NVARCHAR(100)', @pNombre = @Nombre;
```

El valor de `@Nombre` nunca forma parte del texto del SQL. Aunque contenga comillas, punto y coma o `DROP TABLE`, el motor lo trata como **un dato** que se compara con `Nombre`. Esto vale para cualquier valor, sin necesidad de "escapar" comillas a mano (lo que siempre termina fallando con algún caso).

Los parámetros `OUTPUT` funcionan igual que en un procedimiento:

```sql
DECLARE @Total INT;
EXEC sp_executesql
     N'SELECT @pTotal = COUNT(*) FROM dbo.Pedido WHERE ClienteId = @pId',
     N'@pId INT, @pTotal INT OUTPUT',
     @pId = 1, @pTotal = @Total OUTPUT;
SELECT @Total;   -- 2
```

### Los identificadores: validar y `QUOTENAME`

Un nombre de tabla o de columna **no puede** ser un parámetro (`FROM @tabla` no funciona). Si tiene que variar, se concatena, y ahí está el riesgo. La protección tiene dos pasos:

1. **Validar contra el catálogo** (o contra una lista blanca): el objeto tiene que existir.
2. **Escapar con `QUOTENAME`:** encierra el nombre entre corchetes y duplica cualquier `]` que contenga, así que no puede "cerrar" el identificador e inyectar código.

```sql
DECLARE @Esquema SYSNAME = N'dbo', @Tabla SYSNAME = N'Pedido';

IF OBJECT_ID(QUOTENAME(@Esquema) + N'.' + QUOTENAME(@Tabla), N'U') IS NULL
    THROW 50401, N'La tabla no existe.', 1;

DECLARE @sql NVARCHAR(MAX) =
    N'SELECT COUNT(*) AS Filas FROM ' + QUOTENAME(@Esquema) + N'.' + QUOTENAME(@Tabla) + N';';

EXEC sp_executesql @sql;
```

Detalles de `QUOTENAME` que conviene conocer:

* **Devuelve `NULL` si la entrada supera los 128 caracteres.** Un `NULL` concatenado convierte todo el `@sql` en `NULL`, y `sp_executesql` no ejecuta nada (sin error). Valida la longitud, o usa `SYSNAME` (que es `NVARCHAR(128)`) para los identificadores.
* **Escapa identificadores, no valores.** `QUOTENAME(@valor, '''')` existe, pero para los valores la herramienta correcta es un parámetro.
* **Valida esquema y nombre por separado:** `QUOTENAME(N'dbo.Pedido')` produce `[dbo.Pedido]`, una tabla con un punto en el nombre, que no existe.

### Filtros opcionales con SQL dinámico parametrizado

La búsqueda "para todo" de lecciones anteriores, en su versión dinámica:

```sql
CREATE OR ALTER PROCEDURE dbo.BuscarPedidos
    @ClienteId INT          = NULL,
    @Estado    VARCHAR(20)  = NULL,
    @Desde     DATE         = NULL,
    @Debug     BIT          = 0
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT PedidoId, ClienteId, FechaPedido, Estado
        FROM dbo.Pedido
        WHERE 1 = 1';

    IF @ClienteId IS NOT NULL SET @sql += N' AND ClienteId = @pClienteId';
    IF @Estado    IS NOT NULL SET @sql += N' AND Estado = @pEstado';
    IF @Desde     IS NOT NULL SET @sql += N' AND FechaPedido >= @pDesde';

    SET @sql += N' ORDER BY FechaPedido DESC, PedidoId DESC;';

    IF @Debug = 1
    BEGIN
        PRINT @sql;   -- ver el texto armado, sin ejecutarlo
        RETURN;
    END;

    EXEC sp_executesql @sql,
         N'@pClienteId INT, @pEstado VARCHAR(20), @pDesde DATE',
         @pClienteId = @ClienteId, @pEstado = @Estado, @pDesde = @Desde;
END;
```

* Solo se agregan al texto las condiciones de los filtros que llegaron; los **valores** siempre van como parámetros.
* Se declaran **todos** los parámetros en la definición, aunque el texto no los use: `sp_executesql` acepta parámetros sin usar.
* Cada **combinación** de filtros produce un texto distinto y, por lo tanto, su propio plan en caché, adecuado a esa combinación. A diferencia de `OPTION (RECOMPILE)`, no se recompila en cada ejecución.
* `WHERE 1 = 1` permite agregar todas las condiciones con `AND` sin casos especiales.
* `@Debug = 1` muestra el texto armado: es la mejor herramienta para depurar SQL dinámico.

### `ORDER BY` elegido por el usuario: lista blanca

```sql
DECLARE @Columna SYSNAME = CASE @OrdenarPor
    WHEN 'fecha'  THEN N'FechaPedido'
    WHEN 'estado' THEN N'Estado'
    WHEN 'id'     THEN N'PedidoId'
    ELSE N'FechaPedido'                       -- valor por defecto seguro
END;
DECLARE @Direccion NVARCHAR(4) = IIF(@Descendente = 1, N'DESC', N'ASC');

SET @sql += N' ORDER BY ' + QUOTENAME(@Columna) + N' ' + @Direccion + N';';
```

El usuario elige entre opciones conocidas; lo que se concatena **nunca** es lo que el usuario escribió, sino un valor de la lista. La dirección sale de un `BIT`, que solo puede producir dos textos.

### Permisos: el SQL dinámico rompe el encadenamiento

Un procedimiento con permiso `EXECUTE` puede leer tablas sobre las que el usuario no tiene permisos gracias al encadenamiento de propiedad (ver [Procedimientos almacenados](02-Procedimientos%20almacenados.md)). **El SQL dinámico no hereda esa cadena:** se ejecuta en su propio contexto, y el motor verifica los permisos del **usuario que llamó** sobre las tablas que el texto menciona.

Si el usuario de la aplicación solo tiene `EXECUTE`, el procedimiento con SQL dinámico falla con un error de permisos sobre la tabla. Las salidas son darle los permisos de lectura (lo que el procedimiento quería evitar), firmar el procedimiento con un certificado, o `WITH EXECUTE AS OWNER` (con la responsabilidad de que **todo** el SQL dinámico se ejecute con permisos de dueño, lo que hace todavía más crítico no tener inyección).

-----

## Ejemplo completo

Un reporte mensual con una columna por mes (un pivot cuyas columnas dependen de los datos), seguro:

```sql
CREATE OR ALTER PROCEDURE dbo.VentasPorCategoriaYMes
    @Anio  INT,
    @Debug BIT = 0
AS
BEGIN
    SET NOCOUNT ON;

    IF @Anio NOT BETWEEN 2000 AND 2100
        THROW 50402, N'Año fuera de rango.', 1;

    -- 1. Las columnas: los meses con ventas del año, como identificadores escapados
    DECLARE @Columnas NVARCHAR(MAX);

    SELECT @Columnas = STRING_AGG(QUOTENAME(Mes), N', ') WITHIN GROUP (ORDER BY Mes)
    FROM (SELECT DISTINCT FORMAT(FechaPedido, 'yyyy-MM') AS Mes
          FROM dbo.Pedido
          WHERE FechaPedido >= DATEFROMPARTS(@Anio, 1, 1)
            AND FechaPedido <  DATEFROMPARTS(@Anio + 1, 1, 1)) AS m;

    IF @Columnas IS NULL
        THROW 50403, N'No hay ventas en ese año.', 1;

    -- 2. El pivot: las columnas concatenadas (ya escapadas), el año como parámetro
    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT Categoria, ' + @Columnas + N'
        FROM (
            SELECT c.Nombre AS Categoria,
                   FORMAT(p.FechaPedido, ''yyyy-MM'') AS Mes,
                   d.Cantidad * d.PrecioUnitario AS Importe
            FROM dbo.Pedido AS p
            JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
            JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
            JOIN dbo.Categoria AS c ON c.CategoriaId = pr.CategoriaId
            WHERE p.FechaPedido >= DATEFROMPARTS(@pAnio, 1, 1)
              AND p.FechaPedido <  DATEFROMPARTS(@pAnio + 1, 1, 1)
              AND p.Estado <> ''Cancelado''
        ) AS origen
        PIVOT (SUM(Importe) FOR Mes IN (' + @Columnas + N')) AS pvt
        ORDER BY Categoria;';

    IF @Debug = 1 BEGIN PRINT @sql; RETURN; END;

    EXEC sp_executesql @sql, N'@pAnio INT', @pAnio = @Anio;
END;
GO

EXEC dbo.VentasPorCategoriaYMes @Anio = 2026;
```

En `Tienda`:

| Categoria | 2026-03 | 2026-04 | 2026-05 |
| --- | --- | --- | --- |
| Electrónica | 1060.90 | `NULL` | 45.50 |
| Hogar | `NULL` | 75.00 | `NULL` |
| Libros | `NULL` | 190.00 | 95.00 |

Qué decisión hay detrás:

* **Las columnas salen de los datos,** no del usuario, y aun así se escapan con `QUOTENAME`: `[2026-03]` es un identificador válido, y si algún día un valor tuviera caracteres extraños, no podría romper el texto.
* **El año viaja como parámetro (`@pAnio`)**, nunca concatenado, aunque sea un número validado.
* **El pivot necesita SQL dinámico** porque la lista de columnas del `IN (...)` no puede ser un parámetro: es la definición de la forma del resultado.
* **Los `NULL`** son los meses sin ventas de esa categoría (Hogar en marzo); con `COALESCE` en la lista de columnas del `SELECT` exterior se mostrarían como 0, a costa de armar esa lista también.
* **`@Debug`** permite ver el texto exacto que se va a ejecutar, que en un pivot dinámico es la forma más rápida de encontrar un error de sintaxis.
* **`FORMAT` es lento** en volúmenes grandes (ver [Expresiones, CASE y funciones](../03-consultas/04-Expresiones%2C%20CASE%20y%20funciones.md)); en `TiendaGrande`, convendría `CONVERT(CHAR(7), FechaPedido, 120)`, que produce el mismo `yyyy-MM`.

-----

## Errores comunes

**1. Concatenar valores en el texto.**
Qué pasa: inyección de SQL, además de un plan nuevo por cada valor.
Por qué: es la forma "obvia" de armar el texto.
Arreglo: `sp_executesql` con parámetros para todos los valores.

**2. "Escapar" comillas a mano.**
Qué pasa: una protección que falla con algún caso (caracteres Unicode, codificaciones, comentarios).
Por qué: se intenta proteger el texto en lugar de no meter datos en él.
Arreglo: parámetros; para identificadores, `QUOTENAME` y validación.

**3. Identificadores sin validar.**
Qué pasa: un nombre de tabla con código adentro, o errores confusos con nombres que no existen.
Por qué: se concatena lo que llega.
Arreglo: validar contra `sys.objects`, `sys.columns` o una lista blanca, y escapar con `QUOTENAME`.

**4. Un `NULL` que anula todo el texto.**
Qué pasa: `sp_executesql` no hace nada, sin error.
Por qué: `QUOTENAME` de más de 128 caracteres, o un parámetro `NULL` concatenado, convierte todo `@sql` en `NULL`.
Arreglo: validar las entradas, y `@Debug` para ver el texto.

**5. SQL dinámico donde no hacía falta.**
Qué pasa: código más difícil de leer, de mantener y de proteger, sin ninguna ventaja.
Por qué: se usa para variar valores, que se resuelven con parámetros normales.
Arreglo: SQL estático siempre que se pueda.

**6. Olvidar que rompe el encadenamiento de propiedad.**
Qué pasa: errores de permisos en producción, aunque en desarrollo (con un usuario administrador) funcionaba.
Por qué: el SQL dinámico verifica los permisos del llamador.
Arreglo: probar con el usuario real de la aplicación; resolver con certificados o `EXECUTE AS` de forma consciente.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Ejecutar texto | `EXEC (@sql)` | `EXECUTE sql` (en PL/pgSQL) | `PREPARE ... FROM @sql; EXECUTE ...` |
| Con parámetros | `sp_executesql @sql, N'@p INT', @p = 1` | `EXECUTE sql USING valor` | `EXECUTE stmt USING @valor` |
| Escapar identificadores | `QUOTENAME(nombre)` → `[nombre]` | `quote_ident(nombre)` o `format('%I', nombre)` | Comillas invertidas a mano (`` ` ``), con cuidado de duplicarlas |
| Escapar valores (si no hay parámetros) | `QUOTENAME(valor, '''')` | `quote_literal(valor)` o `format('%L', valor)` | `QUOTE(valor)` |
| Pivot | `PIVOT` (columnas fijas en el texto) | `crosstab` (extensión `tablefunc`) o agregación condicional | Agregación condicional |

En PostgreSQL, `format()` con `%I` (identificador) y `%L` (literal) es la forma idiomática de armar SQL dinámico seguro en PL/pgSQL, y `USING` pasa los valores como parámetros.

### En MongoDB

La inyección también existe en MongoDB (*NoSQL injection*): si la aplicación construye un filtro a partir de un JSON del usuario sin validar, un valor como `{ "$ne": null }` en lugar de un texto convierte "busca este email" en "busca cualquier email". La protección es la misma idea: validar los tipos de lo que llega (que un email sea un texto, no un objeto) y nunca armar consultas concatenando texto. Los drivers con parámetros tipados y las bibliotecas de validación de esquemas lo previenen.

-----

## Cuándo sí y cuándo no

**Usa SQL dinámico cuando:**

* Varía un identificador (tabla, columna, esquema) o la forma del resultado (pivot).
* Hay muchos filtros opcionales y quieres un plan por combinación sin recompilar siempre.
* Generas DDL o tareas de mantenimiento sobre objetos variables.

**No lo uses cuando:**

* Lo único que varía son valores: un parámetro normal alcanza.
* `OPTION (RECOMPILE)` resuelve la consulta "para todo" con suficiente rendimiento.

**Siempre:**

* Valores como parámetros de `sp_executesql`; identificadores validados y con `QUOTENAME`; un `@Debug` para ver el texto.

-----

## Resumen en 5 líneas

1. Los valores nunca se concatenan: viajan como parámetros de `sp_executesql`, lo que impide la inyección y reutiliza el plan.
2. Los identificadores no se pueden parametrizar: se validan contra el catálogo o una lista blanca y se escapan con `QUOTENAME`.
3. `QUOTENAME` devuelve `NULL` con más de 128 caracteres, y un `NULL` anula todo el texto: valida y depura con `@Debug`.
4. Filtros opcionales dinámicos parametrizados: un plan por combinación; `ORDER BY` variable, con lista blanca.
5. El SQL dinámico verifica los permisos del llamador: rompe el encadenamiento de propiedad.

-----

## Para profundizar

<details>
<summary>Un catálogo de ataques, para entender qué se previene</summary>

Tu cheat sheet de la sesión 13 resume las variantes clásicas: saltarse una autenticación (`' OR '1'='1`), comentar el resto de la consulta (`admin'--`), encadenar instrucciones (`'; DROP TABLE X;--`), extraer datos con `UNION` (`' UNION SELECT ... FROM ...`) e inferir datos a ciegas (`' AND 1=1 --`, comparando respuestas). Todas tienen la misma causa: un dato que termina interpretado como código. Y todas tienen la misma solución: que el dato nunca forme parte del texto, lo que logran los parámetros. El principio de mínimo privilegio (que el usuario de la aplicación no pueda borrar tablas) limita el daño si algo falla, pero no reemplaza a los parámetros.

</details>

<details>
<summary>Firmar procedimientos con certificados</summary>

Para que un procedimiento con SQL dinámico pueda leer tablas sin dar permisos al usuario, sin usar `EXECUTE AS` (que cambia toda la identidad), SQL Server permite **firmar** el procedimiento con un certificado y dar los permisos a un usuario asociado al certificado. Los permisos se agregan solo dentro de ese procedimiento, la identidad del llamador se conserva para la auditoría, y si alguien modifica el procedimiento, la firma se invalida. Es la técnica recomendada por la documentación de seguridad para este caso.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El SQL dinámico es armar una consulta como texto y ejecutarla. Para que sea seguro, uso `sp_executesql` con parámetros en lugar de concatenar los valores, porque concatenar permite la inyección de SQL. Para los nombres de tablas o columnas, uso `QUOTENAME` y verifico que existan.

### Respuesta ampliada (semi-senior)

Solo uso SQL dinámico cuando varía un identificador, la forma del resultado (un pivot) o la combinación de filtros, nunca para variar valores. La regla es absoluta: los valores viajan como parámetros de `sp_executesql`, lo que impide la inyección y permite reutilizar un plan por texto; los identificadores, que no se pueden parametrizar, se validan contra el catálogo o una lista blanca y se escapan con `QUOTENAME`, sabiendo que devuelve `NULL` con más de 128 caracteres y que un `NULL` anula todo el texto. Para búsquedas con filtros opcionales, agrego al texto solo las condiciones presentes, con todos los valores como parámetros, lo que da un plan por combinación sin recompilar siempre; para un `ORDER BY` variable, uso una lista blanca. Siempre agrego un parámetro `@Debug` para ver el texto, y tengo en cuenta que el SQL dinámico verifica los permisos del llamador, así que rompe el encadenamiento de propiedad; si hace falta, firmo el procedimiento con un certificado o uso `EXECUTE AS` de forma consciente.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `EXEC` y `sp_executesql`?**
`sp_executesql` admite parámetros: los valores no forman parte del texto, lo que evita la inyección y reutiliza el plan. `EXEC` solo ejecuta texto.

**2. ¿Se puede pasar un nombre de tabla como parámetro?**
No. Se concatena, después de validarlo contra el catálogo y escaparlo con `QUOTENAME`.

**3. ¿Cómo evitas la inyección de SQL?**
Con parámetros para todos los valores; validación y `QUOTENAME` para los identificadores; listas blancas para las opciones; y mínimo privilegio para limitar el daño.

**4. ¿Por qué un procedimiento con SQL dinámico puede fallar por permisos aunque el usuario tenga `EXECUTE`?**
Porque el SQL dinámico se ejecuta en su propio contexto y verifica los permisos del llamador sobre las tablas: no hereda el encadenamiento de propiedad.

-----

## Práctica

**Ejercicio 1.** Reescribe el procedimiento `dbo.BuscarClientes` del problema para que sea seguro. ¿Necesita SQL dinámico?

<details>
<summary>Solución</summary>

No lo necesita: lo único que varía es un **valor**.

```sql
CREATE OR ALTER PROCEDURE dbo.BuscarClientes @Nombre NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    SELECT ClienteId, Nombre, Email
    FROM dbo.Cliente
    WHERE Nombre LIKE @Nombre + N'%';
END;
```

Con `@Nombre = N'x'' OR 1=1; DROP TABLE dbo.Pedido; --'`, la consulta busca clientes cuyo nombre empiece con ese texto literal (con comillas, punto y coma y todo), no encuentra ninguno, y no ejecuta nada más. Si además el nombre pudiera contener `%` o `_` y debieran tratarse como literales, habría que escaparlos para el `LIKE` (ver [Filtrar con WHERE](../03-consultas/02-Filtrar%20con%20WHERE.md)); eso es un tema de búsqueda, no de seguridad.

</details>

**Ejercicio 2.** Escribe un procedimiento `dbo.ContarFilas @Esquema SYSNAME, @Tabla SYSNAME` que devuelva la cantidad de filas de una tabla cualquiera, de forma segura.

<details>
<summary>Solución</summary>

```sql
CREATE OR ALTER PROCEDURE dbo.ContarFilas @Esquema SYSNAME, @Tabla SYSNAME
AS
BEGIN
    SET NOCOUNT ON;

    -- 1. Validar contra el catálogo (tabla de usuario)
    IF NOT EXISTS (
        SELECT 1
        FROM sys.tables AS t
        JOIN sys.schemas AS s ON s.schema_id = t.schema_id
        WHERE s.name = @Esquema AND t.name = @Tabla
    )
        THROW 50404, N'La tabla no existe.', 1;

    -- 2. Armar con QUOTENAME, por partes
    DECLARE @sql NVARCHAR(MAX) =
        N'SELECT COUNT_BIG(*) AS Filas FROM ' + QUOTENAME(@Esquema) + N'.' + QUOTENAME(@Tabla) + N';';

    -- 3. Ejecutar
    EXEC sp_executesql @sql;
END;
GO

EXEC dbo.ContarFilas @Esquema = N'dbo', @Tabla = N'Pedido';   -- 5 en Tienda
EXEC dbo.ContarFilas @Esquema = N'dbo', @Tabla = N'Pedido]; DROP TABLE dbo.Cliente; --';   -- Msg 50404
```

El segundo intento falla en la validación (no hay una tabla con ese nombre). Y aunque la validación no existiera, `QUOTENAME` convertiría el texto en `[Pedido]]; DROP TABLE dbo.Cliente; --]`, un único identificador (inexistente) en lugar de código. `SYSNAME` limita los parámetros a 128 caracteres, el límite de `QUOTENAME`. Para conteos sobre tablas grandes, `sys.dm_db_partition_stats` da la cantidad de filas sin recorrer la tabla.

</details>

-----

## Siguiente lección

Con esto termina el módulo de programación en T-SQL. El siguiente módulo trata cómo observar, automatizar y auditar lo que pasa en el servidor:

[DMVs para diagnóstico](../07-monitoreo-y-administracion/01-DMVs%20para%20diagnóstico.md)
