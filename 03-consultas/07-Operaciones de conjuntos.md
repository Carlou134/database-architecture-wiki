# Operaciones de conjuntos

## En una frase

`UNION`, `INTERSECT` y `EXCEPT` combinan los resultados de dos consultas **verticalmente** (filas de una debajo de las de la otra), como operaciones de conjuntos: unión, intersección y diferencia; a diferencia de un `JOIN`, que combina **horizontalmente** (columnas de una al lado de las de la otra).

-----

## Antes de empezar

Conviene que ya sepas:

* Que el álgebra relacional incluye unión, intersección y diferencia, de [El modelo relacional](../01-fundamentos/02-El%20modelo%20relacional.md).
* `JOIN` y el anti join, de [JOIN](06-JOIN.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`UNION`:** filas que están en cualquiera de las dos consultas, sin repetidos.
* **`UNION ALL`:** todas las filas de las dos consultas, con repetidos.
* **`INTERSECT`:** filas que están en las dos consultas.
* **`EXCEPT`:** filas de la primera consulta que no están en la segunda.
* **Compatibilidad de unión:** las dos consultas deben tener la misma cantidad de columnas, con tipos compatibles y en el mismo orden.

-----

## El problema

El área de marketing quiere una sola lista de correos para un boletín: los de los clientes y los de los proveedores. Un `JOIN` no sirve: no hay una relación entre un cliente y un proveedor, y lo que se quiere no es "un cliente al lado de un proveedor", sino **una lista debajo de otra**.

Otras preguntas con la misma forma:

* "¿Qué productos se vendieron en marzo **y también** en abril o mayo?"
* "¿Qué clientes se registraron pero **nunca** compraron?"
* "Mostrar en una sola línea de tiempo los registros de clientes y los pedidos."

Todas son operaciones sobre **conjuntos de filas**, y SQL tiene operadores específicos para ellas.

-----

## Cómo funciona

### Vertical frente a horizontal

```text
JOIN (horizontal)                     UNION (vertical)
┌─────────┬─────────┐                 ┌─────────┐
│ Cliente │ Pedido  │                 │ Consulta│
│ columnas│ columnas│                 │    A    │
└─────────┴─────────┘                 ├─────────┤
 una fila por pareja                  │ Consulta│
                                      │    B    │
                                      └─────────┘
                                 las filas de B debajo de las de A
```

### Las reglas de compatibilidad

Para combinar dos consultas con un operador de conjuntos:

1. Deben tener la **misma cantidad de columnas**.
2. Las columnas en la misma posición deben tener **tipos compatibles** (el resultado toma el de mayor precedencia).
3. Los **nombres** del resultado salen de la **primera** consulta.
4. El `ORDER BY` va **una sola vez, al final**, y ordena el resultado combinado.

Las columnas se emparejan **por posición**, no por nombre. Si en la primera consulta pones `Nombre, Email` y en la segunda `Email, Nombre`, el motor no da error (los dos son texto): mezcla los nombres con los emails.

### `UNION` y `UNION ALL`

```sql
SELECT Ciudad FROM dbo.Cliente WHERE Ciudad IS NOT NULL
UNION
SELECT Ciudad FROM (VALUES (N'Lima'), (N'Trujillo')) AS s (Ciudad)
ORDER BY Ciudad;
```

| Ciudad |
| --- |
| Arequipa |
| Cusco |
| Lima |
| Trujillo |

`UNION` elimina los repetidos: Lima aparece en las dos consultas (y dos veces en la primera, por Ana y Jorge), pero una sola vez en el resultado. Con `UNION ALL`:

```sql
SELECT Ciudad FROM dbo.Cliente WHERE Ciudad IS NOT NULL
UNION ALL
SELECT Ciudad FROM (VALUES (N'Lima'), (N'Trujillo')) AS s (Ciudad)
ORDER BY Ciudad;
```

| Ciudad |
| --- |
| Arequipa |
| Cusco |
| Lima |
| Lima |
| Lima |
| Trujillo |

| | `UNION` | `UNION ALL` |
| --- | --- | --- |
| Repetidos | Los elimina | Los conserva |
| Costo | Tiene que ordenar o agrupar para eliminar repetidos | Solo concatena |
| Cuándo usarlo | Cuando el resultado no debe tener repetidos | Cuando no puede haberlos, o cuando sí los quieres |

Regla práctica: **`UNION ALL` por defecto**, y `UNION` solo cuando eliminar repetidos es parte de la pregunta. Si las dos consultas no pueden devolver la misma fila (por ejemplo, porque cada una incluye una columna `Tipo` distinta), `UNION` hace trabajo inútil.

### `INTERSECT`: lo que está en las dos

```sql
-- Productos vendidos en marzo y también en abril o mayo
SELECT d.ProductoId
FROM dbo.DetallePedido AS d
JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
WHERE p.FechaPedido >= '2026-03-01' AND p.FechaPedido < '2026-04-01'
INTERSECT
SELECT d.ProductoId
FROM dbo.DetallePedido AS d
JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
WHERE p.FechaPedido >= '2026-04-01' AND p.FechaPedido < '2026-06-01';
```

| ProductoId |
| --- |
| 2 |

En marzo se vendieron los productos 1, 2 y 3; en abril y mayo, el 2, el 4, el 5 y el 6. El único que está en las dos listas es el mouse (2).

### `EXCEPT`: lo que está en la primera y no en la segunda

```sql
-- Clientes que nunca hicieron un pedido
SELECT ClienteId FROM dbo.Cliente
EXCEPT
SELECT ClienteId FROM dbo.Pedido;
```

| ClienteId |
| --- |
| 5 |

Es la misma pregunta que el anti join de la lección anterior, escrita como diferencia de conjuntos. El orden importa: `A EXCEPT B` no es lo mismo que `B EXCEPT A`.

Una limitación: `EXCEPT` compara **todas** las columnas del `SELECT`. Si quieres los clientes sin pedidos **con su nombre**, no puedes agregar `Nombre` a la primera consulta (la segunda no lo tiene). Lo habitual es usar el resultado de `EXCEPT` como subconsulta, o directamente escribir `NOT EXISTS`.

### Los operadores tratan los `NULL` como iguales

A diferencia de un `WHERE` o un `JOIN`, donde `NULL = NULL` es `UNKNOWN`, los operadores de conjuntos consideran que dos `NULL` en la misma columna son **la misma fila**:

```sql
SELECT Ciudad FROM dbo.Cliente WHERE ClienteId = 3   -- NULL
INTERSECT
SELECT CAST(NULL AS NVARCHAR(80));                   -- NULL
```

Devuelve una fila con `NULL`. Por eso `EXCEPT` e `INTERSECT` son útiles para comparar dos versiones de una tabla: no hay que escribir `OR (a IS NULL AND b IS NULL)` en cada columna.

### Precedencia y paréntesis

Si combinas varios operadores, `INTERSECT` se evalúa antes que `UNION` y `EXCEPT`, y los demás de izquierda a derecha. Como con `AND` y `OR`, si mezclas operadores, usa paréntesis:

```sql
(SELECT ... UNION SELECT ...)
EXCEPT
SELECT ...;
```

-----

## Ejemplo completo

Una **línea de tiempo** de la tienda: los registros de clientes y los pedidos, en un solo listado ordenado por fecha.

```sql
SELECT FechaRegistro AS Fecha,
       N'Registro'    AS Evento,
       Nombre         AS Detalle
FROM dbo.Cliente

UNION ALL

SELECT p.FechaPedido,
       N'Pedido',
       CONCAT(N'Pedido ', p.PedidoId, N' de ', c.Nombre, N' (', p.Estado, N')')
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId

ORDER BY Fecha;
```

| Fecha | Evento | Detalle |
| --- | --- | --- |
| 2026-01-10 00:00 | Registro | Ana Torres |
| 2026-02-03 00:00 | Registro | Luis Ramos |
| 2026-02-20 00:00 | Registro | María Quispe |
| 2026-03-01 10:15 | Pedido | Pedido 1 de Ana Torres (Entregado) |
| 2026-03-05 16:40 | Pedido | Pedido 2 de Luis Ramos (Enviado) |
| 2026-03-15 00:00 | Registro | Jorge Díaz |
| 2026-04-01 00:00 | Registro | Sofía Mendoza |
| 2026-04-12 09:05 | Pedido | Pedido 3 de Ana Torres (Pagado) |
| 2026-04-20 18:30 | Pedido | Pedido 4 de María Quispe (Cancelado) |
| 2026-05-02 11:00 | Pedido | Pedido 5 de Jorge Díaz (Pendiente) |

Qué decisión hay detrás:

* **`UNION ALL`, no `UNION`:** la columna `Evento` distingue cada consulta, así que no puede haber repetidos entre ellas. `UNION` solo agregaría un ordenamiento inútil.
* **Los nombres de columna salen de la primera consulta:** la segunda no necesita alias, y el `ORDER BY Fecha` usa el nombre de la primera.
* **La segunda consulta puede tener sus propios joins y filtros:** cada parte es una consulta completa; solo las columnas finales tienen que ser compatibles.
* **El `ORDER BY` va una sola vez, al final:** ordena el resultado combinado.

-----

## Errores comunes

**1. Distinta cantidad de columnas.**
Qué pasa: `Msg 205: All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists`.
Por qué: las filas se apilan, así que tienen que tener la misma forma.
Arreglo: iguala las columnas; si una consulta no tiene un dato, usa `NULL` con el tipo correcto (`CAST(NULL AS DATE)`).

**2. Columnas en distinto orden.**
Qué pasa: nombres mezclados con emails, sin ningún error.
Por qué: las columnas se emparejan por posición, no por nombre.
Arreglo: escribe las columnas en el mismo orden en todas las consultas, y revisa el resultado.

**3. `UNION` cuando bastaba `UNION ALL`.**
Qué pasa: una consulta más lenta, y a veces filas legítimamente repetidas que desaparecen (dos ventas idénticas el mismo día).
Por qué: `UNION` elimina repetidos.
Arreglo: `UNION ALL` salvo que la pregunta pida eliminar repetidos.

**4. `ORDER BY` en una de las consultas intermedias.**
Qué pasa: error de sintaxis cerca de `UNION`.
Por qué: el orden solo se aplica al resultado final.
Arreglo: un único `ORDER BY` al final.

**5. Confundir `UNION` con `JOIN`.**
Qué pasa: se intenta "unir" dos tablas relacionadas con `UNION` y se obtiene una lista mezclada sin sentido.
Por qué: los dos se traducen como "unir" en español.
Arreglo: `JOIN` para poner columnas lado a lado (tablas relacionadas); `UNION` para apilar filas (resultados de la misma forma).

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| `UNION`, `UNION ALL` | Sí | Sí | Sí |
| `INTERSECT`, `EXCEPT` | Sí | Sí | Sí, desde 8.0.31 |
| `INTERSECT ALL`, `EXCEPT ALL` (con repetidos) | No | Sí | Sí, desde 8.0.31 |
| Sinónimo de `EXCEPT` | — | — | — (Oracle usa `MINUS`) |
| Precedencia de `INTERSECT` | Mayor que `UNION` y `EXCEPT` | Mayor | Mayor |

En versiones de MySQL anteriores a la 8.0.31, `INTERSECT` se escribe con `INNER JOIN` o `IN`, y `EXCEPT` con un anti join o `NOT IN`/`NOT EXISTS`.

### En MongoDB

El equivalente de `UNION ALL` es la etapa `$unionWith` del *aggregation pipeline* (desde MongoDB 4.4):

```javascript
db.clientes.aggregate([
  { $project: { fecha: "$fechaRegistro", evento: "Registro" } },
  { $unionWith: { coll: "pedidos", pipeline: [
      { $project: { fecha: "$fechaPedido", evento: "Pedido" } }
  ] } },
  { $sort: { fecha: 1 } }
]);
```

Para `UNION` (sin repetidos), se agrega un `$group` después. `INTERSECT` y `EXCEPT` no tienen etapa propia: se resuelven con `$lookup` o con operadores de conjuntos sobre arreglos (`$setIntersection`, `$setDifference`).

-----

## Cuándo sí y cuándo no

**Usa `UNION ALL` cuando:**

* Combinas filas de la misma forma desde varias fuentes: una línea de tiempo, tablas particionadas por año, listas de contactos.

**Usa `INTERSECT` o `EXCEPT` cuando:**

* La pregunta es literalmente "está en las dos" o "está en una y no en la otra", sobre las mismas columnas.
* Comparas dos versiones de una tabla (antes y después de una migración), aprovechando que tratan los `NULL` como iguales.

**Ten en cuenta:**

* Si necesitas columnas extra de una de las tablas, `EXISTS` y `NOT EXISTS` suelen ser más flexibles que `INTERSECT` y `EXCEPT` (ver [Subconsultas](08-Subconsultas.md)).
* Muchas consultas con `UNION` de la misma tabla con distintos filtros se pueden reescribir como una sola consulta con `OR` o con `CASE`, más simple y más rápida.

-----

## Resumen en 5 líneas

1. Los operadores de conjuntos apilan filas (vertical); `JOIN` pone columnas lado a lado (horizontal).
2. Las consultas deben tener la misma cantidad de columnas, tipos compatibles y el mismo orden; los nombres salen de la primera.
3. `UNION` elimina repetidos y cuesta más; `UNION ALL` los conserva y es la opción por defecto.
4. `INTERSECT` es "en las dos"; `EXCEPT` es "en la primera y no en la segunda", y el orden importa.
5. Tratan los `NULL` como iguales, y el `ORDER BY` va una sola vez, al final.

-----

## Para profundizar

<details>
<summary>Comparar dos tablas con EXCEPT</summary>

Después de una migración o de una carga, una forma rápida de verificar que dos tablas tienen exactamente los mismos datos:

```sql
-- Filas que están en la tabla vieja y no en la nueva
SELECT * FROM dbo.ClienteViejo
EXCEPT
SELECT * FROM dbo.ClienteNuevo;

-- Y al revés
SELECT * FROM dbo.ClienteNuevo
EXCEPT
SELECT * FROM dbo.ClienteViejo;
```

Si las dos consultas devuelven cero filas, las tablas coinciden (como conjuntos: si hubiera filas repetidas en una, `EXCEPT` no lo detectaría). Como `EXCEPT` trata los `NULL` como iguales, la comparación es correcta incluso con columnas nullables. Es uno de los pocos casos en que `SELECT *` está justificado.

</details>

<details>
<summary>UNION ALL en vistas particionadas</summary>

Antes de que existieran las tablas particionadas, una técnica común era tener una tabla por año (`Pedido2025`, `Pedido2026`) con un `CHECK` sobre la fecha en cada una, y una vista que las unía con `UNION ALL`. Gracias a los `CHECK`, el optimizador solo lee las tablas que pueden tener filas del rango consultado. Hoy lo habitual es particionar una sola tabla, pero vas a encontrar este patrón en bases heredadas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`UNION` combina los resultados de dos consultas en una sola lista sin repetidos, y `UNION ALL` hace lo mismo pero conserva los repetidos. `INTERSECT` devuelve lo que está en las dos consultas y `EXCEPT` lo que está en la primera y no en la segunda. Las consultas tienen que tener la misma cantidad de columnas y tipos compatibles.

### Respuesta ampliada (semi-senior)

Los operadores de conjuntos apilan filas, a diferencia de un `JOIN`, que combina columnas de tablas relacionadas. Exigen la misma cantidad de columnas, con tipos compatibles y emparejadas por posición, toman los nombres de la primera consulta y admiten un solo `ORDER BY` final. Uso `UNION ALL` por defecto y `UNION` solo cuando eliminar repetidos es parte de la pregunta, porque tiene un costo de ordenamiento y puede eliminar repetidos legítimos. `INTERSECT` y `EXCEPT` tratan los `NULL` como iguales, lo que los hace útiles para comparar dos versiones de una tabla; cuando necesito columnas extra, prefiero `EXISTS` y `NOT EXISTS`. Tengo en cuenta la precedencia (`INTERSECT` antes que los demás) y uso paréntesis al combinarlos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `UNION` y `UNION ALL`?**
`UNION` elimina las filas repetidas y tiene un costo de ordenamiento; `UNION ALL` las conserva y solo concatena.

**2. ¿Qué diferencia hay entre `UNION` y `JOIN`?**
`UNION` apila las filas de dos consultas de la misma forma; `JOIN` combina las columnas de tablas relacionadas según una condición.

**3. ¿Cómo tratan los `NULL` los operadores de conjuntos?**
Como iguales: dos filas con `NULL` en la misma columna se consideran la misma fila.

**4. ¿`A EXCEPT B` es igual a `B EXCEPT A`?**
No. Devuelve las filas de A que no están en B; al revés, las de B que no están en A.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que devuelva los IDs de los productos que **nunca** aparecieron en un pedido cancelado, usando un operador de conjuntos.

<details>
<summary>Solución</summary>

```sql
SELECT ProductoId FROM dbo.Producto
EXCEPT
SELECT d.ProductoId
FROM dbo.DetallePedido AS d
JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
WHERE p.Estado = 'Cancelado';
```

| ProductoId |
| --- |
| 1 |
| 2 |
| 3 |
| 4 |
| 6 |

El único pedido cancelado (el 4) solo tenía la cafetera (5), así que quedan los demás. Fíjate en que la pregunta incluye a los productos que no se vendieron nunca: por eso la primera consulta parte de `Producto` y no de `DetallePedido`.

</details>

**Ejercicio 2.** Una lista de contactos para el boletín: los emails de los clientes y los de estos dos proveedores, sin repetir, con una columna que diga el tipo de contacto. Un proveedor tiene el mismo email que una clienta. ¿Qué pasa con ese email si usas `UNION` con la columna de tipo?

```sql
-- Proveedores (tabla en línea)
(VALUES (N'ventas@andina.com'), (N'ana.torres@correo.com')) AS pr (Email)
```

<details>
<summary>Solución</summary>

```sql
SELECT Email, N'Cliente' AS Tipo FROM dbo.Cliente
UNION
SELECT Email, N'Proveedor'
FROM (VALUES (N'ventas@andina.com'), (N'ana.torres@correo.com')) AS pr (Email)
ORDER BY Email;
```

| Email | Tipo |
| --- | --- |
| ana.torres@correo.com | Cliente |
| ana.torres@correo.com | Proveedor |
| jorge.diaz@correo.com | Cliente |
| luis.ramos@correo.com | Cliente |
| maria.quispe@correo.com | Cliente |
| sofia.mendoza@correo.com | Cliente |
| ventas@andina.com | Proveedor |

`ana.torres@correo.com` aparece **dos veces**: `UNION` elimina filas repetidas **completas**, y las dos filas difieren en `Tipo`. Si el boletín no debe llegar dos veces a la misma persona, la lista tiene que unir solo la columna `Email` (con `UNION`), o agrupar por email y combinar los tipos con `STRING_AGG`.

</details>

-----

## Siguiente lección

[Subconsultas](08-Subconsultas.md)
