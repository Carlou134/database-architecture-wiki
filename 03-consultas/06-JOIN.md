# JOIN

## En una frase

Un `JOIN` combina las filas de dos tablas según una condición (normalmente, que la clave foránea de una coincida con la clave primaria de la otra); el tipo de join decide qué pasa con las filas **sin pareja**: `INNER` las descarta, `LEFT` y `RIGHT` conservan las de un lado, `FULL` las de ambos, y `CROSS` combina todo con todo.

-----

## Antes de empezar

Conviene que ya sepas:

* Claves primarias, foráneas y cardinalidad, de [Claves foráneas](../01-fundamentos/05-Claves%20foráneas%20e%20integridad%20referencial.md) y [Cardinalidad y relaciones](../02-diseno-y-modelado/03-Cardinalidad%20y%20relaciones.md).
* Que en el orden lógico el `FROM` con sus joins va primero, de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).
* Agregados y `GROUP BY`, de [Agregados y GROUP BY](05-Agregados%20y%20GROUP%20BY.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`INNER JOIN`:** devuelve solo las filas que tienen pareja en las dos tablas.
* **`LEFT JOIN` (`LEFT OUTER JOIN`):** devuelve todas las filas de la tabla izquierda, con `NULL` donde no hay pareja en la derecha.
* **`RIGHT JOIN`:** lo mismo, conservando la tabla derecha.
* **`FULL JOIN`:** conserva las filas sin pareja de los dos lados.
* **`CROSS JOIN`:** producto cartesiano: cada fila de una tabla con cada fila de la otra.
* **Self join:** unir una tabla consigo misma.
* **Anti join:** quedarse con las filas de una tabla que **no** tienen pareja en la otra.
* **Multiplicación de filas (fan-out):** el aumento de filas que produce un join con una tabla del lado "muchos".

-----

## El problema

Los datos de la tienda están normalizados: el pedido no guarda el nombre del cliente, solo su `ClienteId`. Para un reporte que diga "quién compró qué", hay que **volver a juntar** lo que la normalización separó.

Combinarlo a mano sería así: para cada pedido, buscar en `Cliente` la fila con el mismo `ClienteId` y copiar el nombre. Eso es exactamente lo que hace un `JOIN`. La pregunta interesante no es la sintaxis, sino qué pasa con los casos borde:

* Sofía no tiene pedidos. ¿Aparece en el reporte "clientes y sus pedidos"?
* Si filtras "pedidos no cancelados", ¿María desaparece de la lista de clientes?
* Ana tiene 2 pedidos con 4 líneas. Después de unir con los detalles, ¿cuántos pedidos tiene?

Cada una de estas preguntas tiene una respuesta precisa, y cada una es una fuente clásica de reportes incorrectos.

-----

## Cómo funciona

### `INNER JOIN`: solo las filas con pareja

```sql
SELECT c.Nombre, p.PedidoId, p.Estado
FROM dbo.Cliente AS c
INNER JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
ORDER BY c.Nombre, p.PedidoId;
```

| Nombre | PedidoId | Estado |
| --- | --- | --- |
| Ana Torres | 1 | Entregado |
| Ana Torres | 3 | Pagado |
| Jorge Díaz | 5 | Pendiente |
| Luis Ramos | 2 | Enviado |
| María Quispe | 4 | Cancelado |

* `ON` define la condición de emparejamiento: casi siempre, clave foránea = clave primaria.
* `INNER` es el valor por defecto: `JOIN` a secas es un `INNER JOIN`.
* **Sofía no aparece:** no tiene ningún pedido con el que emparejarse.
* Ana aparece **dos veces**: una por cada pedido. Un join no "agrega columnas" a una tabla; genera una fila por cada **pareja**.

Los alias (`c`, `p`) son imprescindibles: cuando dos tablas tienen una columna con el mismo nombre (`ClienteId`), hay que decir de cuál se habla. Sin prefijo, la consulta falla con `Msg 209: Ambiguous column name 'ClienteId'`.

### `LEFT JOIN`: conservar el lado izquierdo

```sql
SELECT c.Nombre, p.PedidoId, p.Estado
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
ORDER BY c.Nombre, p.PedidoId;
```

| Nombre | PedidoId | Estado |
| --- | --- | --- |
| Ana Torres | 1 | Entregado |
| Ana Torres | 3 | Pagado |
| Jorge Díaz | 5 | Pendiente |
| Luis Ramos | 2 | Enviado |
| María Quispe | 4 | Cancelado |
| Sofía Mendoza | `NULL` | `NULL` |

Todas las filas de la tabla **izquierda** (la del `FROM`) aparecen; donde no hay pareja, las columnas de la derecha valen `NULL`. Es el join para "todos los X, con sus Y si los tienen".

### Anti join: lo que no tiene pareja

Un `LEFT JOIN` + `WHERE ... IS NULL` sobre la clave de la derecha devuelve las filas **sin** pareja:

```sql
-- Clientes que nunca hicieron un pedido
SELECT c.Nombre
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
WHERE p.PedidoId IS NULL;
```

| Nombre |
| --- |
| Sofía Mendoza |

La columna del `IS NULL` tiene que ser una que **nunca** sea nula cuando hay pareja (la clave primaria de la derecha es lo más seguro). La misma pregunta se puede escribir con `NOT EXISTS`, que suele ser más clara (ver [Subconsultas](08-Subconsultas.md)).

### La trampa: filtrar la tabla derecha en el `WHERE`

"Todos los clientes, con sus pedidos no cancelados":

```sql
-- ❌ El WHERE convierte el LEFT JOIN en un INNER JOIN
SELECT c.Nombre, p.PedidoId, p.Estado
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
WHERE p.Estado <> 'Cancelado';
```

| Nombre | PedidoId | Estado |
| --- | --- | --- |
| Ana Torres | 1 | Entregado |
| Ana Torres | 3 | Pagado |
| Luis Ramos | 2 | Enviado |
| Jorge Díaz | 5 | Pendiente |

María y Sofía desaparecieron. Para Sofía, `p.Estado` es `NULL`, y `NULL <> 'Cancelado'` es `UNKNOWN`; para María, su único pedido está cancelado. El `WHERE` se evalúa **después** del join y descarta justamente las filas que el `LEFT JOIN` había conservado.

```sql
-- ✅ La condición sobre la tabla derecha va en el ON
SELECT c.Nombre, p.PedidoId, p.Estado
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p
       ON p.ClienteId = c.ClienteId
      AND p.Estado <> 'Cancelado'
ORDER BY c.Nombre, p.PedidoId;
```

| Nombre | PedidoId | Estado |
| --- | --- | --- |
| Ana Torres | 1 | Entregado |
| Ana Torres | 3 | Pagado |
| Jorge Díaz | 5 | Pendiente |
| Luis Ramos | 2 | Enviado |
| María Quispe | `NULL` | `NULL` |
| Sofía Mendoza | `NULL` | `NULL` |

La regla, para un `LEFT JOIN`:

* Condición en el **`ON`**: decide qué filas de la derecha se emparejan. Las de la izquierda se conservan siempre.
* Condición en el **`WHERE`**: filtra el resultado final, y si es sobre la tabla derecha, elimina las filas sin pareja.

En un `INNER JOIN` da lo mismo ponerla en el `ON` o en el `WHERE`; en un `LEFT JOIN`, cambia el resultado.

### `RIGHT JOIN` y `FULL JOIN`

`RIGHT JOIN` es un `LEFT JOIN` con las tablas al revés: conserva todas las filas de la tabla de la derecha. `A RIGHT JOIN B` es lo mismo que `B LEFT JOIN A`. En la práctica casi no se usa: escribir siempre `LEFT JOIN`, con la tabla "principal" en el `FROM`, hace las consultas más fáciles de leer.

`FULL JOIN` conserva las filas sin pareja de **los dos** lados. Es útil para comparar dos listas:

```sql
-- Ciudades con clientes frente a ciudades con sucursal
SELECT cli.Ciudad AS CiudadConClientes, suc.Ciudad AS CiudadConSucursal
FROM (SELECT DISTINCT Ciudad FROM dbo.Cliente WHERE Ciudad IS NOT NULL) AS cli
FULL JOIN (VALUES (N'Lima'), (N'Trujillo')) AS suc (Ciudad)
       ON suc.Ciudad = cli.Ciudad;
```

| CiudadConClientes | CiudadConSucursal |
| --- | --- |
| Arequipa | `NULL` |
| Cusco | `NULL` |
| Lima | Lima |
| `NULL` | Trujillo |

En una fila hay pareja (Lima); el resto son ciudades con clientes y sin sucursal, o con sucursal y sin clientes. `VALUES (...) AS suc (Ciudad)` crea una tabla en línea, útil para ejemplos y listas cortas.

### `CROSS JOIN`: todas las combinaciones

```sql
-- Una fila por cada categoría y cada mes del reporte
SELECT cat.Nombre AS Categoria, m.Mes
FROM dbo.Categoria AS cat
CROSS JOIN (VALUES (3), (4), (5)) AS m (Mes)
ORDER BY cat.Nombre, m.Mes;
```

Devuelve 3 categorías × 3 meses = 9 filas, sin condición. Sirve para armar la "grilla" de un reporte (todas las combinaciones, aunque alguna no tenga ventas) y después unirla con los datos reales con un `LEFT JOIN`. Usado sin querer (olvidando la condición), multiplica las tablas y es un error grave: dos tablas de 10.000 filas dan 100 millones.

### Self join: una tabla consigo misma

Se usa la misma tabla dos veces con alias distintos:

```sql
-- Pares de clientes que viven en la misma ciudad
SELECT c1.Nombre AS Cliente1, c2.Nombre AS Cliente2, c1.Ciudad
FROM dbo.Cliente AS c1
JOIN dbo.Cliente AS c2
  ON c2.Ciudad = c1.Ciudad
 AND c2.ClienteId > c1.ClienteId;   -- evita (Ana, Ana) y el par repetido (Jorge, Ana)
```

| Cliente1 | Cliente2 | Ciudad |
| --- | --- | --- |
| Ana Torres | Jorge Díaz | Lima |

El caso más común es la autorreferencia: cada empleado con el nombre de su jefe (`Empleado AS e JOIN Empleado AS j ON j.EmpleadoId = e.JefeId`), con `LEFT JOIN` para incluir a quien no tiene jefe.

### Varios joins y la multiplicación de filas

Las consultas reales encadenan joins siguiendo las claves foráneas:

```sql
SELECT c.Nombre, p.PedidoId, pr.Nombre AS Producto, d.Cantidad
FROM dbo.Cliente AS c
JOIN dbo.Pedido        AS p  ON p.ClienteId   = c.ClienteId
JOIN dbo.DetallePedido AS d  ON d.PedidoId    = p.PedidoId
JOIN dbo.Producto      AS pr ON pr.ProductoId = d.ProductoId
WHERE c.ClienteId = 1;
```

| Nombre | PedidoId | Producto | Cantidad |
| --- | --- | --- | --- |
| Ana Torres | 1 | Teclado mecánico | 1 |
| Ana Torres | 1 | Mouse inalámbrico | 2 |
| Ana Torres | 3 | Clean Code | 2 |
| Ana Torres | 3 | Lámpara de escritorio | 1 |

Cada join hacia una tabla del lado "muchos" **multiplica** las filas: Ana tenía 1 fila en `Cliente`, 2 después de unir con `Pedido` y 4 después de unir con `DetallePedido`. Si después se cuenta o se suma un dato del nivel del pedido, se cuenta varias veces. Por eso `COUNT(*)` diría que Ana tiene 4 pedidos.

Dos soluciones:

* **Contar lo distinto:** `COUNT(DISTINCT p.PedidoId)`. Funciona para conteos.
* **Agregar antes de unir:** calcular el total de cada pedido en una subconsulta y unir el resultado (una fila por pedido). Funciona para sumas, y es la opción más segura cuando se unen **dos** tablas hijas a la vez (por ejemplo, los pedidos y los teléfonos de cada cliente, que se multiplicarían entre sí).

### Sintaxis antigua: joins con coma

En código heredado vas a ver:

```sql
SELECT c.Nombre, p.PedidoId
FROM dbo.Cliente AS c, dbo.Pedido AS p
WHERE p.ClienteId = c.ClienteId;
```

Es un `INNER JOIN` escrito como producto cartesiano + filtro. Funciona, pero mezcla las condiciones de unión con los filtros, y si olvidas una condición obtienes un `CROSS JOIN` sin ningún aviso. Usa siempre la sintaxis `JOIN ... ON`.

-----

## Ejemplo completo

Reporte para gerencia: **todos** los clientes, con la cantidad de pedidos válidos (no cancelados), lo facturado en ellos y su último pedido, incluidos los que no compraron nada.

**Antes:**

```sql
SELECT c.Nombre,
       COUNT(*) AS Pedidos,
       SUM(d.Cantidad * d.PrecioUnitario) AS Facturado,
       MAX(p.FechaPedido) AS UltimoPedido
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
LEFT JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
WHERE p.Estado <> 'Cancelado'
GROUP BY c.ClienteId, c.Nombre;
```

| Nombre | Pedidos | Facturado | UltimoPedido |
| --- | --- | --- | --- |
| Ana Torres | 4 | 476.00 | 2026-04-12 09:05 |
| Luis Ramos | 1 | 849.90 | 2026-03-05 16:40 |
| Jorge Díaz | 2 | 140.50 | 2026-05-02 11:00 |

Tres errores: Ana no tiene 4 pedidos sino 2 (multiplicación por las líneas), Jorge no tiene 2 sino 1, y María y Sofía desaparecieron (el `WHERE` sobre la tabla derecha).

**Después:**

```sql
SELECT c.Nombre,
       COUNT(t.PedidoId)              AS Pedidos,
       COALESCE(SUM(t.Total), 0)      AS Facturado,
       MAX(t.FechaPedido)             AS UltimoPedido
FROM dbo.Cliente AS c
LEFT JOIN (
    SELECT p.PedidoId, p.ClienteId, p.FechaPedido,
           SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.PedidoId, p.ClienteId, p.FechaPedido
) AS t ON t.ClienteId = c.ClienteId
GROUP BY c.ClienteId, c.Nombre
ORDER BY Facturado DESC;
```

| Nombre | Pedidos | Facturado | UltimoPedido |
| --- | --- | --- | --- |
| Luis Ramos | 1 | 849.90 | 2026-03-05 16:40 |
| Ana Torres | 2 | 476.00 | 2026-04-12 09:05 |
| Jorge Díaz | 1 | 140.50 | 2026-05-02 11:00 |
| María Quispe | 0 | 0.00 | `NULL` |
| Sofía Mendoza | 0 | 0.00 | `NULL` |

Qué cambió y por qué:

* **Agregar antes de unir:** la subconsulta `t` devuelve **una fila por pedido**, con su total. Al unirla con `Cliente`, ya no hay líneas que multipliquen el conteo.
* **El filtro de estado dentro de la subconsulta:** se aplica a los pedidos antes del `LEFT JOIN`, así que no elimina clientes.
* **`COUNT(t.PedidoId)`** en lugar de `COUNT(*)`: para María y Sofía, la fila existe pero `t.PedidoId` es `NULL`, y `COUNT(columna)` ignora los nulos; `COUNT(*)` diría 1.
* **`COALESCE(SUM(...), 0)`:** la suma de un cliente sin pedidos es `NULL`; el reporte muestra 0.

-----

## Errores comunes

**1. Filtrar la tabla derecha de un `LEFT JOIN` en el `WHERE`.**
Qué pasa: el `LEFT JOIN` se comporta como un `INNER JOIN` y se pierden las filas sin pareja.
Por qué: el `WHERE` se evalúa después del join, y las columnas de la derecha son `NULL` en esas filas.
Arreglo: la condición va en el `ON`, o en una subconsulta previa.

**2. Contar o sumar después de unir con una tabla "muchos".**
Qué pasa: conteos y totales inflados.
Por qué: cada línea de la tabla hija repite la fila del padre.
Arreglo: `COUNT(DISTINCT ...)`, o agregar antes de unir.

**3. `COUNT(*)` en un `LEFT JOIN`.**
Qué pasa: los clientes sin pedidos aparecen con 1 pedido.
Por qué: la fila del cliente existe, aunque las columnas del pedido sean `NULL`.
Arreglo: `COUNT(p.PedidoId)`.

**4. Olvidar la condición de unión.**
Qué pasa: un producto cartesiano enorme, lento y sin sentido.
Por qué: con la sintaxis de comas, falta una línea del `WHERE`; con `JOIN`, el `ON` es obligatorio y no pasa (salvo con `CROSS JOIN`).
Arreglo: `JOIN ... ON` siempre.

**5. Columnas ambiguas.**
Qué pasa: `Msg 209: Ambiguous column name 'ClienteId'`.
Por qué: la columna existe en las dos tablas.
Arreglo: alias en todas las tablas y prefijo en todas las columnas.

**6. Anti join comprobando una columna que puede ser nula.**
Qué pasa: `LEFT JOIN ... WHERE p.FechaEntrega IS NULL` mezcla "no tiene pedido" con "tiene un pedido sin fecha de entrega".
Por qué: la columna es `NULL` en los dos casos.
Arreglo: comprueba la clave primaria de la derecha, o usa `NOT EXISTS`.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| `INNER`, `LEFT`, `RIGHT`, `CROSS` | Sí | Sí | Sí |
| `FULL JOIN` | Sí | Sí | **No** (se simula con `LEFT JOIN UNION RIGHT JOIN`) |
| `USING (col)` | No | Sí | Sí |
| `NATURAL JOIN` | No | Sí | Sí |
| Join con una subconsulta correlacionada | `CROSS APPLY`, `OUTER APPLY` | `LATERAL` | `LATERAL` (desde 8.0.14) |
| Tabla en línea con `VALUES` | `(VALUES ...) AS t (col)` | Igual | `VALUES ROW(...)` (desde 8.0.19) |

`USING (ClienteId)` es una forma corta de `ON a.ClienteId = b.ClienteId` cuando la columna se llama igual en las dos tablas. `NATURAL JOIN` une por **todas** las columnas con el mismo nombre, lo que es peligroso: si mañana las dos tablas ganan una columna `FechaCreacion`, el join cambia sin que nadie toque la consulta. Evítalo.

### En MongoDB

El equivalente de un `LEFT JOIN` es la etapa `$lookup`:

```javascript
db.clientes.aggregate([
  { $lookup: {
      from: "pedidos",
      localField: "_id",
      foreignField: "clienteId",
      as: "pedidos"            // un arreglo con los pedidos de cada cliente
  } }
]);
```

El resultado no son filas repetidas sino **un documento por cliente con un arreglo** de pedidos (vacío para Sofía). No hay multiplicación de filas salvo que se use `$unwind` para "aplanar" el arreglo. Aun así, en MongoDB los joins se evitan modelando: si los pedidos se leen siempre con su cliente, se embebe lo necesario.

-----

## Cuándo sí y cuándo no

**Usa `INNER JOIN` cuando:**

* Solo te interesan las filas que tienen pareja: "pedidos con su cliente" (todo pedido tiene cliente).

**Usa `LEFT JOIN` cuando:**

* La pregunta empieza por "todos los...": todos los clientes, todos los productos, con sus datos relacionados si existen.

**Usa `CROSS JOIN` cuando:**

* Necesitas todas las combinaciones a propósito: grillas de reportes, calendarios.

**Ten en cuenta:**

* Antes de agregar después de un join, pregúntate: ¿este join multiplica filas?

-----

## Resumen en 5 líneas

1. `INNER JOIN` devuelve solo las parejas; `LEFT JOIN` conserva todo el lado izquierdo con `NULL` donde no hay pareja.
2. En un `LEFT JOIN`, las condiciones sobre la tabla derecha van en el `ON`; en el `WHERE`, eliminan las filas sin pareja.
3. Anti join: `LEFT JOIN ... WHERE derecha.Clave IS NULL`, o `NOT EXISTS`.
4. Cada join con una tabla "muchos" multiplica filas: `COUNT(DISTINCT)` o agregar antes de unir.
5. `CROSS JOIN` para combinaciones a propósito; `RIGHT` y `FULL` existen, pero casi siempre se reescriben como `LEFT`.

-----

## Para profundizar

<details>
<summary>CROSS APPLY y OUTER APPLY</summary>

`APPLY` es un join en el que la tabla de la derecha puede depender de cada fila de la izquierda (en el estándar, `LATERAL`). El caso típico es "los N últimos de cada grupo":

```sql
-- El último pedido de cada cliente
SELECT c.Nombre, ult.PedidoId, ult.FechaPedido
FROM dbo.Cliente AS c
OUTER APPLY (
    SELECT TOP (1) p.PedidoId, p.FechaPedido
    FROM dbo.Pedido AS p
    WHERE p.ClienteId = c.ClienteId
    ORDER BY p.FechaPedido DESC
) AS ult;
```

`CROSS APPLY` se comporta como un `INNER JOIN` (descarta a Sofía) y `OUTER APPLY` como un `LEFT JOIN` (la conserva con `NULL`). La alternativa con funciones de ventana se ve en [Funciones de ventana](10-Funciones%20de%20ventana.md).

</details>

<details>
<summary>Cómo ejecuta el motor un join</summary>

El orden lógico dice **qué** resultado se obtiene; el optimizador elige **cómo**. Hay tres algoritmos físicos: *nested loops* (para cada fila de una tabla, busca sus parejas en la otra; ideal si una es chica y la otra tiene índice), *merge join* (recorre las dos ordenadas por la clave, como un cierre) y *hash join* (arma una tabla hash con una y la recorre con la otra; ideal para tablas grandes sin índices útiles). Los vas a ver en los planes de ejecución del módulo de índices, y el índice sobre la clave foránea es lo que hace barato el *nested loops*.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un `JOIN` combina filas de dos tablas por una condición, normalmente la clave foránea con la primaria. `INNER JOIN` devuelve solo las filas con pareja, `LEFT JOIN` todas las de la tabla izquierda aunque no tengan pareja, `RIGHT JOIN` lo mismo con la derecha, `FULL JOIN` todas las de ambas y `CROSS JOIN` todas las combinaciones.

### Respuesta ampliada (semi-senior)

El tipo de join define qué pasa con las filas sin pareja, y la mayoría de los errores salen de ahí. En un `LEFT JOIN`, una condición sobre la tabla derecha en el `WHERE` lo convierte en `INNER`, porque esas columnas son `NULL` en las filas sin pareja; va en el `ON` o en una subconsulta previa. Cada join con una tabla del lado "muchos" multiplica filas, así que antes de agregar uso `COUNT(DISTINCT)` o agrego en una subconsulta y uno el resultado, que es lo más seguro con varias tablas hijas. En un `LEFT JOIN` cuento con `COUNT(columna)`, no con `COUNT(*)`. Para "lo que no tiene pareja" uso un anti join sobre la clave de la derecha o `NOT EXISTS`. Uso `CROSS JOIN` para armar grillas y `OUTER APPLY` para "el último de cada grupo", y escribo siempre `JOIN ... ON` en lugar de la sintaxis de comas.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre poner una condición en el `ON` o en el `WHERE`?**
En un `INNER JOIN`, ninguna. En un `LEFT JOIN`, en el `ON` decide qué filas de la derecha se emparejan; en el `WHERE` filtra el resultado y elimina las filas sin pareja.

**2. ¿Cómo encuentras los clientes sin pedidos?**
Con `LEFT JOIN Pedido ... WHERE p.PedidoId IS NULL`, o con `WHERE NOT EXISTS (SELECT 1 FROM Pedido WHERE ...)`.

**3. ¿Por qué un total sale inflado después de un join?**
Porque el join con una tabla del lado "muchos" repite cada fila del padre por cada hija, y el valor se suma varias veces.

**4. ¿Cuándo usarías un `CROSS JOIN`?**
Para generar todas las combinaciones a propósito, por ejemplo todas las categorías por todos los meses de un reporte.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que muestre cada producto con el nombre de su categoría y la cantidad total de unidades vendidas, incluidos los productos que no se vendieron nunca (con 0).

<details>
<summary>Solución</summary>

```sql
SELECT pr.Nombre                    AS Producto,
       cat.Nombre                   AS Categoria,
       COALESCE(SUM(d.Cantidad), 0) AS UnidadesVendidas
FROM dbo.Producto AS pr
JOIN dbo.Categoria AS cat ON cat.CategoriaId = pr.CategoriaId
LEFT JOIN dbo.DetallePedido AS d ON d.ProductoId = pr.ProductoId
GROUP BY pr.ProductoId, pr.Nombre, cat.Nombre
ORDER BY UnidadesVendidas DESC, Producto;
```

| Producto | Categoria | UnidadesVendidas |
| --- | --- | --- |
| Clean Code | Libros | 3 |
| Mouse inalámbrico | Electrónica | 3 |
| Cafetera | Hogar | 1 |
| Lámpara de escritorio | Hogar | 1 |
| Monitor 27 pulgadas | Electrónica | 1 |
| Teclado mecánico | Electrónica | 1 |

Con los datos de ejemplo todos los productos se vendieron, pero la consulta ya está preparada para el producto nuevo que todavía no tiene ventas: el `LEFT JOIN` lo conserva y el `COALESCE` muestra 0. El join con `Categoria` es `INNER` porque todo producto tiene categoría (`CategoriaId NOT NULL`).

</details>

**Ejercicio 2.** ¿Qué devuelve esta consulta, y por qué no responde a "clientes con sus pedidos de 2026-04"? Corrígela.

```sql
SELECT c.Nombre, p.PedidoId
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
WHERE p.FechaPedido >= '2026-04-01' AND p.FechaPedido < '2026-05-01';
```

<details>
<summary>Solución</summary>

Devuelve solo **Ana (pedido 3)** y **María (pedido 4)**: el filtro sobre `p.FechaPedido` en el `WHERE` elimina a todos los clientes sin pedidos en abril (para ellos, la fecha es `NULL` o de otro mes), así que el `LEFT JOIN` se comporta como un `INNER JOIN`.

Si la pregunta es "todos los clientes, con sus pedidos de abril si los tienen", el filtro va en el `ON`:

```sql
SELECT c.Nombre, p.PedidoId
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p
       ON p.ClienteId = c.ClienteId
      AND p.FechaPedido >= '2026-04-01'
      AND p.FechaPedido <  '2026-05-01'
ORDER BY c.Nombre;
```

| Nombre | PedidoId |
| --- | --- |
| Ana Torres | 3 |
| Jorge Díaz | `NULL` |
| Luis Ramos | `NULL` |
| María Quispe | 4 |
| Sofía Mendoza | `NULL` |

Si la pregunta era solo "los pedidos de abril con su cliente", entonces lo correcto es un `INNER JOIN` y el filtro puede ir en el `WHERE`. Lo importante es elegir a propósito.

</details>

-----

## Siguiente lección

[Operaciones de conjuntos](07-Operaciones%20de%20conjuntos.md)
