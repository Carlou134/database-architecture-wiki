# Filtrar con WHERE

## En una frase

`WHERE` deja pasar solo las filas cuya condición es **verdadera**, y esa condición se arma con comparaciones (`=`, `<>`, `<`, `>`), rangos (`BETWEEN`), listas (`IN`), patrones de texto (`LIKE`) y nulos (`IS NULL`), combinados con `AND`, `OR` y `NOT` según reglas de precedencia que conviene no dejar al azar.

-----

## Antes de empezar

Conviene que ya sepas:

* El orden lógico de una consulta, de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).
* La lógica de tres valores y la trampa de `NOT IN` con `NULL`, de [NULL y la lógica de tres valores](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Predicado:** una condición que se evalúa como verdadera, falsa o desconocida (`Precio > 100`).
* **Precedencia:** el orden en que se evalúan los operadores cuando no hay paréntesis.
* **Comodín (wildcard):** carácter especial de `LIKE`: `%` (cualquier cantidad de caracteres) y `_` (exactamente uno).
* **Rango semiabierto:** un rango que incluye el inicio y excluye el final (`>= inicio AND < fin`).
* **SARGable:** condición que el motor puede resolver con un índice (*Search ARGument able*).

-----

## El problema

Marketing pide "los productos de la categoría 1 o de la 2 que cuesten menos de 100". La consulta que llega a producción:

```sql
SELECT Nombre, CategoriaId, Precio
FROM dbo.Producto
WHERE CategoriaId = 1 OR CategoriaId = 2 AND Precio < 100;
```

| Nombre | CategoriaId | Precio |
| --- | --- | --- |
| Teclado mecánico | 1 | 120.00 |
| Mouse inalámbrico | 1 | 45.50 |
| Monitor 27 pulgadas | 1 | 899.90 |
| Lámpara de escritorio | 2 | 75.00 |

El monitor de 899.90 está en la lista. La consulta no tiene errores de sintaxis; tiene un error de **precedencia**: `AND` se evalúa antes que `OR`, así que el motor leyó "categoría 1 (a cualquier precio), **o** categoría 2 y menos de 100".

Y ese es solo el primer problema de los filtros. Los otros clásicos: rangos de fechas que pierden el último día, `LIKE` que no encuentra un `%` literal y filtros con `<>` que pierden las filas con `NULL`.

-----

## Cómo funciona

### Operadores de comparación

| Operador | Significado | Ejemplo |
| --- | --- | --- |
| `=` | Igual | `Estado = 'Pagado'` |
| `<>` o `!=` | Distinto | `Estado <> 'Cancelado'` |
| `<`, `<=`, `>`, `>=` | Menor, menor o igual, mayor, mayor o igual | `Precio >= 100` |

`<>` es el estándar; `!=` también funciona en SQL Server, PostgreSQL y MySQL.

Se pueden comparar columnas entre sí, no solo con valores fijos:

```sql
-- Productos cuyo valor en stock supera los 1500
SELECT Nombre, Precio, Stock
FROM dbo.Producto
WHERE Precio * Stock > 1500;
```

| Nombre | Precio | Stock |
| --- | --- | --- |
| Teclado mecánico | 120.00 | 15 |
| Mouse inalámbrico | 45.50 | 40 |
| Monitor 27 pulgadas | 899.90 | 8 |

Los textos se comparan según la intercalación: con la habitual en español (`Modern_Spanish_CI_AS`), `'lima' = 'Lima'` es verdadero (no distingue mayúsculas), pero `'Peru' = 'Perú'` es falso (sí distingue acentos).

### `AND`, `OR`, `NOT` y la precedencia

El orden de evaluación, sin paréntesis, es: **`NOT`, después `AND`, después `OR`**. Igual que en matemática la multiplicación va antes que la suma.

La consulta del problema, corregida con paréntesis:

```sql
SELECT Nombre, CategoriaId, Precio
FROM dbo.Producto
WHERE (CategoriaId = 1 OR CategoriaId = 2) AND Precio < 100;
```

| Nombre | CategoriaId | Precio |
| --- | --- | --- |
| Mouse inalámbrico | 1 | 45.50 |
| Lámpara de escritorio | 2 | 75.00 |

Regla práctica: **si una condición mezcla `AND` y `OR`, usa paréntesis siempre**, aunque la precedencia ya dé el resultado que quieres. Le ahorras al lector tener que recordar la tabla.

### `BETWEEN`: rangos inclusivos

```sql
SELECT Nombre, Precio
FROM dbo.Producto
WHERE Precio BETWEEN 75 AND 120;     -- equivale a Precio >= 75 AND Precio <= 120
```

| Nombre | Precio |
| --- | --- |
| Teclado mecánico | 120.00 |
| Lámpara de escritorio | 75.00 |
| Clean Code | 95.00 |

`BETWEEN` incluye **los dos extremos**. Con números enteros o decimales es cómodo. Con fechas y horas, es una trampa:

```sql
-- "Pedidos de marzo"
SELECT PedidoId, FechaPedido
FROM dbo.Pedido
WHERE FechaPedido BETWEEN '2026-03-01' AND '2026-03-31';
```

`FechaPedido` es `DATETIME2`. El literal `'2026-03-31'` significa `2026-03-31 00:00:00`, así que un pedido del 31 de marzo a las 10:00 **queda afuera**. Con los datos de ejemplo no se nota (no hay pedidos el 31), y por eso este error llega a producción.

La forma correcta para rangos de fechas es el **rango semiabierto**: mayor o igual al inicio, **menor** al día siguiente del final:

```sql
SELECT PedidoId, FechaPedido
FROM dbo.Pedido
WHERE FechaPedido >= '2026-03-01' AND FechaPedido < '2026-04-01';
```

| PedidoId | FechaPedido |
| --- | --- |
| 1 | 2026-03-01 10:15:00 |
| 2 | 2026-03-05 16:40:00 |

Funciona igual con `DATE`, `DATETIME2(0)` o `DATETIME2(7)`, sin pensar en la precisión de la columna.

El `AND` de `BETWEEN x AND y` es parte de la sintaxis de `BETWEEN`, no el operador lógico. `WHERE Precio BETWEEN 75 AND 120 AND Stock > 10` se lee como `(Precio BETWEEN 75 AND 120) AND (Stock > 10)`.

### `IN`: una lista de valores

```sql
SELECT PedidoId, Estado
FROM dbo.Pedido
WHERE Estado IN ('Pagado', 'Enviado', 'Entregado');   -- equivale a tres = unidos con OR
```

| PedidoId | Estado |
| --- | --- |
| 1 | Entregado |
| 2 | Enviado |
| 3 | Pagado |

`NOT IN` excluye la lista. Recuerda la trampa de [NULL](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md): si la lista contiene un `NULL`, `NOT IN` no devuelve ninguna fila. Con una lista escrita a mano se ve; con una subconsulta, no (ver [Subconsultas](08-Subconsultas.md)).

### `LIKE`: patrones de texto

| Comodín | Significado | Ejemplo | Encuentra |
| --- | --- | --- | --- |
| `%` | Cero o más caracteres | `'C%'` | Cafetera, Clean Code |
| `_` | Exactamente un carácter | `'_ouse%'` | Mouse inalámbrico |
| `[abc]` | Uno de los caracteres de la lista (SQL Server) | `'[CL]%'` | Cafetera, Clean Code, Lámpara... |
| `[^abc]` | Ninguno de los de la lista (SQL Server) | `'[^C]%'` | Todo lo que no empieza con C |

```sql
SELECT Nombre FROM dbo.Producto WHERE Nombre LIKE '%de%';
```

| Nombre |
| --- |
| Lámpara de escritorio |
| Clean Code |

"Clean Code" aparece porque "Co**de**" contiene "de": `LIKE` busca el patrón en cualquier parte del texto, no palabras completas.

Para buscar un `%` o un `_` **literal** (por ejemplo, un código `PROMO_50%`), hay dos formas:

```sql
WHERE Codigo LIKE '%50[%]%'               -- el comodín entre corchetes es literal
WHERE Codigo LIKE '%50!%%' ESCAPE '!'     -- el carácter después de ! es literal
```

`LIKE` funciona sobre texto. Si lo usas con un número o una fecha, SQL Server convierte la columna a texto primero: funciona, pero es lento y depende del formato de conversión.

Un detalle de rendimiento que se profundiza en el módulo de índices: `LIKE 'C%'` (prefijo fijo) puede usar un índice; `LIKE '%C'` (comodín al principio) no, porque obliga a revisar cada fila.

### `IS NULL` e `IS NOT NULL`

Cualquier comparación con `NULL` da `UNKNOWN`, y `WHERE` descarta lo desconocido. Para preguntar por nulos:

```sql
SELECT Nombre FROM dbo.Cliente WHERE Ciudad IS NULL;          -- María
SELECT Nombre FROM dbo.Cliente WHERE Ciudad = NULL;           -- ninguna fila, siempre
```

Y cuando filtras con `<>` o `NOT IN` sobre una columna que admite nulos, decide explícitamente qué pasa con ellos:

```sql
SELECT Nombre, Ciudad
FROM dbo.Cliente
WHERE Ciudad <> N'Lima' OR Ciudad IS NULL;    -- incluye a María a propósito
```

¿Conviene guardar los datos faltantes como `NULL`? Sí, cuando el dato realmente falta y no hay un valor por defecto razonable; no conviene usar textos vacíos ni valores mágicos (`'N/A'`, `-1`). Se discute en [NULL y la lógica de tres valores](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md).

### Filtros que pueden usar índices

Una condición es **SARGable** cuando deja la columna "sola" de un lado de la comparación, y el motor puede buscar directamente en un índice. Envolver la columna en una función o una operación lo impide:

| ❌ No SARGable | ✅ SARGable |
| --- | --- |
| `WHERE YEAR(FechaPedido) = 2026` | `WHERE FechaPedido >= '2026-01-01' AND FechaPedido < '2027-01-01'` |
| `WHERE Precio * 1.18 > 200` | `WHERE Precio > 200 / 1.18` |
| `WHERE LEFT(Email, 3) = 'ana'` | `WHERE Email LIKE 'ana%'` |

Con cinco filas da igual; con millones, es la diferencia entre leer unas pocas páginas y leer la tabla entera. Se estudia a fondo en el módulo de índices.

-----

## Ejemplo completo

"Pedidos activos de abril y mayo de 2026: ni cancelados ni pendientes."

**Antes:**

```sql
SELECT PedidoId, ClienteId, FechaPedido, Estado
FROM dbo.Pedido
WHERE FechaPedido BETWEEN '2026-04-01' AND '2026-05-31'
  AND Estado <> 'Cancelado' OR Estado <> 'Pendiente';
```

| PedidoId | ClienteId | FechaPedido | Estado |
| --- | --- | --- | --- |
| 1 | 1 | 2026-03-01 10:15:00 | Entregado |
| 2 | 2 | 2026-03-05 16:40:00 | Enviado |
| 3 | 1 | 2026-04-12 09:05:00 | Pagado |
| 4 | 3 | 2026-04-20 18:30:00 | Cancelado |
| 5 | 4 | 2026-05-02 11:00:00 | Pendiente |

Devuelve los cinco pedidos: el `OR` final se evalúa último, y `Estado <> 'Pendiente'` es verdadero para casi todo. Además, `<> 'Cancelado' OR <> 'Pendiente'` es verdadero para **cualquier** estado: ningún valor es igual a los dos a la vez.

**Después:**

```sql
SELECT PedidoId, ClienteId, FechaPedido, Estado
FROM dbo.Pedido
WHERE FechaPedido >= '2026-04-01'
  AND FechaPedido <  '2026-06-01'
  AND Estado NOT IN ('Cancelado', 'Pendiente');
```

| PedidoId | ClienteId | FechaPedido | Estado |
| --- | --- | --- | --- |
| 3 | 1 | 2026-04-12 09:05:00 | Pagado |

Qué cambió y por qué:

* **Rango semiabierto** en lugar de `BETWEEN`: incluye todo el 31 de mayo, a cualquier hora.
* **`NOT IN` con la lista de exclusiones:** expresa "ni esto ni aquello" sin el error lógico de los `OR`. Es seguro porque la lista no tiene `NULL` y `Estado` es `NOT NULL`.
* **Todo unido con `AND`:** sin `OR` sueltos, no hace falta pensar en la precedencia.

-----

## Errores comunes

**1. Mezclar `AND` y `OR` sin paréntesis.**
Qué pasa: filas que no deberían estar (el monitor de 899.90 en "baratos").
Por qué: `AND` se evalúa antes que `OR`.
Arreglo: paréntesis siempre que se mezclen.

**2. `BETWEEN` con fechas y horas.**
Qué pasa: se pierden las filas del último día después de la medianoche.
Por qué: `'2026-03-31'` es `2026-03-31 00:00:00`, y `BETWEEN` incluye solo hasta ese instante.
Arreglo: `>= inicio AND < día siguiente al final`.

**3. "Ni A ni B" escrito con `OR`.**
Qué pasa: `Estado <> 'A' OR Estado <> 'B'` es verdadero para todas las filas.
Por qué: ningún valor es igual a A y a B a la vez, así que una de las dos desigualdades siempre se cumple.
Arreglo: `Estado <> 'A' AND Estado <> 'B'`, o `Estado NOT IN ('A', 'B')`.

**4. Buscar `%` o `_` literales con `LIKE`.**
Qué pasa: `LIKE '%50%%'` encuentra cualquier texto con un 50.
Por qué: `%` es un comodín.
Arreglo: `[%]` o `ESCAPE`.

**5. Funciones sobre la columna filtrada.**
Qué pasa: `WHERE YEAR(FechaPedido) = 2026` funciona, pero recorre toda la tabla.
Por qué: el motor tiene que calcular la función en cada fila antes de comparar.
Arreglo: reescribe la condición como un rango sobre la columna sin transformar.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| `LIKE` y mayúsculas | Según la intercalación (normalmente no distingue) | **Distingue**; `ILIKE` no distingue | Según la intercalación (normalmente no distingue) |
| `[abc]` en `LIKE` | Sí | No (usar `SIMILAR TO` o `~` con expresiones regulares) | No (usar `REGEXP`) |
| Expresiones regulares | `REGEXP_LIKE` (desde 2025) | `~`, `~*`, `SIMILAR TO` | `REGEXP`, `REGEXP_LIKE` |
| Carácter de escape por defecto en `LIKE` | Ninguno (usar `ESCAPE` o `[ ]`) | `\` | `\` |
| `IS DISTINCT FROM` | Desde 2022 | Sí | `<=>` |

El punto que más sorprende al pasar de SQL Server a PostgreSQL: `WHERE nombre LIKE 'ana%'` no encuentra "Ana Torres", porque PostgreSQL distingue mayúsculas en `LIKE`. Se usa `ILIKE` o se compara en minúsculas (`LOWER(nombre) LIKE 'ana%'`).

### En MongoDB

Los filtros son el primer argumento de `find`, con operadores que empiezan con `$`:

```javascript
db.productos.find({
  categoriaId: { $in: [1, 2] },        // IN
  precio: { $gte: 75, $lte: 120 },     // BETWEEN
  nombre: { $regex: "^C" }             // LIKE 'C%'
});
```

Varias condiciones en el mismo objeto se combinan con `AND`; para `OR` se usa `$or: [ {...}, {...} ]`. No hay lógica de tres valores: `{ ciudad: { $ne: "Lima" } }` **sí** devuelve los documentos sin ciudad.

-----

## Cuándo sí y cuándo no

**Usa `BETWEEN` cuando:**

* El rango es de valores discretos o sin parte horaria (números, columnas `DATE`).

**Usa rangos semiabiertos cuando:**

* Filtras por fecha y hora: siempre.

**Usa `LIKE` cuando:**

* Buscas por prefijo o patrón simple. Para búsquedas de texto libre en tablas grandes ("contiene la palabra..."), existen los índices de texto completo (*full-text search*).

**Ten en cuenta:**

* Cada `<>`, `NOT IN` o `NOT LIKE` sobre una columna que admite `NULL` necesita una decisión explícita sobre los nulos.

-----

## Resumen en 5 líneas

1. `WHERE` solo deja pasar las filas con condición verdadera; lo falso y lo desconocido se descartan.
2. Precedencia: `NOT`, `AND`, `OR`. Si mezclas `AND` y `OR`, usa paréntesis.
3. `BETWEEN` incluye los dos extremos; para fechas con hora, usa `>= inicio AND < fin`.
4. `LIKE`: `%` es cualquier cantidad de caracteres, `_` es uno; los literales se escapan con `[%]` o `ESCAPE`.
5. Nulos con `IS NULL`; y deja la columna sin funciones para que el filtro pueda usar índices.

-----

## Para profundizar

<details>
<summary>Cómo interpreta SQL Server los literales de fecha</summary>

`'2026-03-01'` se interpreta igual con cualquier configuración de idioma cuando la columna es `DATE` o `DATETIME2`. Pero con el tipo antiguo `DATETIME`, el formato `'2026-03-01'` **sí** depende de `SET DATEFORMAT`: con el idioma configurado en un orden día-año-mes, puede leerse como el 3 de enero. Los formatos realmente independientes del idioma son `'20260301'` (sin separadores) y `'2026-03-01T10:15:00'` (con la `T`). Si trabajas con columnas `DATETIME` heredadas, usa esos formatos.

</details>

<details>
<summary>Cortocircuito: el orden de las condiciones no está garantizado</summary>

En C#, `if (x != null && x.Length > 0)` nunca evalúa la segunda parte si la primera es falsa. En SQL **no** hay esa garantía: el optimizador puede evaluar las condiciones de un `WHERE` en cualquier orden. Por eso `WHERE Cantidad <> 0 AND Total / Cantidad > 10` puede fallar con una división por cero. La forma segura es proteger la expresión en sí: `Total / NULLIF(Cantidad, 0) > 10`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`WHERE` filtra las filas con condiciones como `=`, `<>`, `>`, `BETWEEN`, `IN` y `LIKE`, que se combinan con `AND` y `OR`. Para preguntar si algo es nulo se usa `IS NULL`. `AND` tiene más precedencia que `OR`, así que uso paréntesis cuando los mezclo.

### Respuesta ampliada (semi-senior)

`WHERE` solo deja pasar las filas con predicado verdadero, así que cada `<>` o `NOT IN` sobre columnas nullables necesita una decisión explícita sobre los nulos, y `NOT IN` con una subconsulta lo reemplazo por `NOT EXISTS`. Uso paréntesis siempre que mezclo `AND` y `OR`. Para fechas con hora uso rangos semiabiertos (`>= inicio AND < fin`) en lugar de `BETWEEN`, que pierde las filas del último día. Escribo los filtros de forma SARGable, sin funciones sobre la columna (por ejemplo, un rango en lugar de `YEAR(Fecha) = 2026`), para que puedan usar índices, y sé que `LIKE` con un comodín inicial obliga a recorrer la tabla. Tampoco asumo cortocircuito en las condiciones: protejo las divisiones con `NULLIF`.

### Preguntas frecuentes de seguimiento

**1. ¿`BETWEEN` incluye los extremos?**
Sí, los dos. Por eso con fechas y horas conviene un rango semiabierto.

**2. ¿Qué precedencia tienen `AND` y `OR`?**
`AND` se evalúa antes que `OR` (y `NOT` antes que los dos).

**3. ¿Cómo buscas un `%` literal con `LIKE`?**
Con `[%]` en SQL Server, o con la cláusula `ESCAPE`.

**4. ¿Qué es una condición SARGable?**
Una que deja la columna sin transformar de un lado de la comparación, de modo que el motor puede usar un índice para resolverla.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta para cada pedido de información:

1. Productos activos con stock entre 10 y 20 unidades.
2. Clientes cuyo email empieza con una letra entre la `a` y la `l`.
3. Pedidos realizados en abril de 2026.
4. Clientes que no son de Lima ni de Cusco, incluidos los que no informaron ciudad.

<details>
<summary>Solución</summary>

```sql
-- 1
SELECT Nombre, Stock FROM dbo.Producto
WHERE Activo = 1 AND Stock BETWEEN 10 AND 20;

-- 2
SELECT Nombre, Email FROM dbo.Cliente
WHERE Email LIKE '[a-l]%';

-- 3
SELECT PedidoId, FechaPedido FROM dbo.Pedido
WHERE FechaPedido >= '2026-04-01' AND FechaPedido < '2026-05-01';

-- 4
SELECT Nombre, Ciudad FROM dbo.Cliente
WHERE Ciudad NOT IN (N'Lima', N'Cusco') OR Ciudad IS NULL;
```

Resultados:

1. Teclado mecánico (15), Lámpara de escritorio (20), Clean Code (12).
2. Ana Torres, Jorge Díaz, Luis Ramos (las `a`, `j` y `l`; María y Sofía quedan afuera).
3. Pedidos 3 y 4.
4. Luis Ramos (Arequipa) y María Quispe (`NULL`).

Ojo con el 2: un rango como `[a-l]` depende de la intercalación; con las intercalaciones habituales funciona como se espera para letras sin tilde.

</details>

**Ejercicio 2.** Esta consulta debería devolver los productos de la categoría 3 o los que tienen stock 0, pero solo si están activos. ¿Qué devuelve y cómo la corriges?

```sql
SELECT Nombre FROM dbo.Producto
WHERE CategoriaId = 3 OR Stock = 0 AND Activo = 1;
```

<details>
<summary>Solución</summary>

Por la precedencia, se lee como `CategoriaId = 3 OR (Stock = 0 AND Activo = 1)`. Devuelve **Clean Code** (categoría 3). La cafetera tiene stock 0 pero `Activo = 0`, así que no entra por la segunda parte.

Lo pedido es `(CategoriaId = 3 OR Stock = 0) AND Activo = 1`:

```sql
SELECT Nombre FROM dbo.Producto
WHERE (CategoriaId = 3 OR Stock = 0) AND Activo = 1;
```

Con estos datos, el resultado coincide (Clean Code), y ahí está el peligro: la consulta original "funciona" hasta que aparece un producto inactivo de la categoría 3, que la versión sin paréntesis incluiría.

</details>

-----

## Siguiente lección

[Ordenar y paginar](03-Ordenar%20y%20paginar.md)
