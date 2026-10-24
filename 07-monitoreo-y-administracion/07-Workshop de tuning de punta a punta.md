# Workshop de tuning de punta a punta

## En una frase

Optimizar no es probar índices al azar hasta que algo mejore: es un **método** de cinco pasos (**identificar** lo que más cuesta, **analizar** su plan y sus lecturas, **diagnosticar** la causa, **optimizar** con el cambio mínimo y **validar** con números antes y después) que usa todas las herramientas de los módulos anteriores.

-----

## Antes de empezar

Esta lección integra los módulos de [Índices y rendimiento](../04-indices-y-rendimiento/README.md), [Transacciones y concurrencia](../05-transacciones-y-concurrencia/README.md), [Programación T-SQL](../06-programacion-t-sql/README.md) y las lecciones anteriores de este módulo. Se trabaja sobre `TiendaGrande` recién generada, con el script de [Datos de volumen](../04-indices-y-rendimiento/00-Datos%20de%20volumen.md) (sin los índices de las lecciones anteriores).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Línea base (baseline):** las mediciones antes de cambiar nada, contra las que se compara.
* **Cambio mínimo:** la modificación más pequeña que corrige la causa, para poder atribuirle la mejora.

-----

## El problema

El lunes llega el mensaje: "el sistema está lento". El reflejo es abrir la consulta que uno sospecha, agregarle un índice que sugiere SSMS, ver que "ahora anda más rápido" (porque los datos ya están en memoria) y dar el tema por cerrado. Una semana después, el sistema sigue lento, la tabla tiene tres índices nuevos que nadie usa, y nadie sabe cuál de los cambios sirvió para algo.

El tuning sin método falla de tres formas: **optimiza lo que no importa** (la consulta que uno conoce, no la que más consume), **no encuentra la causa** (agrega un índice cuando el problema era una conversión implícita) y **no puede demostrar nada** (sin medición previa, "mejoró" es una impresión).

-----

## Cómo funciona

### Los cinco pasos

```text
1. IDENTIFICAR   ¿Qué consume más? (no lo que sospecho)       Query Store, DMVs, Extended Events
       │
2. ANALIZAR      ¿Cómo se ejecuta y cuánto lee?               Plan real, STATISTICS IO/TIME
       │
3. DIAGNOSTICAR  ¿Por qué? Una causa concreta                 Scans, lookups, conversiones, estimaciones
       │
4. OPTIMIZAR     El cambio mínimo que ataca esa causa         Reescritura SARGable, índice, set-based
       │
5. VALIDAR       ¿Mejoró? ¿Empeoró otra cosa?                 Mismas mediciones, Query Store después
       │
       └──► volver a 1 con la siguiente consulta
```

### Paso 1: identificar

La pregunta no es "¿qué consulta es lenta?", sino "¿qué consulta consume más **en total**?". Una consulta de 50 ms que se ejecuta 200.000 veces por día pesa más que un reporte de 30 segundos que se ejecuta una vez.

```sql
-- Desde Query Store: lo que más CPU consumió en las últimas 24 horas
SELECT TOP (10)
       q.query_id,
       SUM(rs.count_executions)                                    AS Ejecuciones,
       SUM(rs.avg_cpu_time * rs.count_executions) / 1000           AS CpuTotalMs,
       SUM(rs.avg_logical_io_reads * rs.count_executions)          AS LecturasTotales,
       MAX(qt.query_sql_text)                                      AS Texto
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval AS i ON i.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE i.start_time >= DATEADD(hour, -24, SYSUTCDATETIME())
GROUP BY q.query_id
ORDER BY CpuTotalMs DESC;
```

Sin Query Store, `sys.dm_exec_query_stats` (ver [DMVs para diagnóstico](01-DMVs%20para%20diagnóstico.md)), ordenando por `total_worker_time` o `total_logical_reads`, no por el promedio. Los tiempos de esa vista están en **microsegundos**.

### Paso 2: analizar

Con la consulta identificada, se mide con el **plan real** (Ctrl+M en SSMS) y las estadísticas de la sesión:

```sql
SET STATISTICS IO, TIME ON;
-- la consulta
SET STATISTICS IO, TIME OFF;
```

* **Lecturas lógicas** (`logical reads`): páginas leídas de memoria. Es la métrica principal: no depende de si los datos estaban en caché.
* **Tiempo de CPU** frente a **tiempo transcurrido**: mucha diferencia significa esperas (disco, bloqueos, red), no cálculo.
* El plan real, con **filas estimadas frente a reales** en cada operador (ver [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md)).

Ejecutar la consulta dos veces y medir la segunda evita confundir la carga inicial desde disco con el costo de la consulta.

### Paso 3: diagnosticar

Lo que se busca en el plan y su causa habitual:

| Síntoma | Causa habitual | Dónde se vio |
| --- | --- | --- |
| *Scan* con un filtro selectivo | Predicado no SARGable o falta de índice | [Consultas SARGables](../04-indices-y-rendimiento/06-Consultas%20SARGables%20y%20antipatrones.md) |
| Advertencia de conversión implícita | Tipos distintos entre columna y parámetro | Ídem |
| *Key Lookup* con miles de ejecuciones | El índice no cubre la consulta | [Índices compuestos y cubrientes](../04-indices-y-rendimiento/03-Índices%20compuestos,%20cubrientes%20y%20filtrados.md) |
| Filas estimadas muy distintas de las reales | Estadísticas viejas, *parameter sniffing*, variable de tabla | [Planes II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md) |
| *Sort* o *Hash* con advertencia de *spill* | Memoria estimada insuficiente por una mala estimación | Ídem |
| Función escalar que lee tablas | Una consulta escondida por cada fila | [Funciones](../06-programacion-t-sql/03-Funciones.md) |
| *Index Spool* | El optimizador construye un índice temporal en cada ejecución | Falta un índice permanente |
| Tiempo transcurrido muy mayor que la CPU | Esperas: bloqueos, disco | [DMVs](01-DMVs%20para%20diagnóstico.md), [Bloqueos](../05-transacciones-y-concurrencia/03-Bloqueos,%20hints%20y%20NOLOCK.md) |

Un operador "caro" no es un problema por sí mismo: un *Hash Match* sobre un millón de filas es la forma correcta de unir un millón de filas, y un *scan* es lo correcto cuando la consulta necesita casi toda la tabla. Lo que importa es si el trabajo es **proporcional al resultado**.

### Paso 4: optimizar

El orden de preferencia, de menos a más invasivo:

1. **Reescribir la consulta**: hacerla SARGable, quitar la función escalar, corregir el tipo del parámetro. No agrega costo a nadie.
2. **Ajustar o crear un índice**: preferir ampliar uno existente (una columna más en el `INCLUDE`) antes que crear uno nuevo; cada índice cuesta en cada escritura.
3. **Actualizar estadísticas** cuando la estimación es el problema.
4. **Cambiar el diseño**: columnas calculadas persistidas, desnormalización controlada, tablas de resumen.
5. **Hints o plan forzado** como último recurso, documentado, con Query Store (ver [Query Store](04-Query%20Store.md)).

**Un cambio por vez.** Si se cambian tres cosas juntas y mejora, no se sabe cuál sirvió, y las otras dos quedan como costo sin beneficio.

### Paso 5: validar

* Las **mismas mediciones** que en el paso 2, con los mismos parámetros, y también con parámetros distintos (el cliente con muchos pedidos y el que tiene uno).
* El impacto en las **escrituras**, si se creó un índice.
* Después del despliegue, en **Query Store**: la consulta debe bajar en el ranking del paso 1, y ninguna otra debe haber subido (un índice nuevo puede cambiar el plan de otras consultas).
* **Documentar**: qué se encontró, qué se cambió, los números antes y después.

-----

## Ejemplo completo

El reporte de ventas por ciudad del backoffice "tarda una eternidad", y Query Store confirma que está entre las tres consultas con más lecturas totales: se ejecuta desde un procedimiento cada vez que alguien abre el panel. Partimos de `TiendaGrande` sin índices adicionales y con `dbo.fnTotalPedido` de [Funciones](../06-programacion-t-sql/03-Funciones.md) creada en esa base, en un nivel de compatibilidad menor a 150 (o con el *inlining* deshabilitado), que es como corre en muchos servidores.

**Antes:** el procedimiento original.

```sql
CREATE OR ALTER PROCEDURE dbo.VentasPorCiudad
    @Ciudad NVARCHAR(80),
    @Anio   INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT c.ClienteId, c.Nombre,
           COUNT(*)                         AS Pedidos,
           SUM(dbo.fnTotalPedido(p.PedidoId)) AS Total
    FROM dbo.Cliente AS c
    JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
    WHERE ISNULL(c.Ciudad, '') = @Ciudad
      AND YEAR(p.FechaPedido) = @Anio
      AND p.Estado <> 'Cancelado'
    GROUP BY c.ClienteId, c.Nombre
    ORDER BY Total DESC;
END;
```

**Paso 2, analizar.** Con `SET STATISTICS IO, TIME ON` y el plan real, para `@Ciudad = N'Cusco'` y `@Anio = 2026`:

* Un *Clustered Index Scan* de `Pedido` completo (un millón de filas) y otro de `Cliente`.
* El plan no muestra el costo de la función, pero `STATISTICS IO` revela las lecturas sobre `DetallePedido` que hace por cada pedido, y con el panel abierto por varios usuarios, el tiempo de CPU se multiplica.
* El plan es serial: una función escalar no *inlineable* impide el paralelismo.

**Paso 3, diagnosticar.** Tres causas concretas, ninguna se arregla con "un índice":

1. `YEAR(p.FechaPedido) = @Anio` no es SARGable: obliga a leer todas las fechas.
2. `ISNULL(c.Ciudad, '') = @Ciudad` tampoco lo es, y además es innecesario: como `@Ciudad` nunca es una cadena vacía, `ISNULL(Ciudad, '') = 'Cusco'` equivale a `Ciudad = 'Cusco'` (ver [los apuntes corregidos del módulo 04](../04-indices-y-rendimiento/README.md)).
3. `dbo.fnTotalPedido` ejecuta una consulta a `DetallePedido` **por cada pedido**.

**Paso 4, optimizar.** Primero la reescritura, después el índice que la reescritura permite aprovechar:

```sql
CREATE OR ALTER PROCEDURE dbo.VentasPorCiudad
    @Ciudad NVARCHAR(80),
    @Anio   INT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Desde DATETIME2(0) = DATEFROMPARTS(@Anio, 1, 1),
            @Hasta DATETIME2(0) = DATEFROMPARTS(@Anio + 1, 1, 1);

    SELECT c.ClienteId, c.Nombre,
           COUNT(DISTINCT p.PedidoId)              AS Pedidos,
           SUM(d.Cantidad * d.PrecioUnitario)      AS Total
    FROM dbo.Cliente AS c
    JOIN dbo.Pedido AS p        ON p.ClienteId = c.ClienteId
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE c.Ciudad = @Ciudad                                  -- SARGable
      AND p.FechaPedido >= @Desde AND p.FechaPedido < @Hasta  -- rango semiabierto
      AND p.Estado <> 'Cancelado'
    GROUP BY c.ClienteId, c.Nombre
    ORDER BY Total DESC;
END;
GO

-- Índices para los caminos que ahora existen
CREATE INDEX IX_Cliente_Ciudad ON dbo.Cliente (Ciudad) INCLUDE (Nombre);
CREATE INDEX IX_Pedido_ClienteId_FechaPedido ON dbo.Pedido (ClienteId, FechaPedido) INCLUDE (Estado);
CREATE INDEX IX_DetallePedido_PedidoId ON dbo.DetallePedido (PedidoId) INCLUDE (Cantidad, PrecioUnitario);
```

* La función se reemplaza por un `JOIN` y un `SUM`: una sola pasada por las líneas en lugar de una consulta por pedido. Como el `JOIN` con las líneas multiplica las filas, `COUNT(*)` pasa a ser `COUNT(DISTINCT p.PedidoId)`.
* Las fechas se convierten en un rango semiabierto calculado una vez.
* `IX_Pedido_ClienteId_FechaPedido` sigue la regla de igualdad primero y rango después: por cada cliente de la ciudad, un *seek* a su rango de fechas.
* `IX_DetallePedido_PedidoId`: la clave primaria de `DetallePedido` ya empieza por `PedidoId` y es *clustered*, así que este índice solo conviene si las lecturas de la tabla completa son caras; **es un candidato a descartar en la validación** si no aporta.

**Paso 5, validar.** Se repiten las mediciones del paso 2 con los mismos parámetros, y también con `@Ciudad = N'Lima'` (60 % de los clientes, donde un *scan* puede seguir siendo lo correcto). Los números exactos dependen del equipo, así que se anotan en una tabla como esta:

| Medición | Antes | Después |
| --- | --- | --- |
| Lecturas lógicas, `Pedido` | | |
| Lecturas lógicas, `DetallePedido` | | |
| Lecturas lógicas, `Cliente` | | |
| Tiempo de CPU (ms) | | |
| Tiempo transcurrido (ms) | | |
| Plan paralelo | No | |
| Resultado idéntico (`EXCEPT` entre ambas versiones) | — | |

La última fila es la que más se olvida: **una optimización que cambia el resultado es un error**, no una mejora. Guardar el resultado de las dos versiones en tablas temporales y comparar con `EXCEPT` en ambos sentidos lo comprueba.

Qué cambió y por qué:

* **Se atacaron causas, no síntomas:** dos predicados no SARGables y una función escalar con una consulta adentro. El índice vino después, cuando la consulta ya podía usarlo.
* **Un cambio medido por vez:** primero la reescritura, después cada índice, con su medición.
* **Se probó con parámetros distintos:** Cusco y Lima tienen distribuciones muy distintas.
* **Se verificó el resultado:** la reescritura cambió la forma de contar los pedidos, y el `JOIN` con `DetallePedido` excluiría un pedido sin líneas (que la versión original contaba con total `NULL`). En `TiendaGrande` todo pedido tiene líneas, pero es exactamente el tipo de diferencia que la comparación con `EXCEPT` tiene que descartar.

-----

## Errores comunes

**1. Optimizar lo que uno sospecha, no lo que más consume.**
Qué pasa: se mejora una consulta que no pesaba, y el sistema sigue lento.
Por qué: sin medir, se elige por intuición.
Arreglo: el paso 1 con Query Store o las DMVs, ordenando por consumo total.

**2. Medir con el tiempo de una sola ejecución.**
Qué pasa: "bajó de 8 a 1 segundo", pero solo porque la segunda vez los datos estaban en memoria.
Por qué: el tiempo depende de la caché, de la carga y de la red.
Arreglo: lecturas lógicas y CPU, ejecutando varias veces.

**3. Crear todos los índices que sugiere el plan.**
Qué pasa: la tabla acumula índices parecidos, y las escrituras se vuelven lentas.
Por qué: la sugerencia mira una consulta aislada, sin considerar los índices existentes ni el costo de mantenerlos.
Arreglo: evaluar cada sugerencia, preferir ampliar un índice existente y validar su uso después con `sys.dm_db_index_usage_stats`.

**4. Reescribir con `UNION ALL` un `OR` y perder filas.**
Qué pasa: la versión "optimizada" devuelve menos filas.
Por qué: el patrón `... WHERE Col2 = 'B' AND Col1 <> 'A'` excluye las filas donde `Col1` es `NULL` (la comparación da *unknown*).
Arreglo: `AND (Col1 <> 'A' OR Col1 IS NULL)`, y verificar siempre con `EXCEPT` que el resultado es el mismo.

**5. Cambiar varias cosas a la vez.**
Qué pasa: mejora, pero no se sabe por qué, y los cambios inútiles quedan para siempre.
Por qué: apuro.
Arreglo: un cambio, una medición.

**6. No validar después del despliegue.**
Qué pasa: el índice nuevo mejoró una consulta y cambió el plan de otra, que ahora es más lenta.
Por qué: los índices y las estadísticas afectan a todas las consultas de la tabla.
Arreglo: revisar Query Store (consultas que empeoraron) los días siguientes.

-----

## Según el motor

| Paso | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Identificar | Query Store, `sys.dm_exec_query_stats` | `pg_stat_statements` | `performance_schema`, `sys.statement_analysis`, *slow query log* |
| Analizar | Plan real, `STATISTICS IO/TIME` | `EXPLAIN (ANALYZE, BUFFERS)` | `EXPLAIN ANALYZE` (8.0.18+) |
| Índices sin uso | `sys.dm_db_index_usage_stats` | `pg_stat_user_indexes` (`idx_scan`) | `sys.schema_unused_indexes` |
| Estadísticas | `UPDATE STATISTICS` | `ANALYZE` | `ANALYZE TABLE` |

El método es el mismo en todos los motores; cambian las herramientas.

### En MongoDB

El mismo método: el **profiler** de la base (`db.setProfilingLevel(1, { slowms: 100 })`) o el *Performance Advisor* de Atlas identifican las operaciones lentas; `explain("executionStats")` muestra si usó un índice (`IXSCAN`) o recorrió la colección (`COLLSCAN`), con `totalDocsExamined` frente a `nReturned` como equivalente de "trabajo proporcional al resultado"; y la regla ESR guía el diseño de índices compuestos.

-----

## Cuándo sí y cuándo no

**Aplica el método completo cuando:**

* Hay una queja de rendimiento o una consulta que sube en el ranking de consumo.
* Antes de un despliegue con cambios en consultas importantes o en índices.

**No optimices:**

* Lo que no pesa en el consumo total, aunque su plan "se vea feo".
* Sin una línea base: sin números de antes, no hay forma de saber si mejoró.

**Lista de verificación final:**

* [ ] La consulta está entre las que más consumen (paso 1), no es una sospecha.
* [ ] Se midieron lecturas lógicas y CPU, no solo tiempo.
* [ ] El plan real se leyó comparando filas estimadas y reales.
* [ ] Predicados SARGables: sin funciones sobre columnas, sin conversiones implícitas.
* [ ] Sin funciones escalares que lean tablas, sin cursores ni bucles donde basta una operación sobre conjuntos.
* [ ] Índices: se evaluó ampliar uno existente antes de crear otro; sin *Key Lookups* masivos.
* [ ] Estadísticas actualizadas en las tablas involucradas.
* [ ] Probado con parámetros representativos de distintas distribuciones.
* [ ] El resultado es idéntico al original (`EXCEPT` en ambos sentidos).
* [ ] Las operaciones masivas, en lotes; las transacciones, cortas.
* [ ] Validado en Query Store después del despliegue.
* [ ] Documentado: causa, cambio, números antes y después.

-----

## Resumen en 5 líneas

1. Cinco pasos: identificar, analizar, diagnosticar, optimizar y validar; sin saltear ninguno.
2. Se optimiza lo que más consume en total, medido con Query Store o las DMVs, no lo que uno sospecha.
3. Las lecturas lógicas y la CPU son las métricas; el tiempo de una ejecución engaña.
4. Primero la reescritura (SARGable, sin funciones escalares, sin RBAR), después los índices, y un cambio por vez.
5. Se valida con los mismos números, con parámetros distintos, con el resultado idéntico y en Query Store después del despliegue.

-----

## Para profundizar

<details>
<summary>EXISTS frente a IN: el mito</summary>

Tu cheat sheet recomienda `EXISTS` en lugar de `IN` con subconsultas porque sería "más eficiente". En SQL Server, `WHERE ClienteId IN (SELECT ClienteId FROM ...)` y su versión con `EXISTS` producen **el mismo plan**: el optimizador convierte las dos en un *semi join*. La diferencia real está en las versiones negadas: `NOT IN` con una subconsulta que devuelve algún `NULL` no devuelve **ninguna fila** (por la lógica de tres valores, ver [NULL](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md)), mientras que `NOT EXISTS` se comporta como se espera. La recomendación correcta es: `NOT EXISTS` en lugar de `NOT IN`, por **corrección**, no por rendimiento.

</details>

<details>
<summary>Índices sin uso: la consulta que no los encuentra</summary>

La consulta habitual para encontrar índices sin uso hace un `LEFT JOIN` con `sys.dm_db_index_usage_stats` y filtra `WHERE ius.user_seeks = 0 AND ius.user_scans = 0`. Pero un índice que **nunca** se usó no tiene fila en esa vista, así que sus columnas son `NULL`, y `NULL = 0` no es verdadero: la consulta excluye justamente los índices que busca. La versión correcta:

```sql
SELECT OBJECT_NAME(i.object_id) AS Tabla, i.name AS Indice,
       ISNULL(ius.user_seeks + ius.user_scans + ius.user_lookups, 0) AS Lecturas,
       ISNULL(ius.user_updates, 0) AS Escrituras
FROM sys.indexes AS i
LEFT JOIN sys.dm_db_index_usage_stats AS ius
       ON ius.object_id = i.object_id AND ius.index_id = i.index_id AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.type_desc = 'NONCLUSTERED'
  AND i.is_primary_key = 0 AND i.is_unique_constraint = 0
  AND ISNULL(ius.user_seeks + ius.user_scans + ius.user_lookups, 0) = 0
ORDER BY Escrituras DESC;
```

Además, la vista **se reinicia** cuando se reinicia el servidor: antes de borrar un índice, conviene que haya pasado un ciclo completo de uso (incluido el cierre de mes, que corre reportes que no se ejecutan el resto del tiempo).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Primero busco qué consultas consumen más, después miro su plan de ejecución y las lecturas con `STATISTICS IO`, encuentro la causa (un *scan*, un *lookup*, una función en el `WHERE`), aplico el cambio y comparo antes y después.

### Respuesta ampliada (semi-senior)

Sigo cinco pasos. Identifico lo que más consume en total con Query Store o `sys.dm_exec_query_stats`, no lo que sospecho. Analizo con el plan real y `STATISTICS IO, TIME`, usando las lecturas lógicas como métrica principal y comparando filas estimadas con reales. Diagnostico una causa concreta: predicados no SARGables, conversiones implícitas, *lookups* masivos, malas estimaciones por estadísticas o *parameter sniffing*, funciones escalares que leen tablas, procesamiento fila por fila o esperas por bloqueos. Optimizo de lo menos a lo más invasivo (reescritura, ajuste de índices existentes, índices nuevos, estadísticas, diseño, y hints o planes forzados como último recurso), un cambio por vez. Valido con las mismas mediciones y con parámetros de distribuciones distintas, verifico que el resultado sea idéntico con `EXCEPT` y, después del despliegue, reviso en Query Store que ninguna otra consulta haya empeorado. Y documento la causa, el cambio y los números.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué lecturas lógicas y no tiempo?**
Porque el tiempo depende de la caché, la carga y la red; las lecturas lógicas miden el trabajo de la consulta.

**2. ¿`EXISTS` es más rápido que `IN`?**
En SQL Server producen el mismo plan. La diferencia importante es `NOT IN` con `NULL`, que no devuelve filas; por eso se prefiere `NOT EXISTS`.

**3. ¿Crearías el índice que sugiere el plan?**
Lo evalúo: comparo con los índices existentes, considero ampliar uno antes que crear otro y mido el costo en las escrituras.

**4. ¿Cómo sabes que la optimización no rompió nada?**
Comparando el resultado de las dos versiones con `EXCEPT` en ambos sentidos, y revisando en Query Store las demás consultas de la tabla.

-----

## Práctica

**Ejercicio 1.** Una consulta busca pedidos por cliente o por estado. Alguien la "optimizó" así. ¿Devuelve lo mismo que la original?

```sql
-- Original (sobre una vista que expone Pedido con una columna Canal NULL para los pedidos antiguos)
SELECT PedidoId FROM dbo.vPedido WHERE Canal = 'Web' OR Estado = 'Pendiente';

-- "Optimizada"
SELECT PedidoId FROM dbo.vPedido WHERE Canal = 'Web'
UNION ALL
SELECT PedidoId FROM dbo.vPedido WHERE Estado = 'Pendiente' AND Canal <> 'Web';
```

<details>
<summary>Solución</summary>

No. Los pedidos pendientes con `Canal` en `NULL` cumplen la original (`Estado = 'Pendiente'` es verdadero, y `unknown OR true` es verdadero), pero en la segunda parte de la "optimizada", `NULL <> 'Web'` es *unknown* y la fila se descarta. La corrección:

```sql
SELECT PedidoId FROM dbo.vPedido WHERE Canal = 'Web'
UNION ALL
SELECT PedidoId FROM dbo.vPedido WHERE Estado = 'Pendiente' AND (Canal <> 'Web' OR Canal IS NULL);
```

Y la comprobación de que ahora coinciden, en ambos sentidos:

```sql
(SELECT PedidoId FROM dbo.vPedido WHERE Canal = 'Web' OR Estado = 'Pendiente')
EXCEPT
(SELECT PedidoId FROM dbo.vPedido WHERE Canal = 'Web'
 UNION ALL
 SELECT PedidoId FROM dbo.vPedido WHERE Estado = 'Pendiente' AND (Canal <> 'Web' OR Canal IS NULL));
-- y lo mismo con las dos partes invertidas: ambas deben devolver 0 filas
```

Antes de hacer esta reescritura conviene comprobar que haga falta: el optimizador ya puede resolver un `OR` con dos *seeks* unidos (*index union*) si hay un índice por cada columna.

</details>

**Ejercicio 2.** Aplica el paso 1 con las DMVs (sin Query Store): escribe una consulta que muestre las 10 consultas con más lecturas lógicas **totales** desde que están en caché, con su promedio por ejecución.

<details>
<summary>Solución</summary>

```sql
SELECT TOP (10)
       qs.execution_count                                   AS Ejecuciones,
       qs.total_logical_reads                               AS LecturasTotales,
       qs.total_logical_reads / qs.execution_count          AS LecturasPromedio,
       qs.total_worker_time / 1000                          AS CpuTotalMs,      -- microsegundos → ms
       SUBSTRING(st.text, qs.statement_start_offset / 2 + 1,
                 (CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2 + 1) AS Sentencia,
       qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_logical_reads DESC;
```

Ordenar por el **total** pone arriba lo que más carga al servidor; el promedio ayuda a distinguir entre "una consulta pesada" y "una consulta liviana que se ejecuta muchísimo", que se optimizan de forma distinta. Los offsets están en bytes de texto Unicode, de ahí la división por 2. Recuerda que esta vista solo contiene lo que sigue en caché: lo que se recompiló o salió de la caché no aparece.

</details>

-----

## Siguiente lección

Con este workshop termina el módulo. El siguiente es [NoSQL con MongoDB](../README.md#-estructura), que está planificado.
