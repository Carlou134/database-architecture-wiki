# Agregados y GROUP BY

## En una frase

Las funciones de agregado (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) resumen muchas filas en un valor; `GROUP BY` hace ese resumen **por grupo** (por cliente, por categoría, por mes) y `HAVING` filtra los grupos después de calcularlos, igual que `WHERE` filtra las filas antes.

-----

## Antes de empezar

Conviene que ya sepas:

* El orden lógico (`WHERE` antes de `GROUP BY`, `HAVING` después), de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).
* Cómo tratan los `NULL` los agregados, de [NULL y la lógica de tres valores](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md).
* `CASE` y la división entera, de [Expresiones, CASE y funciones](04-Expresiones%2C%20CASE%20y%20funciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Función de agregado:** función que recibe un conjunto de filas y devuelve un valor (`SUM`, `COUNT`).
* **`GROUP BY`:** agrupa las filas que comparten los mismos valores en las columnas indicadas.
* **`HAVING`:** filtra grupos según el resultado de sus agregados.
* **Agregación condicional:** un agregado sobre un `CASE`, para contar o sumar solo las filas que cumplen una condición.
* **`STRING_AGG`:** función que concatena los textos de un grupo en un solo valor.
* **Embudo (funnel):** proceso en varias etapas en el que en cada etapa quedan menos casos.

-----

## El problema

Gerencia quiere saber cuánto compró cada cliente:

```sql
SELECT c.Nombre, SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.Cliente AS c
JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId;
```

```text
Msg 8120, Level 16, State 1
Column 'dbo.Cliente.Nombre' is invalid in the select list because it is not contained
in either an aggregate function or the GROUP BY clause.
```

La consulta pide dos cosas incompatibles: `SUM` resume **todas** las filas en una, pero `Nombre` tiene un valor distinto por fila. ¿Qué nombre debería mostrar junto al total general? Falta decir **por qué** se agrupa. Y una vez agregado el `GROUP BY`, aparecen las preguntas de verdad: ¿el pedido cancelado de María cuenta como compra? ¿Por qué el promedio de unidades da 1 si hay pedidos de 2?

-----

## Cómo funciona

### Las funciones de agregado

| Función | Qué devuelve | Con `NULL` |
| --- | --- | --- |
| `COUNT(*)` | Cantidad de filas | Las cuenta |
| `COUNT(col)` | Cantidad de valores no nulos | Los ignora |
| `COUNT(DISTINCT col)` | Cantidad de valores distintos no nulos | Los ignora |
| `SUM(col)` | Suma | Los ignora; sin valores, `NULL` |
| `AVG(col)` | Promedio | Los ignora (divide por los no nulos) |
| `MIN(col)`, `MAX(col)` | Mínimo y máximo (también en textos y fechas) | Los ignora |

Sin `GROUP BY`, el agregado resume **toda** la tabla (o todo lo que pase el `WHERE`) en una sola fila:

```sql
SELECT COUNT(*)                               AS Lineas,
       COUNT(DISTINCT PedidoId)               AS Pedidos,
       SUM(Cantidad)                          AS Unidades,
       SUM(Cantidad * PrecioUnitario)         AS Facturado,
       MIN(PrecioUnitario)                    AS PrecioMinimo,
       MAX(PrecioUnitario)                    AS PrecioMaximo
FROM dbo.DetallePedido;
```

| Lineas | Pedidos | Unidades | Facturado | PrecioMinimo | PrecioMaximo |
| --- | --- | --- | --- | --- | --- |
| 8 | 5 | 10 | 1676.40 | 45.50 | 849.90 |

### La trampa de `AVG` con enteros

```sql
SELECT AVG(Cantidad)         AS PromedioEntero,
       AVG(Cantidad * 1.0)   AS PromedioDecimal
FROM dbo.DetallePedido;
```

| PromedioEntero | PromedioDecimal |
| --- | --- |
| 1 | 1.250000 |

En SQL Server, `AVG` devuelve el **mismo tipo** que la columna: con una columna `INT`, el promedio también es entero y se trunca (10 / 8 = 1,25 → 1). Es el mismo problema de la división entera, escondido dentro de una función. Convierte antes de promediar.

Y para redondear el resultado:

```sql
SELECT ROUND(AVG(Precio), 2) AS PrecioPromedio FROM dbo.Producto;   -- 240.900000
```

### `GROUP BY`: un resultado por grupo

```sql
SELECT c.Nombre,
       COUNT(DISTINCT p.PedidoId)          AS Pedidos,
       SUM(d.Cantidad * d.PrecioUnitario)  AS Total
FROM dbo.Cliente AS c
JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY c.ClienteId, c.Nombre
ORDER BY Total DESC;
```

| Nombre | Pedidos | Total |
| --- | --- | --- |
| Luis Ramos | 1 | 849.90 |
| Ana Torres | 2 | 476.00 |
| María Quispe | 1 | 210.00 |
| Jorge Díaz | 1 | 140.50 |

La regla que explica el error del problema: **toda columna del `SELECT` que no esté dentro de un agregado tiene que estar en el `GROUP BY`**. Cada fila del resultado es un grupo, y cada columna tiene que tener un único valor por grupo.

Detalles que importan:

* **Se agrupa por `ClienteId` y `Nombre`**, no solo por `Nombre`: dos clientes pueden llamarse igual, y agrupando solo por nombre se sumarían juntos.
* **`COUNT(DISTINCT p.PedidoId)`**, no `COUNT(*)`: después del join hay una fila por **línea** de pedido. Ana tiene 2 pedidos pero 4 líneas; `COUNT(*)` diría 4. Se profundiza en [JOIN](06-JOIN.md).
* **Se puede agrupar por expresiones:** `GROUP BY YEAR(FechaPedido), MONTH(FechaPedido)`.
* **Los `NULL` forman un grupo:** `GROUP BY Ciudad` tiene un grupo para los clientes sin ciudad.
* En SQL Server **no** se puede agrupar por posición (`GROUP BY 1`) ni por un alias del `SELECT`, porque `GROUP BY` se procesa antes.

### `WHERE` frente a `HAVING`

| | `WHERE` | `HAVING` |
| --- | --- | --- |
| Cuándo se evalúa | Antes de agrupar | Después de agrupar |
| Qué filtra | Filas | Grupos |
| Puede usar agregados | No | Sí |

```sql
-- Clientes con más de un pedido no cancelado
SELECT p.ClienteId, COUNT(*) AS Pedidos
FROM dbo.Pedido AS p
WHERE p.Estado <> 'Cancelado'          -- filas: fuera los pedidos cancelados
GROUP BY p.ClienteId
HAVING COUNT(*) > 1;                   -- grupos: solo los que tienen más de uno
```

| ClienteId | Pedidos |
| --- | --- |
| 1 | 2 |

Si una condición se puede escribir en el `WHERE` (porque no depende de un agregado), va en el `WHERE`: filtra antes y el motor agrupa menos filas.

### `STRING_AGG`: concatenar un grupo

El `GROUP_CONCAT` de SQLite y MySQL se llama `STRING_AGG` en SQL Server (desde 2017):

```sql
SELECT d.PedidoId,
       STRING_AGG(pr.Nombre, N', ') WITHIN GROUP (ORDER BY pr.Nombre) AS Productos
FROM dbo.DetallePedido AS d
JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
GROUP BY d.PedidoId;
```

| PedidoId | Productos |
| --- | --- |
| 1 | Mouse inalámbrico, Teclado mecánico |
| 2 | Monitor 27 pulgadas |
| 3 | Clean Code, Lámpara de escritorio |
| 4 | Cafetera |
| 5 | Clean Code, Mouse inalámbrico |

`WITHIN GROUP (ORDER BY ...)` define el orden dentro del texto; sin él, el orden no está garantizado. Es útil para reportes y exportaciones; para la aplicación suele ser mejor devolver las filas y armar la lista en el código.

### Agregación condicional

Un `CASE` dentro del agregado cuenta o suma solo las filas que cumplen una condición, y permite calcular varias métricas en una sola pasada:

```sql
SELECT COUNT(*)                                                    AS PedidosTotales,
       SUM(CASE WHEN Estado = 'Cancelado' THEN 1 ELSE 0 END)       AS Cancelados,
       SUM(CASE WHEN Estado IN ('Enviado', 'Entregado') THEN 1 ELSE 0 END) AS Despachados
FROM dbo.Pedido;
```

| PedidosTotales | Cancelados | Despachados |
| --- | --- | --- |
| 5 | 1 | 2 |

Es la técnica para convertir filas en columnas (un "pivot" manual): una columna por estado, por mes o por categoría.

### `ROLLUP`: subtotales

`GROUP BY ROLLUP (...)` agrega filas de subtotal y total general:

```sql
SELECT cat.Nombre AS Categoria,
       SUM(d.Cantidad * d.PrecioUnitario) AS Facturado
FROM dbo.DetallePedido AS d
JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
JOIN dbo.Categoria AS cat ON cat.CategoriaId = pr.CategoriaId
GROUP BY ROLLUP (cat.Nombre);
```

| Categoria | Facturado |
| --- | --- |
| Electrónica | 1106.40 |
| Hogar | 285.00 |
| Libros | 285.00 |
| `NULL` | 1676.40 |

La fila con `NULL` es el **total general**. Para distinguirla de un grupo que realmente tiene `NULL`, se usa `GROUPING(cat.Nombre)`, que vale 1 en las filas de subtotal.

-----

## Ejemplo completo

El **embudo de ventas** de la tienda: de todos los pedidos creados, ¿cuántos se pagaron, cuántos se enviaron y cuántos se entregaron? Es el mismo análisis de embudo de tus apuntes (navegar → iniciar compra → comprar), aplicado a los estados del pedido.

```sql
SELECT COUNT(*) AS Creados,
       SUM(CASE WHEN Estado IN ('Pagado', 'Enviado', 'Entregado') THEN 1 ELSE 0 END) AS Pagados,
       SUM(CASE WHEN Estado IN ('Enviado', 'Entregado')           THEN 1 ELSE 0 END) AS Enviados,
       SUM(CASE WHEN Estado = 'Entregado'                         THEN 1 ELSE 0 END) AS Entregados,
       CAST(100.0 * SUM(CASE WHEN Estado IN ('Pagado', 'Enviado', 'Entregado') THEN 1 ELSE 0 END)
            / COUNT(*) AS DECIMAL(5,1))                                              AS PctPagados
FROM dbo.Pedido;
```

| Creados | Pagados | Enviados | Entregados | PctPagados |
| --- | --- | --- | --- | --- |
| 5 | 3 | 2 | 1 | 60.0 |

Qué decisión hay detrás:

* **Cada etapa incluye las siguientes:** un pedido entregado también fue pagado y enviado. Por eso "Pagados" cuenta Pagado, Enviado **y** Entregado. Si se contara solo `Estado = 'Pagado'`, el embudo diría que se pagó 1 pedido de 5.
* **`100.0 *`:** sin el decimal, `100 * 3 / 5` funcionaría (60), pero `3 / 5 * 100` daría 0 por la división entera. Poner el `100.0` al principio evita depender del orden.
* **Una sola pasada:** la agregación condicional calcula las cuatro etapas leyendo la tabla una vez.
* **Lo que falta:** el embudo usa el estado **actual**. Con un historial de estados (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)), se podría medir también cuánto tarda cada etapa.

-----

## Errores comunes

**1. Columnas sin agregar fuera del `GROUP BY`.**
Qué pasa: `Msg 8120: Column ... is invalid in the select list`.
Por qué: cada grupo devuelve una fila, y esa columna tendría varios valores.
Arreglo: agrégala al `GROUP BY` o envuélvela en un agregado.

**2. Agrupar solo por un nombre.**
Qué pasa: dos clientes llamados igual se suman como uno.
Por qué: el nombre no es clave.
Arreglo: agrupa por la clave (`ClienteId`) y, además, por las columnas que quieres mostrar.

**3. `AVG` de una columna entera.**
Qué pasa: el promedio de 1, 2, 1, 2... da 1.
Por qué: `AVG` devuelve el tipo de la columna.
Arreglo: `AVG(col * 1.0)` o `AVG(CAST(col AS DECIMAL(10,2)))`.

**4. Filtrar agregados en el `WHERE`.**
Qué pasa: `WHERE COUNT(*) > 1` da `Msg 147: An aggregate may not appear in the WHERE clause`.
Por qué: el `WHERE` se evalúa antes de agrupar.
Arreglo: `HAVING COUNT(*) > 1`.

**5. `COUNT(*)` después de un join que multiplica filas.**
Qué pasa: "Ana tiene 4 pedidos".
Por qué: el join con los detalles genera una fila por línea.
Arreglo: `COUNT(DISTINCT p.PedidoId)`, o agregar los detalles antes de unir (ver [JOIN](06-JOIN.md)).

**6. Esperar `0` de un `SUM` sin filas.**
Qué pasa: el total de un cliente sin pedidos es `NULL`.
Por qué: la suma de ningún valor es `NULL`.
Arreglo: `COALESCE(SUM(...), 0)`.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| `AVG` de enteros | Entero (trunca) | `numeric` (con decimales) | Decimal |
| Concatenar un grupo | `STRING_AGG(col, sep) WITHIN GROUP (ORDER BY ...)` | `string_agg(col, sep ORDER BY ...)` | `GROUP_CONCAT(col ORDER BY ... SEPARATOR sep)` |
| `GROUP BY` por posición o alias | No | Sí | Sí |
| Columnas no agregadas fuera del `GROUP BY` | Error | Error (salvo que dependan de la PK agrupada) | Error con `ONLY_FULL_GROUP_BY` (por defecto desde 5.7); sin ese modo, devuelve un valor arbitrario |
| Agregación condicional | `SUM(CASE ...)` | `COUNT(*) FILTER (WHERE ...)` además de `CASE` | `SUM(condición)` (los booleanos valen 1 o 0) |
| Subtotales | `ROLLUP`, `CUBE`, `GROUPING SETS` | Los mismos | `WITH ROLLUP` |

El `AVG` de enteros es una diferencia que cambia resultados: la misma consulta devuelve 1 en SQL Server y 1.25 en PostgreSQL.

### En MongoDB

El equivalente de `GROUP BY` es la etapa `$group` del *aggregation pipeline*:

```javascript
db.pedidos.aggregate([
  { $match: { estado: { $ne: "Cancelado" } } },          // WHERE
  { $group: { _id: "$clienteId", pedidos: { $sum: 1 } } }, // GROUP BY + COUNT
  { $match: { pedidos: { $gt: 1 } } }                     // HAVING
]);
```

Fíjate en que `HAVING` no existe como tal: es simplemente otro `$match` **después** del `$group`. El *pipeline* hace explícito el orden lógico que en SQL está implícito.

-----

## Cuándo sí y cuándo no

**Agrega en la base cuando:**

* Necesitas totales, conteos o promedios: la base agrega millones de filas mucho más rápido que traerlas a la aplicación.

**Ten en cuenta:**

* Los agregados sobre tablas grandes se benefician de índices y, para reportes repetidos, de vistas indexadas (ver [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md)).
* `STRING_AGG` es cómodo para reportes, pero para la aplicación suele ser mejor devolver filas.
* Si necesitas el agregado **junto** a cada fila (cada pedido con el total del cliente), no uses `GROUP BY`: usa funciones de ventana ([Funciones de ventana](10-Funciones%20de%20ventana.md)).

-----

## Resumen en 5 líneas

1. `COUNT(*)` cuenta filas; `COUNT(col)` y los demás agregados ignoran los `NULL`; `SUM` sin valores da `NULL`.
2. En SQL Server, `AVG` de una columna entera devuelve un entero: convierte antes.
3. Toda columna del `SELECT` que no esté agregada debe estar en el `GROUP BY`; agrupa por la clave.
4. `WHERE` filtra filas antes de agrupar; `HAVING` filtra grupos después.
5. `SUM(CASE ...)` calcula varias métricas en una pasada; `STRING_AGG` concatena; `ROLLUP` agrega subtotales.

-----

## Para profundizar

<details>
<summary>GROUPING SETS y CUBE</summary>

`ROLLUP (a, b)` produce los grupos `(a, b)`, `(a)` y `()`: subtotales jerárquicos. `CUBE (a, b)` produce todas las combinaciones: `(a, b)`, `(a)`, `(b)` y `()`. `GROUPING SETS ((a), (b))` permite elegir exactamente qué agrupaciones calcular, en una sola consulta. Son la base de los reportes con subtotales por varias dimensiones, y equivalen a varias consultas unidas con `UNION ALL`, pero leyendo los datos una sola vez.

</details>

<details>
<summary>Tasa de cancelación (churn)</summary>

En tus apuntes, la tasa de cancelación de un mes se calcula como cancelaciones / (clientes al inicio + clientes nuevos). En SQL, ese cálculo es un caso típico de agregación condicional sobre una tabla de suscripciones con fecha de inicio y de fin: para cada mes, se cuentan los clientes activos al inicio y los que cancelaron durante el mes, con `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`, y se divide con `100.0 *` para no caer en la división entera. Con una CTE que genere la lista de meses (ver [CTE](09-CTE.md)), se obtiene la serie completa en una sola consulta.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las funciones de agregado como `COUNT`, `SUM` y `AVG` resumen varias filas en un valor. Con `GROUP BY` las calculo por grupo, por ejemplo el total por cliente. `WHERE` filtra filas antes de agrupar y `HAVING` filtra grupos después, como "clientes con más de un pedido".

### Respuesta ampliada (semi-senior)

Toda columna no agregada del `SELECT` tiene que estar en el `GROUP BY`, y agrupo por la clave para no mezclar entidades con el mismo nombre. Distingo `COUNT(*)` de `COUNT(col)` y de `COUNT(DISTINCT col)`, y sé que los agregados ignoran los nulos y que `SUM` sin filas devuelve `NULL`. En SQL Server, `AVG` de una columna entera trunca, así que convierto antes. Pongo en el `WHERE` todo lo que no dependa de un agregado y dejo en el `HAVING` solo los filtros sobre grupos. Uso agregación condicional con `SUM(CASE ...)` para calcular varias métricas en una pasada, como un embudo o un pivot, `STRING_AGG` con `WITHIN GROUP` para concatenar y `ROLLUP` o `GROUPING SETS` para los subtotales. Y cuido que los joins no multipliquen filas antes de agregar.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `WHERE` y `HAVING`?**
`WHERE` filtra filas antes de agrupar y no puede usar agregados; `HAVING` filtra grupos después y sí puede.

**2. ¿Qué diferencia hay entre `COUNT(*)` y `COUNT(columna)`?**
`COUNT(*)` cuenta filas; `COUNT(columna)` cuenta los valores no nulos de esa columna.

**3. ¿Cuál es el equivalente de `GROUP_CONCAT` en SQL Server?**
`STRING_AGG`, desde SQL Server 2017.

**4. ¿Cómo cuentas cuántas filas cumplen una condición dentro de un grupo?**
Con agregación condicional: `SUM(CASE WHEN condición THEN 1 ELSE 0 END)`.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta con la cantidad de clientes por ciudad, mostrando "Sin ciudad" para los que no la informaron, ordenada de mayor a menor cantidad.

<details>
<summary>Solución</summary>

```sql
SELECT COALESCE(Ciudad, N'Sin ciudad') AS Ciudad,
       COUNT(*)                        AS Clientes
FROM dbo.Cliente
GROUP BY Ciudad
ORDER BY Clientes DESC, Ciudad;
```

| Ciudad | Clientes |
| --- | --- |
| Lima | 2 |
| Arequipa | 1 |
| Cusco | 1 |
| Sin ciudad | 1 |

Se agrupa por la columna original y se aplica `COALESCE` solo para mostrar. El `ORDER BY ... Ciudad` usa el alias (que ya existe en el `ORDER BY`), así que "Sin ciudad" se ordena alfabéticamente con las demás.

</details>

**Ejercicio 2.** Escribe una consulta con el facturado por categoría, **sin** contar los pedidos cancelados, y quédate solo con las categorías que facturaron más de 200.

<details>
<summary>Solución</summary>

```sql
SELECT cat.Nombre                          AS Categoria,
       SUM(d.Cantidad * d.PrecioUnitario)  AS Facturado
FROM dbo.DetallePedido AS d
JOIN dbo.Pedido    AS p   ON p.PedidoId = d.PedidoId
JOIN dbo.Producto  AS pr  ON pr.ProductoId = d.ProductoId
JOIN dbo.Categoria AS cat ON cat.CategoriaId = pr.CategoriaId
WHERE p.Estado <> 'Cancelado'
GROUP BY cat.CategoriaId, cat.Nombre
HAVING SUM(d.Cantidad * d.PrecioUnitario) > 200
ORDER BY Facturado DESC;
```

| Categoria | Facturado |
| --- | --- |
| Electrónica | 1106.40 |
| Libros | 285.00 |

Hogar facturaba 285.00 en total, pero 210.00 eran de la cafetera del pedido cancelado: sin él, queda en 75.00 y el `HAVING` la descarta. El filtro por estado va en el `WHERE` (es sobre filas) y el de facturado en el `HAVING` (es sobre grupos).

</details>

-----

## Siguiente lección

[JOIN](06-JOIN.md)
