# NULL y la lógica de tres valores

## En una frase

`NULL` no es un valor sino la **ausencia** de valor ("no se sabe" o "no aplica"), y por eso cualquier comparación con `NULL` no da verdadero ni falso sino **desconocido**: SQL trabaja con una lógica de tres valores, y no entenderla produce consultas que pierden filas en silencio.

-----

## Antes de empezar

Conviene que ya sepas:

* `NOT NULL`, `CHECK` y `UNIQUE`, de [Restricciones](06-Restricciones.md).
* Los operadores lógicos `AND`, `OR` y `NOT`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`NULL`:** marcador de que una columna no tiene valor.
* **`UNKNOWN` (desconocido):** el tercer valor de verdad, junto a `TRUE` y `FALSE`.
* **Lógica de tres valores (3VL):** lógica donde una condición puede ser verdadera, falsa o desconocida.
* **`IS NULL` / `IS NOT NULL`:** los únicos operadores que preguntan correctamente si algo es nulo.
* **`COALESCE`:** función que devuelve el primer argumento que no sea nulo.

-----

## El problema

Del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md): María no informó su ciudad (`Ciudad = NULL`). Marketing pide "todos los clientes que **no** son de Lima":

```sql
SELECT Nombre, Ciudad
FROM dbo.Cliente
WHERE Ciudad <> N'Lima';
```

| Nombre | Ciudad |
| --- | --- |
| Luis Ramos | Arequipa |
| Sofía Mendoza | Cusco |

**María no aparece.** No es de Lima (o al menos, no sabemos que lo sea), pero la consulta la descartó sin ningún error ni aviso. Y si pides los que **sí** son de Lima, tampoco aparece. María desaparece de las dos mitades:

```text
WHERE Ciudad =  'Lima'  → Ana, Jorge          (2)
WHERE Ciudad <> 'Lima'  → Luis, Sofía         (2)
                                     Total:    4 de 5 clientes
```

Esto no es un error de SQL Server: es cómo funciona `NULL` en todos los motores. Y es una de las fuentes de errores más frecuentes en reportes, porque nadie se da cuenta de que faltan filas.

-----

## Cómo funciona

### `NULL` significa "sin valor", no "vacío" ni "cero"

| Valor | Significado |
| --- | --- |
| `NULL` | No hay valor: no se sabe o no aplica |
| `''` (texto vacío) | Hay un valor: un texto de longitud 0 |
| `0` | Hay un valor: el número cero |

`Ciudad = NULL` significa "no sabemos dónde vive". `Ciudad = ''` significaría "vive en un lugar sin nombre", que no tiene sentido. Mezclar las dos cosas en la misma columna obliga a cada consulta a contemplar los dos casos.

### Comparar con `NULL` da `UNKNOWN`

Si no sabemos la ciudad de María, ¿es igual a `'Lima'`? No lo sabemos. ¿Es distinta? Tampoco lo sabemos. Por eso:

| Expresión | Resultado |
| --- | --- |
| `NULL = 'Lima'` | `UNKNOWN` |
| `NULL <> 'Lima'` | `UNKNOWN` |
| `NULL = NULL` | `UNKNOWN` (¡dos valores desconocidos no son necesariamente iguales!) |
| `NULL > 5` | `UNKNOWN` |
| `NULL + 10` | `NULL` |

Y la regla clave: **`WHERE` solo deja pasar las filas donde la condición es `TRUE`**. Las filas con `FALSE` **y** las filas con `UNKNOWN` se descartan.

### Las tablas de verdad

| `AND` | TRUE | FALSE | UNKNOWN |
| --- | --- | --- | --- |
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

| `OR` | TRUE | FALSE | UNKNOWN |
| --- | --- | --- | --- |
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

| `NOT` | Resultado |
| --- | --- |
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | **UNKNOWN** |

La forma de razonarlo: piensa `UNKNOWN` como "podría ser verdadero o falso". `FALSE AND (cualquier cosa)` es falso sin importar lo que valga la otra parte; `TRUE OR (cualquier cosa)` es verdadero. En los demás casos, el resultado depende del valor desconocido, así que queda desconocido. Y `NOT UNKNOWN` sigue siendo `UNKNOWN`: negar algo que no sabes no te da información.

### `IS NULL` e `IS NOT NULL`

Para preguntar si algo es nulo, `= NULL` **nunca** funciona, porque da `UNKNOWN`. Los operadores correctos son estos:

```sql
SELECT Nombre FROM dbo.Cliente WHERE Ciudad = NULL;     -- ❌ 0 filas, siempre
SELECT Nombre FROM dbo.Cliente WHERE Ciudad IS NULL;    -- ✅ María
```

Y la consulta del problema, corregida:

```sql
SELECT Nombre, Ciudad
FROM dbo.Cliente
WHERE Ciudad <> N'Lima' OR Ciudad IS NULL;
```

| Nombre | Ciudad |
| --- | --- |
| Luis Ramos | Arequipa |
| María Quispe | `NULL` |
| Sofía Mendoza | Cusco |

Si María debe aparecer o no es una **decisión de negocio** ("¿los clientes sin ciudad cuentan como de fuera de Lima?"). Lo que no puede pasar es que la decida el `NULL` sin que nadie lo note.

### La trampa de `NOT IN`

Esta es la versión más peligrosa del problema:

```sql
SELECT Nombre FROM dbo.Cliente WHERE Ciudad IN (N'Lima', NULL);       -- Ana, Jorge
SELECT Nombre FROM dbo.Cliente WHERE Ciudad NOT IN (N'Lima', NULL);   -- 0 filas
```

`NOT IN (N'Lima', NULL)` equivale a `Ciudad <> 'Lima' AND Ciudad <> NULL`. La segunda parte siempre es `UNKNOWN`, así que el `AND` nunca puede dar `TRUE`. **Si la lista de un `NOT IN` contiene un `NULL`, la consulta devuelve cero filas.**

Con una lista escrita a mano se nota. Con una subconsulta, no:

```sql
-- "Clientes cuya ciudad no tiene ningún vendedor asignado"
SELECT Nombre FROM dbo.Cliente
WHERE Ciudad NOT IN (SELECT Ciudad FROM dbo.Vendedor);   -- si un vendedor tiene Ciudad NULL → 0 filas
```

El día que alguien registra un vendedor sin ciudad, el reporte queda vacío. La alternativa segura es `NOT EXISTS`, que no tiene este problema:

```sql
SELECT c.Nombre FROM dbo.Cliente AS c
WHERE NOT EXISTS (SELECT 1 FROM dbo.Vendedor AS v WHERE v.Ciudad = c.Ciudad);
```

### `NULL` en las funciones de agregado

Las funciones de agregado **ignoran** los `NULL`:

```sql
SELECT COUNT(*)      AS Clientes,          -- cuenta filas
       COUNT(Ciudad) AS ConCiudad,         -- cuenta valores no nulos
       COUNT(DISTINCT Ciudad) AS Ciudades
FROM dbo.Cliente;
```

| Clientes | ConCiudad | Ciudades |
| --- | --- | --- |
| 5 | 4 | 3 |

* `COUNT(*)` cuenta filas, sin importar su contenido.
* `COUNT(columna)` cuenta solo los valores no nulos.
* `SUM`, `AVG`, `MIN` y `MAX` ignoran los nulos. `AVG` divide por la cantidad de valores **no nulos**, no por la cantidad de filas: si 3 de 10 productos no tienen calificación, el promedio es de los 7 que la tienen.
* Si **todos** los valores son nulos (o no hay filas), `SUM` devuelve `NULL`, no `0`.

### `NULL` en otros lugares

| Dónde | Comportamiento en SQL Server |
| --- | --- |
| `GROUP BY` | Todos los `NULL` forman **un** grupo |
| `DISTINCT` | Todos los `NULL` cuentan como **un** valor |
| `ORDER BY ... ASC` | Los `NULL` van **primero** |
| `UNIQUE` | Admite **un** `NULL` (ver [Restricciones](06-Restricciones.md)) |
| `CHECK` | `UNKNOWN` **pasa**: solo se rechaza `FALSE` |
| `JOIN ... ON a.x = b.x` | Las filas con `NULL` en `x` **nunca** se emparejan |

Fíjate en la asimetría: en `WHERE` y en los `JOIN`, `UNKNOWN` se trata como falso (la fila no entra); en `CHECK`, se trata como verdadero (la fila entra). Y en `GROUP BY` y `DISTINCT`, los `NULL` se tratan como iguales entre sí, aunque `NULL = NULL` sea `UNKNOWN`.

### Reemplazar `NULL`: `COALESCE` e `ISNULL`

`COALESCE` devuelve el primer argumento no nulo:

```sql
SELECT Nombre, COALESCE(Ciudad, N'Sin ciudad') AS Ciudad
FROM dbo.Cliente;
```

| Nombre | Ciudad |
| --- | --- |
| Ana Torres | Lima |
| Luis Ramos | Arequipa |
| María Quispe | Sin ciudad |
| Jorge Díaz | Lima |
| Sofía Mendoza | Cusco |

SQL Server también tiene `ISNULL(a, b)`, que parece igual pero no lo es:

| | `COALESCE` | `ISNULL` |
| --- | --- | --- |
| Estándar SQL | Sí | No, solo SQL Server |
| Argumentos | Dos o más | Exactamente dos |
| Tipo del resultado | El de mayor precedencia entre los argumentos | El del **primer** argumento |

La diferencia de tipo puede truncar:

```sql
DECLARE @codigo VARCHAR(3) = NULL;

SELECT ISNULL(@codigo, 'Desconocido')   AS ConIsnull,     -- 'Des'
       COALESCE(@codigo, 'Desconocido') AS ConCoalesce;   -- 'Desconocido'
```

Prefiere `COALESCE`, salvo que tengas un motivo concreto para usar `ISNULL`.

### Texto y aritmética con `NULL`

```sql
SELECT N'Cliente: ' + Ciudad        FROM dbo.Cliente WHERE ClienteId = 3;   -- NULL
SELECT CONCAT(N'Cliente: ', Ciudad) FROM dbo.Cliente WHERE ClienteId = 3;   -- 'Cliente: '
SELECT 100 + NULL;                                                          -- NULL
```

El operador `+` propaga el `NULL`; `CONCAT` (desde SQL Server 2012) trata los `NULL` como texto vacío.

### Diseñar para no necesitar `NULL`

La mejor forma de manejar `NULL` es no tenerlo donde no hace falta:

* **`NOT NULL` por defecto.** Cada columna nullable debería poder justificar por qué.
* **`DEFAULT` cuando hay un valor razonable:** `Stock DEFAULT 0`, `Activo DEFAULT 1`.
* **Un solo significado.** `NULL` puede significar "no se sabe" o "no aplica" (un cliente persona no tiene razón social). Si la diferencia importa, el diseño debe distinguirlos, por ejemplo con un tipo de cliente o con tablas separadas por subtipo (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)).

-----

## Ejemplo completo

Un reporte por ciudad que contempla los `NULL` de forma explícita:

**Antes:**

```sql
SELECT Ciudad, COUNT(*) AS Clientes
FROM dbo.Cliente
WHERE Ciudad <> N'Cusco'
GROUP BY Ciudad
ORDER BY Ciudad;
```

| Ciudad | Clientes |
| --- | --- |
| Arequipa | 1 |
| Lima | 2 |

**Después:**

```sql
SELECT COALESCE(Ciudad, N'(Sin ciudad)') AS CiudadReporte,
       COUNT(*) AS Clientes
FROM dbo.Cliente
WHERE Ciudad <> N'Cusco' OR Ciudad IS NULL
GROUP BY Ciudad
ORDER BY CASE WHEN Ciudad IS NULL THEN 1 ELSE 0 END, Ciudad;
```

| CiudadReporte | Clientes |
| --- | --- |
| Arequipa | 1 |
| Lima | 2 |
| (Sin ciudad) | 1 |

Qué cambió y por qué:

* **`OR Ciudad IS NULL`:** María deja de desaparecer en el filtro.
* **`COALESCE` en la salida:** el reporte muestra un texto claro en lugar de una celda vacía, que el lector confundiría con un error.
* **`ORDER BY CASE`:** SQL Server pone los `NULL` primero en orden ascendente; el `CASE` los manda al final, donde se espera ver "los que faltan".
* **`GROUP BY Ciudad`:** se agrupa por la columna original, no por la expresión con `COALESCE`, para no mezclar a quien tenga escrito literalmente "(Sin ciudad)" con quien tenga `NULL`.

-----

## Errores comunes

**1. `WHERE columna = NULL`.**
Qué pasa: la consulta devuelve cero filas, siempre, sin error.
Por qué: cualquier comparación con `NULL` da `UNKNOWN`.
Arreglo: `IS NULL` / `IS NOT NULL`.

**2. Filtros con `<>` que pierden los nulos.**
Qué pasa: `WHERE Ciudad <> 'Lima'` excluye a quienes no tienen ciudad.
Por qué: `NULL <> 'Lima'` es `UNKNOWN`, y `WHERE` descarta lo desconocido.
Arreglo: decide qué pasa con los nulos y escríbelo: `OR Ciudad IS NULL`.

**3. `NOT IN` con una subconsulta que puede devolver `NULL`.**
Qué pasa: el resultado queda vacío el día que aparece un `NULL` en la subconsulta.
Por qué: `x <> NULL` es `UNKNOWN`, y anula el `AND` implícito del `NOT IN`.
Arreglo: `NOT EXISTS`, o filtra los nulos en la subconsulta (`WHERE Ciudad IS NOT NULL`).

**4. Confundir `COUNT(*)` con `COUNT(columna)`.**
Qué pasa: "tenemos 4 clientes" cuando son 5.
Por qué: `COUNT(columna)` ignora los nulos.
Arreglo: `COUNT(*)` para contar filas; `COUNT(columna)` solo si quieres contar los valores informados.

**5. Esperar `0` de un `SUM` sin filas.**
Qué pasa: `SELECT SUM(Cantidad) ... WHERE PedidoId = 999` devuelve `NULL`, y la aplicación falla al leer un `decimal` no nulo.
Por qué: la suma de ningún valor es `NULL`.
Arreglo: `COALESCE(SUM(Cantidad), 0)`.

**6. Usar `''` o valores mágicos en lugar de `NULL`.**
Qué pasa: `Ciudad = ''`, `FechaBaja = '1900-01-01'` o `Edad = -1` para decir "no hay dato", y cada consulta tiene que conocer el truco.
Por qué: se intenta evitar el `NULL` sin cambiar el diseño.
Arreglo: `NULL` cuando de verdad no hay dato, o un diseño que no necesite la columna opcional.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Lógica de tres valores | Sí | Sí | Sí |
| Comparación segura con nulos | `IS [NOT] DISTINCT FROM` (desde 2022) | `IS [NOT] DISTINCT FROM` | `<=>` (`NULL <=> NULL` es verdadero) |
| `NULL` en `ORDER BY ASC` | Primero | **Último** | Primero |
| Controlar la posición | `CASE` en el `ORDER BY` | `NULLS FIRST` / `NULLS LAST` | `ORDER BY Col IS NULL, Col` |
| Concatenar con `NULL` | `+` da `NULL`; `CONCAT` ignora el `NULL` | `\|\|` da `NULL`; `concat()` ignora el `NULL` | `CONCAT` da **`NULL`**; `CONCAT_WS` ignora el `NULL` |
| Reemplazar nulos | `COALESCE`, `ISNULL` | `COALESCE` | `COALESCE`, `IFNULL` |
| `''` frente a `NULL` | Distintos | Distintos | Distintos |

`IS DISTINCT FROM` es el operador que compara tratando los `NULL` como iguales: `NULL IS NOT DISTINCT FROM NULL` es verdadero. Es útil para detectar cambios entre dos versiones de una fila, donde `a <> b` falla si uno de los dos es nulo.

En **Oracle** (por si te lo encuentras), `''` y `NULL` son lo mismo: un texto vacío se guarda como `NULL`. Es una excepción conocida que trae problemas al migrar.

### En MongoDB

MongoDB distingue dos casos que SQL no distingue: un campo con valor `null` y un campo que **no existe** en el documento. La consulta `{ ciudad: null }` encuentra **los dos**: los documentos con `ciudad: null` y los que no tienen el campo `ciudad`. Para separarlos se usa `{ ciudad: { $exists: false } }` (no existe) o `{ ciudad: { $type: "null" } }` (existe y es `null`). No hay lógica de tres valores: las comparaciones devuelven verdadero o falso.

-----

## Cuándo sí y cuándo no

**Usa `NULL` cuando:**

* El dato realmente puede faltar y no hay un valor por defecto razonable (la ciudad de un cliente que no la informó, la fecha de entrega de un pedido que no se entregó).

**Evita `NULL` cuando:**

* El dato es obligatorio para que la fila tenga sentido: `NOT NULL`.
* Hay un valor por defecto natural: `DEFAULT`.
* La columna solo aplica a un subtipo de filas: considera una tabla aparte.

**Herramientas que ayudan:**

* Revisar en cada consulta las columnas nullables de los `WHERE`, los `JOIN` y los `NOT IN` es la mejor costumbre que puedes adquirir.
* `SELECT COUNT(*) - COUNT(Col) AS Nulos FROM Tabla;` cuenta rápidamente los nulos de una columna.

-----

## Resumen en 5 líneas

1. `NULL` es ausencia de valor; cualquier comparación con `NULL` da `UNKNOWN`, incluso `NULL = NULL`.
2. `WHERE` y `JOIN` solo aceptan `TRUE`: las filas con `UNKNOWN` desaparecen en silencio.
3. Usa `IS NULL`; nunca `= NULL`. Evita `NOT IN` con subconsultas que puedan devolver `NULL`: usa `NOT EXISTS`.
4. Los agregados ignoran los `NULL`: `COUNT(*)` ≠ `COUNT(col)`, y `SUM` sin valores da `NULL`.
5. Prefiere `COALESCE` a `ISNULL`, y diseña con `NOT NULL` y `DEFAULT` para no necesitar `NULL` donde no aplica.

-----

## Para profundizar

<details>
<summary>ANSI_NULLS: la configuración que no debes cambiar</summary>

SQL Server tiene una opción de sesión `SET ANSI_NULLS OFF` que hace que `= NULL` funcione como `IS NULL`. Existe por compatibilidad con versiones muy antiguas, está marcada como obsoleta y en una versión futura será siempre `ON`. Si la encuentras en un procedimiento heredado, es una señal de código que depende de un comportamiento no estándar. Escribe siempre `IS NULL`.

</details>

<details>
<summary>Por qué Codd quería dos tipos de NULL</summary>

En 1990, Codd propuso distinguir dos marcadores: *A-mark* ("valor aplicable pero desconocido": no sabemos la ciudad) e *I-mark* ("valor inaplicable": una persona no tiene razón social). Eso llevaría a una lógica de **cuatro** valores. Ningún motor lo implementó, y C. J. Date argumenta que hasta el `NULL` simple es un error del modelo. En la práctica, la lección es la misma: cuanto menos dependa el diseño de los `NULL`, menos sorpresas hay.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`NULL` significa que no hay valor. No es lo mismo que cero ni que un texto vacío. Para preguntar si algo es nulo se usa `IS NULL`, porque `= NULL` nunca es verdadero. Las funciones como `COUNT(columna)` o `AVG` ignoran los nulos, y para reemplazarlos se usa `COALESCE`.

### Respuesta ampliada (semi-senior)

SQL usa una lógica de tres valores: cualquier comparación con `NULL` da `UNKNOWN`, y `WHERE` y `ON` solo dejan pasar `TRUE`. Por eso un filtro `<>` pierde las filas con nulos en silencio, y un `NOT IN` cuya subconsulta devuelve un `NULL` da cero filas; en esos casos uso `NOT EXISTS`. En cambio, un `CHECK` deja pasar `UNKNOWN`, y por eso una columna obligatoria necesita `NOT NULL` además del `CHECK`. Los agregados ignoran los nulos (`COUNT(*)` frente a `COUNT(col)`, `AVG` sobre los valores informados, `SUM` vacío da `NULL`), mientras que `GROUP BY` y `DISTINCT` agrupan todos los nulos juntos. Prefiero `COALESCE` a `ISNULL`, porque es estándar y no trunca al tipo del primer argumento. En el diseño, `NOT NULL` es el valor por defecto, con `DEFAULT` donde hay un valor razonable, y si `NULL` empieza a significar dos cosas distintas, lo resuelvo con el modelo, no con valores mágicos.

### Preguntas frecuentes de seguimiento

**1. ¿Cuánto vale `NULL = NULL`?**
`UNKNOWN`. Dos valores desconocidos no tienen por qué ser iguales. Para compararlos como iguales: `IS NOT DISTINCT FROM` (SQL Server 2022, PostgreSQL) o `<=>` (MySQL).

**2. ¿Por qué un `NOT IN` puede devolver cero filas?**
Porque si la lista contiene `NULL`, la condición equivale a un `AND` de desigualdades donde una es siempre `UNKNOWN`, y el resultado nunca llega a `TRUE`.

**3. ¿Qué diferencia hay entre `COUNT(*)` y `COUNT(columna)`?**
`COUNT(*)` cuenta filas; `COUNT(columna)` cuenta los valores no nulos de esa columna.

**4. ¿`COALESCE` o `ISNULL`?**
`COALESCE`: es estándar, acepta varios argumentos y su tipo de resultado no trunca. `ISNULL` toma el tipo del primer argumento.

**5. ¿Cómo trata `NULL` un `CHECK`?**
Lo deja pasar: el `CHECK` solo rechaza las filas donde la condición es `FALSE`, y con `NULL` es `UNKNOWN`.

-----

## Práctica

**Ejercicio 1.** Con los datos del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md), ¿cuántas filas devuelve cada consulta?

```sql
-- a
SELECT * FROM dbo.Cliente WHERE Ciudad = N'Lima' OR Ciudad <> N'Lima';
-- b
SELECT * FROM dbo.Cliente WHERE NOT (Ciudad = N'Cusco');
-- c
SELECT * FROM dbo.Cliente WHERE Ciudad IN (N'Lima', N'Cusco', NULL);
-- d
SELECT * FROM dbo.Cliente WHERE Ciudad NOT IN (N'Lima', N'Cusco', NULL);
-- e
SELECT COUNT(*), COUNT(Ciudad), COUNT(DISTINCT Ciudad) FROM dbo.Cliente WHERE ClienteId IN (1, 3, 4);
```

<details>
<summary>Solución</summary>

* **a: 4 filas.** Parece una tautología, pero para María las dos partes son `UNKNOWN`, y `UNKNOWN OR UNKNOWN` es `UNKNOWN`.
* **b: 3 filas** (Ana, Luis, Jorge). Para María, `NOT UNKNOWN` sigue siendo `UNKNOWN`.
* **c: 3 filas** (Ana, Jorge, Sofía). El `NULL` de la lista no empareja con nada, ni siquiera con la ciudad nula de María.
* **d: 0 filas.** El `NULL` de la lista hace que el `NOT IN` nunca sea verdadero.
* **e: `3`, `2`, `1`.** Tres filas (Ana, María, Jorge); dos ciudades informadas (Lima, Lima); una ciudad distinta (Lima).

</details>

**Ejercicio 2.** Escribe una consulta que devuelva el total gastado por cada cliente, incluidos los que no tienen pedidos (con total `0.00`). Pista: con un `LEFT JOIN`, los clientes sin pedidos quedan con columnas nulas.

<details>
<summary>Solución</summary>

```sql
SELECT c.Nombre,
       COALESCE(SUM(d.Cantidad * d.PrecioUnitario), 0) AS TotalGastado
FROM dbo.Cliente AS c
LEFT JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
LEFT JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY c.ClienteId, c.Nombre
ORDER BY TotalGastado DESC;
```

| Nombre | TotalGastado |
| --- | --- |
| Luis Ramos | 849.90 |
| Ana Torres | 476.00 |
| María Quispe | 210.00 |
| Jorge Díaz | 140.50 |
| Sofía Mendoza | 0.00 |

Sofía no tiene pedidos: el `LEFT JOIN` la conserva con `d.Cantidad` y `d.PrecioUnitario` nulos, `SUM` devuelve `NULL` y `COALESCE` lo convierte en `0`. (Esta consulta cuenta también el pedido cancelado de María; filtrar por estado es un ejercicio del módulo de consultas.)

</details>

-----

## Siguiente lección

Con esto terminan los fundamentos. El siguiente módulo usa todo lo anterior para diseñar una base desde cero:

[Proceso de diseño](../02-diseno-y-modelado/01-Proceso%20de%20diseño.md)
