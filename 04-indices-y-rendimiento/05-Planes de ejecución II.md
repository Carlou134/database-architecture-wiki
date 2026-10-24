# Planes de ejecución II

## En una frase

El optimizador elige el plan con las **estadísticas** (un resumen de cómo se distribuyen los valores de cada columna) y lo **reutiliza** mientras está en caché; cuando las estadísticas mienten o el plan se compiló para un valor atípico (*parameter sniffing*), el plan es malo, y la solución va, en ese orden, por índices, reescritura y estadísticas, y solo al final por los *hints*.

-----

## Antes de empezar

Conviene que ya sepas:

* Leer un plan y comparar filas estimadas con reales, de [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md).
* El punto de inflexión entre *seek* + *lookup* y *scan*, de [Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Estadísticas:** objeto que resume la distribución de los valores de una o varias columnas; el optimizador lo usa para estimar cuántas filas devuelve cada filtro.
* **Histograma:** la parte de las estadísticas que divide los valores de la primera columna en hasta 200 tramos, con la cantidad de filas de cada uno.
* **Estimación de cardinalidad:** el cálculo de cuántas filas devuelve cada operador.
* **Caché de planes:** memoria donde se guardan los planes compilados para reutilizarlos.
* **Parameter sniffing:** compilar el plan con el valor del parámetro de la primera ejecución y reutilizarlo para valores que necesitarían otro plan.
* **Hint:** instrucción que fuerza al optimizador a una decisión concreta (un índice, un tipo de join, un orden).

-----

## El problema

Un procedimiento que lista las líneas de los pedidos de un cliente funciona perfecto durante semanas: 30 lecturas, 1 milisegundo. Un lunes, sin ningún cambio de código, la misma pantalla tarda 4 segundos para **todos** los clientes.

Alguien ejecuta `DBCC FREEPROCCACHE` (vaciar la caché de planes) y todo vuelve a la normalidad... hasta el jueves. Lo que pasó: el lunes a primera hora, el servidor se reinició, y la **primera** persona que abrió la pantalla fue el cliente mayorista, con 50.000 pedidos. El plan se compiló para él, y se reutilizó para todos los demás.

Este es el problema más famoso de rendimiento en SQL Server, y entenderlo exige saber dos cosas: con qué información elige el optimizador y por qué reutiliza los planes.

-----

## Cómo funciona

### Las estadísticas

Para decidir entre un *seek* y un *scan*, el optimizador necesita saber **cuántas filas** devuelve `WHERE ClienteId = 1`, **antes** de ejecutar la consulta. No las cuenta: las estima con las estadísticas.

Cada índice tiene estadísticas sobre sus columnas, y SQL Server crea más automáticamente sobre las columnas que aparecen en los filtros (`AUTO_CREATE_STATISTICS`, activado por defecto).

```sql
DBCC SHOW_STATISTICS ('dbo.Pedido', 'IX_Pedido_Cliente_Fecha') WITH HISTOGRAM;
```

| RANGE_HI_KEY | RANGE_ROWS | EQ_ROWS | DISTINCT_RANGE_ROWS | AVG_RANGE_ROWS |
| --- | --- | --- | --- | --- |
| 1 | 0 | 50000 | 0 | 1 |
| 512 | 4823 | 12 | 509 | 9,48 |
| 1031 | 4917 | 8 | 517 | 9,51 |
| ... | ... | ... | ... | ... |

* El **histograma** tiene hasta 200 tramos (*steps*) de la **primera** columna del índice.
* **`EQ_ROWS`:** filas con exactamente el valor `RANGE_HI_KEY`. El cliente 1, con 50.000 filas, tiene su propio tramo.
* **`AVG_RANGE_ROWS`:** filas promedio por valor **dentro** del tramo. Para `ClienteId = 700`, el optimizador estima ~9,5 filas.

Con esto, la estimación de `WHERE ClienteId = 1` es 50.000, y la de `WHERE ClienteId = 700` es ~9,5. El optimizador elige un plan distinto para cada uno.

### Estadísticas desactualizadas

Las estadísticas se calculan en un momento y envejecen. SQL Server las actualiza automáticamente (`AUTO_UPDATE_STATISTICS`) cuando cambió una cantidad suficiente de filas: con el nivel de compatibilidad 130 o superior (SQL Server 2016 en adelante), el umbral es dinámico y, para tablas grandes, es aproximadamente la raíz cuadrada de 1.000 × filas (para un millón de filas, unas 31.600 modificaciones).

El caso típico de estadísticas engañosas: una tabla de pedidos con estadísticas calculadas ayer, y una consulta por los pedidos **de hoy**. El histograma no tiene ningún tramo para hoy, así que el optimizador estima muy pocas filas aunque haya miles (el problema de la "clave ascendente"). El cardinality estimator moderno mitiga este caso, pero sigue siendo una de las causas clásicas de malas estimaciones.

Actualizarlas a mano:

```sql
UPDATE STATISTICS dbo.Pedido IX_Pedido_Cliente_Fecha WITH FULLSCAN;   -- un índice, leyendo todo
UPDATE STATISTICS dbo.Pedido;                                         -- todas las de la tabla, con muestreo
EXEC sp_updatestats;                                                   -- toda la base, solo las que cambiaron
```

Y ver cuándo se actualizaron por última vez y cuánto cambió desde entonces:

```sql
SELECT s.name, sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID('dbo.Pedido');
```

### La caché de planes

Compilar un plan cuesta CPU (de milisegundos a segundos en consultas complejas). Por eso SQL Server guarda los planes compilados en la **caché de planes** y los reutiliza cuando llega la misma consulta. Para un procedimiento almacenado o una consulta parametrizada, "la misma consulta" significa el mismo texto, **con cualquier valor del parámetro**.

```text
Primera ejecución:  EXEC dbo.LineasDeCliente @ClienteId = 700
                    → compila con @ClienteId = 700 (estima ~9,5 pedidos)
                    → elige Nested Loops + seeks
                    → guarda el plan en caché

Siguientes:         EXEC dbo.LineasDeCliente @ClienteId = 1
                    → reutiliza el plan de 700 (no recompila)
                    → 50.000 seeks en lugar de un scan
```

### Parameter sniffing

"Olfatear" el parámetro (*sniffing*) es, en sí, **bueno**: compilar con el valor real permite al optimizador usar el histograma en lugar de una estimación promedio. El problema aparece cuando la distribución es muy desigual y un mismo plan no sirve para todos los valores:

| Plan compilado para | Bueno para | Malo para |
| --- | --- | --- |
| Cliente común (~10 pedidos): *Nested Loops* + *seeks* | Los 99.999 clientes comunes | El mayorista: 50.000 *seeks* |
| Mayorista (50.000 pedidos): *Hash Match* + *scans* | El mayorista | Los demás: un *scan* de 9.000 páginas para leer 25 líneas |

Los síntomas que menciona tu cheat sheet son los característicos: el rendimiento cambia sin cambios de código, a veces es rápido y a veces lento con los mismos parámetros, y vaciar la caché lo "arregla" por un tiempo.

### Soluciones al parameter sniffing

| Solución | Cómo | Ventaja | Costo |
| --- | --- | --- | --- |
| **Mejor índice** | Un índice cubriente que sirva para todos los valores | Ataca la causa | No siempre existe |
| **`OPTION (RECOMPILE)`** | Compila un plan nuevo en cada ejecución | Siempre el plan adecuado para el valor | CPU de compilación en cada ejecución |
| **`OPTION (OPTIMIZE FOR (@p = valor))`** | Compila siempre para un valor elegido | Un plan estable | Hay que elegir bien el valor, y mantenerlo |
| **`OPTION (OPTIMIZE FOR UNKNOWN)`** | Compila con la estimación promedio, sin mirar el valor | Estable | El promedio puede no servir a nadie |
| **Separar los casos** | Un procedimiento (o una rama `IF`) para los clientes grandes | Cada caso con su plan | Más código |
| **Forzar un plan con Query Store** | Elegir un plan bueno y fijarlo | Sin cambiar el código | Requiere monitorear (módulo de monitoreo) |
| **PSP (SQL Server 2022)** | Con el nivel de compatibilidad 160, el motor puede guardar **varios planes** para una consulta según el rango del valor | Automático | Solo para predicados de igualdad, en casos que el motor detecta |

`OPTION (RECOMPILE)` es la solución más usada para procedimientos que no se ejecutan miles de veces por segundo: el costo de compilar suele ser pequeño comparado con el de un plan malo.

Copiar el parámetro a una variable local (`DECLARE @c INT = @ClienteId`) también "evita" el *sniffing*, porque el optimizador no conoce el valor de una variable al compilar y usa el promedio. Es equivalente a `OPTIMIZE FOR UNKNOWN`, pero escondido: mejor escribir el hint explícito.

`DBCC FREEPROCCACHE` (vaciar toda la caché) **no es una solución**: obliga a recompilar todas las consultas del servidor y deja el problema listo para repetirse. Para descartar un solo plan, se usa el `plan_handle` concreto: `DBCC FREEPROCCACHE (plan_handle)`.

### Otras fuentes de malas estimaciones

| Causa | Qué pasa | Qué hacer |
| --- | --- | --- |
| **Variables de tabla** (`@tabla`) | No tienen estadísticas. Antes de 2019 se estimaba siempre 1 fila; desde SQL Server 2019 (compatibilidad 150), la *compilación diferida* usa la cantidad real de filas en la primera compilación, pero sigue sin histograma | Tablas temporales (`#tabla`) para volúmenes grandes (ver [Tablas temporales](../06-programacion-t-sql/05-Tablas%20temporales%20y%20variables%20de%20tabla.md)) |
| **Funciones escalares en la consulta** | Se ejecutan fila por fila y ocultan su costo | *Inlining* de funciones (2019) o reescribir como función de tabla en línea |
| **Columnas correlacionadas** (`Ciudad = 'Lima' AND Region = 'Lima'`) | El optimizador asume que los filtros son independientes y subestima | Estadísticas de varias columnas, o un índice que las combine |
| **Funciones sobre columnas** (`YEAR(Fecha) = 2026`) | No puede usar el histograma | Reescribir de forma SARGable ([próxima lección](06-Consultas%20SARGables%20y%20antipatrones.md)) |

### Hints: el último recurso

Los *hints* reemplazan la decisión del optimizador por la tuya. Tu cheat sheet tiene el orden correcto, y vale la pena repetirlo:

1. Crear o mejorar índices.
2. Reescribir la consulta.
3. Actualizar estadísticas.
4. **Último recurso:** hints.

Un hint es una decisión **congelada**: lo que hoy es correcto, con otros datos mañana puede ser el peor plan, y el optimizador ya no puede corregirlo.

| Tipo | Ejemplos | Uso razonable |
| --- | --- | --- |
| **De consulta** (`OPTION (...)`) | `RECOMPILE`, `OPTIMIZE FOR`, `MAXDOP 1`, `FORCE ORDER` | `RECOMPILE` y `OPTIMIZE FOR` para el *sniffing*; `MAXDOP` para consultas que se paralelizan sin necesidad |
| **De tabla** (`WITH (...)`) | `INDEX(nombre)`, `FORCESEEK`, `FORCESCAN`, `NOLOCK`, `UPDLOCK`, `READPAST` | Los de bloqueo, con criterio (ver el módulo de transacciones); los de índice, casi nunca |
| **De join** | `INNER LOOP JOIN`, `INNER HASH JOIN`, `INNER MERGE JOIN` | Casi nunca: además, un hint de join **fuerza el orden de todos los joins** de la consulta |

`USE HINT (...)` (desde 2016) permite hints más finos y documentados, como `USE HINT ('DISABLE_PARAMETER_SNIFFING')` o `USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION')`, en lugar de los *trace flags* (`QUERYTRACEON`).

-----

## Ejemplo completo

El problema del lunes, reproducido en `TiendaGrande`:

```sql
USE TiendaGrande;
GO

CREATE OR ALTER PROCEDURE dbo.LineasDeCliente
    @ClienteId INT
AS
BEGIN
    SELECT p.PedidoId, d.ProductoId, d.Cantidad
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.ClienteId = @ClienteId;
END;
GO

SET STATISTICS IO ON;

-- 1. Primera ejecución con un cliente común: se compila y se guarda el plan
EXEC dbo.LineasDeCliente @ClienteId = 700;     -- Nested Loops: ~10 seeks en DetallePedido

-- 2. El mayorista reutiliza ese plan
EXEC dbo.LineasDeCliente @ClienteId = 1;       -- Nested Loops: 50.000 seeks → ~150.000+ lecturas

-- 3. Al revés: descartar el plan y empezar por el mayorista
EXEC sp_recompile 'dbo.LineasDeCliente';
EXEC dbo.LineasDeCliente @ClienteId = 1;       -- Hash Match + scan de DetallePedido: ~9.000 lecturas
EXEC dbo.LineasDeCliente @ClienteId = 700;     -- reutiliza el Hash Match: ~9.000 lecturas para ~25 líneas

SET STATISTICS IO OFF;
```

| Ejecución | Plan en caché compilado para | Lecturas aproximadas en `DetallePedido` |
| --- | --- | --- |
| Cliente 700 (primero) | 700 | ~30 (correcto) |
| Cliente 1 | 700 | ~150.000 (malo) |
| Cliente 1 (primero) | 1 | ~9.000 (correcto) |
| Cliente 700 | 1 | ~9.000 (malo: 300 veces más de lo necesario) |

En el plan real de la ejecución mala, el operador sobre `Pedido` muestra la pista: *Estimated Number of Rows* ~9,5 y *Actual Number of Rows* 50.000. Y en las propiedades del `SELECT` raíz, *Parameter List* muestra los dos valores: `Parameter Compiled Value = 700` y `Parameter Runtime Value = 1`.

**La solución**, para un procedimiento que se ejecuta unas pocas veces por segundo:

```sql
CREATE OR ALTER PROCEDURE dbo.LineasDeCliente
    @ClienteId INT
AS
BEGIN
    SELECT p.PedidoId, d.ProductoId, d.Cantidad
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.ClienteId = @ClienteId
    OPTION (RECOMPILE);
END;
```

Qué decisión hay detrás:

* **`RECOMPILE` en la consulta, no en el procedimiento:** `CREATE PROCEDURE ... WITH RECOMPILE` recompila todo el procedimiento; `OPTION (RECOMPILE)` solo la consulta afectada. Además, con `OPTION (RECOMPILE)` el optimizador puede usar el valor del parámetro como si fuera un literal.
* **Por qué no `OPTIMIZE FOR (@ClienteId = 700)`:** daría el plan bueno para los clientes comunes y el malo para el mayorista, de forma estable. Puede ser aceptable si el mayorista es uno solo y tolera la lentitud, pero es una decisión de negocio.
* **Por qué no un índice:** la diferencia no está en el acceso a `Pedido` (que ya tiene un índice cubriente) sino en cómo unir con `DetallePedido`, y ahí el mejor algoritmo realmente depende de la cantidad de filas.
* **Si el procedimiento se ejecutara miles de veces por segundo,** el costo de compilar cada vez sería relevante, y convendría separar el caso del mayorista (otro procedimiento, o una rama por cantidad de pedidos) o forzar un plan con Query Store.

-----

## Errores comunes

**1. "Arreglar" el rendimiento con `DBCC FREEPROCCACHE`.**
Qué pasa: todo mejora un rato y el problema vuelve.
Por qué: el plan malo se descarta, pero la causa (un plan que depende del primer valor) sigue ahí.
Arreglo: diagnosticar con el plan real (valor compilado frente a valor de ejecución) y aplicar una de las soluciones.

**2. Hints como primera opción.**
Qué pasa: un `WITH (INDEX(...))` que hoy acelera y en seis meses, con otros datos, es el peor plan posible.
Por qué: el hint congela una decisión que el optimizador ya no puede corregir.
Arreglo: índices, reescritura y estadísticas primero; el hint, documentado y revisado periódicamente.

**3. Ignorar las estadísticas después de una carga masiva.**
Qué pasa: un proceso que carga un millón de filas y después consulta, con planes estimados para la tabla de antes.
Por qué: la actualización automática puede no haberse disparado todavía.
Arreglo: `UPDATE STATISTICS` al final de las cargas grandes.

**4. Variables de tabla con miles de filas.**
Qué pasa: joins con *Nested Loops* que se ejecutan miles de veces.
Por qué: sin histograma, la estimación no refleja la distribución real.
Arreglo: tablas temporales para volúmenes grandes.

**5. Variables locales para "evitar el sniffing" sin saber lo que hacen.**
Qué pasa: un plan mediocre para todos los valores, sin que nadie sepa por qué.
Por qué: el optimizador usa la densidad promedio, igual que con `OPTIMIZE FOR UNKNOWN`.
Arreglo: el hint explícito, documentado, si esa es la intención.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Estadísticas | Por índice y por columna, con histograma de 200 tramos | Por columna (`pg_stats`), con valores más frecuentes e histograma | Histogramas opcionales (desde 8.0, `ANALYZE TABLE ... UPDATE HISTOGRAM`) y estadísticas de índices |
| Actualizarlas | `UPDATE STATISTICS`, automático por umbral | `ANALYZE`, automático con *autovacuum* | `ANALYZE TABLE`, automático (InnoDB persistente) |
| Caché de planes | Sí, compartida para consultas parametrizadas | Por sesión, para sentencias preparadas | No guarda planes entre ejecuciones |
| Parameter sniffing | Sí (problema clásico) | Sentencias preparadas: plan "genérico" tras 5 ejecuciones (`plan_cache_mode`) | No aplica de la misma forma |
| Hints | `OPTION (...)`, `WITH (...)`, `USE HINT` | **No nativos** (extensión `pg_hint_plan`); se ajusta con parámetros (`enable_nestloop`...) | Hints de optimizador (`/*+ ... */`) e índices (`USE INDEX`, `FORCE INDEX`) |
| Fijar planes | Query Store, *plan guides* | Extensiones | No |

La ausencia de hints en PostgreSQL es una decisión de diseño de su comunidad: preferir que se corrijan las estadísticas y el optimizador antes que permitir decisiones congeladas.

### En MongoDB

MongoDB elige el plan **probando**: cuando llega una forma de consulta nueva, ejecuta en paralelo los planes candidatos durante un rato, se queda con el que produjo resultados más rápido y lo guarda en la caché de planes para esa forma de consulta. Tiene su propio equivalente del *sniffing*: un plan elegido con valores poco representativos puede quedar en caché. `hint()` fuerza un índice (`db.pedidos.find({...}).hint({ clienteId: 1 })`), con las mismas advertencias que en SQL Server.

-----

## Cuándo sí y cuándo no

**Sospecha de parameter sniffing cuando:**

* El rendimiento cambia sin cambios de código, o depende de qué valor se usó primero.
* El plan real muestra un valor compilado distinto del de ejecución y filas estimadas muy distintas de las reales.

**Usa hints cuando:**

* Ya probaste índices, reescritura y estadísticas, tienes la medición que lo justifica y documentas por qué.

**Ten en cuenta:**

* Cambiar el nivel de compatibilidad de la base cambia el *cardinality estimator* y puede cambiar los planes de todas las consultas. Se hace con pruebas, y Query Store permite comparar antes y después.

-----

## Resumen en 5 líneas

1. El optimizador estima filas con las estadísticas: un histograma de hasta 200 tramos de la primera columna de cada índice.
2. Las estadísticas envejecen; actualízalas después de cambios grandes, y revisa `modification_counter` y `last_updated`.
3. Los planes se reutilizan desde la caché: un plan compilado para un valor atípico se aplica a todos (*parameter sniffing*).
4. Soluciones: mejor índice, `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, separar casos o forzar un plan con Query Store; nunca `FREEPROCCACHE` como arreglo.
5. Hints al final, documentados: congelan una decisión que el optimizador ya no puede corregir.

-----

## Para profundizar

<details>
<summary>Cardinality estimator: el viejo y el nuevo</summary>

SQL Server 2014 introdujo un nuevo modelo de estimación de cardinalidad (CE 120), que cambió supuestos importantes: por ejemplo, el viejo asumía que los filtros sobre columnas distintas eran totalmente independientes (multiplicaba sus selectividades), y el nuevo asume una correlación parcial (*exponential backoff*). El CE que se usa depende del nivel de compatibilidad de la base. Al migrar de versión, algunas consultas mejoran y otras empeoran; por eso existen la opción `LEGACY_CARDINALITY_ESTIMATION` de la base y el hint `USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION')`, que permiten volver al viejo para casos puntuales.

</details>

<details>
<summary>Intelligent Query Processing</summary>

Desde SQL Server 2017, una familia de mejoras llamada *Intelligent Query Processing* corrige automáticamente algunos de estos problemas: *memory grant feedback* (ajusta la memoria reservada según las ejecuciones anteriores), *adaptive joins* (elige entre *Nested Loops* y *Hash Match* durante la ejecución, según las filas reales), *table variable deferred compilation* y *scalar UDF inlining* (2019), y *Parameter Sensitive Plan optimization* y *DOP feedback* (2022). La mayoría se activan con el nivel de compatibilidad más reciente, y es una de las mejores razones para mantenerlo actualizado.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El optimizador usa estadísticas para estimar cuántas filas devuelve cada filtro y elegir el plan. Si las estadísticas están desactualizadas, elige mal, y se arreglan con `UPDATE STATISTICS`. El *parameter sniffing* es cuando un procedimiento se compila con el primer valor del parámetro y ese plan se usa para todos, aunque no les sirva.

### Respuesta ampliada (semi-senior)

El optimizador estima la cardinalidad con estadísticas (histograma de 200 tramos de la primera columna y densidades), y elige el plan según esas estimaciones; por eso, cuando el plan real muestra filas estimadas muy distintas de las reales, investigo estadísticas desactualizadas (claves ascendentes, cargas masivas), variables de tabla, funciones escalares o columnas correlacionadas. Los planes de consultas parametrizadas se reutilizan desde la caché, y si la distribución es muy desigual, el plan compilado para el primer valor puede ser malo para los demás: lo confirmo comparando el valor compilado con el de ejecución en el plan real. Según la frecuencia de ejecución, lo resuelvo con un mejor índice, `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, separando los casos atípicos o forzando un plan con Query Store; en SQL Server 2022, PSP puede resolver algunos casos solo. No uso `FREEPROCCACHE` como solución, y dejo los hints como último recurso, documentados, porque congelan decisiones que el optimizador ya no puede corregir cuando cambian los datos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es el parameter sniffing y por qué no es siempre malo?**
Es compilar el plan con el valor real del primer parámetro. Es bueno porque permite usar el histograma; es malo cuando la distribución es tan desigual que un plan no sirve para todos los valores.

**2. ¿Qué diferencia hay entre `OPTION (RECOMPILE)` y `WITH RECOMPILE` en el procedimiento?**
El primero recompila solo esa consulta (y puede tratar el parámetro como literal); el segundo recompila todo el procedimiento en cada ejecución.

**3. ¿Cuándo se actualizan las estadísticas automáticamente?**
Cuando la cantidad de modificaciones supera un umbral; desde SQL Server 2016 (compatibilidad 130) es dinámico, aproximadamente la raíz de 1.000 × filas en tablas grandes.

**4. ¿Por qué un hint puede ser peligroso?**
Porque fuerza una decisión fija; si los datos cambian, el optimizador ya no puede elegir un plan mejor.

-----

## Práctica

**Ejercicio 1.** Con el procedimiento `dbo.LineasDeCliente` original (sin `RECOMPILE`), reproduce el *parameter sniffing* y encuentra, en el plan real, los valores compilado y de ejecución.

<details>
<summary>Solución</summary>

```sql
EXEC sp_recompile 'dbo.LineasDeCliente';   -- empezar de cero
EXEC dbo.LineasDeCliente @ClienteId = 700;
-- Ctrl + M
EXEC dbo.LineasDeCliente @ClienteId = 1;
```

En el plan real de la segunda ejecución, clic en el operador `SELECT` raíz → `F4` (propiedades) → *Parameter List* → `@ClienteId`:

* *Parameter Compiled Value:* `(700)`
* *Parameter Runtime Value:* `(1)`

Y en el operador que lee `Pedido`, *Estimated Number of Rows* ~9,5 frente a *Actual Number of Rows* 50.000. Las dos pistas juntas confirman el *sniffing*.

</details>

**Ejercicio 2.** Escribe una consulta que muestre las estadísticas de `dbo.Pedido` con su última actualización y las modificaciones acumuladas desde entonces. Después, carga 50.000 pedidos nuevos del día de hoy y vuelve a consultarla. ¿Qué esperas ver?

<details>
<summary>Solución</summary>

```sql
SELECT s.name AS Estadistica, sp.last_updated, sp.rows, sp.modification_counter
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID('dbo.Pedido');

INSERT INTO dbo.Pedido (ClienteId, FechaPedido, Estado)
SELECT 2 + n % 99999, SYSUTCDATETIME(), 'Pendiente'
FROM dbo.Numero WHERE n <= 50000;
```

Después de la carga, `modification_counter` de cada estadística aumenta en 50.000 y `last_updated` no cambia todavía: la actualización automática no se dispara al insertar, sino la próxima vez que una consulta **use** esas estadísticas y el umbral se haya superado (con un millón de filas, unas 31.600 modificaciones). Una consulta como `SELECT COUNT(*) FROM dbo.Pedido WHERE FechaPedido >= CAST(SYSUTCDATETIME() AS DATE)` la dispara. Para no depender de eso al final de una carga, `UPDATE STATISTICS dbo.Pedido;`.

</details>

-----

## Siguiente lección

[Consultas SARGables y antipatrones](06-Consultas%20SARGables%20y%20antipatrones.md)
