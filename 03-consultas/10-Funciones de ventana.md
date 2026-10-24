# Funciones de ventana

## En una frase

Una función de ventana calcula un valor **sobre un grupo de filas relacionadas con la fila actual** (su "ventana") **sin colapsarlas**: a diferencia de `GROUP BY`, que devuelve una fila por grupo, `OVER (...)` conserva cada fila y le agrega el ranking, el acumulado, el total del grupo o el valor de la fila anterior.

-----

## Antes de empezar

Conviene que ya sepas:

* Agregados y `GROUP BY`, de [Agregados y GROUP BY](05-Agregados%20y%20GROUP%20BY.md).
* El orden lógico (las funciones de ventana se evalúan en el `SELECT`), de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).
* CTE, de [CTE](09-CTE.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Función de ventana:** función que se calcula sobre un conjunto de filas definido con `OVER`, sin agruparlas.
* **`OVER`:** cláusula que define la ventana.
* **`PARTITION BY`:** divide las filas en grupos independientes; la ventana no cruza de un grupo a otro.
* **`ORDER BY` de la ventana:** el orden dentro de cada partición; define qué es "anterior", "siguiente" y "acumulado".
* **Marco (frame):** el subconjunto de filas de la partición que entra en el cálculo para la fila actual (`ROWS BETWEEN ...`).
* **Ranking:** funciones que numeran filas: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`.

-----

## El problema

Tres pedidos de información que con lo visto hasta ahora se resuelven mal:

1. **"Cada pedido con el total del cliente al lado."** Con `GROUP BY ClienteId` se obtiene el total, pero se pierde el detalle de cada pedido. Hay que calcular el total en una subconsulta y unirlo de vuelta.
2. **"El ingreso acumulado día a día."** Con una subconsulta correlacionada que suma todos los pedidos anteriores a cada fila: correcta, pero con un costo que crece con cada fila.
3. **"El producto más caro de cada categoría."** Con `TOP (1)` se obtiene el más caro de **todos**, no uno por categoría.

El patrón común: se quiere un valor **calculado sobre un grupo** sin perder **las filas individuales**. Eso es exactamente lo que hacen las funciones de ventana, y por eso son una de las preguntas más frecuentes en entrevistas semi-senior de SQL.

-----

## Cómo funciona

Para los ejemplos, una CTE con el total de cada pedido:

```sql
WITH Pedidos AS (
    SELECT p.PedidoId, p.ClienteId, p.FechaPedido, p.Estado,
           SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    GROUP BY p.PedidoId, p.ClienteId, p.FechaPedido, p.Estado
)
SELECT ... FROM Pedidos ...
```

| PedidoId | ClienteId | FechaPedido | Total |
| --- | --- | --- | --- |
| 1 | 1 | 2026-03-01 10:15 | 211.00 |
| 2 | 2 | 2026-03-05 16:40 | 849.90 |
| 3 | 1 | 2026-04-12 09:05 | 265.00 |
| 4 | 3 | 2026-04-20 18:30 | 210.00 |
| 5 | 4 | 2026-05-02 11:00 | 140.50 |

(En los ejemplos siguientes se omite la CTE y se escribe solo la consulta sobre `Pedidos`.)

### `OVER ()`: un agregado sin agrupar

```sql
SELECT PedidoId, Total,
       SUM(Total) OVER ()                                     AS TotalGeneral,
       CAST(100.0 * Total / SUM(Total) OVER () AS DECIMAL(5,2)) AS PctDelTotal
FROM Pedidos;
```

| PedidoId | Total | TotalGeneral | PctDelTotal |
| --- | --- | --- | --- |
| 1 | 211.00 | 1676.40 | 12.59 |
| 2 | 849.90 | 1676.40 | 50.70 |
| 3 | 265.00 | 1676.40 | 15.81 |
| 4 | 210.00 | 1676.40 | 12.53 |
| 5 | 140.50 | 1676.40 | 8.38 |

`SUM(Total) OVER ()` es la suma de **todas** las filas, repetida en cada una. Las cinco filas se conservan: no hay `GROUP BY`.

### `PARTITION BY`: el agregado de mi grupo

```sql
SELECT PedidoId, ClienteId, Total,
       SUM(Total) OVER (PARTITION BY ClienteId)   AS TotalDelCliente,
       COUNT(*)   OVER (PARTITION BY ClienteId)   AS PedidosDelCliente
FROM Pedidos
ORDER BY ClienteId, PedidoId;
```

| PedidoId | ClienteId | Total | TotalDelCliente | PedidosDelCliente |
| --- | --- | --- | --- | --- |
| 1 | 1 | 211.00 | 476.00 | 2 |
| 3 | 1 | 265.00 | 476.00 | 2 |
| 2 | 2 | 849.90 | 849.90 | 1 |
| 4 | 3 | 210.00 | 210.00 | 1 |
| 5 | 4 | 140.50 | 140.50 | 1 |

`PARTITION BY` es como un `GROUP BY` que no colapsa: cada fila ve el agregado **de su propio grupo**. Es la solución al pedido 1 del problema, sin subconsultas ni joins de vuelta.

### Funciones de ranking

Para numerar filas, la ventana necesita un **orden**:

```sql
SELECT p.Nombre, v.Unidades,
       ROW_NUMBER() OVER (ORDER BY v.Unidades DESC, p.Nombre) AS NumeroFila,
       RANK()       OVER (ORDER BY v.Unidades DESC)           AS Rango,
       DENSE_RANK() OVER (ORDER BY v.Unidades DESC)           AS RangoDenso
FROM (SELECT ProductoId, SUM(Cantidad) AS Unidades
      FROM dbo.DetallePedido GROUP BY ProductoId) AS v
JOIN dbo.Producto AS p ON p.ProductoId = v.ProductoId
ORDER BY NumeroFila;
```

| Nombre | Unidades | NumeroFila | Rango | RangoDenso |
| --- | --- | --- | --- | --- |
| Clean Code | 3 | 1 | 1 | 1 |
| Mouse inalámbrico | 3 | 2 | 1 | 1 |
| Cafetera | 1 | 3 | 3 | 2 |
| Lámpara de escritorio | 1 | 4 | 3 | 2 |
| Monitor 27 pulgadas | 1 | 5 | 3 | 2 |
| Teclado mecánico | 1 | 6 | 3 | 2 |

| Función | Con empates | Ejemplo |
| --- | --- | --- |
| `ROW_NUMBER()` | Numera igual, sin repetir: el desempate es arbitrario si el orden no es único | 1, 2, 3, 4 |
| `RANK()` | Mismo número para los empatados, y **salta** los siguientes | 1, 1, 3, 3 |
| `DENSE_RANK()` | Mismo número para los empatados, **sin saltos** | 1, 1, 2, 2 |
| `NTILE(n)` | Divide las filas en n grupos de tamaño parecido | 1, 1, 2, 2 (con `NTILE(2)`) |

Como con `TOP`, `ROW_NUMBER` necesita un orden determinista para dar siempre el mismo resultado: por eso lleva `p.Nombre` como desempate.

### Top N por grupo

El pedido 3 del problema, "el producto más caro de cada categoría", es el uso más famoso de `ROW_NUMBER`:

```sql
WITH Ranking AS (
    SELECT CategoriaId, Nombre, Precio,
           ROW_NUMBER() OVER (PARTITION BY CategoriaId ORDER BY Precio DESC, ProductoId) AS Puesto
    FROM dbo.Producto
)
SELECT CategoriaId, Nombre, Precio
FROM Ranking
WHERE Puesto = 1;
```

| CategoriaId | Nombre | Precio |
| --- | --- | --- |
| 1 | Monitor 27 pulgadas | 899.90 |
| 2 | Cafetera | 210.00 |
| 3 | Clean Code | 95.00 |

La CTE es **necesaria**: las funciones de ventana se evalúan en el `SELECT`, que va después del `WHERE`, así que no se puede filtrar por ellas directamente:

```text
Msg 4108, Level 15, State 1
Windowed functions can only appear in the SELECT or ORDER BY clauses.
```

Con `WHERE Puesto <= 3`, el mismo patrón da los 3 primeros de cada grupo. Y con `RANK()` en lugar de `ROW_NUMBER()`, conserva los empates del primer puesto.

### `LAG` y `LEAD`: la fila anterior y la siguiente

```sql
SELECT ClienteId, PedidoId, FechaPedido, Total,
       LAG(FechaPedido) OVER (PARTITION BY ClienteId ORDER BY FechaPedido) AS PedidoAnterior,
       DATEDIFF(day,
                LAG(FechaPedido) OVER (PARTITION BY ClienteId ORDER BY FechaPedido),
                FechaPedido)                                               AS DiasDesdeAnterior
FROM Pedidos
ORDER BY ClienteId, FechaPedido;
```

| ClienteId | PedidoId | FechaPedido | Total | PedidoAnterior | DiasDesdeAnterior |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 2026-03-01 10:15 | 211.00 | `NULL` | `NULL` |
| 1 | 3 | 2026-04-12 09:05 | 265.00 | 2026-03-01 10:15 | 42 |
| 2 | 2 | 2026-03-05 16:40 | 849.90 | `NULL` | `NULL` |
| 3 | 4 | 2026-04-20 18:30 | 210.00 | `NULL` | `NULL` |
| 4 | 5 | 2026-05-02 11:00 | 140.50 | `NULL` | `NULL` |

* `LAG(col)` devuelve el valor de la fila **anterior** en el orden de la ventana; `LEAD(col)`, el de la **siguiente**.
* La primera fila de cada partición no tiene anterior: `NULL`. Se puede dar un valor por defecto: `LAG(Total, 1, 0)` (desplazamiento 1, valor por defecto 0).
* `LAG(col, 2)` mira dos filas atrás.

Es la herramienta para "tiempo entre compras", "variación respecto del mes anterior" o "el siguiente estado de un pedido", y para análisis de retención: un cliente cuyo `DiasDesdeAnterior` crece es un cliente que se está yendo.

### Acumulados: `ORDER BY` y el marco

Con `ORDER BY` dentro de `OVER`, un agregado se vuelve **acumulado**:

```sql
SELECT PedidoId, FechaPedido, Total,
       SUM(Total) OVER (ORDER BY FechaPedido
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS Acumulado
FROM Pedidos
ORDER BY FechaPedido;
```

| PedidoId | FechaPedido | Total | Acumulado |
| --- | --- | --- | --- |
| 1 | 2026-03-01 10:15 | 211.00 | 211.00 |
| 2 | 2026-03-05 16:40 | 849.90 | 1060.90 |
| 3 | 2026-04-12 09:05 | 265.00 | 1325.90 |
| 4 | 2026-04-20 18:30 | 210.00 | 1535.90 |
| 5 | 2026-05-02 11:00 | 140.50 | 1676.40 |

El **marco** (`ROWS BETWEEN ... AND ...`) dice qué filas entran en el cálculo de cada fila:

| Marco | Qué suma |
| --- | --- |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Desde la primera fila hasta la actual (acumulado) |
| `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` | La actual y las 2 anteriores (promedio móvil de 3) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Toda la partición |

### La trampa del marco por defecto: `RANGE` frente a `ROWS`

Si escribes `ORDER BY` en la ventana **sin** marco, SQL Server usa `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. La diferencia con `ROWS` aparece cuando el orden tiene **empates**: `RANGE` trata todas las filas con el mismo valor de orden como una sola.

```sql
SELECT PedidoId, MONTH(FechaPedido) AS Mes, Total,
       SUM(Total) OVER (ORDER BY MONTH(FechaPedido))                         AS ConRange,
       SUM(Total) OVER (ORDER BY MONTH(FechaPedido), PedidoId
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)    AS ConRows
FROM Pedidos
ORDER BY Mes, PedidoId;
```

| PedidoId | Mes | Total | ConRange | ConRows |
| --- | --- | --- | --- | --- |
| 1 | 3 | 211.00 | 1060.90 | 211.00 |
| 2 | 3 | 849.90 | 1060.90 | 1060.90 |
| 3 | 4 | 265.00 | 1535.90 | 1325.90 |
| 4 | 4 | 210.00 | 1535.90 | 1535.90 |
| 5 | 5 | 140.50 | 1676.40 | 1676.40 |

Con `RANGE`, los dos pedidos de marzo muestran el acumulado de **todo** marzo, porque empatan en el orden. Además, `RANGE` suele ser más lento (usa una tabla de trabajo en disco). Regla: **para acumulados, escribe siempre `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** y un orden sin empates.

La misma trampa afecta a `LAST_VALUE`: con el marco por defecto, "la última fila" es la fila actual, y `LAST_VALUE(col) OVER (ORDER BY ...)` devuelve el propio valor de la fila. Hay que extender el marco hasta `UNBOUNDED FOLLOWING`, o usar `FIRST_VALUE` con el orden invertido.

### `COUNT(*) OVER ()` para paginar

La paginación de [Ordenar y paginar](03-Ordenar%20y%20paginar.md) necesitaba una segunda consulta para el total de filas. Con una función de ventana, va en la misma:

```sql
SELECT ProductoId, Nombre, Precio,
       COUNT(*) OVER () AS TotalFilas
FROM dbo.Producto
ORDER BY Precio DESC, ProductoId
OFFSET 0 ROWS FETCH NEXT 2 ROWS ONLY;
```

`COUNT(*) OVER ()` se calcula antes del `OFFSET`, así que vale 6 aunque la página tenga 2 filas.

-----

## Ejemplo completo

El reporte mensual de [CTE](09-CTE.md), con la variación respecto del mes anterior y el acumulado del año, reescrito con funciones de ventana:

```sql
WITH Meses AS (
    SELECT CAST('2026-01-01' AS DATE) AS Mes
    UNION ALL
    SELECT DATEADD(month, 1, Mes) FROM Meses WHERE Mes < '2026-06-01'
),
VentasPorMes AS (
    SELECT DATEFROMPARTS(YEAR(p.FechaPedido), MONTH(p.FechaPedido), 1) AS Mes,
           SUM(d.Cantidad * d.PrecioUnitario)                           AS Vendido
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY DATEFROMPARTS(YEAR(p.FechaPedido), MONTH(p.FechaPedido), 1)
)
SELECT m.Mes,
       COALESCE(v.Vendido, 0)                                                    AS Vendido,
       COALESCE(v.Vendido, 0) - LAG(COALESCE(v.Vendido, 0), 1, 0) OVER (ORDER BY m.Mes) AS Variacion,
       SUM(COALESCE(v.Vendido, 0)) OVER (ORDER BY m.Mes
                                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS Acumulado
FROM Meses AS m
LEFT JOIN VentasPorMes AS v ON v.Mes = m.Mes
ORDER BY m.Mes;
```

| Mes | Vendido | Variacion | Acumulado |
| --- | --- | --- | --- |
| 2026-01-01 | 0.00 | 0.00 | 0.00 |
| 2026-02-01 | 0.00 | 0.00 | 0.00 |
| 2026-03-01 | 1060.90 | 1060.90 | 1060.90 |
| 2026-04-01 | 265.00 | -795.90 | 1325.90 |
| 2026-05-01 | 140.50 | -124.50 | 1466.40 |
| 2026-06-01 | 0.00 | -140.50 | 1466.40 |

Qué cambió y por qué:

* **`LAG(..., 1, 0)` en lugar de la subconsulta correlacionada:** lee el mes anterior en la misma pasada, y el tercer argumento (0) evita el `NULL` del primer mes.
* **La CTE `Serie` desapareció:** las funciones de ventana se aplican directamente sobre el resultado del `LEFT JOIN`.
* **`ROWS BETWEEN ...` explícito en el acumulado:** aunque los meses no tienen empates, escribir el marco evita depender del `RANGE` por defecto.
* **Una sola lectura de los datos:** la versión con subconsulta correlacionada volvía a evaluar la serie para cada mes.

-----

## Errores comunes

**1. Filtrar por una función de ventana en el `WHERE`.**
Qué pasa: `Msg 4108: Windowed functions can only appear in the SELECT or ORDER BY clauses`.
Por qué: el `WHERE` se evalúa antes que el `SELECT`.
Arreglo: calcula la función en una CTE o tabla derivada y filtra afuera.

**2. Acumulados con el marco por defecto.**
Qué pasa: filas empatadas con el mismo acumulado, y consultas más lentas.
Por qué: sin marco, se usa `RANGE`, que agrupa los empates.
Arreglo: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` y un orden único.

**3. `ROW_NUMBER` con orden no determinista.**
Qué pasa: el "producto más caro" de una categoría cambia entre ejecuciones cuando hay dos con el mismo precio.
Por qué: entre filas empatadas, el número se asigna arbitrariamente.
Arreglo: agrega un desempate al `ORDER BY` de la ventana, o usa `RANK` si quieres conservar los empates.

**4. `LAST_VALUE` que devuelve la fila actual.**
Qué pasa: "el último precio de la partición" muestra el precio de cada fila.
Por qué: el marco por defecto termina en la fila actual.
Arreglo: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, o `FIRST_VALUE` con el orden invertido.

**5. Confundir el `ORDER BY` de la ventana con el de la consulta.**
Qué pasa: el acumulado es correcto pero las filas salen desordenadas.
Por qué: el `ORDER BY` dentro de `OVER` solo define el cálculo; no ordena el resultado.
Arreglo: un `ORDER BY` al final de la consulta para el orden de presentación.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Funciones de ventana | Desde 2005 (ranking); marcos y `LAG`/`LEAD` desde 2012 | Desde 8.4 | Desde **8.0** |
| Marcos `ROWS`, `RANGE` | Sí (`RANGE` solo con `UNBOUNDED` y `CURRENT ROW`) | `ROWS`, `RANGE` con desplazamientos, `GROUPS` | `ROWS`, `RANGE` con desplazamientos |
| Ventanas con nombre (`WINDOW w AS (...)`) | Desde 2022 | Sí | Sí |
| Filtrar por la ventana directamente | No (CTE) | No (CTE) | No (CTE) |
| `QUALIFY` (filtrar por la ventana sin CTE) | No | No | No (sí en Snowflake, BigQuery, DuckDB) |
| `IGNORE NULLS` en `FIRST_VALUE`/`LAST_VALUE` | Desde 2022 | No | No |

Las ventanas con nombre evitan repetir la misma definición:

```sql
SELECT ..., SUM(Total) OVER w AS Acumulado, AVG(Total) OVER w AS PromedioMovil
FROM Pedidos
WINDOW w AS (ORDER BY FechaPedido ROWS BETWEEN 2 PRECEDING AND CURRENT ROW);
```

### En MongoDB

Desde la versión 5.0, el *aggregation pipeline* tiene la etapa `$setWindowFields`, que es el equivalente directo:

```javascript
db.pedidos.aggregate([
  { $setWindowFields: {
      partitionBy: "$clienteId",
      sortBy: { fechaPedido: 1 },
      output: {
        numeroDePedido: { $documentNumber: {} },                     // ROW_NUMBER
        acumulado: { $sum: "$total",
                     window: { documents: ["unbounded", "current"] } } // ROWS BETWEEN ...
      }
  } }
]);
```

-----

## Cuándo sí y cuándo no

**Usa funciones de ventana cuando:**

* Necesitas un valor del grupo **junto** a cada fila (porcentaje del total, total del cliente).
* La pregunta es de ranking, top N por grupo, acumulados, promedios móviles o comparación con la fila anterior.

**Usa `GROUP BY` cuando:**

* Solo quieres una fila por grupo, sin el detalle.

**Ten en cuenta:**

* Cada `OVER` con un `PARTITION BY` u `ORDER BY` distinto puede necesitar su propio ordenamiento. Un índice que coincida con la partición y el orden lo evita.

-----

## Resumen en 5 líneas

1. `OVER (...)` calcula sobre un grupo de filas sin colapsarlas: conserva el detalle y agrega el valor del grupo.
2. `PARTITION BY` define los grupos; el `ORDER BY` de la ventana define el orden para rankings, acumulados y `LAG`/`LEAD`.
3. `ROW_NUMBER` numera sin repetir; `RANK` repite y salta; `DENSE_RANK` repite sin saltar. Top N por grupo: `ROW_NUMBER` en una CTE.
4. No se pueden usar en el `WHERE`: se calculan en una CTE y se filtra afuera.
5. Para acumulados, marco explícito `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`: el `RANGE` por defecto agrupa los empates.

-----

## Para profundizar

<details>
<summary>Islas y huecos (gaps and islands)</summary>

Un problema clásico que se resuelve con ventanas: encontrar rachas consecutivas (días seguidos con ventas, rangos de números sin huecos). El truco es restar a cada valor su `ROW_NUMBER()`: dentro de una racha consecutiva, la diferencia es constante.

```sql
-- Valores: 1, 2, 3, 7, 8, 10
-- ROW_NUMBER: 1, 2, 3, 4, 5, 6
-- Diferencia: 0, 0, 0, 3, 3, 4  → tres islas: (1-3), (7-8), (10)
SELECT MIN(n) AS Desde, MAX(n) AS Hasta
FROM (SELECT n, n - ROW_NUMBER() OVER (ORDER BY n) AS Grupo
      FROM (VALUES (1), (2), (3), (7), (8), (10)) AS t (n)) AS x
GROUP BY Grupo;
```

Con fechas, se resta `DATEADD(day, -ROW_NUMBER() OVER (...), fecha)`.

</details>

<details>
<summary>Eliminar duplicados con ROW_NUMBER</summary>

Para borrar filas repetidas dejando una de cada grupo, se numeran las filas de cada grupo de duplicados y se borran las que no son la primera, a través de una CTE:

```sql
WITH Duplicados AS (
    SELECT ROW_NUMBER() OVER (PARTITION BY Email ORDER BY ClienteId) AS n
    FROM dbo.ClienteImportado
)
DELETE FROM Duplicados WHERE n > 1;
```

En SQL Server, un `DELETE` sobre una CTE de una sola tabla borra las filas de esa tabla. Conviene ejecutarlo primero como `SELECT`, y dentro de una transacción (ver [Modificar datos](11-Modificar%20datos.md)).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las funciones de ventana calculan valores sobre un grupo de filas sin agruparlas, con la cláusula `OVER`. Por ejemplo, `ROW_NUMBER()` numera filas, `RANK()` hace rankings, `SUM() OVER (PARTITION BY ...)` da el total del grupo en cada fila y `LAG()` trae el valor de la fila anterior.

### Respuesta ampliada (semi-senior)

A diferencia de `GROUP BY`, una función de ventana conserva cada fila. `PARTITION BY` define los grupos, el `ORDER BY` de la ventana el orden para rankings, `LAG`/`LEAD` y acumulados, y el marco qué filas entran en el cálculo. Distingo `ROW_NUMBER`, `RANK` y `DENSE_RANK` por cómo tratan los empates, y le doy a `ROW_NUMBER` un orden determinista. Como se evalúan en el `SELECT`, no se pueden filtrar en el `WHERE`: el top N por grupo lo resuelvo con `ROW_NUMBER` en una CTE y un filtro afuera. Para acumulados escribo el marco explícito con `ROWS`, porque sin marco SQL Server usa `RANGE`, que agrupa los empates y es más lento; la misma trampa hace que `LAST_VALUE` devuelva la fila actual. También las uso para porcentajes del total, promedios móviles, tiempo entre eventos con `LAG`, el total de filas en una paginación con `COUNT(*) OVER ()` y problemas de islas y huecos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `RANK` y `DENSE_RANK`?**
Los dos dan el mismo número a los empatados; `RANK` salta los números siguientes (1, 1, 3) y `DENSE_RANK` no (1, 1, 2).

**2. ¿Cómo obtienes el registro más reciente de cada cliente?**
Con `ROW_NUMBER() OVER (PARTITION BY ClienteId ORDER BY Fecha DESC)` en una CTE, y filtrando las filas con número 1.

**3. ¿Por qué no puedes usar una función de ventana en el `WHERE`?**
Porque se evalúa en el `SELECT`, que va después del `WHERE` en el orden lógico.

**4. ¿Qué diferencia hay entre `GROUP BY` y `PARTITION BY`?**
`GROUP BY` devuelve una fila por grupo; `PARTITION BY` conserva todas las filas y le agrega a cada una el cálculo de su grupo.

**5. ¿Qué diferencia hay entre `ROWS` y `RANGE`?**
`ROWS` cuenta filas físicas; `RANGE` agrupa las filas con el mismo valor de orden. Con empates, los acumulados difieren.

-----

## Práctica

**Ejercicio 1.** Para cada pedido, muestra su total, el número de pedido del cliente (1 para su primer pedido, 2 para el segundo...) y el total del pedido anterior del mismo cliente.

<details>
<summary>Solución</summary>

```sql
WITH Pedidos AS (
    SELECT p.PedidoId, p.ClienteId, p.FechaPedido,
           SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    GROUP BY p.PedidoId, p.ClienteId, p.FechaPedido
)
SELECT ClienteId, PedidoId, Total,
       ROW_NUMBER() OVER (PARTITION BY ClienteId ORDER BY FechaPedido, PedidoId) AS NumeroDePedido,
       LAG(Total)   OVER (PARTITION BY ClienteId ORDER BY FechaPedido, PedidoId) AS TotalAnterior
FROM Pedidos
ORDER BY ClienteId, NumeroDePedido;
```

| ClienteId | PedidoId | Total | NumeroDePedido | TotalAnterior |
| --- | --- | --- | --- | --- |
| 1 | 1 | 211.00 | 1 | `NULL` |
| 1 | 3 | 265.00 | 2 | 211.00 |
| 2 | 2 | 849.90 | 1 | `NULL` |
| 3 | 4 | 210.00 | 1 | `NULL` |
| 4 | 5 | 140.50 | 1 | `NULL` |

</details>

**Ejercicio 2.** Lista, para cada categoría, sus productos con el puesto según el precio (del más caro al más barato) y el porcentaje que representa su precio sobre la suma de los precios de la categoría.

<details>
<summary>Solución</summary>

```sql
SELECT c.Nombre AS Categoria,
       p.Nombre AS Producto,
       p.Precio,
       RANK() OVER (PARTITION BY p.CategoriaId ORDER BY p.Precio DESC) AS Puesto,
       CAST(100.0 * p.Precio / SUM(p.Precio) OVER (PARTITION BY p.CategoriaId)
            AS DECIMAL(5,2))                                          AS PctDeLaCategoria
FROM dbo.Producto AS p
JOIN dbo.Categoria AS c ON c.CategoriaId = p.CategoriaId
ORDER BY c.Nombre, Puesto;
```

| Categoria | Producto | Precio | Puesto | PctDeLaCategoria |
| --- | --- | --- | --- | --- |
| Electrónica | Monitor 27 pulgadas | 899.90 | 1 | 84.47 |
| Electrónica | Teclado mecánico | 120.00 | 2 | 11.26 |
| Electrónica | Mouse inalámbrico | 45.50 | 3 | 4.27 |
| Hogar | Cafetera | 210.00 | 1 | 73.68 |
| Hogar | Lámpara de escritorio | 75.00 | 2 | 26.32 |
| Libros | Clean Code | 95.00 | 1 | 100.00 |

Las dos funciones de ventana tienen la misma partición pero distinto orden: `RANK` necesita el orden por precio; el `SUM` de la categoría no necesita orden (sin `ORDER BY`, la ventana es toda la partición).

</details>

-----

## Siguiente lección

[Modificar datos](11-Modificar%20datos.md)
