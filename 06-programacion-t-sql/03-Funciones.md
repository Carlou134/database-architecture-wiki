# Funciones

## En una frase

Una función definida por el usuario recibe parámetros y **devuelve** un valor (escalar) o una tabla, y se usa **dentro** de una consulta; la función de tabla **en línea** es la mejor de las tres (el optimizador la expande como si fuera una vista con parámetros), mientras que las escalares y las de varias instrucciones pueden ocultar costos enormes porque se evalúan fila por fila o con estimaciones a ciegas.

-----

## Antes de empezar

Conviene que ya sepas:

* `CROSS APPLY` y `OUTER APPLY`, de [JOIN](../03-consultas/06-JOIN.md).
* Vistas, de [Vistas](01-Vistas.md).
* Por qué una función sobre una columna impide un *seek*, de [Consultas SARGables y antipatrones](../04-indices-y-rendimiento/06-Consultas%20SARGables%20y%20antipatrones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Función definida por el usuario (UDF):** función propia, creada con `CREATE FUNCTION`.
* **Función escalar:** devuelve un solo valor (un número, un texto).
* **Función de tabla en línea (inline TVF):** devuelve una tabla definida por **una sola** consulta `RETURN (SELECT ...)`.
* **Función de tabla de varias instrucciones (multi-statement TVF):** devuelve una tabla que se llena con varias instrucciones.
* **Función determinista:** devuelve siempre el mismo resultado para los mismos parámetros.
* **Inlining:** reemplazar la llamada a la función por su definición dentro de la consulta, para optimizarlas juntas.

-----

## El problema

Alguien escribió una función para no repetir el cálculo del total de un pedido:

```sql
CREATE FUNCTION dbo.fnTotalPedido (@PedidoId INT)
RETURNS DECIMAL(12,2)
AS
BEGIN
    RETURN (SELECT SUM(Cantidad * PrecioUnitario) FROM dbo.DetallePedido WHERE PedidoId = @PedidoId);
END;
```

Y la usa en el listado de pedidos:

```sql
SELECT PedidoId, FechaPedido, dbo.fnTotalPedido(PedidoId) AS Total
FROM dbo.Pedido
WHERE FechaPedido >= '2026-01-01';
```

En `Tienda`, perfecto. En `TiendaGrande`, con 140.000 pedidos de 2026, la consulta tarda varias veces más que la misma lógica escrita con un `JOIN` y un `GROUP BY`. En las versiones anteriores a SQL Server 2019 (o con un nivel de compatibilidad menor a 150), la función se ejecuta **140.000 veces**, una por fila, cada una con su propia consulta a `DetallePedido`; el plan de ejecución muestra un costo casi nulo para ella (mintiendo), y toda la consulta pierde la posibilidad de ejecutarse en paralelo.

El código quedó más prolijo, y la consulta, mucho más lenta. Las funciones son útiles, pero solo una de sus tres variantes es "gratis" para el optimizador.

-----

## Cómo funciona

### Los tres tipos

| Tipo | Devuelve | Cómo se define | Cómo la trata el optimizador |
| --- | --- | --- | --- |
| **Escalar** | Un valor | `RETURNS tipo` + bloque `BEGIN ... END` | Fila por fila (salvo *inlining*, desde 2019) |
| **De tabla en línea** | Una tabla | `RETURNS TABLE AS RETURN (SELECT ...)` | **Se expande en la consulta**, como una vista con parámetros |
| **De tabla de varias instrucciones** | Una tabla | `RETURNS @t TABLE (...)` + bloque que llena `@t` | Caja negra, con una estimación fija de filas |

### Funciones escalares

```sql
CREATE OR ALTER FUNCTION dbo.fnPrecioConIgv (@Precio DECIMAL(10,2))
RETURNS DECIMAL(12,2)
WITH SCHEMABINDING
AS
BEGIN
    RETURN ROUND(@Precio * 1.18, 2);
END;
GO

SELECT Nombre, Precio, dbo.fnPrecioConIgv(Precio) AS PrecioConIgv
FROM dbo.Producto;
```

Una función escalar que **solo calcula** (sin leer tablas), como esta, es razonable: su costo por fila es pequeño. El problema es la que **lee tablas**, como `fnTotalPedido`: esconde una consulta dentro de cada fila.

Lo que cambió en SQL Server 2019: con el nivel de compatibilidad 150 o superior, el **inlining de funciones escalares** convierte muchas funciones en expresiones dentro del plan, como si se hubieran escrito en línea, y elimina el costo fila por fila. No todas son *inlineables* (por ejemplo, las que usan `GETDATE()`, variables de tabla o bucles); se puede verificar:

```sql
SELECT OBJECT_NAME(object_id) AS Funcion, is_inlineable
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('dbo.fnTotalPedido');
```

Aun con *inlining*, la recomendación sigue siendo la misma: si la función lee tablas, conviene escribirla como función de tabla en línea.

### Funciones de tabla en línea: la buena

```sql
CREATE OR ALTER FUNCTION dbo.fnTotalesPedido (@PedidoId INT)
RETURNS TABLE
AS
RETURN (
    SELECT SUM(Cantidad * PrecioUnitario) AS Total,
           COUNT(*)                       AS Lineas
    FROM dbo.DetallePedido
    WHERE PedidoId = @PedidoId
);
GO

SELECT p.PedidoId, p.FechaPedido, t.Total, t.Lineas
FROM dbo.Pedido AS p
CROSS APPLY dbo.fnTotalesPedido(p.PedidoId) AS t
WHERE p.FechaPedido >= '2026-01-01';
```

* El cuerpo es **una sola** consulta, entre paréntesis, después de `RETURN`.
* El optimizador **reemplaza** la llamada por la consulta, igual que con una vista, y optimiza todo junto: puede convertir el `CROSS APPLY` en un join con agregado, usar índices, paralelizar.
* Se usa en el `FROM`, con `CROSS APPLY` (una fila de la función por cada fila de afuera) o como una tabla con parámetros fijos (`SELECT * FROM dbo.fnPedidosDeCliente(1)`).

Es, literalmente, una **vista con parámetros**, y es la forma recomendada de encapsular lógica que lee tablas.

### Funciones de tabla de varias instrucciones: la caja negra

```sql
CREATE OR ALTER FUNCTION dbo.fnPedidosDeCliente (@ClienteId INT)
RETURNS @Resultado TABLE (PedidoId INT, Total DECIMAL(12,2))
AS
BEGIN
    INSERT INTO @Resultado (PedidoId, Total)
    SELECT p.PedidoId, SUM(d.Cantidad * d.PrecioUnitario)
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.ClienteId = @ClienteId
    GROUP BY p.PedidoId;

    RETURN;
END;
```

Permite lógica de varios pasos (`IF`, varias consultas), pero el optimizador no ve lo que hay adentro:

* Hasta SQL Server 2014 estimaba **1 fila**; desde 2014 (CE 120), **100 filas**, sin importar cuántas devuelva de verdad.
* Desde SQL Server 2017, la *ejecución intercalada* (*interleaved execution*) ejecuta la función antes de optimizar el resto de la consulta y usa la cantidad real de filas, pero solo en algunos casos (consultas de solo lectura, sin que la función esté dentro de un `CROSS APPLY` correlacionado).

Si una función de varias instrucciones se puede reescribir como **en línea** (una sola consulta, con CTE si hace falta), casi siempre conviene.

### Lo que una función no puede hacer

Las funciones son para **calcular**, no para cambiar cosas:

* No pueden modificar tablas permanentes (`INSERT`, `UPDATE`, `DELETE`), salvo la variable de tabla que devuelven.
* No pueden usar `TRY...CATCH`, ni `THROW`, ni ejecutar procedimientos almacenados.
* No pueden usar funciones no deterministas con efectos secundarios, como `NEWID()` o `RAND()` (sí `GETDATE()`, que es no determinista pero sin efectos secundarios).
* No pueden usar tablas temporales (`#tabla`) ni SQL dinámico.

Si necesitas cualquiera de estas cosas, lo que quieres es un **procedimiento**.

### `SCHEMABINDING` en las funciones

Como en las vistas, `WITH SCHEMABINDING` impide cambiar los objetos de los que depende la función. En las funciones escalares que no leen tablas tiene un beneficio extra: SQL Server puede marcarlas como **deterministas y sin acceso a datos**, lo que permite usarlas en columnas calculadas persistidas e índices, y evita cierto trabajo de protección en los planes que modifican datos.

### Funciones en el `WHERE`

```sql
SELECT PedidoId FROM dbo.Pedido WHERE dbo.fnAnio(FechaPedido) = 2026;   -- no SARGable
```

Una función sobre la columna del filtro, propia o del sistema, impide el *seek* (ver [Consultas SARGables](../04-indices-y-rendimiento/06-Consultas%20SARGables%20y%20antipatrones.md)). Con una función propia, además, se suma su costo por fila. La reescritura es la misma: aplicar la lógica al **valor**, no a la columna.

-----

## Ejemplo completo

El listado del problema, en `TiendaGrande`, en tres versiones (crea antes en esa base las funciones `dbo.fnTotalPedido` y `dbo.fnTotalesPedido` de esta lección):

```sql
USE TiendaGrande;
SET STATISTICS IO, TIME ON;

-- 1. Función escalar que lee tablas
SELECT p.PedidoId, p.FechaPedido, dbo.fnTotalPedido(p.PedidoId) AS Total
FROM dbo.Pedido AS p
WHERE p.FechaPedido >= '2026-01-01';

-- 2. Función de tabla en línea con CROSS APPLY
SELECT p.PedidoId, p.FechaPedido, t.Total
FROM dbo.Pedido AS p
CROSS APPLY dbo.fnTotalesPedido(p.PedidoId) AS t
WHERE p.FechaPedido >= '2026-01-01';

-- 3. La consulta escrita a mano, como referencia
SELECT p.PedidoId, p.FechaPedido, SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.Pedido AS p
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
WHERE p.FechaPedido >= '2026-01-01'
GROUP BY p.PedidoId, p.FechaPedido;

SET STATISTICS IO, TIME OFF;
```

| Versión | Qué muestra el plan | Comportamiento |
| --- | --- | --- |
| 1, compatibilidad < 150 | Un *Compute Scalar* con un costo casi nulo; `STATISTICS IO` **no** muestra las lecturas de la función | La función se ejecuta ~140.000 veces; mucho más lenta y sin paralelismo |
| 1, compatibilidad ≥ 150 | La función aparece *inlineada*: el plan incluye el acceso a `DetallePedido` | Mucho mejor, parecida a la versión 3 |
| 2 | El mismo plan que la versión 3 (el optimizador expandió la función) | Igual de rápida que escribir la consulta a mano |
| 3 | Join + agregado | La referencia |

Qué decisión hay detrás:

* **La función en línea da el mismo plan que la consulta escrita a mano:** reutilizar lógica sin pagar nada por ello.
* **La escalar depende de la versión y del nivel de compatibilidad:** el mismo código puede ser aceptable en SQL Server 2022 y desastroso en una base con compatibilidad 130. La en línea funciona bien en todas.
* **La versión 1 engaña al medir:** `STATISTICS IO` no cuenta las lecturas que hace la función por dentro. Para verlas, hay que mirar `sys.dm_exec_function_stats` o Extended Events.

-----

## Errores comunes

**1. Funciones escalares que leen tablas.**
Qué pasa: una consulta por cada fila, escondida en el plan.
Por qué: la función se evalúa fila por fila (sin *inlining*).
Arreglo: reescribirla como función de tabla en línea y usarla con `CROSS APPLY`, o integrar la lógica en la consulta.

**2. Funciones de varias instrucciones donde alcanzaba una en línea.**
Qué pasa: estimaciones fijas (1 o 100 filas) y planes malos con muchas filas.
Por qué: el optimizador no ve el contenido de la función.
Arreglo: una sola consulta con `RETURN (SELECT ...)`, usando CTE si hace falta.

**3. Funciones sobre columnas en el `WHERE`.**
Qué pasa: *scans* en lugar de *seeks*, más el costo de la función por fila.
Por qué: el índice no está ordenado por el resultado de la función.
Arreglo: aplicar la lógica al valor del filtro.

**4. Confiar en `STATISTICS IO` para medir una consulta con funciones escalares.**
Qué pasa: una consulta "barata" que tarda minutos.
Por qué: las lecturas de la función no se cuentan en la consulta que la llama.
Arreglo: medir el tiempo real y revisar `sys.dm_exec_function_stats`.

**5. Querer modificar datos o manejar errores en una función.**
Qué pasa: `Invalid use of a side-effecting operator 'INSERT' within a function`.
Por qué: las funciones no pueden tener efectos secundarios.
Arreglo: un procedimiento almacenado.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Escalares | Sí (*inlining* desde 2019) | Sí; las funciones SQL simples se *inlinean* | Sí |
| Que devuelven tablas | En línea y de varias instrucciones | `RETURNS TABLE` / `RETURNS SETOF`; las de lenguaje `SQL` simples se *inlinean* | **No** |
| Modificar datos dentro | No | **Sí** (las funciones pueden tener efectos secundarios) | No (salvo en casos limitados) |
| Volatilidad | Determinista o no (lo deduce el motor) | Se declara: `IMMUTABLE`, `STABLE`, `VOLATILE` | `DETERMINISTIC` / `NOT DETERMINISTIC` (declarado) |
| Lenguajes | T-SQL (y CLR) | SQL, PL/pgSQL, Python, JavaScript y otros | SQL |

En PostgreSQL, declarar bien la volatilidad (`IMMUTABLE` para una función pura) permite usar la función en índices de expresión, y es la información que el optimizador usa para decidir si puede *inlinearla* o calcularla una sola vez.

### En MongoDB

MongoDB permite funciones de JavaScript en algunas etapas (`$function`, `$accumulator`), pero son lentas y la documentación las desaconseja salvo que no haya alternativa. La forma idiomática es expresar los cálculos con los operadores del *aggregation pipeline*, que el motor ejecuta de forma nativa.

-----

## Cuándo sí y cuándo no

**Usa una función de tabla en línea cuando:**

* Quieres reutilizar una consulta que necesita parámetros (una "vista con parámetros").

**Usa una función escalar cuando:**

* Es un cálculo puro sobre sus parámetros, sin leer tablas (una conversión, un formato de código).

**Evita las funciones de varias instrucciones cuando:**

* La lógica cabe en una sola consulta.

**Usa un procedimiento cuando:**

* Necesitas modificar datos, manejar errores, usar tablas temporales o SQL dinámico.

-----

## Resumen en 5 líneas

1. Hay tres tipos: escalar (un valor), de tabla en línea (una consulta) y de tabla de varias instrucciones (varios pasos).
2. La función en línea se expande en la consulta como una vista con parámetros: es la opción por defecto para reutilizar lógica que lee tablas.
3. Las escalares que leen tablas se ejecutan fila por fila (sin *inlining*, antes de 2019 o con compatibilidad < 150) y ocultan su costo.
4. Las de varias instrucciones son cajas negras con estimaciones fijas de filas.
5. Las funciones no modifican datos ni manejan errores: para eso, procedimientos.

-----

## Para profundizar

<details>
<summary>Medir el costo real de las funciones</summary>

```sql
SELECT OBJECT_NAME(object_id) AS Funcion,
       execution_count,
       total_elapsed_time / 1000 AS TiempoTotalMs,
       total_logical_reads
FROM sys.dm_exec_function_stats
WHERE database_id = DB_ID()
ORDER BY total_elapsed_time DESC;
```

Esta DMV (desde SQL Server 2016) acumula las ejecuciones de las funciones escalares desde que su plan entró en la caché. Una función con millones de ejecuciones y millones de lecturas es la candidata obvia a reescribir, aunque los planes de las consultas que la usan la muestren como "gratis".

</details>

<details>
<summary>Convertir una función escalar en una en línea</summary>

La receta para reemplazar `dbo.fnTotalPedido(PedidoId)` sin cambiar el resultado:

1. Crear la versión en línea, que devuelve una tabla con una columna: `RETURNS TABLE AS RETURN (SELECT SUM(...) AS Total FROM ... WHERE PedidoId = @PedidoId)`.
2. En las consultas, reemplazar `dbo.fnTotalPedido(p.PedidoId)` por `t.Total` y agregar `OUTER APPLY dbo.fnTotalesPedido(p.PedidoId) AS t`.
3. Usar `OUTER APPLY` (no `CROSS`) si la escalar podía devolver `NULL` y la fila debía conservarse; con un agregado sin `GROUP BY` siempre hay una fila, así que en este caso los dos dan lo mismo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una función definida por el usuario recibe parámetros y devuelve un valor o una tabla, y se usa dentro de una consulta. Hay funciones escalares, que devuelven un valor, y funciones de tabla. A diferencia de un procedimiento, una función no puede modificar datos.

### Respuesta ampliada (semi-senior)

Distingo tres tipos con comportamientos muy distintos para el optimizador. La función de tabla en línea, con una sola consulta en el `RETURN`, se expande en la consulta que la usa, igual que una vista, así que es la forma de reutilizar lógica con parámetros sin costo; la uso con `CROSS APPLY` u `OUTER APPLY`. Las escalares que leen tablas se ejecutaban fila por fila, ocultaban su costo en el plan y en `STATISTICS IO` e impedían el paralelismo; desde SQL Server 2019, con compatibilidad 150, muchas se *inlinean*, pero no todas, y verifico `is_inlineable`. Las de varias instrucciones son cajas negras con una estimación fija de 100 filas (1 antes de 2014), mejorada en parte por la ejecución intercalada de 2017. Por eso evito las escalares que leen tablas y las de varias instrucciones cuando la lógica cabe en una consulta, y uso `SCHEMABINDING` en las escalares puras. Y como las funciones no pueden modificar datos ni usar `TRY...CATCH`, todo lo que tenga efectos secundarios va en procedimientos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre una función y un procedimiento?**
La función devuelve un valor o una tabla, se usa dentro de una consulta y no puede modificar datos; el procedimiento se ejecuta con `EXEC`, puede modificar datos, manejar errores y devolver varios resultados.

**2. ¿Por qué las funciones escalares pueden ser lentas?**
Porque sin *inlining* se ejecutan una vez por fila, y si leen tablas, cada ejecución es una consulta; además, su costo no aparece en el plan.

**3. ¿Qué es una función de tabla en línea?**
Una función cuyo cuerpo es una sola consulta; el optimizador la expande en la consulta que la llama, como una vista con parámetros.

**4. ¿Qué cambió en SQL Server 2019 con las funciones escalares?**
El *inlining* de funciones escalares: muchas se convierten en expresiones dentro del plan, eliminando el costo fila por fila.

-----

## Práctica

**Ejercicio 1.** Escribe una función de tabla en línea `dbo.fnVentasCategoria` que reciba un rango de fechas y devuelva, por categoría, el facturado sin pedidos cancelados. Úsala para marzo de 2026 en `Tienda`.

<details>
<summary>Solución</summary>

```sql
CREATE OR ALTER FUNCTION dbo.fnVentasCategoria (@Desde DATE, @Hasta DATE)
RETURNS TABLE
AS
RETURN (
    SELECT c.CategoriaId, c.Nombre AS Categoria,
           SUM(d.Cantidad * d.PrecioUnitario) AS Facturado
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
    JOIN dbo.Categoria AS c ON c.CategoriaId = pr.CategoriaId
    WHERE p.FechaPedido >= @Desde AND p.FechaPedido < @Hasta
      AND p.Estado <> 'Cancelado'
    GROUP BY c.CategoriaId, c.Nombre
);
GO

SELECT Categoria, Facturado
FROM dbo.fnVentasCategoria('2026-03-01', '2026-04-01')
ORDER BY Facturado DESC;
```

| Categoria | Facturado |
| --- | --- |
| Electrónica | 1060.90 |

En marzo solo hubo los pedidos 1 (teclado y mouse) y 2 (monitor), todos de Electrónica: 211.00 + 849.90. El rango semiabierto (`@Hasta` exclusivo) evita perder el último día; la función queda como una vista con parámetros que el optimizador expande.

</details>

**Ejercicio 2.** ¿Por qué esta función no se puede crear, y qué usarías en su lugar?

```sql
CREATE FUNCTION dbo.fnRegistrarVisita (@ClienteId INT)
RETURNS INT
AS
BEGIN
    INSERT INTO dbo.Visita (ClienteId, Fecha) VALUES (@ClienteId, SYSUTCDATETIME());
    RETURN SCOPE_IDENTITY();
END;
```

<details>
<summary>Solución</summary>

Falla con `Invalid use of a side-effecting operator 'INSERT' within a function`: las funciones no pueden modificar tablas permanentes. Una función se puede invocar desde un `SELECT`, una vez por fila, en un orden que decide el optimizador; si pudiera insertar, una simple consulta podría producir una cantidad imprevisible de escrituras.

Lo correcto es un procedimiento con un parámetro `OUTPUT`:

```sql
CREATE OR ALTER PROCEDURE dbo.RegistrarVisita @ClienteId INT, @VisitaId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.Visita (ClienteId, Fecha) VALUES (@ClienteId, SYSUTCDATETIME());
    SET @VisitaId = SCOPE_IDENTITY();
END;
```

</details>

-----

## Siguiente lección

[Triggers](04-Triggers.md)
