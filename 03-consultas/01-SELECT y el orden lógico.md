# SELECT y el orden lógico

## En una frase

`SELECT` describe **qué columnas** quieres y `FROM` **de dónde** salen; lo importante es que el motor no procesa la consulta en el orden en que se escribe (`SELECT` primero) sino en un **orden lógico** (`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`), y ese orden explica la mitad de los errores de SQL.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo conectarte y pararte en la base `Tienda`, de [Entorno de trabajo](00-Entorno%20de%20trabajo.md).
* Que una consulta recibe tablas y devuelve una tabla, de [El modelo relacional](../01-fundamentos/02-El%20modelo%20relacional.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Consulta (query):** instrucción que pide datos a la base.
* **Cláusula:** cada parte de una consulta (`SELECT`, `FROM`, `WHERE`...).
* **Alias:** nombre alternativo para una columna o tabla, con `AS`.
* **`DISTINCT`:** elimina las filas repetidas del resultado.
* **Orden lógico de procesamiento:** el orden conceptual en que el motor evalúa las cláusulas, distinto del orden en que se escriben.

-----

## El problema

Quieres los productos con su precio con IGV (18 %) y quedarte con los que superan los 200 soles:

```sql
SELECT Nombre, Precio * 1.18 AS PrecioConIgv
FROM dbo.Producto
WHERE PrecioConIgv > 200;
```

```text
Msg 207, Level 16, State 1
Invalid column name 'PrecioConIgv'.
```

Pero si usas el mismo alias en el `ORDER BY`, funciona:

```sql
SELECT Nombre, Precio * 1.18 AS PrecioConIgv
FROM dbo.Producto
ORDER BY PrecioConIgv DESC;   -- ✅
```

¿Por qué el alias existe en el `ORDER BY` y no en el `WHERE`? No es un capricho de SQL Server, y la respuesta no está en la sintaxis: está en el **orden en que el motor procesa** las cláusulas.

-----

## Cómo funciona

### `SELECT` y `FROM`

```sql
SELECT Nombre, Precio        -- qué columnas
FROM dbo.Producto;           -- de qué tabla
```

| Nombre | Precio |
| --- | --- |
| Teclado mecánico | 120.00 |
| Mouse inalámbrico | 45.50 |
| Monitor 27 pulgadas | 899.90 |
| Lámpara de escritorio | 75.00 |
| Cafetera | 210.00 |
| Clean Code | 95.00 |

`SELECT *` devuelve todas las columnas. Es útil para explorar una tabla, pero en código de una aplicación conviene listar las columnas:

* Si alguien agrega una columna, `SELECT *` empieza a devolverla, y el código que lee por posición se rompe.
* Trae datos que no necesitas (columnas grandes, columnas sensibles).
* Impide que un índice "cubra" la consulta (se ve en el módulo de índices).

Las palabras clave **no distinguen mayúsculas**: `select`, `SELECT` y `Select` son lo mismo. Escribirlas en mayúsculas es una convención de legibilidad. Lo que sí puede distinguir mayúsculas son los **datos** (según la intercalación) y, en otros motores, los nombres de las tablas.

### Expresiones y alias

En la lista del `SELECT` puedes poner columnas, expresiones o valores fijos, y darle a cada uno un nombre con `AS`:

```sql
SELECT Nombre                 AS Producto,
       Precio                 AS PrecioSinIgv,
       Precio * 1.18          AS PrecioConIgv,
       N'PEN'                 AS Moneda,
       Stock * Precio         AS ValorInventario
FROM dbo.Producto;
```

| Producto | PrecioSinIgv | PrecioConIgv | Moneda | ValorInventario |
| --- | --- | --- | --- | --- |
| Teclado mecánico | 120.00 | 141.6000 | PEN | 1800.00 |
| Mouse inalámbrico | 45.50 | 53.6900 | PEN | 1820.00 |
| Monitor 27 pulgadas | 899.90 | 1061.8820 | PEN | 7199.20 |
| Lámpara de escritorio | 75.00 | 88.5000 | PEN | 1500.00 |
| Cafetera | 210.00 | 247.8000 | PEN | 0.00 |
| Clean Code | 95.00 | 112.1000 | PEN | 1140.00 |

* `AS` es opcional (`Precio Producto` también funciona), pero escribirlo evita confusiones: si olvidas una coma entre dos columnas, `SELECT Nombre Precio` devuelve una sola columna `Nombre` llamada "Precio", sin error.
* Un alias con espacios o con una palabra reservada va entre corchetes: `AS [Precio con IGV]`. Mejor evitarlo.
* La multiplicación `DECIMAL(10,2) * 1.18` devuelve 4 decimales: la escala del resultado se suma. El redondeo se ve en [Expresiones, CASE y funciones](04-Expresiones%2C%20CASE%20y%20funciones.md).

Las tablas también pueden tener alias, lo que se vuelve imprescindible con los joins:

```sql
SELECT p.Nombre, p.Precio
FROM dbo.Producto AS p;
```

### `DISTINCT`: sin filas repetidas

```sql
SELECT DISTINCT Ciudad
FROM dbo.Cliente;
```

| Ciudad |
| --- |
| `NULL` |
| Arequipa |
| Cusco |
| Lima |

Lima aparecía dos veces (Ana y Jorge) y ahora aparece una. Fíjate en el `NULL`: `DISTINCT` trata todos los `NULL` como un solo valor.

`DISTINCT` se aplica a la **fila completa** del resultado, no a la primera columna:

```sql
SELECT DISTINCT Ciudad, Nombre FROM dbo.Cliente;   -- 5 filas: ninguna combinación se repite
```

Y no es gratis: para eliminar duplicados, el motor tiene que ordenar o agrupar el resultado. Úsalo cuando el resultado de verdad puede tener duplicados y no los quieres, no "por las dudas".

### El orden lógico de procesamiento

Así se **escribe** una consulta completa:

```sql
SELECT   ...      -- 5
FROM     ...      -- 1
WHERE    ...      -- 2
GROUP BY ...      -- 3
HAVING   ...      -- 4
ORDER BY ...      -- 6
```

Y así se **procesa**, lógicamente:

| Paso | Cláusula | Qué hace | Qué ve |
| --- | --- | --- | --- |
| 1 | `FROM` (y los `JOIN`) | Arma el conjunto de filas de partida | Las tablas |
| 2 | `WHERE` | Filtra filas | Las columnas de las tablas |
| 3 | `GROUP BY` | Agrupa las filas | Las columnas de las tablas |
| 4 | `HAVING` | Filtra grupos | Los grupos y los agregados |
| 5 | `SELECT` | Calcula las expresiones y **crea los alias** | Todo lo anterior |
| 5b | `DISTINCT` | Elimina filas repetidas | El resultado del `SELECT` |
| 6 | `ORDER BY` | Ordena | **Los alias del `SELECT`** |
| 7 | `TOP` / `OFFSET` | Limita la cantidad de filas | El resultado ordenado |

Con esta tabla se explica el problema del principio: el `WHERE` (paso 2) se evalúa **antes** que el `SELECT` (paso 5), así que el alias `PrecioConIgv` todavía no existe. El `ORDER BY` (paso 6) se evalúa después, y ya lo ve.

Las soluciones:

```sql
-- 1. Repetir la expresión
SELECT Nombre, Precio * 1.18 AS PrecioConIgv
FROM dbo.Producto
WHERE Precio * 1.18 > 200;

-- 2. Calcularla en un paso previo (tabla derivada o CTE)
SELECT Nombre, PrecioConIgv
FROM (SELECT Nombre, Precio * 1.18 AS PrecioConIgv FROM dbo.Producto) AS t
WHERE PrecioConIgv > 200;
```

| Nombre | PrecioConIgv |
| --- | --- |
| Monitor 27 pulgadas | 1061.8820 |
| Cafetera | 247.8000 |

La segunda forma se ve en [Subconsultas](08-Subconsultas.md) y [CTE](09-CTE.md).

Ojo con una aclaración: el orden **lógico** define qué ve cada cláusula y qué resultado se obtiene. El orden **físico** (cómo el motor ejecuta realmente la consulta) lo decide el optimizador, que puede filtrar mientras lee, usar un índice o reordenar los joins, siempre que el resultado sea el mismo.

### La consulta como respuesta a preguntas

Una forma práctica de escribir una consulta es responder una pregunta por cláusula:

| Cláusula | Pregunta |
| --- | --- |
| `SELECT` | ¿Qué quiero mostrar? |
| `FROM` | ¿De dónde salen los datos? |
| `WHERE` | ¿Qué filas me interesan? |
| `GROUP BY` | ¿Por qué quiero agrupar? |
| `HAVING` | ¿Qué grupos me interesan? |
| `ORDER BY` | ¿En qué orden lo presento? |

-----

## Ejemplo completo

Reporte de inventario: productos activos con su valor en stock, del más valioso al menos valioso, con nombres de columna legibles.

```sql
SELECT p.Nombre              AS Producto,
       p.Stock               AS Unidades,
       p.Precio              AS PrecioUnitario,
       p.Stock * p.Precio    AS ValorInventario
FROM dbo.Producto AS p
WHERE p.Activo = 1
ORDER BY ValorInventario DESC;
```

| Producto | Unidades | PrecioUnitario | ValorInventario |
| --- | --- | --- | --- |
| Monitor 27 pulgadas | 8 | 899.90 | 7199.20 |
| Mouse inalámbrico | 40 | 45.50 | 1820.00 |
| Teclado mecánico | 15 | 120.00 | 1800.00 |
| Lámpara de escritorio | 20 | 75.00 | 1500.00 |
| Clean Code | 12 | 95.00 | 1140.00 |

Recorrido en orden lógico:

1. **`FROM`:** las 6 filas de `Producto`, con el alias `p`.
2. **`WHERE`:** se descarta la cafetera (`Activo = 0`); quedan 5.
3. **`SELECT`:** se calcula `ValorInventario` y se crean los alias.
4. **`ORDER BY`:** se ordena usando el alias, que ya existe.

-----

## Errores comunes

**1. Usar un alias del `SELECT` en el `WHERE`.**
Qué pasa: `Msg 207: Invalid column name`.
Por qué: el `WHERE` se procesa antes que el `SELECT`.
Arreglo: repite la expresión, o calcúlala en una tabla derivada o una CTE.

**2. Olvidar una coma entre columnas.**
Qué pasa: `SELECT Nombre Precio FROM ...` devuelve una sola columna, `Nombre`, con el título "Precio".
Por qué: sin coma, la segunda palabra se interpreta como alias.
Arreglo: escribe siempre `AS` en los alias; así una palabra suelta salta a la vista.

**3. `SELECT *` en código de aplicación.**
Qué pasa: el código se rompe o se vuelve lento cuando alguien agrega columnas a la tabla.
Por qué: el resultado depende de la estructura actual de la tabla.
Arreglo: lista las columnas que necesitas.

**4. `DISTINCT` para tapar duplicados de un join mal hecho.**
Qué pasa: el resultado "se ve bien", pero la consulta hace trabajo de más y esconde el error de fondo (un join que multiplica filas).
Por qué: se elimina el síntoma sin entender la causa.
Arreglo: entiende por qué aparecen los duplicados (ver [JOIN](06-JOIN.md)) y corrige el join.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Alias con espacios | `[Precio con IGV]` o `"..."` | `"Precio con IGV"` | `` `Precio con IGV` `` o `"..."` |
| Alias del `SELECT` en `WHERE` | No | No | No |
| Alias del `SELECT` en `GROUP BY` | **No** | Sí | Sí |
| Alias del `SELECT` en `HAVING` | No | No | Sí |
| `SELECT` sin `FROM` | Sí | Sí | Sí (o `FROM DUAL`) |
| Mayúsculas en nombres de tabla | Según la intercalación | Se pasan a minúsculas sin comillas | Según el sistema operativo |

PostgreSQL y MySQL son más permisivos con los alias en `GROUP BY` porque los resuelven de forma especial, pero el orden lógico es el mismo. Si tu SQL tiene que funcionar en varios motores, no uses alias fuera del `ORDER BY`.

### En MongoDB

El equivalente de `SELECT` es la **proyección** de `find`:

```javascript
db.productos.find(
  { activo: true },                 // equivalente al WHERE
  { nombre: 1, precio: 1, _id: 0 }  // proyección: qué campos devolver
);
```

Para expresiones calculadas (el precio con IGV) se usa el *aggregation pipeline*, con una etapa `$project` o `$addFields`. En el *pipeline*, el orden **sí** es el que escribes: cada etapa recibe la salida de la anterior.

-----

## Cuándo sí y cuándo no

**Usa `SELECT *` cuando:**

* Exploras una tabla por primera vez, en una consulta que nadie más va a usar.

**Lista las columnas cuando:**

* La consulta va en una aplicación, un reporte o una vista: siempre.

**Usa `DISTINCT` cuando:**

* La pregunta es literalmente "¿qué valores distintos hay?" (las ciudades de los clientes).

**Ten en cuenta:**

* Pensar en el orden lógico antes de escribir evita la mayoría de los errores de "columna inválida" y de alias.

-----

## Resumen en 5 líneas

1. `SELECT` elige columnas y expresiones; `FROM` dice de qué tabla; `AS` les da nombre.
2. Las palabras clave no distinguen mayúsculas; los datos, según la intercalación, sí pueden.
3. `DISTINCT` elimina filas repetidas completas y trata todos los `NULL` como uno.
4. Orden lógico: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `TOP`.
5. Por eso los alias del `SELECT` no existen en el `WHERE` pero sí en el `ORDER BY`.

-----

## Para profundizar

<details>
<summary>Por qué SQL se escribe en un orden y se procesa en otro</summary>

SQL se diseñó en los años 70 para parecerse al inglés ("select name from customers where..."), no para reflejar el orden de evaluación. Lenguajes más recientes invierten el orden para que coincida con el procesamiento: LINQ en C# empieza por `from` (`from p in productos where p.Activo select p.Nombre`), y por eso el IntelliSense puede sugerir las columnas mientras escribes el `select`. Si ya usaste LINQ, el orden lógico de SQL te va a resultar familiar.

</details>

<details>
<summary>SELECT sin FROM y las funciones del sistema</summary>

En SQL Server, `SELECT` puede evaluar expresiones sin tabla: `SELECT 1 + 1`, `SELECT GETDATE()`, `SELECT @@VERSION`. Es la forma habitual de probar una función o una expresión antes de usarla en una consulta real. Oracle exige `FROM DUAL` (una tabla de una sola fila), y MySQL la acepta por compatibilidad.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`SELECT` indica las columnas que quiero y `FROM` la tabla. Puedo calcular expresiones y darles nombre con `AS`, y usar `DISTINCT` para no repetir filas. Aunque se escribe primero, el `SELECT` se procesa después del `FROM`, el `WHERE` y el `GROUP BY`; por eso no puedo usar un alias en el `WHERE`.

### Respuesta ampliada (semi-senior)

El orden lógico de procesamiento es `FROM` (con los joins), `WHERE`, `GROUP BY`, `HAVING`, `SELECT`, `DISTINCT`, `ORDER BY` y finalmente `TOP` u `OFFSET`. De ahí salen varias reglas: los alias del `SELECT` solo existen en el `ORDER BY`; el `WHERE` filtra filas antes de agrupar y el `HAVING`, grupos después; y las funciones de ventana, que se evalúan en el `SELECT`, no pueden usarse en el `WHERE`. Para reutilizar una expresión en un filtro, la calculo en una tabla derivada o una CTE. El orden lógico define la semántica; el físico lo decide el optimizador. En código de aplicación evito `SELECT *` por estabilidad y rendimiento, y no uso `DISTINCT` para tapar joins que multiplican filas.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no puedo usar un alias en el `WHERE`?**
Porque el `WHERE` se procesa antes que el `SELECT`, que es donde se crean los alias.

**2. ¿Por qué sí puedo usarlo en el `ORDER BY`?**
Porque el `ORDER BY` se procesa después del `SELECT`.

**3. ¿`DISTINCT` se aplica a una columna o a la fila?**
A la fila completa del resultado.

**4. ¿Qué problemas tiene `SELECT *`?**
Trae columnas innecesarias, cambia cuando cambia la tabla y puede impedir que un índice cubra la consulta.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que muestre el nombre de cada cliente y su email, con los títulos `Cliente` y `Correo`, solo para los clientes registrados desde marzo de 2026, ordenados por nombre.

<details>
<summary>Solución</summary>

```sql
SELECT Nombre AS Cliente,
       Email  AS Correo
FROM dbo.Cliente
WHERE FechaRegistro >= '2026-03-01'
ORDER BY Cliente;
```

| Cliente | Correo |
| --- | --- |
| Jorge Díaz | jorge.diaz@correo.com |
| Sofía Mendoza | sofia.mendoza@correo.com |

El `ORDER BY` puede usar el alias `Cliente`; el `WHERE` usa la columna original.

</details>

**Ejercicio 2.** ¿Qué devuelve esta consulta y por qué? Corrígela para que devuelva lo que su autor quería: las categorías distintas de los productos activos.

```sql
SELECT DISTINCT CategoriaId Nombre
FROM dbo.Producto
WHERE Activo = 1;
```

<details>
<summary>Solución</summary>

Devuelve una sola columna, llamada `Nombre`, con los valores **de `CategoriaId`**: falta una coma, así que `Nombre` se interpretó como alias.

| Nombre |
| --- |
| 1 |
| 2 |
| 3 |

Si lo que se quería eran las categorías distintas, la consulta correcta es:

```sql
SELECT DISTINCT CategoriaId
FROM dbo.Producto
WHERE Activo = 1;
```

Que devuelve 1, 2 y 3 con el título `CategoriaId`. (El resultado coincide por casualidad, porque cada categoría tiene al menos un producto activo; el título engañoso es lo que delata el error.) Para mostrar el **nombre** de la categoría hace falta un join, que se ve en [JOIN](06-JOIN.md).

</details>

-----

## Siguiente lección

[Filtrar con WHERE](02-Filtrar%20con%20WHERE.md)
