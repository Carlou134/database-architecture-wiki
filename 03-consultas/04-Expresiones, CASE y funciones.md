# Expresiones, CASE y funciones

## En una frase

Además de leer columnas, una consulta puede **calcular**: operaciones aritméticas, decisiones con `CASE`, y funciones de texto, fecha y conversión; cada motor tiene su propio catálogo de funciones, y en SQL Server conviene conocer sus trampas (división entera, `LEN` y los espacios, `FORMAT` lento, conversiones que fallan).

-----

## Antes de empezar

Conviene que ya sepas:

* Expresiones y alias en el `SELECT`, de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).
* Los tipos de SQL Server y la conversión implícita, de [Tipos de datos](../01-fundamentos/03-Tipos%20de%20datos.md).
* `COALESCE` y `NULL` en expresiones, de [NULL y la lógica de tres valores](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Expresión:** combinación de columnas, valores, operadores y funciones que produce un valor.
* **`CASE`:** expresión condicional: devuelve un valor distinto según una condición.
* **Función escalar:** función que recibe valores de una fila y devuelve un valor (`UPPER`, `YEAR`).
* **Precedencia de tipos:** regla que decide a qué tipo se convierte una expresión que mezcla tipos distintos.
* **`CAST` / `CONVERT` / `TRY_CAST`:** funciones de conversión de tipo; la versión `TRY_` devuelve `NULL` en lugar de fallar.

-----

## El problema

En tus apuntes de Codecademy (SQLite) hay consultas como estas:

```sql
SELECT strftime('%Y', fecha_pedido) AS anio FROM pedidos;
SELECT nombre || ' - ' || ciudad FROM clientes;
SELECT ROUND(AVG(precio), 2) FROM productos;
```

En SQL Server, la primera falla (`strftime` no existe), la segunda solo funciona desde SQL Server 2025 (que agregó `||`; en versiones anteriores es un error de sintaxis) y la tercera funciona, pero puede devolver un resultado inesperado si `precio` fuera entero. Y hay sorpresas que no dan error:

```sql
SELECT 5 / 2;                       -- 2, no 2.5
SELECT LEN('Lima   ');              -- 4, no 7
SELECT N'Total: ' + 100;            -- error: no puede convertir 'Total: ' a número
```

Las funciones son la parte de SQL que **más cambia entre motores**. Esta lección arma el catálogo de T-SQL que vas a usar a diario, con la traducción desde SQLite.

-----

## Cómo funciona

### Aritmética y la división entera

| Operador | Significado |
| --- | --- |
| `+`, `-`, `*`, `/` | Suma, resta, multiplicación, división |
| `%` | Resto de la división (módulo) |

La regla que sorprende: **si los dos operandos son enteros, la división es entera** (se descartan los decimales).

```sql
SELECT 5 / 2        AS Entera,       -- 2
       5 / 2.0      AS Decimal1,     -- 2.500000
       5 * 1.0 / 2  AS Decimal2,     -- 2.500000
       CAST(5 AS DECIMAL(10,2)) / 2 AS Decimal3;   -- 2.500000
```

Esto afecta a porcentajes y promedios: "pedidos entregados / pedidos totales" con dos `COUNT(*)` (enteros) da 0 en lugar de 0.2. Multiplica por `1.0` o convierte uno de los operandos.

Dividir por cero da error (`Msg 8134: Divide by zero error encountered`). La protección habitual es `NULLIF`, que convierte el divisor en `NULL` si es cero:

```sql
SELECT Total / NULLIF(Cantidad, 0)   -- NULL en lugar de error
```

### Concatenar texto

```sql
SELECT Nombre + N' (' + Ciudad + N')' AS ConMas,          -- NULL si Ciudad es NULL
       CONCAT(Nombre, N' (', Ciudad, N')') AS ConConcat,  -- 'María Quispe ()' si Ciudad es NULL
       CONCAT_WS(N' - ', Nombre, Ciudad, Email) AS ConSeparador
FROM dbo.Cliente
WHERE ClienteId IN (1, 3);
```

| ConMas | ConConcat | ConSeparador |
| --- | --- | --- |
| Ana Torres (Lima) | Ana Torres (Lima) | Ana Torres - Lima - ana.torres@correo.com |
| `NULL` | María Quispe () | María Quispe - maria.quispe@correo.com |

* `+` propaga el `NULL` y exige que todo sea texto (`N'Total: ' + 100` falla porque intenta convertir el texto a número, por precedencia de tipos).
* `CONCAT` trata los `NULL` como texto vacío y convierte los números automáticamente.
* `CONCAT_WS` (desde 2017) pone un separador entre los valores y **omite** los `NULL` sin dejar separadores dobles.

### `CASE`: decisiones dentro de una expresión

Tiene dos formas. La **buscada**, con una condición por rama:

```sql
SELECT Nombre, Precio,
       CASE
           WHEN Precio >= 500 THEN N'Premium'
           WHEN Precio >= 100 THEN N'Medio'
           ELSE N'Económico'
       END AS Segmento
FROM dbo.Producto;
```

| Nombre | Precio | Segmento |
| --- | --- | --- |
| Teclado mecánico | 120.00 | Medio |
| Mouse inalámbrico | 45.50 | Económico |
| Monitor 27 pulgadas | 899.90 | Premium |
| Lámpara de escritorio | 75.00 | Económico |
| Cafetera | 210.00 | Medio |
| Clean Code | 95.00 | Económico |

Y la **simple**, que compara una expresión con valores:

```sql
SELECT PedidoId,
       CASE Estado
           WHEN 'Pendiente' THEN N'Esperando pago'
           WHEN 'Cancelado' THEN N'Anulado'
           ELSE N'En curso o terminado'
       END AS Situacion
FROM dbo.Pedido;
```

Reglas de `CASE`:

* Las ramas se evalúan **en orden** y gana la primera verdadera. Por eso el monitor (899.90) es "Premium" y no "Medio", aunque también cumple `>= 100`.
* Sin `ELSE`, las filas que no cumplen ninguna rama dan `NULL`.
* Todas las ramas deben devolver **tipos compatibles**: el resultado toma el tipo de mayor precedencia. `CASE WHEN ... THEN 'Sin stock' ELSE Stock END` falla, porque intenta convertir `'Sin stock'` a `INT`. Convierte la rama numérica: `ELSE CAST(Stock AS NVARCHAR(10))`.
* `CASE` es una **expresión**, no una instrucción: puede ir en el `SELECT`, en el `WHERE`, en el `ORDER BY`, dentro de un `SUM` (agregación condicional, en la [próxima lección](05-Agregados%20y%20GROUP%20BY.md)).

`IIF(condición, si, no)` (desde 2012) es un atajo para un `CASE` de dos ramas: `IIF(Stock = 0, N'Agotado', N'Disponible')`.

### Funciones de texto

| Función | Qué hace | Ejemplo | Resultado |
| --- | --- | --- | --- |
| `LEN(s)` | Longitud, **sin** los espacios finales | `LEN(N'Lima   ')` | 4 |
| `DATALENGTH(s)` | Bytes que ocupa | `DATALENGTH(N'Lima')` | 8 (NVARCHAR: 2 bytes por carácter) |
| `UPPER(s)`, `LOWER(s)` | Mayúsculas, minúsculas | `UPPER(N'ana')` | ANA |
| `TRIM(s)` | Quita espacios al inicio y al final (desde 2017) | `TRIM(N'  Lima ')` | Lima |
| `LTRIM(s)`, `RTRIM(s)` | Quita espacios a la izquierda / derecha | | |
| `LEFT(s, n)`, `RIGHT(s, n)` | Primeros / últimos n caracteres | `LEFT(N'Teclado', 3)` | Tec |
| `SUBSTRING(s, inicio, n)` | n caracteres desde la posición inicio (empieza en 1) | `SUBSTRING(N'Teclado', 2, 3)` | ecl |
| `CHARINDEX(buscar, s)` | Posición de la primera aparición (0 si no está) | `CHARINDEX(N'@', N'ana@x.com')` | 4 |
| `REPLACE(s, viejo, nuevo)` | Reemplaza todas las apariciones | `REPLACE(N'a-b-c', N'-', N'/')` | a/b/c |
| `REPLICATE(s, n)` | Repite n veces | `REPLICATE(N'0', 3)` | 000 |
| `STRING_SPLIT(s, sep)` | Divide en filas (desde 2016) | Ver la nota sobre 1FN en [Normalización](../02-diseno-y-modelado/06-Normalización%201FN%202FN%203FN.md) | |

Ejemplo: separar el usuario y el dominio de un email.

```sql
SELECT Email,
       LEFT(Email, CHARINDEX('@', Email) - 1)               AS Usuario,
       SUBSTRING(Email, CHARINDEX('@', Email) + 1, 254)     AS Dominio
FROM dbo.Cliente
WHERE ClienteId = 1;
```

| Email | Usuario | Dominio |
| --- | --- | --- |
| ana.torres@correo.com | ana.torres | correo.com |

### Funciones de fecha y hora

| Función | Qué hace | Ejemplo con `'2026-04-12T09:05:00'` |
| --- | --- | --- |
| `SYSDATETIME()`, `GETDATE()` | Fecha y hora actual del servidor | |
| `SYSUTCDATETIME()` | Fecha y hora actual en UTC | |
| `YEAR(f)`, `MONTH(f)`, `DAY(f)` | Partes de la fecha | 2026, 4, 12 |
| `DATEPART(parte, f)` | Cualquier parte: `quarter`, `week`, `weekday`, `hour`... | `DATEPART(quarter, f)` → 2 |
| `DATENAME(parte, f)` | El nombre de la parte (según el idioma de la sesión) | `DATENAME(month, f)` → April / abril |
| `DATEADD(parte, n, f)` | Suma n unidades | `DATEADD(day, 30, f)` → 2026-05-12 09:05 |
| `DATEDIFF(parte, f1, f2)` | Cantidad de **límites** cruzados entre dos fechas | `DATEDIFF(day, '2026-03-01', f)` → 42 |
| `EOMONTH(f)` | Último día del mes | 2026-04-30 |
| `DATEFROMPARTS(a, m, d)` | Arma una fecha | `DATEFROMPARTS(2026, 4, 1)` → 2026-04-01 |
| `DATETRUNC(parte, f)` | Trunca a la parte indicada (desde 2022) | `DATETRUNC(month, f)` → 2026-04-01 00:00 |
| `CAST(f AS DATE)` | Quita la hora | 2026-04-12 |

Un detalle de `DATEDIFF`: cuenta **límites cruzados**, no períodos completos. `DATEDIFF(year, '2025-12-31', '2026-01-01')` da 1, aunque pasó un día. Por eso calcular la edad con `DATEDIFF(year, ...)` necesita una corrección (está en [Del modelo ER a tablas](../02-diseno-y-modelado/04-Del%20modelo%20ER%20a%20tablas.md)).

### Conversión: `CAST`, `CONVERT`, `TRY_CAST` y `FORMAT`

```sql
SELECT CAST(Precio AS INT)                              AS PrecioEntero,      -- trunca: 899.90 → 899
       CAST(Precio AS NVARCHAR(20))                     AS PrecioTexto,
       CONVERT(VARCHAR(10), SYSDATETIME(), 23)          AS FechaIso,          -- 'yyyy-mm-dd'
       CONVERT(VARCHAR(10), SYSDATETIME(), 103)         AS FechaPeru,         -- 'dd/mm/yyyy'
       TRY_CAST(N'abc' AS INT)                          AS ConversionFallida  -- NULL, sin error
FROM dbo.Producto
WHERE ProductoId = 3;
```

| Función | Para qué | Nota |
| --- | --- | --- |
| `CAST(x AS tipo)` | Conversión estándar | Falla si no se puede convertir |
| `CONVERT(tipo, x, estilo)` | Conversión con **estilo** de formato para fechas | Estilos frecuentes: 23 (`yyyy-mm-dd`), 103 (`dd/mm/yyyy`), 120 (`yyyy-mm-dd hh:mi:ss`) |
| `TRY_CAST`, `TRY_CONVERT` | Conversión que devuelve `NULL` si falla | Útil para limpiar datos importados |
| `FORMAT(x, patrón, cultura)` | Formato flexible al estilo .NET | `FORMAT(Precio, 'N2', 'es-PE')`. **Lento**: evítalo en consultas sobre muchas filas |
| `ROUND(x, n)` | Redondea a n decimales | `ROUND(899.95, 1)` → 900.00 |

Una regla de arquitectura: el **formato de presentación** (separadores de miles, símbolo de moneda, nombre del mes) pertenece a la aplicación, no a la consulta. La consulta devuelve números y fechas tipados; la interfaz los formatea según el idioma del usuario.

### De SQLite a T-SQL

| SQLite (tus apuntes) | SQL Server |
| --- | --- |
| `strftime('%Y', fecha)` | `YEAR(fecha)` o `DATEPART(year, fecha)` |
| `strftime('%m', fecha)` | `MONTH(fecha)` |
| `strftime('%Y-%m', fecha)` | `FORMAT(fecha, 'yyyy-MM')` o `CONVERT(CHAR(7), fecha, 23)` |
| `strftime('%H', fecha)` | `DATEPART(hour, fecha)` |
| `date('now')` | `CAST(SYSDATETIME() AS DATE)` |
| `a \|\| b` | `a + b` o `CONCAT(a, b)` (`\|\|` solo desde SQL Server 2025) |
| `length(s)` | `LEN(s)` |
| `substr(s, i, n)` | `SUBSTRING(s, i, n)` |
| `instr(s, x)` | `CHARINDEX(x, s)` (¡argumentos al revés!) |
| `ifnull(a, b)` | `COALESCE(a, b)` o `ISNULL(a, b)` |
| `LIMIT n` | `TOP (n)` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col, ',')` |

-----

## Ejemplo completo

Un listado de pedidos para el área comercial, con todo el cálculo hecho en la consulta:

```sql
SELECT p.PedidoId,
       CAST(p.FechaPedido AS DATE)                           AS Fecha,
       DATENAME(weekday, p.FechaPedido)                      AS DiaSemana,
       CONCAT(N'T', DATEPART(quarter, p.FechaPedido), N'-', YEAR(p.FechaPedido)) AS Trimestre,
       DATEDIFF(day, p.FechaPedido, '2026-06-01')            AS DiasDesdePedido,
       CASE
           WHEN p.Estado IN ('Entregado', 'Enviado') THEN N'Despachado'
           WHEN p.Estado = 'Cancelado'               THEN N'Anulado'
           ELSE N'Por despachar'
       END                                                   AS Situacion
FROM dbo.Pedido AS p
ORDER BY p.FechaPedido;
```

Con la sesión en español (`SET LANGUAGE Spanish;`):

| PedidoId | Fecha | DiaSemana | Trimestre | DiasDesdePedido | Situacion |
| --- | --- | --- | --- | --- | --- |
| 1 | 2026-03-01 | domingo | T1-2026 | 92 | Despachado |
| 2 | 2026-03-05 | jueves | T1-2026 | 88 | Despachado |
| 3 | 2026-04-12 | domingo | T2-2026 | 50 | Por despachar |
| 4 | 2026-04-20 | lunes | T2-2026 | 42 | Anulado |
| 5 | 2026-05-02 | sábado | T2-2026 | 30 | Por despachar |

Qué decisión hay en cada columna:

* **`CAST(... AS DATE)`:** quita la hora sin convertir a texto; la columna sigue siendo una fecha, que la aplicación puede formatear.
* **`DATENAME(weekday, ...)`:** depende del idioma de la sesión (`domingo` o `Sunday`). Si el resultado lo consume una aplicación, conviene devolver `DATEPART(weekday, ...)` (un número) y traducir en la interfaz.
* **`CONCAT` con números:** convierte el trimestre y el año a texto sin `CAST`.
* **Fecha fija en `DATEDIFF`:** para que el ejemplo sea reproducible; en un reporte real iría `SYSDATETIME()`.
* **`CASE` con `IN`:** agrupa estados en categorías de negocio sin tocar los datos.

-----

## Errores comunes

**1. División entera sin querer.**
Qué pasa: `COUNT(*) / 3` o `Entregados / Total` devuelven 0 o un número truncado.
Por qué: entero dividido entero da entero.
Arreglo: `* 1.0`, o `CAST` de un operando a `DECIMAL`.

**2. Concatenar con `+` columnas que pueden ser `NULL`.**
Qué pasa: la expresión completa da `NULL`.
Por qué: `+` propaga el `NULL`.
Arreglo: `CONCAT` o `CONCAT_WS`.

**3. Ramas de `CASE` con tipos incompatibles.**
Qué pasa: `Conversion failed when converting the nvarchar value 'Sin stock' to data type int`.
Por qué: el resultado de `CASE` toma el tipo de mayor precedencia entre las ramas.
Arreglo: convierte las ramas a un tipo común.

**4. `FORMAT` sobre miles de filas.**
Qué pasa: una consulta que tarda varias veces más que la misma sin `FORMAT`.
Por qué: `FORMAT` usa el motor de formato de .NET fila por fila.
Arreglo: `CONVERT` con estilo, o formatear en la aplicación.

**5. Funciones de otro motor.**
Qué pasa: `'strftime' is not a recognized built-in function name`.
Por qué: el catálogo de funciones no es estándar.
Arreglo: la tabla de traducción de esta lección, y siempre indicar el motor al buscar ayuda (o al pedírsela a una IA, ver [SQL con IA](12-SQL%20con%20IA.md)).

**6. `CAST` a `INT` para "redondear".**
Qué pasa: `CAST(899.90 AS INT)` da 899, no 900.
Por qué: la conversión a entero **trunca**.
Arreglo: `ROUND(x, 0)` si quieres redondear.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| División entera `5 / 2` | 2 | 2 | **2.5000** (MySQL siempre devuelve decimal con `/`; `DIV` es la entera) |
| Concatenar | `+`, `CONCAT` (y `\|\|` desde 2025) | `\|\|`, `concat` | `CONCAT` (`\|\|` es `OR` por defecto) |
| Longitud | `LEN` (sin espacios finales) | `length` / `char_length` | `CHAR_LENGTH` (caracteres), `LENGTH` (bytes) |
| Posición de un texto | `CHARINDEX(x, s)` | `position(x in s)`, `strpos(s, x)` | `LOCATE(x, s)`, `INSTR(s, x)` |
| Año de una fecha | `YEAR(f)` | `EXTRACT(YEAR FROM f)` | `YEAR(f)` |
| Sumar días | `DATEADD(day, 7, f)` | `f + INTERVAL '7 days'` | `DATE_ADD(f, INTERVAL 7 DAY)` |
| Diferencia en días | `DATEDIFF(day, a, b)` | `b::date - a::date` | `DATEDIFF(b, a)` (¡orden inverso!) |
| Truncar al mes | `DATETRUNC(month, f)` | `date_trunc('month', f)` | `DATE_FORMAT(f, '%Y-%m-01')` |
| Formatear | `FORMAT`, `CONVERT` con estilo | `to_char(f, 'YYYY-MM')` | `DATE_FORMAT(f, '%Y-%m')` |
| Conversión segura | `TRY_CAST` | No nativo (se usa una función propia o una expresión regular) | No nativo |

`DATEDIFF` es la trampa clásica al migrar: en SQL Server lleva la unidad y el orden es `(inicio, fin)`; en MySQL no lleva unidad (siempre días) y el orden es `(fin, inicio)`.

### En MongoDB

Las expresiones se escriben como operadores del *aggregation pipeline*, dentro de una etapa como `$project`:

```javascript
db.productos.aggregate([
  { $project: {
      nombre: 1,
      segmento: {
        $switch: {
          branches: [
            { case: { $gte: ["$precio", 500] }, then: "Premium" },
            { case: { $gte: ["$precio", 100] }, then: "Medio" }
          ],
          default: "Económico"
        }
      },
      anio: { $year: "$fechaRegistro" }
  } }
]);
```

`$switch` y `$cond` son el equivalente de `CASE`; `$concat`, `$toUpper`, `$substr`, `$year`, `$dateAdd` y `$dateDiff`, los de las funciones de texto y fecha.

-----

## Cuándo sí y cuándo no

**Calcula en la consulta cuando:**

* El cálculo reduce los datos que viajan (filtrar, agregar, clasificar con `CASE`).
* El resultado se usa para ordenar o filtrar.

**Deja el cálculo para la aplicación cuando:**

* Es formato de presentación: moneda, separadores, nombres de meses en el idioma del usuario.
* Es lógica de negocio compleja que cambia a menudo y conviene probar con pruebas unitarias.

**Ten en cuenta:**

* Una función sobre una columna en el `WHERE` impide usar índices (ver [Filtrar con WHERE](02-Filtrar%20con%20WHERE.md)). En el `SELECT`, no tiene ese costo.

-----

## Resumen en 5 líneas

1. Entero dividido entero es entero en SQL Server: multiplica por `1.0`; y protege los divisores con `NULLIF`.
2. Concatena con `CONCAT` o `CONCAT_WS`, que manejan los `NULL` y convierten números.
3. `CASE` evalúa en orden, gana la primera rama verdadera, sin `ELSE` da `NULL`, y todas las ramas necesitan tipos compatibles.
4. Fechas: `YEAR`, `DATEPART`, `DATEADD`, `DATEDIFF` (cuenta límites, no períodos), `EOMONTH`, `DATETRUNC`.
5. Convierte con `CAST`, `CONVERT` (estilos) o `TRY_CAST`; evita `FORMAT` en volumen y deja la presentación a la aplicación.

-----

## Para profundizar

<details>
<summary>Precedencia de tipos de datos</summary>

Cuando una expresión mezcla tipos, SQL Server convierte el de menor precedencia al de mayor. Un extracto del orden, de mayor a menor: `datetime2` > `datetime` > `float` > `decimal` > `money` > `bigint` > `int` > `smallint` > `tinyint` > `bit` > `nvarchar` > `varchar`. Por eso `N'Total: ' + 100` intenta convertir el texto a `int` (el entero tiene más precedencia) y falla, y por eso una columna `VARCHAR` comparada con un parámetro `NVARCHAR` se convierte entera a `NVARCHAR`, lo que impide usar su índice.

</details>

<details>
<summary>Funciones nuevas de SQL Server 2022 y 2025</summary>

SQL Server 2022 agregó `GREATEST` y `LEAST` (el mayor o el menor de varios valores), `DATETRUNC`, `IS DISTINCT FROM`, `GENERATE_SERIES` (genera una serie de números) y mejoras en `TRIM`, `LTRIM` y `RTRIM` (quitar caracteres específicos). SQL Server 2025 agregó, entre otras, funciones de expresiones regulares (`REGEXP_LIKE`, `REGEXP_REPLACE`, `REGEXP_SUBSTR`) y el tipo `json` nativo. Antes de usarlas, verifica la versión del servidor con `SELECT @@VERSION;`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso `CASE` para clasificar valores dentro de una consulta, funciones de texto como `UPPER`, `SUBSTRING` y `CONCAT`, y de fecha como `YEAR`, `DATEADD` y `DATEDIFF`. Para convertir tipos uso `CAST` o `CONVERT`. Tengo en cuenta que dividir dos enteros devuelve un entero.

### Respuesta ampliada (semi-senior)

En T-SQL cuido la división entera (multiplico por `1.0` o convierto) y protejo los divisores con `NULLIF`. Concateno con `CONCAT` o `CONCAT_WS` porque manejan los nulos, mientras que `+` los propaga y falla con números por precedencia de tipos. Uso `CASE` sabiendo que evalúa en orden, que sin `ELSE` devuelve `NULL` y que el tipo del resultado sale de la precedencia entre las ramas. Para fechas uso `DATEADD`, `EOMONTH`, `DATETRUNC` y `DATEDIFF`, recordando que este último cuenta límites cruzados y no períodos completos. Convierto con `CAST` o con `CONVERT` y estilo, uso `TRY_CAST` para limpiar datos importados y evito `FORMAT` en consultas de volumen, porque es lento; el formato de presentación lo dejo a la aplicación. Y como el catálogo de funciones no es estándar, conozco las diferencias principales con PostgreSQL y MySQL, como el orden de los argumentos de `DATEDIFF`.

### Preguntas frecuentes de seguimiento

**1. ¿Cuánto da `SELECT 5 / 2` en SQL Server?**
2. Los dos operandos son enteros, así que la división es entera.

**2. ¿Qué diferencia hay entre `CAST` y `CONVERT`?**
`CAST` es estándar; `CONVERT` es de T-SQL y acepta un estilo, útil para formatear fechas.

**3. ¿Por qué evitarías `FORMAT`?**
Porque es mucho más lento que `CONVERT` en consultas sobre muchas filas, y porque el formato de presentación pertenece a la aplicación.

**4. ¿Qué devuelve un `CASE` sin `ELSE` cuando ninguna rama se cumple?**
`NULL`.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que muestre, para cada producto: el nombre en mayúsculas, el precio con IGV redondeado a 2 decimales y una columna `Disponibilidad` con "Agotado" si el stock es 0, "Últimas unidades" si es menor a 10 y "Disponible" en otro caso.

<details>
<summary>Solución</summary>

```sql
SELECT UPPER(Nombre)            AS Producto,
       ROUND(Precio * 1.18, 2)  AS PrecioConIgv,
       CASE
           WHEN Stock = 0  THEN N'Agotado'
           WHEN Stock < 10 THEN N'Últimas unidades'
           ELSE N'Disponible'
       END                      AS Disponibilidad
FROM dbo.Producto
ORDER BY ProductoId;
```

| Producto | PrecioConIgv | Disponibilidad |
| --- | --- | --- |
| TECLADO MECÁNICO | 141.6000 | Disponible |
| MOUSE INALÁMBRICO | 53.6900 | Disponible |
| MONITOR 27 PULGADAS | 1061.8800 | Últimas unidades |
| LÁMPARA DE ESCRITORIO | 88.5000 | Disponible |
| CAFETERA | 247.8000 | Agotado |
| CLEAN CODE | 112.1000 | Disponible |

`ROUND` redondea, pero no cambia la escala del tipo: el resultado sigue mostrando 4 decimales (con ceros). Para mostrar exactamente 2: `CAST(Precio * 1.18 AS DECIMAL(10,2))`, que además redondea. El orden de las ramas importa: `Stock = 0` va antes que `Stock < 10`, porque 0 también es menor que 10.

</details>

**Ejercicio 2.** Escribe una consulta que muestre cuántos días pasaron entre el registro de cada cliente y su primer pedido. Pista: el primer pedido es el de menor fecha. (Usa `MIN` y `GROUP BY`, que se profundizan en la próxima lección.)

<details>
<summary>Solución</summary>

```sql
SELECT c.Nombre,
       c.FechaRegistro,
       MIN(p.FechaPedido)                                    AS PrimerPedido,
       DATEDIFF(day, c.FechaRegistro, MIN(p.FechaPedido))   AS DiasHastaPrimerPedido
FROM dbo.Cliente AS c
JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
GROUP BY c.ClienteId, c.Nombre, c.FechaRegistro
ORDER BY DiasHastaPrimerPedido;
```

| Nombre | FechaRegistro | PrimerPedido | DiasHastaPrimerPedido |
| --- | --- | --- | --- |
| Luis Ramos | 2026-02-03 | 2026-03-05 16:40 | 30 |
| Jorge Díaz | 2026-03-15 | 2026-05-02 11:00 | 48 |
| Ana Torres | 2026-01-10 | 2026-03-01 10:15 | 50 |
| María Quispe | 2026-02-20 | 2026-04-20 18:30 | 59 |

Sofía no aparece porque no tiene pedidos (el `JOIN` solo conserva las filas que tienen pareja; ver [JOIN](06-JOIN.md)). `DATEDIFF(day, ...)` cuenta los cambios de día, sin importar la hora.

</details>

-----

## Siguiente lección

[Agregados y GROUP BY](05-Agregados%20y%20GROUP%20BY.md)
