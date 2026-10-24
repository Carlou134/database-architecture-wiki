# Cursores frente a operaciones sobre conjuntos

## En una frase

Un cursor recorre un resultado **fila por fila**, como un bucle `foreach` de C#, y SQL Server está diseñado para lo contrario: describir **qué** hacer con todo un conjunto de filas en una sola instrucción; casi todo lo que se hace con un cursor se puede escribir como una operación sobre conjuntos decenas o cientos de veces más rápida, y los pocos casos legítimos de cursor tienen una forma correcta de escribirse.

-----

## Antes de empezar

Conviene que ya sepas:

* `UPDATE` con `JOIN`, `CASE`, funciones de ventana y CTE, del módulo de [Consultas](../03-consultas/README.md).
* Tablas temporales, de [Tablas temporales y variables de tabla](05-Tablas%20temporales%20y%20variables%20de%20tabla.md).
* Procesamiento por lotes, de [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Cursor:** mecanismo para recorrer las filas de un resultado una por una.
* **Operación sobre conjuntos (set-based):** instrucción que procesa todas las filas que cumplen una condición de una vez.
* **RBAR (*Row By Agonizing Row*):** nombre irónico para el procesamiento fila por fila, con cursores o bucles.
* **`FETCH`:** instrucción que lee la siguiente fila de un cursor.
* **`@@FETCH_STATUS`:** resultado del último `FETCH`: 0 si leyó una fila.
* **`FAST_FORWARD`:** tipo de cursor de solo lectura y solo hacia adelante, el más eficiente.

-----

## El problema

Hay que clasificar a los 100.000 clientes de `TiendaGrande` en segmentos según lo que compraron (Oro, Plata, Bronce). La primera versión, escrita por alguien que viene de programar en C#:

```sql
DECLARE @ClienteId INT, @Total DECIMAL(14,2);

DECLARE c CURSOR FOR SELECT ClienteId FROM dbo.Cliente;
OPEN c;
FETCH NEXT FROM c INTO @ClienteId;

WHILE @@FETCH_STATUS = 0
BEGIN
    SELECT @Total = SUM(d.Cantidad * d.PrecioUnitario)
    FROM dbo.Pedido AS p JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.ClienteId = @ClienteId;

    UPDATE dbo.Cliente
    SET Segmento = CASE WHEN @Total >= 80000 THEN 'Oro'
                        WHEN @Total >= 40000 THEN 'Plata'
                        ELSE 'Bronce' END
    WHERE ClienteId = @ClienteId;

    FETCH NEXT FROM c INTO @ClienteId;
END;

CLOSE c;
DEALLOCATE c;
```

Es correcta, se entiende, y tarda **varios minutos**: son 100.000 consultas de suma y 100.000 `UPDATE` de una fila, cada uno con su propia ejecución, sus propios bloqueos y su propia escritura en el log. La versión basada en conjuntos (al final de la lección) hace lo mismo en **segundos**, con una sola instrucción.

-----

## Cómo funciona

### Por qué el procesamiento fila por fila es lento

| | Cursor (fila por fila) | Operación sobre conjuntos |
| --- | --- | --- |
| Ejecuciones | Una (o varias) por fila: N planes ejecutados | Una |
| Acceso a los datos | N búsquedas pequeñas | Un *scan* o un join eficiente sobre todo el conjunto |
| Optimizador | Optimiza cada instrucción suelta, sin ver el conjunto | Elige el mejor algoritmo para el conjunto completo (*Hash*, *Merge*, paralelismo) |
| Log y bloqueos | N transacciones pequeñas (en modo automático) | Una instrucción, que se puede partir en lotes si es enorme |
| Paralelismo | No | Sí |

Tus apuntes de la sesión 12 dan una cifra típica: de 10 a 100 veces más rápido. Con volúmenes grandes, la diferencia puede ser mayor.

La razón de fondo es la misma que se vio en [SELECT y el orden lógico](../03-consultas/01-SELECT%20y%20el%20orden%20lógico.md): SQL es **declarativo**. Con un cursor, el programador decide el algoritmo (recorrer, buscar, actualizar); con una instrucción sobre conjuntos, el optimizador elige el mejor algoritmo con las estadísticas y los índices disponibles.

### El `WHILE` no es la alternativa

Una "solución" frecuente (que también aparece en tu cheat sheet) es reemplazar el cursor por un bucle `WHILE` sobre una tabla numerada:

```sql
SELECT ROW_NUMBER() OVER (ORDER BY ClienteId) AS n, ClienteId INTO #c FROM dbo.Cliente;
DECLARE @i INT = 1, @max INT = (SELECT COUNT(*) FROM #c);
WHILE @i <= @max
BEGIN
    -- ... lo mismo que dentro del cursor, para la fila @i ...
    SET @i += 1;
END;
```

Esto **sigue siendo fila por fila**: las mismas 100.000 iteraciones, cada una con sus consultas. A veces es incluso más lento que un cursor `FAST_FORWARD`, porque cada iteración busca su fila en la tabla temporal. Cambiar `CURSOR` por `WHILE` no resuelve el problema; el problema es el **procesamiento fila por fila**.

El `WHILE` sí es la herramienta correcta para otra cosa: recorrer **lotes** de miles de filas, cada uno procesado con una instrucción sobre conjuntos (ver [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md)).

### Cómo traducir a conjuntos

| Lo que hace el cursor | La operación sobre conjuntos |
| --- | --- |
| `UPDATE` con un `IF` dentro, fila por fila | Un `UPDATE` con `CASE` |
| Calcular un agregado por fila y guardarlo | `UPDATE ... FROM` con un join a una CTE o tabla derivada agregada |
| `INSERT` fila por fila | `INSERT ... SELECT` |
| Llevar un acumulado o un total corrido | `SUM(...) OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)` |
| Numerar filas | `ROW_NUMBER() OVER (...)` |
| Comparar con la fila anterior | `LAG()` / `LEAD()` |
| Concatenar valores | `STRING_AGG` |
| Recorrer una jerarquía | CTE recursiva |
| Ejecutar una consulta por cada fila | `CROSS APPLY` con una función de tabla en línea |
| "Si existe, actualizar; si no, insertar" | `UPDATE` + `INSERT ... WHERE NOT EXISTS`, o `MERGE` (con sus advertencias) |

La forma de pensar es: en lugar de "para cada cliente, calcula su total y actualízalo", "**actualiza a todos los clientes con su total**". El *qué*, no el *cómo*.

### Los casos legítimos de un cursor

Hay situaciones en las que el procesamiento fila por fila es lo correcto:

* **Ejecutar un procedimiento por cada fila,** cuando el procedimiento no se puede reescribir para recibir un conjunto (por ejemplo, un procedimiento de un proveedor externo).
* **Tareas de administración:** recorrer todas las bases o todas las tablas para hacer un `BACKUP`, un `DBCC CHECKDB` o un `ALTER INDEX` en cada una.
* **Lógica en la que cada fila depende del resultado de la anterior** de una forma que las funciones de ventana no pueden expresar (algunos cálculos de saldos con reglas complejas, asignaciones con capacidad limitada).
* **Generar SQL dinámico** objeto por objeto.

Y aun en esos casos, a menudo hay que procesar pocas filas (unas decenas de bases, unos cientos de tablas), así que el costo no importa.

### Cómo escribir bien un cursor

Si el cursor es necesario, la forma correcta es la de tu cheat sheet:

```sql
DECLARE @Base SYSNAME;

DECLARE cBases CURSOR LOCAL FAST_FORWARD FOR
    SELECT name FROM sys.databases WHERE database_id > 4 AND state_desc = 'ONLINE';

OPEN cBases;
FETCH NEXT FROM cBases INTO @Base;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT CONCAT(N'Procesando ', @Base);
    -- EXEC dbo.MantenimientoBase @Base;

    FETCH NEXT FROM cBases INTO @Base;
END;

CLOSE cBases;
DEALLOCATE cBases;
```

| Opción | Por qué |
| --- | --- |
| **`LOCAL`** | El cursor solo existe en este lote o procedimiento; un cursor `GLOBAL` (el valor por defecto de la base) queda visible para toda la conexión y choca si otro procedimiento usa el mismo nombre |
| **`FAST_FORWARD`** | Solo lectura y solo hacia adelante: el tipo más eficiente |
| **`STATIC`** | Si necesitas que el cursor no vea los cambios hechos mientras se recorre (copia el resultado a `tempdb`) |
| **`CLOSE` + `DEALLOCATE`** | Liberan el cursor; con `LOCAL`, se liberan solos al terminar el lote, pero cerrarlos explícitamente es más claro |

`@@FETCH_STATUS` vale 0 si el `FETCH` leyó una fila, -1 si no hay más filas y -2 si la fila se borró (en cursores `KEYSET`). El patrón `FETCH` antes del bucle y `FETCH` al final del cuerpo asegura que el bucle termine cuando no quedan filas.

Para que un error no deje el cursor abierto (con un cursor `GLOBAL`), se cierra en el `CATCH` verificando su estado con `CURSOR_STATUS`, como en tu cheat sheet.

-----

## Ejemplo completo

La segmentación de clientes del problema, en `TiendaGrande`, basada en conjuntos:

```sql
USE TiendaGrande;

ALTER TABLE dbo.Cliente ADD Segmento VARCHAR(10) NULL;   -- una vez
GO

SET STATISTICS TIME ON;

WITH TotalCliente AS (
    SELECT p.ClienteId, SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.ClienteId
)
UPDATE c
SET c.Segmento = CASE WHEN t.Total >= 80000 THEN 'Oro'
                      WHEN t.Total >= 40000 THEN 'Plata'
                      ELSE 'Bronce' END
FROM dbo.Cliente AS c
LEFT JOIN TotalCliente AS t ON t.ClienteId = c.ClienteId;

SET STATISTICS TIME OFF;

SELECT Segmento, COUNT(*) AS Clientes
FROM dbo.Cliente
GROUP BY Segmento
ORDER BY Clientes DESC;
```

Con los datos generados, un cliente común compra en total unos 45.000 de media (unos 9 pedidos de ~5.000 cada uno), así que con estos umbrales los clientes se reparten entre los tres segmentos; el cliente 1, con 50.000 pedidos, es Oro con holgura. Las cantidades exactas dependen de los valores pseudoaleatorios del script.

| Versión | Ejecuciones | Tiempo aproximado |
| --- | --- | --- |
| Cursor | ~200.000 instrucciones (una suma y un `UPDATE` por cliente) | Minutos |
| Conjuntos | 1 instrucción | Segundos |

Qué cambió y por qué:

* **La CTE calcula el total de todos los clientes de una vez,** con un join y un agregado que el optimizador resuelve con *Hash Match* sobre los dos millones y medio de líneas, en paralelo si conviene.
* **El `UPDATE ... FROM` con `LEFT JOIN`** asigna el segmento a todos los clientes, incluidos los que no compraron (su total es `NULL`, y el `CASE` los manda a Bronce).
* **El `CASE` reemplaza al `IF`** que había dentro del cursor.
* **Si la tabla fuera enorme,** esta misma instrucción se aplicaría por lotes de clientes (por rangos de `ClienteId`), sin volver al fila por fila.

-----

## Errores comunes

**1. Cursores para lo que se resuelve con una instrucción.**
Qué pasa: procesos que tardan minutos u horas.
Por qué: se traslada la forma de pensar de un lenguaje imperativo (`foreach`) a SQL.
Arreglo: pensar en el conjunto: "actualiza todas las filas que...".

**2. Reemplazar el cursor por un `WHILE`.**
Qué pasa: el proceso sigue igual de lento (o más).
Por qué: el problema es el fila por fila, no la palabra `CURSOR`.
Arreglo: una operación sobre conjuntos; el `WHILE`, solo para recorrer lotes.

**3. Cursores sin `LOCAL` ni `FAST_FORWARD`.**
Qué pasa: un cursor global que choca con otro del mismo nombre y un tipo más caro de lo necesario.
Por qué: las opciones por defecto no son las más eficientes.
Arreglo: `CURSOR LOCAL FAST_FORWARD`.

**4. Olvidar el `FETCH` dentro del bucle.**
Qué pasa: un bucle infinito que procesa siempre la misma fila.
Por qué: `@@FETCH_STATUS` nunca cambia.
Arreglo: el patrón con `FETCH` antes del bucle y al final del cuerpo.

**5. Llamar a la base fila por fila desde la aplicación.**
Qué pasa: el mismo problema del cursor, más la latencia de red de cada llamada (el problema N+1).
Por qué: un `foreach` en C# que ejecuta una consulta por elemento.
Arreglo: una consulta que procese el conjunto, o un TVP con todas las filas.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Cursores en el servidor | `DECLARE c CURSOR ... FETCH` | En PL/pgSQL: `FOR fila IN SELECT ... LOOP` (más simple) | En procedimientos: `DECLARE ... CURSOR`, `FETCH`, `HANDLER FOR NOT FOUND` |
| `UPDATE` con join | `UPDATE ... FROM ... JOIN` | `UPDATE ... FROM ...` | `UPDATE t JOIN ...` |
| Recomendación | Operaciones sobre conjuntos | Operaciones sobre conjuntos | Operaciones sobre conjuntos |

El principio es universal: todos los motores relacionales están optimizados para operaciones sobre conjuntos, y en todos el procesamiento fila por fila es la excepción.

### En MongoDB

Un cursor de MongoDB es otra cosa: es el mecanismo normal con el que el driver recibe los resultados de un `find` en lotes, y no implica procesamiento fila por fila en el servidor. Lo que sí es el mismo antipatrón es recorrer los documentos en la aplicación y hacer un `updateOne` por cada uno, en lugar de un `updateMany` o un *aggregation pipeline* con `$merge`. Para operaciones distintas por documento, `bulkWrite` envía todas en una sola llamada.

-----

## Cuándo sí y cuándo no

**Usa operaciones sobre conjuntos cuando:**

* Siempre que la lógica se pueda expresar como "hacer X con todas las filas que cumplen Y": casi siempre.

**Usa un cursor cuando:**

* Tienes que ejecutar un procedimiento o un comando por cada elemento (bases, tablas, procedimientos de terceros), y no hay versión basada en conjuntos.
* Cada fila depende del resultado de la anterior de una forma que las funciones de ventana no resuelven.

**Ten en cuenta:**

* La claridad también cuenta: un cursor de 20 filas para un proceso administrativo es perfectamente razonable. El problema son los cursores sobre tablas de negocio con miles o millones de filas.

-----

## Resumen en 5 líneas

1. Un cursor procesa fila por fila; SQL Server está optimizado para procesar conjuntos completos en una instrucción.
2. La diferencia típica es de 10 a 100 veces o más: una ejecución optimizada contra miles de ejecuciones pequeñas.
3. Reemplazar el cursor por un `WHILE` no resuelve nada: sigue siendo fila por fila.
4. Casi todo se traduce: `IF` → `CASE`, agregados por fila → join con una CTE, acumulados → funciones de ventana.
5. Si el cursor es necesario: `LOCAL FAST_FORWARD`, `FETCH` antes y al final del bucle, `CLOSE` y `DEALLOCATE`.

-----

## Para profundizar

<details>
<summary>El "quirky update" y por qué no usarlo</summary>

Antes de las funciones de ventana con marcos (SQL Server 2012), existía un truco para calcular acumulados sin cursor: `UPDATE t SET @acum = Acumulado = @acum + Monto`, confiando en que el `UPDATE` recorriera las filas en el orden del índice agrupado. Funciona casi siempre, pero el orden **no está garantizado** (puede romperse con paralelismo o con otro plan), y Microsoft no lo documenta como comportamiento soportado. Desde 2012, `SUM(Monto) OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)` hace lo mismo de forma correcta y garantizada. Si encuentras este patrón en código heredado, es un buen candidato a reescribir.

</details>

<details>
<summary>Cursores del lado del cliente</summary>

Las aplicaciones también pueden recorrer resultados fila por fila (un `SqlDataReader` que lee de a una fila, mientras la aplicación hace otras consultas por cada una). El efecto es el mismo que un cursor, con la latencia de red agregada. La versión correcta es la misma que en el servidor: una consulta que devuelve todo lo necesario de una vez (con joins o con varios conjuntos de resultados), o un procedimiento que procesa el conjunto completo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un cursor recorre filas una por una, y es lento porque ejecuta una instrucción por cada fila. Casi siempre se puede reemplazar por una sola instrucción que procese todas las filas a la vez, como un `UPDATE` con un `CASE` o un `JOIN`. Si hace falta un cursor, se usa `LOCAL FAST_FORWARD`.

### Respuesta ampliada (semi-senior)

SQL es declarativo: con una operación sobre conjuntos, el optimizador elige el algoritmo (joins hash, merge, paralelismo) para todo el conjunto en una ejecución; con un cursor, el algoritmo lo decido yo, fila por fila, con miles de ejecuciones, búsquedas y escrituras pequeñas. Por eso traduzco los cursores a conjuntos: los `IF` por fila a `CASE`, los agregados por fila a un `UPDATE ... FROM` con una CTE agregada, los acumulados y comparaciones con la fila anterior a funciones de ventana, las jerarquías a CTE recursivas y las consultas por fila a `CROSS APPLY` con funciones en línea. Sé que cambiar el cursor por un `WHILE` no soluciona nada, y reservo el `WHILE` para procesar lotes basados en conjuntos. Uso cursores solo en casos legítimos (ejecutar un procedimiento o un comando por cada base o tabla, lógica secuencial que las ventanas no expresan), siempre `LOCAL FAST_FORWARD`. Y aplico el mismo criterio en la aplicación, donde el equivalente es el problema N+1.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué los cursores son lentos?**
Porque ejecutan una o varias instrucciones por fila, sin que el optimizador pueda elegir un algoritmo para el conjunto completo ni paralelizar.

**2. ¿Un `WHILE` es mejor que un cursor?**
No, si procesa fila por fila: es el mismo problema. El `WHILE` es útil para recorrer lotes procesados con operaciones sobre conjuntos.

**3. ¿Cuándo usarías un cursor?**
Para ejecutar un procedimiento o un comando por cada elemento (por ejemplo, cada base de datos) o para lógica secuencial que no se puede expresar con funciones de ventana.

**4. ¿Qué opciones de cursor usas?**
`LOCAL FAST_FORWARD`: solo visible en el lote, de solo lectura y solo hacia adelante.

-----

## Práctica

**Ejercicio 1.** Reescribe este cursor como una operación sobre conjuntos:

```sql
DECLARE @Id INT, @Stock INT;
DECLARE c CURSOR LOCAL FAST_FORWARD FOR SELECT ProductoId, Stock FROM dbo.Producto;
OPEN c;
FETCH NEXT FROM c INTO @Id, @Stock;
WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Stock = 0
        UPDATE dbo.Producto SET Activo = 0 WHERE ProductoId = @Id;
    ELSE IF @Stock < 10
        UPDATE dbo.Producto SET Precio = Precio * 1.05 WHERE ProductoId = @Id;
    FETCH NEXT FROM c INTO @Id, @Stock;
END;
CLOSE c; DEALLOCATE c;
```

<details>
<summary>Solución</summary>

```sql
UPDATE dbo.Producto
SET Activo = CASE WHEN Stock = 0 THEN 0 ELSE Activo END,
    Precio = CASE WHEN Stock > 0 AND Stock < 10 THEN Precio * 1.05 ELSE Precio END
WHERE Stock < 10;
```

Una sola instrucción, que solo toca los productos con stock menor a 10 (el `WHERE`), y aplica a cada uno la regla que le corresponde con `CASE`. Cada `CASE` deja la columna igual (`ELSE Activo`, `ELSE Precio`) en las filas en las que no aplica. Con los datos de ejemplo: la cafetera (stock 0) queda inactiva y el monitor (stock 8) sube a 944.90 (899.90 × 1,05 = 944.895, redondeado a 2 decimales por el tipo de la columna).

</details>

**Ejercicio 2.** Escribe, con operaciones sobre conjuntos, el saldo acumulado de facturación de cada cliente pedido por pedido, en orden de fecha (lo que un cursor haría llevando una variable `@acumulado`).

<details>
<summary>Solución</summary>

```sql
WITH TotalPedido AS (
    SELECT p.ClienteId, p.PedidoId, p.FechaPedido,
           SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    GROUP BY p.ClienteId, p.PedidoId, p.FechaPedido
)
SELECT ClienteId, PedidoId, FechaPedido, Total,
       SUM(Total) OVER (PARTITION BY ClienteId
                        ORDER BY FechaPedido, PedidoId
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS Acumulado
FROM TotalPedido
ORDER BY ClienteId, FechaPedido;
```

En `Tienda`, Ana tiene 211.00 y luego 476.00 de acumulado; los demás clientes, un solo pedido cada uno. `PARTITION BY ClienteId` reinicia el acumulado por cliente (lo que un cursor haría poniendo `@acumulado = 0` al cambiar de cliente), y el marco `ROWS` explícito evita la trampa del `RANGE` por defecto (ver [Funciones de ventana](../03-consultas/10-Funciones%20de%20ventana.md)).

</details>

-----

## Siguiente lección

[SQL dinámico seguro](07-SQL%20dinámico%20seguro.md)
