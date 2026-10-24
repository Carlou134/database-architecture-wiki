# Subconsultas

## En una frase

Una subconsulta es una consulta dentro de otra; según dónde va y qué devuelve, funciona como un **valor** (escalar), como una **lista** (con `IN`), como una **pregunta de existencia** (con `EXISTS`) o como una **tabla** (en el `FROM`), y puede ser independiente o **correlacionada** con cada fila de la consulta exterior.

-----

## Antes de empezar

Conviene que ya sepas:

* `WHERE`, `IN` y la trampa de `NOT IN` con `NULL`, de [Filtrar con WHERE](02-Filtrar%20con%20WHERE.md) y [NULL y la lógica de tres valores](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md).
* Agregados y el anti join, de [Agregados y GROUP BY](05-Agregados%20y%20GROUP%20BY.md) y [JOIN](06-JOIN.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Subconsulta (subquery, consulta anidada):** un `SELECT` dentro de otra instrucción, entre paréntesis.
* **Subconsulta escalar:** devuelve exactamente una fila y una columna; se usa como un valor.
* **Subconsulta correlacionada:** hace referencia a columnas de la consulta exterior, así que conceptualmente se evalúa una vez por cada fila de afuera.
* **Tabla derivada:** subconsulta en el `FROM`, que se usa como una tabla y lleva alias obligatorio.
* **`EXISTS` / `NOT EXISTS`:** verdadero si la subconsulta devuelve al menos una fila / ninguna.
* **Semi join:** quedarse con las filas que **tienen** pareja, sin duplicarlas (lo que hace `EXISTS`).

-----

## El problema

"¿Qué productos cuestan más que el promedio?" Dos pasos a mano:

```sql
SELECT AVG(Precio) FROM dbo.Producto;                    -- 240.900000
SELECT Nombre, Precio FROM dbo.Producto WHERE Precio > 240.90;
```

Funciona hoy. Mañana alguien cambia un precio, el promedio ya no es 240.90 y la segunda consulta da un resultado viejo. Y no se puede escribir `WHERE Precio > AVG(Precio)`: un agregado no puede ir en el `WHERE`.

Lo que hace falta es **usar el resultado de una consulta dentro de otra**, en el mismo momento. Y una vez que se puede, aparecen otras preguntas: ¿y si quiero el promedio **de su propia categoría**? ¿Y si la subconsulta devuelve varias filas? ¿Y si devuelve un `NULL`?

-----

## Cómo funciona

### Subconsulta escalar: un valor

Devuelve **una fila y una columna**, y se puede usar en cualquier lugar donde iría un valor:

```sql
SELECT Nombre, Precio
FROM dbo.Producto
WHERE Precio > (SELECT AVG(Precio) FROM dbo.Producto);
```

| Nombre | Precio |
| --- | --- |
| Monitor 27 pulgadas | 899.90 |

También en el `SELECT`, para mostrar el valor al lado de cada fila:

```sql
SELECT Nombre,
       Precio,
       Precio - (SELECT AVG(Precio) FROM dbo.Producto) AS DiferenciaConPromedio
FROM dbo.Producto;
```

Si una subconsulta escalar devuelve **más de una fila**, la consulta falla:

```text
Msg 512, Level 16, State 1
Subquery returned more than 1 value. This is not permitted when the subquery follows
=, !=, <, <= , >, >= or when the subquery is used as an expression.
```

Y si no devuelve ninguna fila, vale `NULL`, con todo lo que eso implica en una comparación.

### Subconsulta con `IN`: una lista

```sql
-- Clientes que compraron "Clean Code" (producto 6)
SELECT Nombre
FROM dbo.Cliente
WHERE ClienteId IN (
    SELECT p.ClienteId
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE d.ProductoId = 6
);
```

| Nombre |
| --- |
| Ana Torres |
| Jorge Díaz |

La subconsulta devuelve una lista de una columna, y `IN` pregunta si el valor está en ella. Una ventaja frente a un `JOIN`: si Ana hubiera comprado Clean Code en dos pedidos, con un `JOIN` aparecería dos veces; con `IN`, una.

### La trampa de `NOT IN` con una subconsulta

```sql
-- Clientes que no están en la lista negra (importada de un Excel, con una fila vacía)
SELECT Nombre
FROM dbo.Cliente
WHERE ClienteId NOT IN (
    SELECT ClienteId FROM (VALUES (2), (NULL)) AS ListaNegra (ClienteId)
);
```

Resultado: **ninguna fila**. Si la subconsulta devuelve un solo `NULL`, `NOT IN` equivale a `ClienteId <> 2 AND ClienteId <> NULL`, y la segunda parte siempre es `UNKNOWN`. Con una lista escrita a mano se ve; con una subconsulta sobre una columna nullable, aparece el día que alguien carga una fila incompleta.

La alternativa segura es `NOT EXISTS`.

### `EXISTS` y `NOT EXISTS`: preguntas de existencia

```sql
-- Clientes con al menos un pedido pagado, enviado o entregado
SELECT c.Nombre
FROM dbo.Cliente AS c
WHERE EXISTS (
    SELECT 1
    FROM dbo.Pedido AS p
    WHERE p.ClienteId = c.ClienteId
      AND p.Estado IN ('Pagado', 'Enviado', 'Entregado')
);
```

| Nombre |
| --- |
| Ana Torres |
| Luis Ramos |

* `EXISTS` es verdadero si la subconsulta devuelve **al menos una fila**; no importa qué columnas devuelva. Por eso se escribe `SELECT 1` (o `SELECT *`, que en un `EXISTS` no lee columnas).
* El motor puede **detenerse en la primera fila** que encuentra: no necesita contar todas.
* `EXISTS` nunca da `UNKNOWN`: es verdadero o falso.

Y su negación, sin la trampa del `NULL`:

```sql
-- Clientes sin ningún pedido
SELECT c.Nombre
FROM dbo.Cliente AS c
WHERE NOT EXISTS (
    SELECT 1 FROM dbo.Pedido AS p WHERE p.ClienteId = c.ClienteId
);
```

| Nombre |
| --- |
| Sofía Mendoza |

| Para preguntar... | Opciones | Recomendada |
| --- | --- | --- |
| "¿Tiene al menos uno?" | `IN`, `EXISTS`, `JOIN` + `DISTINCT` | `EXISTS` o `IN` |
| "¿No tiene ninguno?" | `NOT IN`, `NOT EXISTS`, `LEFT JOIN ... IS NULL`, `EXCEPT` | **`NOT EXISTS`** (sin trampa con `NULL`) |

### Subconsultas correlacionadas

Las dos consultas con `EXISTS` de arriba son **correlacionadas**: la subconsulta usa `c.ClienteId`, una columna de la consulta exterior. Conceptualmente, se evalúa una vez **por cada cliente**.

Otro ejemplo: productos que cuestan más que el promedio **de su propia categoría**.

```sql
SELECT pr.Nombre, pr.CategoriaId, pr.Precio
FROM dbo.Producto AS pr
WHERE pr.Precio > (
    SELECT AVG(otro.Precio)
    FROM dbo.Producto AS otro
    WHERE otro.CategoriaId = pr.CategoriaId      -- correlación con la fila exterior
);
```

| Nombre | CategoriaId | Precio |
| --- | --- | --- |
| Monitor 27 pulgadas | 1 | 899.90 |
| Cafetera | 2 | 210.00 |

El promedio de Electrónica es 355.13, el de Hogar 142.50 y el de Libros 95.00. Clean Code es el único libro, así que no supera su propio promedio.

"Una vez por cada fila" es el modelo **lógico**. En la práctica, el optimizador suele reescribir las subconsultas correlacionadas como joins, y no hay que temerles por rendimiento. Lo que sí conviene es usar alias distintos (`pr` y `otro`) para dejar claro qué columna es de qué consulta.

### Tablas derivadas: subconsultas en el `FROM`

Una subconsulta en el `FROM` funciona como una tabla temporal, y **necesita un alias**:

```sql
SELECT t.CategoriaId, t.PrecioPromedio
FROM (
    SELECT CategoriaId, AVG(Precio) AS PrecioPromedio
    FROM dbo.Producto
    GROUP BY CategoriaId
) AS t
WHERE t.PrecioPromedio > 100;
```

| CategoriaId | PrecioPromedio |
| --- | --- |
| 1 | 355.133333 |
| 2 | 142.500000 |

Sirven para trabajar en pasos: calcular algo (un agregado, un alias) y filtrarlo o unirlo después. Ya se usaron en [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md) (filtrar por un alias) y en [JOIN](06-JOIN.md) (agregar antes de unir). Cuando la consulta crece, las CTE de la [próxima lección](09-CTE.md) hacen lo mismo de forma más legible.

### `ANY` y `ALL`

Comparan un valor con **todos** los de una lista:

```sql
-- Productos más caros que todos los de la categoría Hogar (2)
SELECT Nombre, Precio
FROM dbo.Producto
WHERE Precio > ALL (SELECT Precio FROM dbo.Producto WHERE CategoriaId = 2);
```

| Nombre | Precio |
| --- | --- |
| Monitor 27 pulgadas | 899.90 |

`> ALL (...)` equivale a `> (SELECT MAX(...))`, y `> ANY (...)` a `> (SELECT MIN(...))`. Se usan poco: la versión con `MAX` o `MIN` suele ser más clara. Ojo: `> ALL` con una subconsulta vacía es **verdadero** (no hay ningún valor que lo contradiga).

### Subconsultas en `INSERT`, `UPDATE` y `DELETE`

Las subconsultas también sirven para decidir qué filas modificar:

```sql
-- Subir un 10 % los libros, buscando la categoría por nombre
UPDATE dbo.Producto
SET Precio = Precio * 1.10
WHERE CategoriaId = (SELECT CategoriaId FROM dbo.Categoria WHERE Nombre = N'Libros');

-- Borrar las categorías que no tienen productos
DELETE FROM dbo.Categoria
WHERE NOT EXISTS (SELECT 1 FROM dbo.Producto AS p WHERE p.CategoriaId = Categoria.CategoriaId);

-- Copiar a una tabla de archivo los pedidos cancelados
INSERT INTO dbo.PedidoArchivado (PedidoId, ClienteId, FechaPedido)
SELECT PedidoId, ClienteId, FechaPedido
FROM dbo.Pedido
WHERE Estado = 'Cancelado';
```

Se ven con más detalle, y con cómo ejecutarlas de forma segura, en [Modificar datos](11-Modificar%20datos.md).

-----

## Ejemplo completo

"Clientes que gastaron más que el promedio por cliente", sin contar los pedidos cancelados. La pregunta parece simple, pero tiene una decisión escondida: **¿el promedio incluye a los clientes que no gastaron nada?**

```sql
SELECT c.Nombre, g.Gastado
FROM dbo.Cliente AS c
JOIN (
    SELECT p.ClienteId, SUM(d.Cantidad * d.PrecioUnitario) AS Gastado
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.ClienteId
) AS g ON g.ClienteId = c.ClienteId
WHERE g.Gastado > (
    -- Promedio sobre TODOS los clientes, incluidos los que gastaron 0
    SELECT SUM(d.Cantidad * d.PrecioUnitario) / (SELECT COUNT(*) FROM dbo.Cliente)
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
)
ORDER BY g.Gastado DESC;
```

| Nombre | Gastado |
| --- | --- |
| Luis Ramos | 849.90 |
| Ana Torres | 476.00 |

Los números detrás:

| Definición del promedio | Cálculo | Promedio | Quiénes lo superan |
| --- | --- | --- | --- |
| Entre **todos** los clientes (5) | 1466.40 / 5 | 293.28 | Luis, Ana |
| Solo entre los que **compraron** (3) | 1466.40 / 3 | 488.80 | Luis |

Qué decisión hay detrás:

* **Una tabla derivada (`g`)** calcula el gasto por cliente, y una **subconsulta escalar** calcula el promedio. Cada pieza responde una pregunta.
* **La definición del promedio cambia el resultado:** con `AVG(g.Gastado)` se obtendría el segundo promedio (solo los clientes con compras). Ninguno es "el correcto": es una pregunta para el negocio, y la consulta tiene que dejar claro cuál eligió.
* **Sin división entera:** `SUM(...)` es `DECIMAL`, así que la división por `COUNT(*)` conserva los decimales.
* **El cálculo del gasto aparece dos veces.** Es la señal de que esta consulta quedaría mejor como CTE: se reescribe en la [próxima lección](09-CTE.md).

-----

## Errores comunes

**1. Una subconsulta escalar que devuelve varias filas.**
Qué pasa: `Msg 512: Subquery returned more than 1 value`.
Por qué: se usó `=` con una subconsulta que puede devolver varias filas (por ejemplo, buscar una categoría por un nombre que no es único).
Arreglo: `IN` si puede haber varias; o garantiza una sola fila (por clave, con `TOP (1) ... ORDER BY`, o con un agregado).

**2. `NOT IN` con una subconsulta que puede devolver `NULL`.**
Qué pasa: el resultado queda vacío, sin error.
Por qué: `x <> NULL` es `UNKNOWN`.
Arreglo: `NOT EXISTS`.

**3. Correlación con la tabla equivocada.**
Qué pasa: `WHERE CategoriaId = CategoriaId` dentro de la subconsulta compara la tabla interior consigo misma, y siempre es verdadero.
Por qué: sin alias, una columna se resuelve primero contra la tabla más cercana.
Arreglo: alias distintos en las dos consultas y prefijo en cada columna.

**4. Tabla derivada sin alias.**
Qué pasa: un error de sintaxis (`Msg 102` o `Msg 156: Incorrect syntax near...`) justo después de la subconsulta.
Por qué: una subconsulta en el `FROM` necesita un nombre.
Arreglo: `(...) AS t`.

**5. `ORDER BY` dentro de una subconsulta.**
Qué pasa: `Msg 1033: The ORDER BY clause is invalid in views, inline functions, derived tables, subqueries, and common table expressions, unless TOP, OFFSET or FOR XML is also specified`.
Por qué: el orden de una subconsulta no se conserva en el resultado exterior.
Arreglo: ordena en la consulta exterior.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Subconsultas escalares, `IN`, `EXISTS`, correlacionadas | Sí | Sí | Sí |
| Alias obligatorio en tablas derivadas | Sí | No, desde la 16 | Sí |
| Comparar varias columnas: `(a, b) IN (SELECT a, b ...)` | **No** (usar `EXISTS`) | Sí | Sí |
| Subconsulta sobre la misma tabla en un `UPDATE`/`DELETE` | Sí | Sí | Limitado: `You can't specify target table for update in FROM clause` (se esquiva con una tabla derivada) |
| `LATERAL` / `APPLY` | `CROSS APPLY`, `OUTER APPLY` | `LATERAL` | `LATERAL` (desde 8.0.14) |

La comparación de varias columnas con `IN` (*row constructors*) es una de las diferencias que más se notan al venir de PostgreSQL o MySQL. En SQL Server se escribe con `EXISTS`:

```sql
WHERE EXISTS (SELECT 1 FROM dbo.DetallePedido AS d
              WHERE d.PedidoId = x.PedidoId AND d.ProductoId = x.ProductoId)
```

### En MongoDB

No hay subconsultas en `find`. Lo equivalente se resuelve con el *aggregation pipeline*: un `$lookup` con un `pipeline` interno funciona como una subconsulta correlacionada, y un `$match` con `$expr` permite comparar con valores calculados en etapas anteriores. Para "más caro que el promedio", se calcula el promedio con `$group` y después se filtra, en el mismo pipeline o en dos consultas desde la aplicación.

-----

## Cuándo sí y cuándo no

**Usa una subconsulta escalar cuando:**

* Necesitas comparar con un valor calculado (el promedio, el máximo, una configuración).

**Usa `EXISTS` / `NOT EXISTS` cuando:**

* La pregunta es "tiene alguno" o "no tiene ninguno": es la forma más clara y la más segura con nulos.

**Usa una tabla derivada cuando:**

* Necesitas calcular algo en un paso y usarlo en el siguiente. Si se vuelve larga o se repite, pásala a una CTE.

**Ten en cuenta:**

* Una subconsulta correlacionada en el `SELECT` de una consulta con millones de filas puede ser cara si el optimizador no la convierte en un join. Si una consulta es lenta, el plan de ejecución lo muestra.

-----

## Resumen en 5 líneas

1. Escalar: una fila y una columna, se usa como un valor; si devuelve varias, falla (`Msg 512`); si ninguna, es `NULL`.
2. `IN` compara con una lista; `NOT IN` con una subconsulta que puede devolver `NULL` no devuelve nada.
3. `EXISTS` / `NOT EXISTS` preguntan por existencia, se detienen en la primera fila y nunca dan `UNKNOWN`.
4. Correlacionada: usa columnas de la consulta exterior; lógicamente se evalúa por fila, físicamente el optimizador suele convertirla en join.
5. Tabla derivada: subconsulta en el `FROM`, con alias obligatorio, para trabajar por pasos.

-----

## Para profundizar

<details>
<summary>Semi join y anti join en el plan de ejecución</summary>

Cuando escribes `EXISTS` o `IN`, el optimizador suele ejecutarlo como un **semi join**: une las dos tablas, pero devuelve cada fila de la izquierda **una sola vez**, en cuanto encuentra una pareja. `NOT EXISTS` se ejecuta como un **anti semi join**: devuelve las filas de la izquierda que **no** encuentran pareja. En el plan de ejecución vas a ver operadores llamados *Left Semi Join* y *Left Anti Semi Join*. `NOT IN` con una columna nullable obliga al motor a comprobar además si hay nulos, lo que genera un plan más complejo: otra razón para preferir `NOT EXISTS`.

</details>

<details>
<summary>Subconsultas y cortocircuito</summary>

Con `WHERE a = 1 OR EXISTS (...)`, no hay garantía de que el motor evite ejecutar la subconsulta cuando `a = 1`. Tampoco de que un `CASE WHEN x = 0 THEN 0 ELSE (SELECT ...) END` no evalúe la subconsulta. En la práctica, `CASE` evalúa en orden en la mayoría de los casos, pero la documentación de SQL Server advierte excepciones con los agregados. Si una subconsulta puede fallar (por ejemplo, con una división por cero), protégela por dentro con `NULLIF` en lugar de confiar en el orden de evaluación.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una subconsulta es una consulta dentro de otra. Puede devolver un valor, como el promedio de precios para compararlo en el `WHERE`, o una lista que se usa con `IN`. `EXISTS` comprueba si la subconsulta devuelve alguna fila. Una subconsulta correlacionada usa columnas de la consulta de afuera.

### Respuesta ampliada (semi-senior)

Distingo cuatro usos: escalar (una fila y una columna; falla si devuelve más y es `NULL` si no devuelve nada), de lista con `IN`, de existencia con `EXISTS` y tabla derivada en el `FROM`. Para "no tiene ninguno" uso `NOT EXISTS`, porque `NOT IN` con una subconsulta que puede devolver `NULL` devuelve cero filas, y además genera planes más complejos. `EXISTS` se resuelve como semi join y puede detenerse en la primera coincidencia. Las correlacionadas se evalúan lógicamente por fila, pero el optimizador suele convertirlas en joins; lo que cuido es usar alias distintos para no correlacionar con la tabla equivocada. Cuando una consulta encadena varias tablas derivadas o repite un cálculo, la paso a CTE. Y en SQL Server, para comparar varias columnas contra una subconsulta, uso `EXISTS`, porque no admite `(a, b) IN (...)`.

### Preguntas frecuentes de seguimiento

**1. ¿`IN` o `EXISTS`?**
Para "tiene alguno", ambos suelen generar el mismo plan. Para "no tiene ninguno", `NOT EXISTS`, por la trampa de `NOT IN` con `NULL`.

**2. ¿Qué es una subconsulta correlacionada?**
Una que usa columnas de la consulta exterior, así que su resultado depende de cada fila de afuera.

**3. ¿Qué pasa si una subconsulta escalar devuelve varias filas?**
Falla con el error 512. Si no devuelve ninguna, su valor es `NULL`.

**4. ¿Subconsulta o `JOIN`?**
Para traer columnas de otra tabla, `JOIN`. Para filtrar por existencia sin duplicar filas, `EXISTS` o `IN`. Muchas veces el optimizador genera el mismo plan; lo que decide es la claridad.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta para cada caso, usando subconsultas:

1. Los pedidos cuyo total supera el total del pedido 3.
2. Las categorías que tienen al menos un producto sin stock.
3. Los productos que nunca se vendieron en un pedido de Ana Torres.

<details>
<summary>Solución</summary>

```sql
-- 1
SELECT d.PedidoId, SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.DetallePedido AS d
GROUP BY d.PedidoId
HAVING SUM(d.Cantidad * d.PrecioUnitario) > (
    SELECT SUM(Cantidad * PrecioUnitario) FROM dbo.DetallePedido WHERE PedidoId = 3
);

-- 2
SELECT cat.Nombre
FROM dbo.Categoria AS cat
WHERE EXISTS (
    SELECT 1 FROM dbo.Producto AS p
    WHERE p.CategoriaId = cat.CategoriaId AND p.Stock = 0
);

-- 3
SELECT pr.Nombre
FROM dbo.Producto AS pr
WHERE NOT EXISTS (
    SELECT 1
    FROM dbo.DetallePedido AS d
    JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
    JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
    WHERE d.ProductoId = pr.ProductoId
      AND c.Email = N'ana.torres@correo.com'
);
```

Resultados:

1. Solo el **pedido 2** (849.90): el pedido 3 suma 265.00, y los demás son menores (211.00, 210.00 y 140.50).
2. **Hogar**: la cafetera tiene stock 0.
3. **Monitor 27 pulgadas** y **Cafetera**: Ana compró el teclado y el mouse (pedido 1), y Clean Code y la lámpara (pedido 3). Se busca a Ana por su email, que es único, y no por su nombre.

</details>

**Ejercicio 2.** ¿Qué devuelve esta consulta y por qué? Corrígela para que devuelva los productos que no están en ningún pedido cancelado.

```sql
SELECT Nombre FROM dbo.Producto
WHERE ProductoId NOT IN (
    SELECT d.ProductoId
    FROM dbo.Pedido AS p
    LEFT JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado = 'Cancelado'
);
```

<details>
<summary>Solución</summary>

Con los datos de ejemplo devuelve lo esperado (todos menos la cafetera), porque el pedido cancelado tiene un detalle. Pero tiene una bomba de tiempo: si alguna vez se cancela un pedido **sin detalles**, el `LEFT JOIN` devuelve una fila con `d.ProductoId = NULL`, y el `NOT IN` pasa a devolver **cero filas**.

Versión segura:

```sql
SELECT pr.Nombre
FROM dbo.Producto AS pr
WHERE NOT EXISTS (
    SELECT 1
    FROM dbo.DetallePedido AS d
    JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
    WHERE d.ProductoId = pr.ProductoId
      AND p.Estado = 'Cancelado'
);
```

| Nombre |
| --- |
| Teclado mecánico |
| Mouse inalámbrico |
| Monitor 27 pulgadas |
| Lámpara de escritorio |
| Clean Code |

`NOT EXISTS` no tiene el problema, y el `INNER JOIN` dentro de la subconsulta expresa mejor la pregunta: solo interesan los detalles que existen.

</details>

-----

## Siguiente lección

[CTE](09-CTE.md)
