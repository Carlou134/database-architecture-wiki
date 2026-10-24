# Tablas temporales y variables de tabla

## En una frase

Las tablas temporales (`#Tabla`), las variables de tabla (`@Tabla`) y los parámetros con valor de tabla (TVP) guardan resultados intermedios en `tempdb` durante una sesión o un lote; se diferencian en el **alcance**, en cómo se comportan ante un `ROLLBACK` y, sobre todo, en que las tablas temporales tienen **estadísticas** y las variables de tabla no, lo que decide cuál conviene según la cantidad de filas.

-----

## Antes de empezar

Conviene que ya sepas:

* CTE y por qué no se materializan, de [CTE](../03-consultas/09-CTE.md).
* Estadísticas y estimación de filas, de [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md).
* Procedimientos almacenados, de [Procedimientos almacenados](02-Procedimientos%20almacenados.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`tempdb`:** base del sistema donde viven los objetos temporales, las versiones de filas y las operaciones intermedias de las consultas.
* **Tabla temporal local (`#Tabla`):** tabla visible solo para la sesión (y los procedimientos que llama) que la creó.
* **Tabla temporal global (`##Tabla`):** tabla visible para todas las sesiones.
* **Variable de tabla (`@Tabla`):** variable con forma de tabla, visible solo en el lote o procedimiento donde se declara.
* **Parámetro con valor de tabla (TVP):** parámetro de un procedimiento cuyo valor es una tabla, de un tipo definido con `CREATE TYPE ... AS TABLE`.
* **Compilación diferida de variables de tabla:** mejora de SQL Server 2019 que compila la consulta con la cantidad real de filas de la variable.

-----

## El problema

Un reporte mensual de `TiendaGrande` encadena cuatro pasos: los pedidos del mes, sus totales, los clientes de esos pedidos y el ranking. Escrito como una sola consulta con cinco CTE anidadas, tarda mucho, y el plan muestra que algunas CTE se evalúan **varias veces** (como se vio en [CTE](../03-consultas/09-CTE.md), SQL Server no las materializa).

Alguien lo reescribe guardando cada paso en una **variable de tabla**, y en SQL Server 2017 tarda **más**: el optimizador estima que cada variable tiene **una** fila, elige *Nested Loops* para unir 25.000 pedidos y los ejecuta 25.000 veces.

Con **tablas temporales**, el mismo reporte tarda una fracción: cada paso se calcula una vez, y el optimizador sabe cuántas filas tiene cada tabla intermedia.

La diferencia entre las tres versiones no está en la lógica, sino en dónde se guardan los resultados intermedios y qué sabe el optimizador sobre ellos.

-----

## Cómo funciona

### Las tres opciones, comparadas

| Característica | `#Tabla` (temporal local) | `##Tabla` (temporal global) | `@Tabla` (variable de tabla) |
| --- | --- | --- | --- |
| Dónde vive | `tempdb` | `tempdb` | **`tempdb`** (no "en memoria") |
| Visible para | La sesión y los procedimientos que llama | Todas las sesiones | Solo el lote o procedimiento donde se declara |
| Se elimina | Al terminar la sesión, o el procedimiento que la creó | Cuando termina la sesión creadora y nadie más la usa | Al terminar el lote |
| **Estadísticas** | **Sí** | Sí | **No** (ver abajo) |
| Índices | Todos, también después de crearla | Todos | Solo los declarados en la definición |
| `ALTER TABLE` | Sí | Sí | No |
| Afectada por `ROLLBACK` | Sí | Sí | **No** |
| Provoca recompilaciones | Puede (cambios de estadísticas, DDL) | Puede | No |

Dos correcciones sobre tu cheat sheet de la sesión 11:

* **Las variables de tabla no viven "en memoria"**: ocupan `tempdb` igual que las tablas temporales (si son chicas, sus páginas pueden quedarse en el buffer pool, como cualquier otra página).
* **Las variables de tabla no tienen estadísticas**, ni siquiera desde SQL Server 2019. Lo que agregó 2019 es la **compilación diferida**, que es otra cosa (ver más abajo).

### Tablas temporales locales

```sql
DROP TABLE IF EXISTS #PedidosMes;   -- SQL Server 2016+

SELECT PedidoId, ClienteId, FechaPedido
INTO #PedidosMes
FROM dbo.Pedido
WHERE FechaPedido >= '2026-03-01' AND FechaPedido < '2026-04-01';

CREATE CLUSTERED INDEX CIX_PedidosMes ON #PedidosMes (PedidoId);
```

* **`SELECT ... INTO #Tabla`** crea la tabla con las columnas del resultado. Es la forma más rápida de cargarla (y puede tener registro mínimo).
* **Índices después de cargar:** en cargas grandes, crear el índice al final es más barato que mantenerlo fila por fila.
* **Estadísticas automáticas:** el optimizador sabe cuántas filas tiene y cómo se distribuyen, igual que con una tabla normal.

Alcance: una `#Tabla` creada en la sesión es visible en todos los procedimientos que esa sesión llama. Una creada **dentro** de un procedimiento es visible en los procedimientos que él llama, y desaparece cuando ese procedimiento termina:

```text
Sesión
 └─ crea #A ─────────────► visible aquí y en todo lo que se llame desde aquí
 └─ EXEC dbo.Padre
       └─ crea #B ───────► visible en dbo.Padre y en dbo.Hijo
       └─ EXEC dbo.Hijo  (ve #A y #B)
 └─ (de vuelta en la sesión: #B ya no existe; #A sí)
```

### Variables de tabla

```sql
DECLARE @Clientes TABLE (
    ClienteId INT NOT NULL PRIMARY KEY,
    Pedidos   INT NOT NULL,
    INDEX IX_Pedidos NONCLUSTERED (Pedidos)   -- índices en la declaración (desde 2014)
);

INSERT INTO @Clientes (ClienteId, Pedidos)
SELECT ClienteId, COUNT(*) FROM #PedidosMes GROUP BY ClienteId;
```

**El problema de las estimaciones.** Como no tienen estadísticas, el optimizador no sabe cuántas filas hay:

| Versión / compatibilidad | Estimación de filas de una variable de tabla |
| --- | --- |
| Hasta SQL Server 2017 (o compatibilidad < 150) | **1 fila**, siempre (salvo con `OPTION (RECOMPILE)`) |
| SQL Server 2019+ con compatibilidad ≥ 150 | La cantidad **real** de filas al momento de la primera compilación (*compilación diferida*) |

Con 2019, el optimizador sabe **cuántas** filas hay, pero sigue sin un **histograma**: no sabe cómo se distribuyen los valores. Para un join por una columna con valores muy desiguales, una tabla temporal sigue estimando mejor.

**Inmunes al `ROLLBACK`.** Los datos de una variable de tabla no se deshacen con la transacción. Es el truco del "log inmune al rollback" de tus apuntes: registrar los pasos de un proceso en una variable de tabla, hacer el `ROLLBACK` y después insertar el registro en una tabla permanente (ver [Manejo de errores](../05-transacciones-y-concurrencia/06-Manejo%20de%20errores.md)).

### Parámetros con valor de tabla (TVP)

Para pasarle a un procedimiento una **lista** de valores (los productos de un pedido, una lista de IDs), en lugar de un texto separado por comas:

```sql
CREATE TYPE dbo.TipoLineaPedido AS TABLE (
    ProductoId INT NOT NULL PRIMARY KEY,
    Cantidad   INT NOT NULL CHECK (Cantidad > 0)
);
GO

CREATE OR ALTER PROCEDURE dbo.RegistrarPedidoCompleto
    @ClienteId INT,
    @Lineas    dbo.TipoLineaPedido READONLY      -- READONLY es obligatorio
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (@ClienteId, 'Pagado');
        DECLARE @PedidoId INT = SCOPE_IDENTITY();

        INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
        SELECT @PedidoId, l.ProductoId, l.Cantidad, p.Precio
        FROM @Lineas AS l
        JOIN dbo.Producto AS p ON p.ProductoId = l.ProductoId;
    COMMIT;
END;
GO

DECLARE @l dbo.TipoLineaPedido;
INSERT INTO @l VALUES (1, 1), (2, 2);
EXEC dbo.RegistrarPedidoCompleto @ClienteId = 5, @Lineas = @l;
```

Un pedido con todas sus líneas en **una** llamada, con tipos y restricciones validados por el motor. Desde .NET, se pasa un `DataTable` (o una lista de `SqlDataRecord`) con `SqlDbType.Structured`. Un TVP se comporta como una variable de tabla, con las mismas limitaciones de estimación.

### Cuál usar

| Situación | Opción |
| --- | --- |
| Resultado intermedio de unas pocas filas, usado una vez | CTE, o variable de tabla |
| Unas pocas decenas o cientos de filas, sin joins delicados | Variable de tabla (o temporal: cualquiera sirve) |
| Miles de filas o más, usadas en joins o varias veces | **Tabla temporal** |
| El resultado se usa en procedimientos llamados | Tabla temporal (la variable no es visible en ellos) |
| Datos que deben sobrevivir a un `ROLLBACK` | Variable de tabla |
| Pasar una lista de valores a un procedimiento | TVP |
| Compartir datos entre sesiones | `##Tabla` (rara vez: una tabla permanente de trabajo suele ser más clara) |

### Cuidar `tempdb`

`tempdb` es compartida por **todas** las bases del servidor: tablas temporales, variables de tabla, versiones de filas (RCSI), ordenamientos que no entran en memoria. Una tabla temporal de 20 GB creada por un reporte afecta a todo el servidor. Buenas prácticas:

* Cargar en las tablas temporales solo las columnas y filas necesarias.
* Borrarlas explícitamente (`DROP TABLE`) cuando ya no hacen falta en procesos largos, en lugar de esperar al final de la sesión.
* Monitorear su espacio (`sys.dm_db_session_space_usage` muestra cuánto usa cada sesión).

-----

## Ejemplo completo

El reporte mensual del problema en `TiendaGrande`: los 10 clientes que más compraron en marzo de 2026, con su total y su cantidad de pedidos.

```sql
USE TiendaGrande;
SET STATISTICS IO, TIME ON;

DROP TABLE IF EXISTS #PedidosMes, #TotalesPedido;

-- 1. Los pedidos del mes, sin cancelados
SELECT PedidoId, ClienteId
INTO #PedidosMes
FROM dbo.Pedido
WHERE FechaPedido >= '2026-03-01' AND FechaPedido < '2026-04-01'
  AND Estado <> 'Cancelado';

CREATE CLUSTERED INDEX CIX_PedidosMes ON #PedidosMes (PedidoId);

-- 2. El total de cada pedido
SELECT d.PedidoId, SUM(d.Cantidad * d.PrecioUnitario) AS Total
INTO #TotalesPedido
FROM dbo.DetallePedido AS d
JOIN #PedidosMes AS pm ON pm.PedidoId = d.PedidoId
GROUP BY d.PedidoId;

-- 3. El ranking de clientes
SELECT TOP (10) c.ClienteId, c.Nombre,
       COUNT(*)      AS Pedidos,
       SUM(t.Total)  AS Comprado
FROM #PedidosMes AS pm
JOIN #TotalesPedido AS t ON t.PedidoId = pm.PedidoId
JOIN dbo.Cliente AS c ON c.ClienteId = pm.ClienteId
GROUP BY c.ClienteId, c.Nombre
ORDER BY Comprado DESC;

SET STATISTICS IO, TIME OFF;
```

Comparación con la misma lógica guardando los pasos en variables de tabla:

| Versión | Estimación de filas del paso 1 | Join elegido con `DetallePedido` | Comportamiento |
| --- | --- | --- | --- |
| `#Tabla` | ~24.000 (real) | *Hash Match* o *Merge Join* | Cada paso una vez, con buenas estimaciones |
| `@Tabla`, compatibilidad < 150 | **1** | *Nested Loops*, ejecutado ~24.000 veces | Mucho más lenta |
| `@Tabla`, compatibilidad ≥ 150 | ~24.000 (compilación diferida) | Similar a la temporal | Aceptable; sin histograma |

Qué decisión hay detrás:

* **Cada paso se calcula una vez** y se reutiliza: a diferencia de una CTE referenciada varias veces.
* **Las tablas temporales tienen estadísticas,** así que cada join se planifica con la cantidad real de filas.
* **El índice agrupado en `#PedidosMes`** después de cargarla favorece los joins por `PedidoId`.
* **`DROP TABLE IF EXISTS` al principio** permite volver a ejecutar el script en la misma sesión sin errores.
* **Para un reporte que se ejecuta seguido,** este patrón va dentro de un procedimiento, donde las tablas temporales desaparecen solas al terminar.

-----

## Errores comunes

**1. Variables de tabla con miles de filas en joins (antes de 2019).**
Qué pasa: estimación de 1 fila, *Nested Loops* que se ejecutan miles de veces.
Por qué: las variables de tabla no tienen estadísticas.
Arreglo: tabla temporal, `OPTION (RECOMPILE)` en la consulta, o compatibilidad 150+.

**2. Creer que las variables de tabla viven en memoria.**
Qué pasa: se eligen "por rendimiento" para volúmenes grandes.
Por qué: un mito muy extendido.
Arreglo: las dos viven en `tempdb`; se elige por estadísticas, alcance y comportamiento ante el `ROLLBACK`.

**3. Pasar listas como texto separado por comas.**
Qué pasa: `WHERE ProductoId IN (SELECT value FROM STRING_SPLIT(@Lista, ','))` sin validación de tipos, con estimaciones malas y riesgo de errores de conversión.
Por qué: es lo más fácil de enviar desde la aplicación.
Arreglo: un TVP.

**4. Olvidar el alcance de las tablas temporales.**
Qué pasa: `Invalid object name '#Paso1'` en un procedimiento, o en la sesión después de que terminó el procedimiento que la creó.
Por qué: la tabla desaparece al terminar el procedimiento que la creó.
Arreglo: crearla en el nivel que la necesita, o pasar los datos como TVP.

**5. Abusar de `tempdb`.**
Qué pasa: `tempdb` llena y todo el servidor afectado.
Por qué: tablas temporales con todas las columnas y filas "por si acaso", que nunca se borran en procesos largos.
Arreglo: solo lo necesario, y `DROP TABLE` cuando ya no se usa.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Tabla temporal | `#Tabla` (`CREATE TABLE #t` o `SELECT ... INTO #t`) | `CREATE TEMP TABLE t ...` | `CREATE TEMPORARY TABLE t ...` |
| Visible para | La sesión y los procedimientos llamados | La sesión | La sesión |
| Al terminar la transacción | Persiste | Configurable: `ON COMMIT DROP` / `DELETE ROWS` / `PRESERVE ROWS` | Persiste |
| Variable de tabla | `DECLARE @t TABLE (...)` | No (se usan tablas temporales o arreglos) | No |
| Tabla global | `##Tabla` | No | No |
| Pasar una tabla a un procedimiento | TVP | Arreglos o tipos compuestos (`int[]`) | JSON o tablas temporales |
| Estadísticas de la temporal | Automáticas | Hay que ejecutar `ANALYZE` (*autovacuum* no procesa las temporales) | Automáticas |

En PostgreSQL, el detalle de las estadísticas es importante: una tabla temporal grande sin `ANALYZE` deja al optimizador sin información, el mismo problema que las variables de tabla de SQL Server.

### En MongoDB

No hay tablas temporales: los resultados intermedios viven dentro del *aggregation pipeline*, que pasa los documentos de una etapa a la siguiente. Para materializar un resultado intermedio y reutilizarlo, se escribe en una colección con `$out` o `$merge`, y se borra después; es el equivalente de una tabla de trabajo permanente.

-----

## Cuándo sí y cuándo no

**Usa tablas temporales cuando:**

* Un proceso de varios pasos reutiliza resultados intermedios grandes, o los une con otras tablas.

**Usa variables de tabla cuando:**

* El conjunto es chico, o necesitas que sobreviva a un `ROLLBACK`.

**Usa un TVP cuando:**

* Una aplicación tiene que pasar una lista de filas a un procedimiento.

**No uses ninguna cuando:**

* Una sola consulta (con una CTE que se usa una vez) resuelve el problema: materializar tiene un costo.

-----

## Resumen en 5 líneas

1. `#Tabla`, `##Tabla` y `@Tabla` viven todas en `tempdb`; difieren en alcance, estadísticas y comportamiento ante el `ROLLBACK`.
2. Las tablas temporales tienen estadísticas e índices completos: son la opción para volúmenes grandes y joins.
3. Las variables de tabla no tienen estadísticas: antes de 2019 se estiman en 1 fila; desde 2019, con la cantidad real, pero sin histograma.
4. Los datos de una variable de tabla sobreviven al `ROLLBACK`; los de una temporal, no.
5. Para pasar listas a un procedimiento, TVP; y cuida `tempdb`, que es compartida por todo el servidor.

-----

## Para profundizar

<details>
<summary>Caché de tablas temporales</summary>

Cuando un procedimiento crea una tabla temporal, SQL Server puede **reutilizarla** en la siguiente ejecución en lugar de crearla y borrarla cada vez, lo que reduce la contención en las páginas de asignación de `tempdb`. Ciertas operaciones impiden esa caché: crear índices con nombre después de crear la tabla (`CREATE INDEX` aparte), usar `ALTER TABLE` o crear restricciones con nombre. En procedimientos que se ejecutan miles de veces por segundo, conviene declarar los índices **dentro** del `CREATE TABLE` (`INDEX IX_... NONCLUSTERED (...)`) para que la tabla se pueda reutilizar.

</details>

<details>
<summary>La contención en tempdb</summary>

Con muchas sesiones creando objetos temporales a la vez, pueden aparecer esperas `PAGELATCH_UP` sobre las páginas de asignación de `tempdb` (PFS, GAM, SGAM). La solución clásica es tener varios archivos de datos de `tempdb` del mismo tamaño (el instalador de SQL Server lo configura desde 2016, uno por núcleo hasta 8), y desde SQL Server 2019 existe la opción de metadatos de `tempdb` optimizados para memoria (`ALTER SERVER CONFIGURATION SET MEMORY_OPTIMIZED TEMPDB_METADATA = ON`).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las tablas temporales (`#Tabla`) y las variables de tabla (`@Tabla`) guardan resultados intermedios. Las temporales tienen estadísticas e índices y se usan para muchas filas; las variables de tabla son más simples y se usan para pocas filas. Las dos viven en `tempdb`.

### Respuesta ampliada (semi-senior)

Elijo según volumen, alcance y comportamiento transaccional. Las tablas temporales tienen estadísticas automáticas e índices completos, son visibles en los procedimientos llamados y las uso para resultados intermedios grandes o que se unen con otras tablas, creando los índices después de cargarlas. Las variables de tabla no tienen estadísticas: antes de SQL Server 2019 se estiman en una fila, lo que produce *Nested Loops* desastrosos con volumen; desde 2019, la compilación diferida usa la cantidad real de filas, pero sigue sin histograma. Las uso para conjuntos chicos o cuando necesito datos que sobrevivan a un `ROLLBACK`, como un registro de pasos. Las dos viven en `tempdb`, no en memoria. Para pasar listas a un procedimiento uso TVP en lugar de textos separados por comas, y cuido `tempdb` porque la comparte todo el servidor.

### Preguntas frecuentes de seguimiento

**1. ¿Las variables de tabla viven en memoria?**
No: igual que las tablas temporales, viven en `tempdb`.

**2. ¿Qué cambió en SQL Server 2019 con las variables de tabla?**
La compilación diferida: la consulta se compila con la cantidad real de filas de la variable, en lugar de suponer una. Siguen sin estadísticas.

**3. ¿Qué pasa con los datos de una variable de tabla ante un `ROLLBACK`?**
Se conservan: las variables de tabla no participan de la transacción.

**4. ¿Cómo pasas una lista de valores a un procedimiento?**
Con un parámetro con valor de tabla (TVP), de un tipo creado con `CREATE TYPE ... AS TABLE` y declarado `READONLY`.

-----

## Práctica

**Ejercicio 1.** ¿Qué devuelve el último `SELECT`, y por qué?

```sql
CREATE TABLE #t (Id INT);
DECLARE @t TABLE (Id INT);

BEGIN TRANSACTION;
    INSERT INTO #t VALUES (1);
    INSERT INTO @t VALUES (1);
ROLLBACK;

SELECT (SELECT COUNT(*) FROM #t) AS Temporal, (SELECT COUNT(*) FROM @t) AS Variable;
```

<details>
<summary>Solución</summary>

| Temporal | Variable |
| --- | --- |
| 0 | 1 |

El `ROLLBACK` deshace el `INSERT` en la tabla temporal (participa de la transacción como cualquier tabla), pero no el de la variable de tabla, que no participa de las transacciones. Es exactamente lo que permite usar una variable de tabla como registro de pasos que sobrevive a un error.

</details>

**Ejercicio 2.** Crea un TVP `dbo.TipoListaIds` y un procedimiento `dbo.ProductosPorIds` que reciba la lista y devuelva nombre y precio de esos productos. Llámalo con los productos 1, 3 y 99.

<details>
<summary>Solución</summary>

```sql
CREATE TYPE dbo.TipoListaIds AS TABLE (Id INT NOT NULL PRIMARY KEY);
GO

CREATE OR ALTER PROCEDURE dbo.ProductosPorIds @Ids dbo.TipoListaIds READONLY
AS
BEGIN
    SET NOCOUNT ON;
    SELECT p.ProductoId, p.Nombre, p.Precio
    FROM dbo.Producto AS p
    JOIN @Ids AS i ON i.Id = p.ProductoId
    ORDER BY p.ProductoId;
END;
GO

DECLARE @lista dbo.TipoListaIds;
INSERT INTO @lista VALUES (1), (3), (99);
EXEC dbo.ProductosPorIds @Ids = @lista;
```

| ProductoId | Nombre | Precio |
| --- | --- | --- |
| 1 | Teclado mecánico | 120.00 |
| 3 | Monitor 27 pulgadas | 899.90 |

El 99 no existe y simplemente no aparece (es un `INNER JOIN`). La `PRIMARY KEY` del tipo impide IDs repetidos en la lista, y el tipo `INT` impide que llegue un texto: dos validaciones que un `STRING_SPLIT` no hace.

</details>

-----

## Siguiente lección

[Cursores frente a operaciones sobre conjuntos](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md)
