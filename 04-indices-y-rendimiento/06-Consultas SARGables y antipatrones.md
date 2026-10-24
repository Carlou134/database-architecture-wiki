# Consultas SARGables y antipatrones

## En una frase

Un índice solo sirve si la consulta lo deja usar: una condición **SARGable** deja la columna sola, sin funciones ni operaciones, de un lado de la comparación, y la mayoría de las consultas lentas con índices "que no se usan" se explican por un puñado de antipatrones conocidos: funciones sobre columnas, conversiones implícitas, comodines al principio, `OR` entre columnas distintas y filtros "para todo".

-----

## Antes de empezar

Conviene que ya sepas:

* Filtros con `WHERE` y la idea de SARGable, de [Filtrar con WHERE](../03-consultas/02-Filtrar%20con%20WHERE.md).
* Leer planes y estadísticas, de [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md) y [II](05-Planes%20de%20ejecución%20II.md).
* La precedencia de tipos, de [Expresiones, CASE y funciones](../03-consultas/04-Expresiones%2C%20CASE%20y%20funciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **SARGable (*Search ARGument able*):** condición que el motor puede resolver con un *seek* en un índice.
* **Conversión implícita:** conversión de tipo que el motor agrega por su cuenta; si afecta a la columna, impide el *seek*.
* **Consulta "para todo" (catch-all):** consulta con filtros opcionales del tipo `@p IS NULL OR Col = @p`.
* **Problema N+1:** ejecutar una consulta por cada fila de un resultado anterior, en lugar de una sola consulta.
* **Función escalar definida por el usuario (UDF):** función propia que devuelve un valor y se evalúa fila por fila.

-----

## El problema

La tabla `Pedido` de `TiendaGrande` tiene un índice sobre `FechaPedido`. Esta consulta, para el reporte de 2026, lo ignora y recorre las 4.500 páginas:

```sql
SELECT COUNT(*) FROM dbo.Pedido WHERE YEAR(FechaPedido) = 2026;
```

La misma pregunta, escrita de otra forma, usa el índice y lee unas pocas centenas de páginas:

```sql
SELECT COUNT(*) FROM dbo.Pedido WHERE FechaPedido >= '2026-01-01' AND FechaPedido < '2027-01-01';
```

El índice existe en los dos casos. Lo que cambia es si la consulta **le permite** al motor buscar en él. Y este patrón (un índice correcto que la consulta no deja usar) es la causa más frecuente de consultas lentas en sistemas que "ya tienen los índices".

-----

## Cómo funciona

### Por qué una función impide el seek

El índice está ordenado por `FechaPedido`. Para responder `FechaPedido >= '2026-01-01'`, el motor salta a la primera entrada de 2026 y lee hacia adelante. Pero el índice **no** está ordenado por `YEAR(FechaPedido)`: para saber qué filas tienen `YEAR(...) = 2026`, el motor tiene que calcular la función **en cada fila**, es decir, recorrer todo el índice.

La fórmula general que resume tu cheat sheet:

```text
función(COLUMNA) = valor        →  COLUMNA = función_inversa(valor)
       NO SARGable                         SARGable
```

### La tabla de reescrituras

| ❌ No SARGable | ✅ SARGable |
| --- | --- |
| `WHERE YEAR(Fecha) = 2026` | `WHERE Fecha >= '2026-01-01' AND Fecha < '2027-01-01'` |
| `WHERE MONTH(Fecha) = 3 AND YEAR(Fecha) = 2026` | `WHERE Fecha >= '2026-03-01' AND Fecha < '2026-04-01'` |
| `WHERE CAST(Fecha AS DATE) = @Dia` | `WHERE Fecha >= @Dia AND Fecha < DATEADD(day, 1, @Dia)` |
| `WHERE DATEADD(day, 30, Fecha) > @Hoy` | `WHERE Fecha > DATEADD(day, -30, @Hoy)` |
| `WHERE DATEDIFF(day, Fecha, @Hoy) < 7` | `WHERE Fecha > DATEADD(day, -7, @Hoy)` (con cuidado: `DATEDIFF` cuenta límites de día) |
| `WHERE Precio * 1.18 > 236` | `WHERE Precio > 236 / 1.18` |
| `WHERE LEFT(Email, 7) = 'cliente'` | `WHERE Email LIKE 'cliente%'` |
| `WHERE SUBSTRING(Codigo, 1, 2) = 'PE'` | `WHERE Codigo LIKE 'PE%'` |
| `WHERE ISNULL(Ciudad, N'') = N'Lima'` | `WHERE Ciudad = N'Lima'` |
| `WHERE ISNULL(Descuento, 0) = 0` | `WHERE Descuento = 0 OR Descuento IS NULL` |

Una corrección importante sobre tus apuntes: la tabla de la sesión 3 reescribe `ISNULL(Col, 0) = 5` como `Col = 5 OR Col IS NULL`, y eso **cambia el resultado**. `ISNULL(Col, 0) = 5` solo es verdadero cuando `Col` vale 5 (un `NULL` se convierte en 0, que no es 5), así que equivale a `Col = 5` a secas. La versión con `OR Col IS NULL` corresponde a `ISNULL(Col, 0) = 0`. Al reescribir, verifica siempre que la nueva condición devuelva **exactamente** las mismas filas.

Un caso que funciona aunque parezca no SARGable: `CAST(Fecha AS DATE) = @Dia` sobre una columna `DATETIME2` puede usar un *seek*, porque SQL Server tiene una optimización específica para esa conversión. Aun así, el rango explícito es más claro y funciona en todos los motores.

### Conversiones implícitas

Cuando se comparan tipos distintos, el de menor precedencia se convierte al de mayor. Si el que se convierte es **la columna**, se pierde el *seek*:

| Columna | Valor | ¿Qué se convierte? | ¿Seek? |
| --- | --- | --- | --- |
| `VARCHAR` | `N'abc'` (`NVARCHAR`) | **La columna** a `NVARCHAR` | ❌ (o un *seek* degradado, según la intercalación) |
| `NVARCHAR` | `'abc'` (`VARCHAR`) | El valor | ✅ |
| `VARCHAR` (con números: `'12345'`) | `12345` (`INT`) | **La columna** a `INT` (y falla si algún valor no es numérico) | ❌ |
| `INT` | `'12345'` (`VARCHAR`) | El valor | ✅ |
| `DECIMAL(10,2)` | `99.99` | Nada problemático: el literal es `NUMERIC` | ✅ |
| `DATETIME2` | `'2026-03-28'` | El valor (el literal de texto se convierte a fecha) | ✅ |

Otra corrección sobre los apuntes: los dos últimos casos aparecen como problemas en el cheat sheet de la sesión 3, y no lo son. `99.99` no es un `FLOAT` sino un `NUMERIC(4,2)`, y un literal de texto comparado con una columna de fecha se convierte el literal, no la columna. El problema real es siempre el mismo: **la columna** del lado de menor precedencia.

El caso más frecuente en la práctica viene de la aplicación: muchos ORM y drivers envían los textos como `NVARCHAR` por defecto. Si la columna es `VARCHAR`, cada consulta convierte la columna. En el plan aparece la advertencia `Type conversion in expression (CONVERT_IMPLICIT(nvarchar, ...)) may affect "SeekPlan"`. El arreglo está en la aplicación (en EF Core, `.IsUnicode(false)` o `HasColumnType("varchar(20)")` en la propiedad).

### `LIKE` con comodín al principio

`LIKE 'cliente5%'` es SARGable: el índice está ordenado por texto, y los valores que empiezan con "cliente5" están juntos. `LIKE '%5@correo.com'` no: los valores que **terminan** en algo están repartidos por todo el índice.

Para búsquedas de "contiene" sobre textos largos y tablas grandes, la herramienta es la búsqueda de texto completo (*Full-Text Search*), con su propio tipo de índice. Para sufijos, un truco es guardar el texto invertido en una columna calculada indexada y buscar el prefijo del texto invertido.

### `OR` entre columnas distintas

```sql
SELECT PedidoId FROM dbo.Pedido WHERE ClienteId = 700 OR FechaPedido >= '2026-06-29';
```

Un índice por `ClienteId` no sirve para la parte de la fecha, y uno por fecha no sirve para la del cliente. El optimizador a veces combina dos índices (*index union*), pero a menudo recorre la tabla. La reescritura con `UNION` permite que cada parte use su índice:

```sql
SELECT PedidoId FROM dbo.Pedido WHERE ClienteId = 700
UNION   -- UNION, no UNION ALL: un pedido puede cumplir las dos condiciones
SELECT PedidoId FROM dbo.Pedido WHERE FechaPedido >= '2026-06-29';
```

Un `OR` sobre la **misma** columna (`Estado = 'Pagado' OR Estado = 'Enviado'`, o un `IN`) no tiene este problema: es un *seek* con varios rangos.

### Consultas "para todo"

Un patrón muy común en pantallas de búsqueda con filtros opcionales:

```sql
SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE (@ClienteId IS NULL OR ClienteId = @ClienteId)
  AND (@Estado    IS NULL OR Estado    = @Estado)
  AND (@Desde     IS NULL OR FechaPedido >= @Desde);
```

El plan se compila **una vez** y tiene que servir para cualquier combinación de parámetros (con cliente, sin cliente, con fecha...), así que el optimizador elige un plan genérico, casi siempre un *scan*. Las soluciones:

* **`OPTION (RECOMPILE)`:** con el valor conocido en cada ejecución, el optimizador simplifica las condiciones (`@ClienteId IS NULL` se vuelve verdadero o falso) y elige un plan para esa combinación. Es la solución más simple.
* **SQL dinámico parametrizado:** armar la consulta con solo las condiciones que llegaron, y ejecutarla con `sp_executesql` y parámetros. Cada combinación tiene su propio plan en caché. Se ve en [SQL dinámico seguro](../06-programacion-t-sql/07-SQL%20dinámico%20seguro.md).

### Otros antipatrones frecuentes

| Antipatrón | Problema | Alternativa |
| --- | --- | --- |
| **Funciones escalares propias en el `SELECT` o el `WHERE`** | Se ejecutan fila por fila, ocultan su costo en el plan e impedían el paralelismo antes de 2019 | Expresión en línea, función de tabla en línea o *scalar UDF inlining* (2019+) |
| **`SELECT *`** | Trae columnas innecesarias e impide que un índice cubra la consulta | Listar las columnas |
| **`DISTINCT` o `ORDER BY` innecesarios** | Ordenamientos costosos que nadie necesita | Quitarlos; corregir el join que genera duplicados |
| **N+1 desde la aplicación** | Mil consultas de una fila en lugar de una consulta de mil filas: mil viajes de red y mil ejecuciones | Un join, o una consulta con `IN` / una lista de valores |
| **`NOT IN`, `<>` y `NOT LIKE` sobre tablas grandes** | Rara vez selectivos: devuelven casi todo | Expresar la condición en positivo si se puede (`Estado IN (...)`) |
| **Comparar columnas de tipos distintos en un join** | Conversión implícita en cada fila del join | Mismo tipo en la FK y en la PK (ver [Tipos de datos](../01-fundamentos/03-Tipos%20de%20datos.md)) |

### Cuando la función es inevitable: columna calculada indexada

Si una consulta frecuente necesita de verdad filtrar por una expresión, se puede materializar como columna calculada e indexarla. Tu cheat sheet tiene el ejemplo:

```sql
ALTER TABLE dbo.Pedido ADD AnioPedido AS YEAR(FechaPedido) PERSISTED;
CREATE INDEX IX_Pedido_AnioPedido ON dbo.Pedido (AnioPedido);
```

Ahora `WHERE AnioPedido = 2026` (y también `WHERE YEAR(FechaPedido) = 2026`, porque el optimizador reconoce la expresión) puede usar el índice. Para expresiones deterministas, ni siquiera hace falta `PERSISTED` para indexarlas; `PERSISTED` guarda el valor y evita recalcularlo al leer.

-----

## Ejemplo completo

La pantalla de "pedidos por mes" de `TiendaGrande`, con tres problemas a la vez:

**Antes:**

```sql
DECLARE @Anio INT = 2026, @Mes INT = 3, @Estado NVARCHAR(20) = N'Entregado';

SELECT PedidoId, ClienteId, FechaPedido
FROM dbo.Pedido
WHERE YEAR(FechaPedido) = @Anio
  AND MONTH(FechaPedido) = @Mes
  AND Estado = @Estado;
```

Con el índice `IX_Pedido_Fecha (FechaPedido) INCLUDE (Estado)` de [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md), el plan es un **scan** de ~4.500 lecturas, con dos señales:

* El filtro de fecha aparece como *Predicate* (residual), no como *Seek Predicate*.
* Una advertencia `CONVERT_IMPLICIT(nvarchar(20), [Estado], 0)`: la columna `Estado` (`VARCHAR`) se convierte para compararla con `@Estado` (`NVARCHAR`).

**Después:**

```sql
DECLARE @Anio INT = 2026, @Mes INT = 3, @Estado VARCHAR(20) = 'Entregado';
DECLARE @Desde DATE = DATEFROMPARTS(@Anio, @Mes, 1);
DECLARE @Hasta DATE = DATEADD(month, 1, @Desde);

SELECT PedidoId, ClienteId, FechaPedido
FROM dbo.Pedido
WHERE FechaPedido >= @Desde
  AND FechaPedido <  @Hasta
  AND Estado = @Estado;
```

Ahora el filtro de fecha **puede** resolverse con un *seek* y la advertencia de conversión desaparece. Pero el plan probablemente siga siendo un *scan*: el mes tiene unos 24.000 pedidos (unos 20.000 entregados), y `ClienteId` no está en el índice, así que cada fila necesitaría un *Key Lookup*, muy por encima del punto de inflexión. Hace falta el segundo paso:

```sql
DROP INDEX IX_Pedido_Fecha ON dbo.Pedido;
CREATE INDEX IX_Pedido_Fecha ON dbo.Pedido (FechaPedido) INCLUDE (Estado, ClienteId);
```

Con el índice cubriente, la consulta es un *Index Seek* sobre el rango del mes, sin *lookups*, con un centenar de lecturas en lugar de ~4.500.

Qué cambió y por qué:

* **Rango semiabierto calculado una vez:** `DATEFROMPARTS` y `DATEADD` se aplican a las **variables**, no a la columna; la columna queda sola en la comparación.
* **`@Estado VARCHAR(20)`:** el mismo tipo que la columna, sin conversión implícita.
* **SARGable es necesario, pero no suficiente:** la reescritura le permitió al optimizador usar el índice; el `INCLUDE` hizo que le convenga. Si siempre se filtra por estado, `(Estado, FechaPedido) INCLUDE (ClienteId)` sería todavía más preciso. El orden es siempre el mismo: primero la consulta SARGable, después el índice.

-----

## Errores comunes

**1. Funciones sobre la columna del filtro.**
Qué pasa: el índice existe y el plan muestra un *scan*.
Por qué: el índice no está ordenado por el resultado de la función.
Arreglo: aplicar la función inversa al valor, o una columna calculada indexada.

**2. Reescribir y cambiar el resultado.**
Qué pasa: la versión SARGable devuelve filas de más o de menos (como el `ISNULL` de los apuntes).
Por qué: la equivalencia se supuso sin verificarla, sobre todo con `NULL` y con límites de fechas.
Arreglo: comparar las dos versiones con `EXCEPT` en los dos sentidos antes de reemplazar la consulta.

**3. Parámetros `NVARCHAR` contra columnas `VARCHAR`.**
Qué pasa: conversión implícita en cada fila, *scans* en lugar de *seeks*.
Por qué: el driver o el ORM envían Unicode por defecto.
Arreglo: configurar el tipo del parámetro en la aplicación igual al de la columna.

**4. Consultas "para todo" sin `RECOMPILE`.**
Qué pasa: una pantalla de búsqueda que siempre recorre la tabla, con cualquier filtro.
Por qué: un solo plan para todas las combinaciones de parámetros.
Arreglo: `OPTION (RECOMPILE)` o SQL dinámico parametrizado.

**5. Arreglar con un índice lo que es un problema de la consulta.**
Qué pasa: se crea un índice sobre `YEAR(FechaPedido)`... que no se usa, porque la consulta usa `MONTH()`.
Por qué: se trata el síntoma.
Arreglo: primero hacer la consulta SARGable; después, si hace falta, el índice.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Indexar una expresión | Columna calculada + índice | **Índice de expresión directo**: `CREATE INDEX ON pedido ((extract(year from fecha_pedido)))` | Índice funcional (desde 8.0.13): `CREATE INDEX ... ((YEAR(fecha_pedido)))` |
| Búsqueda "contiene" | Full-Text Search | Índices `pg_trgm` (trigramas) con GIN, que **sí** aceleran `LIKE '%x%'` | `FULLTEXT` |
| `LIKE` sin distinguir mayúsculas | Según la intercalación | `ILIKE` (no SARGable con un índice común; sí con `lower()` indexado) | Según la intercalación |
| Conversión implícita en el plan | Advertencia `CONVERT_IMPLICIT` | Se ve en el `EXPLAIN` como una conversión en el filtro | `EXPLAIN` + `SHOW WARNINGS` |

PostgreSQL es especialmente flexible aquí: un índice sobre `lower(email)` hace SARGable `WHERE lower(email) = 'x'`, y los índices de trigramas resuelven el "contiene" sin un motor de texto completo aparte.

### En MongoDB

Los mismos principios aplican: un filtro con `$regex` anclado al principio (`/^cliente5/`) puede usar un índice; uno sin ancla (`/correo/`) recorre todas las claves del índice o la colección. Las consultas con `$expr` que aplican funciones a los campos (`{ $expr: { $eq: [{ $year: "$fechaPedido" }, 2026] } }`) no pueden usar el índice como una búsqueda por rango; la versión SARGable es `{ fechaPedido: { $gte: ISODate("2026-01-01"), $lt: ISODate("2027-01-01") } }`. MongoDB no tiene conversión implícita entre tipos: un `"12345"` (texto) y un `12345` (número) son valores distintos, y la consulta simplemente no encuentra nada.

-----

## Cuándo sí y cuándo no

**Haz SARGable una consulta cuando:**

* Filtra o une tablas grandes por una columna con índice: siempre.

**No te preocupes tanto cuando:**

* La tabla es chica, o la función está en el `SELECT` (calcular `YEAR()` para mostrarlo no impide nada).

**Ten en cuenta:**

* El checklist de tu cheat sheet es una excelente rutina de revisión: tipos de los parámetros, funciones en el `WHERE`, `LIKE` con comodín inicial, rangos de fechas en lugar de funciones, *seek* o *scan* en el plan, advertencias de conversión y lecturas lógicas.

-----

## Resumen en 5 líneas

1. SARGable = la columna sola de un lado de la comparación; `función(columna)` obliga a recorrer el índice.
2. Reescribe aplicando la función inversa al valor, y verifica que el resultado sea idéntico (sobre todo con `NULL` y fechas).
3. Conversión implícita: el problema es cuando se convierte la columna; iguala el tipo del parámetro al de la columna.
4. `LIKE 'x%'` sí, `LIKE '%x'` no; `OR` entre columnas distintas, `UNION`; consultas "para todo", `RECOMPILE` o SQL dinámico.
5. Si la expresión es inevitable, una columna calculada indexada la vuelve SARGable.

-----

## Para profundizar

<details>
<summary>Verificar que dos versiones de una consulta son equivalentes</summary>

Antes de reemplazar una consulta por su versión SARGable en producción, compara los resultados:

```sql
-- Filas que devuelve la original y no la nueva, y al revés
(SELECT PedidoId FROM dbo.Pedido WHERE YEAR(FechaPedido) = 2026
 EXCEPT
 SELECT PedidoId FROM dbo.Pedido WHERE FechaPedido >= '2026-01-01' AND FechaPedido < '2027-01-01')
UNION ALL
(SELECT PedidoId FROM dbo.Pedido WHERE FechaPedido >= '2026-01-01' AND FechaPedido < '2027-01-01'
 EXCEPT
 SELECT PedidoId FROM dbo.Pedido WHERE YEAR(FechaPedido) = 2026);
```

Si no devuelve filas, son equivalentes en los datos actuales. Para casos con `NULL`, agrega filas de prueba con nulos y límites (el 31 de diciembre a las 23:59:59, el 1 de enero a las 00:00:00) antes de comparar.

</details>

<details>
<summary>Las funciones escalares y el *inlining*</summary>

Antes de SQL Server 2019, una función escalar propia en una consulta se ejecutaba una vez por fila, en un contexto separado, el plan la mostraba como un costo casi nulo (mintiendo sobre su costo real) e impedía el paralelismo de toda la consulta. Desde 2019, con el nivel de compatibilidad 150, el *scalar UDF inlining* convierte muchas funciones escalares en expresiones dentro del plan, como si se hubieran escrito en línea. No todas son *inlineables* (por ejemplo, las que usan `GETDATE()` o bucles); `sys.sql_modules.is_inlineable` lo indica.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una consulta SARGable es la que permite usar un índice para buscar. Si aplico una función a la columna, como `YEAR(Fecha) = 2026`, el índice no se puede usar para buscar y se recorre toda la tabla. Lo correcto es escribir un rango: `Fecha >= '2026-01-01' AND Fecha < '2027-01-01'`.

### Respuesta ampliada (semi-senior)

Para que un índice se use con un *seek*, la columna tiene que quedar sin transformar de un lado de la comparación. Reescribo las funciones sobre columnas aplicando la inversa al valor (rangos de fechas en lugar de `YEAR` o `CAST`, prefijos con `LIKE` en lugar de `LEFT`) y verifico la equivalencia con `EXCEPT`, porque es fácil cambiar el resultado con `NULL` o con los límites de las fechas. Reviso las conversiones implícitas, que solo son un problema cuando se convierte la columna, típicamente un parámetro `NVARCHAR` del ORM contra una columna `VARCHAR`. Para `OR` entre columnas distintas uso `UNION`, para las búsquedas con filtros opcionales `OPTION (RECOMPILE)` o SQL dinámico parametrizado, y si una expresión es inevitable, la materializo como columna calculada indexada. También busco los antipatrones de la aplicación, como el N+1. Y el orden es siempre el mismo: primero la consulta SARGable, después el índice.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `WHERE YEAR(Fecha) = 2026` no usa el índice?**
Porque el índice está ordenado por `Fecha`, no por `YEAR(Fecha)`; para evaluar la función, el motor tiene que calcularla en cada fila.

**2. ¿Cuándo una conversión implícita impide el uso de un índice?**
Cuando se convierte la columna, porque su tipo tiene menor precedencia que el del valor; por ejemplo, una columna `VARCHAR` comparada con un parámetro `NVARCHAR`.

**3. ¿Qué es una consulta "para todo" y cómo la optimizas?**
Una consulta con filtros opcionales del tipo `@p IS NULL OR Col = @p`; se optimiza con `OPTION (RECOMPILE)` o con SQL dinámico parametrizado.

**4. ¿Qué es el problema N+1?**
Ejecutar una consulta por cada fila de un resultado anterior (por ejemplo, los pedidos de cada cliente, uno por uno), en lugar de una sola consulta con un join.

-----

## Práctica

**Ejercicio 1.** Reescribe de forma SARGable y equivalente:

```sql
-- a
WHERE DATEDIFF(day, FechaPedido, '2026-06-30') <= 30
-- b
WHERE CONVERT(VARCHAR(7), FechaPedido, 120) = '2026-05'
-- c
WHERE ISNULL(Ciudad, N'Sin ciudad') <> N'Lima'
-- d
WHERE Precio + 10 > 100
```

<details>
<summary>Solución</summary>

```sql
-- a
WHERE FechaPedido >= '2026-05-31'

-- b: mayo de 2026
WHERE FechaPedido >= '2026-05-01' AND FechaPedido < '2026-06-01'

-- c: los que no son de Lima, incluidos los nulos
WHERE Ciudad <> N'Lima' OR Ciudad IS NULL

-- d
WHERE Precio > 90
```

* **a:** `DATEDIFF(day, x, '2026-06-30')` cuenta límites de día: el 31 de mayo, a cualquier hora, da 30, así que entra. Y las fechas **posteriores** al 30 de junio dan un número negativo, que también es `<= 30`: la condición original no tiene límite superior. Por eso la equivalencia exacta es `>= '2026-05-31'`, sin `< '2026-07-01'`. Si la intención era "solo hasta el 30 de junio", la consulta original tenía un error, y la reescritura es el momento de descubrirlo. Detalles así son los que hay que verificar con `EXCEPT`.
* **c:** `ISNULL(Ciudad, 'Sin ciudad') <> 'Lima'` es verdadero para los nulos (porque 'Sin ciudad' no es 'Lima'). La reescritura tiene que incluirlos explícitamente. Ojo: un `<>` rara vez es selectivo, así que aun siendo SARGable, el motor probablemente recorra la tabla. La ventaja aquí es de claridad y estimación, más que de *seek*.

</details>

**Ejercicio 2.** Encuentra los antipatrones de esta consulta de una pantalla de búsqueda y propón una versión mejor.

```sql
CREATE PROCEDURE dbo.BuscarClientes @Nombre NVARCHAR(100) = NULL, @Ciudad NVARCHAR(80) = NULL
AS
SELECT *
FROM dbo.Cliente
WHERE (@Nombre IS NULL OR UPPER(Nombre) LIKE '%' + UPPER(@Nombre) + '%')
  AND (@Ciudad IS NULL OR ISNULL(Ciudad, N'') = @Ciudad);
```

<details>
<summary>Solución</summary>

Antipatrones:

1. **`SELECT *`:** trae todas las columnas e impide cubrir la consulta.
2. **Consulta "para todo"** sin `RECOMPILE`: un solo plan para cualquier combinación de filtros.
3. **`UPPER(Nombre)`:** innecesario con una intercalación insensible a mayúsculas, y no SARGable.
4. **`LIKE '%...%'`:** el comodín inicial obliga a recorrer todo el índice o la tabla.
5. **`ISNULL(Ciudad, N'') = @Ciudad`:** no SARGable y equivalente a `Ciudad = @Ciudad` (si `@Ciudad` no está vacío).

Versión mejorada:

```sql
CREATE OR ALTER PROCEDURE dbo.BuscarClientes @Nombre NVARCHAR(100) = NULL, @Ciudad NVARCHAR(80) = NULL
AS
SELECT ClienteId, Nombre, Email, Ciudad
FROM dbo.Cliente
WHERE (@Nombre IS NULL OR Nombre LIKE @Nombre + N'%')     -- búsqueda por prefijo
  AND (@Ciudad IS NULL OR Ciudad = @Ciudad)
OPTION (RECOMPILE);
```

Cambiar "contiene" por "empieza con" es una **decisión de producto**, no solo técnica: hay que acordarla. Si el negocio necesita "contiene" sobre una tabla grande, la respuesta es Full-Text Search. Con `RECOMPILE`, cada combinación de filtros obtiene su plan, y un índice sobre `(Nombre)` y otro sobre `(Ciudad)` pueden usarse cuando corresponde.

</details>

-----

## Siguiente lección

[Mantenimiento de índices](07-Mantenimiento%20de%20índices.md)
