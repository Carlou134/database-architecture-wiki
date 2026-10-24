# DMVs para diagnóstico

## En una frase

Las vistas de administración dinámica (DMV) son la "telemetría" interna de SQL Server, consultable con T-SQL: qué se está ejecutando ahora, qué consultas consumieron más recursos, en qué se pasa el tiempo esperando el servidor, cómo responden los discos y qué índices se usan; son el primer lugar donde mirar ante cualquier problema de rendimiento, antes de adivinar.

-----

## Antes de empezar

Conviene que ya sepas:

* Planes de ejecución y caché de planes, de [Planes de ejecución I](../04-indices-y-rendimiento/04-Planes%20de%20ejecución%20I.md) y [II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md).
* Bloqueos y esperas, de [Bloqueos, hints y NOLOCK](../05-transacciones-y-concurrencia/03-Bloqueos%2C%20hints%20y%20NOLOCK.md).
* Las DMV de índices, de [Mantenimiento de índices](../04-indices-y-rendimiento/07-Mantenimiento%20de%20índices.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DMV / DMF:** vista / función de administración dinámica; empiezan con `sys.dm_`.
* **Espera (wait):** tiempo en que una tarea no avanza porque necesita algo (un bloqueo, una página del disco, memoria, CPU).
* **Estadísticas de espera (wait stats):** el acumulado de todas las esperas del servidor desde el último reinicio, por tipo.
* **Sesión / petición (session / request):** una conexión / una instrucción que se está ejecutando en ella.
* **`sql_handle` / `plan_handle`:** identificadores con los que se obtiene el texto y el plan de una consulta en caché.
* **Latencia de E/S:** cuánto tarda, en promedio, una lectura o escritura en un archivo de la base.

-----

## El problema

"El sistema está lento." Es todo lo que dice el ticket. Sin datos, el equipo empieza a adivinar: que es la red, que falta un índice, que hay que agregar memoria, que el servidor es viejo. Cada hipótesis cuesta días de pruebas, y la mitad de las veces la causa real era otra.

SQL Server registra internamente, todo el tiempo, casi todo lo que hace falta para responder: qué consultas se ejecutaron y cuánto costaron, en qué estuvo esperando, qué sesiones bloquean a otras, qué tan rápido responden los discos. Las DMV son la forma de **preguntárselo**. La regla de oro del diagnóstico es: **medir antes de cambiar**.

-----

## Cómo funciona

### Qué son y cómo se organizan

Las DMV son vistas (y funciones, las DMF) del esquema `sys` cuyo nombre empieza con `dm_`. Su prefijo dice de qué hablan:

| Prefijo | Sobre qué | Ejemplos |
| --- | --- | --- |
| `sys.dm_exec_*` | Ejecución: sesiones, peticiones, consultas en caché, planes | `dm_exec_requests`, `dm_exec_query_stats`, `dm_exec_sql_text` |
| `sys.dm_os_*` | El sistema: esperas, memoria, CPU, *schedulers* | `dm_os_wait_stats`, `dm_os_performance_counters` |
| `sys.dm_db_*` | La base: índices, estadísticas, espacio | `dm_db_index_usage_stats`, `dm_db_index_physical_stats` |
| `sys.dm_tran_*` | Transacciones y bloqueos | `dm_tran_locks`, `dm_tran_active_transactions` |
| `sys.dm_io_*` | Entrada y salida a disco | `dm_io_virtual_file_stats` |

Tres cosas a tener en cuenta:

* **Permisos:** consultar la mayoría requiere `VIEW SERVER STATE` (en SQL Server 2022, también los permisos más granulares `VIEW SERVER PERFORMANCE STATE` y similares).
* **Se reinician:** la mayoría acumula desde el último reinicio del servidor (o desde que el plan entró en caché). Un dato de "las consultas más costosas" de hace 2 horas no es lo mismo que el de hace 2 meses.
* **Son una foto:** muestran el estado en el momento de consultarlas. Para ver tendencias hay que guardarlas periódicamente, o usar Query Store (ver [Query Store](04-Query%20Store.md)).

### Qué se está ejecutando ahora

```sql
SELECT r.session_id,
       s.login_name,
       s.host_name,
       r.status,
       r.wait_type,
       r.wait_time / 1000                AS SegundosEsperando,
       r.blocking_session_id             AS BloqueadoPor,
       r.cpu_time / 1000                 AS SegundosCpu,
       r.logical_reads,
       DB_NAME(r.database_id)            AS Base,
       SUBSTRING(t.text, r.statement_start_offset / 2 + 1,
                 (CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(t.text)
                       ELSE r.statement_end_offset END - r.statement_start_offset) / 2 + 1) AS Instruccion
FROM sys.dm_exec_requests AS r
JOIN sys.dm_exec_sessions AS s ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE s.is_user_process = 1
  AND r.session_id <> @@SPID
ORDER BY r.cpu_time DESC;
```

Es la primera consulta ante un "está lento **ahora**". Muestra cada instrucción en ejecución, qué está esperando y si otra sesión la bloquea. Los *offsets* recortan el texto completo del lote para mostrar la **instrucción** concreta que se está ejecutando.

En la práctica, mucha gente usa `sp_WhoIsActive`, un procedimiento gratuito de Adam Machanic que hace lo mismo con mucha más información (el plan, el texto completo, la cadena de bloqueos).

### Las consultas que más consumieron

`sys.dm_exec_query_stats` acumula, para cada instrucción con un plan en caché, cuántas veces se ejecutó y cuánto consumió:

```sql
SELECT TOP (10)
       qs.execution_count                                   AS Ejecuciones,
       qs.total_worker_time / 1000                          AS CpuTotalMs,
       qs.total_worker_time / qs.execution_count / 1000     AS CpuPromedioMs,
       qs.total_logical_reads                               AS LecturasTotales,
       qs.total_logical_reads / qs.execution_count          AS LecturasPromedio,
       qs.total_elapsed_time / qs.execution_count / 1000    AS DuracionPromedioMs,
       SUBSTRING(t.text, qs.statement_start_offset / 2 + 1,
                 (CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(t.text)
                       ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2 + 1) AS Instruccion,
       qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS t
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_worker_time DESC;
```

* **Ordenar por el total** (`total_worker_time`, `total_logical_reads`) encuentra lo que más carga al servidor: una consulta de 50 ms que se ejecuta un millón de veces pesa más que una de 30 segundos que se ejecuta una vez.
* **Ordenar por el promedio** encuentra las consultas individualmente lentas.
* **`query_plan`** es un enlace al plan en caché (el plan **estimado**, sin filas reales).
* Solo ve lo que está **en caché**: las consultas con `OPTION (RECOMPILE)` o cuyo plan se descartó no aparecen.

`sys.dm_exec_procedure_stats` hace lo mismo a nivel de procedimiento almacenado, y `sys.dm_exec_function_stats` para las funciones (ver [Funciones](../06-programacion-t-sql/03-Funciones.md)).

### En qué espera el servidor

Cada vez que una tarea no puede avanzar (espera un bloqueo, una página del disco, memoria), SQL Server registra el tipo y la duración de la espera. Sumadas desde el último reinicio, las **estadísticas de espera** son el mejor resumen de dónde se va el tiempo del servidor:

```sql
SELECT TOP (10)
       wait_type,
       wait_time_ms / 1000                          AS SegundosEsperados,
       waiting_tasks_count                          AS Esperas,
       100.0 * wait_time_ms / SUM(wait_time_ms) OVER () AS Porcentaje
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (   -- esperas "benignas" de procesos en segundo plano (lista parcial)
    'SLEEP_TASK', 'LAZYWRITER_SLEEP', 'SQLTRACE_BUFFER_FLUSH', 'XE_TIMER_EVENT',
    'XE_DISPATCHER_WAIT', 'REQUEST_FOR_DEADLOCK_SEARCH', 'CHECKPOINT_QUEUE',
    'BROKER_TO_FLUSH', 'BROKER_TASK_STOP', 'WAITFOR', 'DIRTY_PAGE_POLL',
    'HADR_FILESTREAM_IOMGR_IOCOMPLETION', 'SP_SERVER_DIAGNOSTICS_SLEEP',
    'QDS_PERSIST_TASK_MAIN_LOOP_SLEEP', 'QDS_ASYNC_QUEUE')
  AND wait_type NOT LIKE 'SLEEP_%'
ORDER BY wait_time_ms DESC;
```

| Espera | Qué suele significar | Por dónde investigar |
| --- | --- | --- |
| `PAGEIOLATCH_SH` / `_EX` | Esperando leer páginas del disco | Consultas que leen mucho (índices que faltan), memoria insuficiente, discos lentos |
| `LCK_M_*` | Esperando un bloqueo | Bloqueos y transacciones largas ([Bloqueos](../05-transacciones-y-concurrencia/03-Bloqueos%2C%20hints%20y%20NOLOCK.md)) |
| `WRITELOG` | Esperando escribir el log al confirmar | Disco del log lento, muchísimas transacciones pequeñas |
| `CXPACKET` / `CXCONSUMER` | Paralelismo | Planes paralelos innecesarios (configuración de `MAXDOP` y `cost threshold for parallelism`), o consultas grandes legítimas |
| `SOS_SCHEDULER_YIELD` | Las tareas ceden la CPU: hay presión de CPU | Consultas con mucho cálculo o muchas lecturas lógicas |
| `RESOURCE_SEMAPHORE` | Esperando memoria para ejecutar | *Memory grants* grandes, por estimaciones infladas |
| `ASYNC_NETWORK_IO` | El **cliente** no lee los resultados lo bastante rápido | La aplicación procesa fila por fila, o pide más filas de las que necesita |

Las esperas no dicen **qué** consulta es la culpable, pero sí **dónde mirar**: un servidor dominado por `PAGEIOLATCH` tiene un problema distinto de uno dominado por `LCK_M_X`. Como son acumuladas, conviene compararlas en dos momentos (guardarlas, esperar una hora y calcular la diferencia) para ver qué pasa **ahora**.

### Cómo responden los discos

```sql
SELECT DB_NAME(vfs.database_id)                                  AS Base,
       mf.type_desc                                              AS Tipo,
       mf.physical_name                                          AS Archivo,
       vfs.io_stall_read_ms  / NULLIF(vfs.num_of_reads, 0)       AS LecturaPromedioMs,
       vfs.io_stall_write_ms / NULLIF(vfs.num_of_writes, 0)      AS EscrituraPromedioMs
FROM sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN sys.master_files AS mf ON mf.database_id = vfs.database_id AND mf.file_id = vfs.file_id
ORDER BY LecturaPromedioMs DESC;
```

Como referencia aproximada: en almacenamiento SSD, latencias de un dígito de milisegundos son normales; decenas de milisegundos sostenidas en los archivos de datos, o más de unos pocos en el log, indican un problema de almacenamiento (o una carga que lee mucho más de lo necesario).

### Otras consultas útiles

| Pregunta | DMV |
| --- | --- |
| ¿Quién bloquea a quién? | `sys.dm_exec_requests` (`blocking_session_id`), `sys.dm_tran_locks` |
| ¿Qué transacciones llevan más tiempo abiertas? | `sys.dm_tran_active_transactions`, `sys.dm_tran_session_transactions` |
| ¿Qué índices no se usan? ¿Cuáles faltan? | `sys.dm_db_index_usage_stats`, `sys.dm_db_missing_index_*` |
| ¿Qué tan fragmentados están los índices? | `sys.dm_db_index_physical_stats` |
| ¿Cuánta memoria usa cada base en la caché? | `sys.dm_os_buffer_descriptors` |
| ¿Cuánto `tempdb` usa cada sesión? | `sys.dm_db_session_space_usage` |
| ¿Cuándo se actualizaron las estadísticas? | `sys.dm_db_stats_properties` |

-----

## Ejemplo completo

Una rutina de diagnóstico de primeros auxilios ante "el sistema está lento", en tres preguntas, sobre `TiendaGrande` con carga (por ejemplo, ejecutando en otra ventana el procedimiento `dbo.LineasDeCliente` con *parameter sniffing* de [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md)):

**1. ¿Qué se está ejecutando ahora, y qué espera?** (la consulta de `dm_exec_requests`)

| session_id | status | wait_type | BloqueadoPor | SegundosCpu | logical_reads | Instruccion |
| --- | --- | --- | --- | --- | --- | --- |
| 63 | running | `NULL` | 0 | 12 | 2.400.000 | `SELECT p.PedidoId, d.ProductoId ... WHERE p.ClienteId = @ClienteId` |

Una instrucción con millones de lecturas lógicas y sin esperar a nadie: no es un bloqueo, es una consulta que trabaja de más.

**2. ¿Es un caso aislado o algo que se repite?** (la consulta de `dm_exec_query_stats`, ordenada por lecturas totales)

| Ejecuciones | LecturasPromedio | DuracionPromedioMs | Instruccion |
| --- | --- | --- | --- |
| 340 | 150.000 | 900 | `SELECT p.PedidoId, d.ProductoId ...` |

La misma instrucción, 340 ejecuciones con 150.000 lecturas de media: para un cliente común debería leer unas decenas de páginas.

**3. ¿Qué plan tiene en caché?** (clic en `query_plan`)

El plan muestra un *Nested Loops* con *seeks* en `DetallePedido`, compilado para un cliente común, que se aplica también al mayorista. El diagnóstico apunta a *parameter sniffing*, y la solución (`OPTION (RECOMPILE)`, o forzar un plan con Query Store) se elige con lo visto en el módulo de índices.

Qué muestra la rutina:

* **Tres consultas a las DMV llevaron de "está lento" a "esta instrucción, con este plan".** Sin cambiar nada, sin adivinar.
* **Primero lo que pasa ahora, después lo acumulado, después el plan:** el orden habitual de un diagnóstico.
* **Las esperas y la latencia de disco** completan el cuadro cuando la consulta lenta no es la causa (por ejemplo, si todo espera `WRITELOG`, el problema es el disco del log).

(Las cifras son ilustrativas: dependen del equipo y de la carga.)

-----

## Errores comunes

**1. Cambiar cosas antes de medir.**
Qué pasa: índices nuevos, memoria agregada o reinicios que no resuelven nada.
Por qué: se actúa por intuición.
Arreglo: consultas activas, consultas costosas, esperas y discos, antes de cualquier cambio.

**2. Interpretar datos acumulados sin saber desde cuándo.**
Qué pasa: se persigue una consulta que fue costosa hace tres semanas, o se ignora una nueva.
Por qué: las DMV acumulan desde el último reinicio o desde que el plan entró en caché.
Arreglo: mirar `sqlserver_start_time` en `sys.dm_os_sys_info` y `creation_time` en `dm_exec_query_stats`; comparar fotos en dos momentos.

**3. Ordenar solo por el promedio.**
Qué pasa: se optimiza una consulta de 20 segundos que corre una vez al mes, y se ignora una de 100 ms que corre un millón de veces al día.
Por qué: el promedio no mide el impacto total.
Arreglo: mirar los totales y los promedios.

**4. No filtrar las esperas benignas.**
Qué pasa: la espera "más importante" resulta ser `SLEEP_TASK`, que no significa nada.
Por qué: muchas tareas en segundo plano registran esperas mientras duermen.
Arreglo: excluir las esperas benignas (las listas de la comunidad, como la de Paul Randal, son el estándar).

**5. Creer que las DMV muestran todo.**
Qué pasa: una consulta lenta no aparece en `dm_exec_query_stats`.
Por qué: su plan ya salió de la caché, o se recompila siempre.
Arreglo: Query Store o Extended Events para lo que la caché no conserva.

-----

## Según el motor

| Pregunta | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Qué se ejecuta ahora | `sys.dm_exec_requests` | `pg_stat_activity` | `SHOW PROCESSLIST`, `performance_schema.threads` |
| Consultas más costosas | `sys.dm_exec_query_stats` | `pg_stat_statements` (extensión) | `performance_schema.events_statements_summary_by_digest`, `sys.statement_analysis` |
| Esperas | `sys.dm_os_wait_stats` | `wait_event` en `pg_stat_activity` (instantáneo; sin acumulado nativo) | `performance_schema.events_waits_summary_*` |
| Bloqueos | `sys.dm_tran_locks` | `pg_locks` | `performance_schema.data_locks` |
| Uso de índices | `sys.dm_db_index_usage_stats` | `pg_stat_user_indexes` | `sys.schema_index_statistics` |
| E/S por archivo | `sys.dm_io_virtual_file_stats` | `pg_stat_io` (desde la 16) | `performance_schema.file_summary_by_instance` |

`pg_stat_statements` en PostgreSQL y el esquema `sys` de MySQL cumplen el mismo papel que `dm_exec_query_stats`, y suelen ser lo primero que se activa al administrar esos motores.

### En MongoDB

El equivalente de las DMV son los comandos de diagnóstico: `db.currentOp()` (operaciones en curso, como `dm_exec_requests`), `db.serverStatus()` (métricas del servidor, conexiones, memoria, operaciones), `db.collection.aggregate([{ $indexStats: {} }])` (uso de índices) y el *Database Profiler* (`db.setProfilingLevel(1, { slowms: 100 })`), que guarda las operaciones lentas en la colección `system.profile`. Las herramientas `mongostat` y `mongotop` muestran actividad en tiempo real.

-----

## Cuándo sí y cuándo no

**Usa las DMV cuando:**

* Hay un problema de rendimiento, ahora o recurrente: siempre son el primer paso.
* Quieres un inventario rápido del estado del servidor (esperas, discos, índices).

**Complementa con otras herramientas cuando:**

* Necesitas historia: Query Store (planes y tiempos por consulta a lo largo del tiempo).
* Necesitas capturar eventos concretos (errores, *deadlocks*, consultas de más de n segundos): Extended Events.

**Herramientas que ayudan:**

* `sp_WhoIsActive` (Adam Machanic) para lo que se ejecuta ahora.
* El *First Responder Kit* de Brent Ozar (`sp_Blitz`, `sp_BlitzCache`, `sp_BlitzFirst`), gratuito, que empaqueta muchas de estas consultas con recomendaciones.

-----

## Resumen en 5 líneas

1. Las DMV (`sys.dm_*`) son la telemetría interna de SQL Server, consultable con T-SQL; requieren `VIEW SERVER STATE`.
2. `dm_exec_requests` muestra lo que se ejecuta ahora; `dm_exec_query_stats`, lo que más consumió desde que está en caché.
3. `dm_os_wait_stats` dice dónde se va el tiempo del servidor: `PAGEIOLATCH`, `LCK_M_*`, `WRITELOG`, `CXPACKET`...
4. `dm_io_virtual_file_stats` mide la latencia de los discos por archivo.
5. Acumulan desde el reinicio y solo ven la caché: mide antes de cambiar, mira totales y promedios, y compara fotos.

-----

## Para profundizar

<details>
<summary>Guardar fotos de las esperas para ver qué pasa ahora</summary>

Como `dm_os_wait_stats` acumula desde el reinicio, una espera que dominó durante una carga de hace un mes puede ocultar lo que pasa hoy. La técnica es guardar la foto, esperar y calcular la diferencia:

```sql
SELECT wait_type, wait_time_ms, waiting_tasks_count
INTO #Antes FROM sys.dm_os_wait_stats;

WAITFOR DELAY '00:05:00';   -- 5 minutos de carga real

SELECT d.wait_type,
       d.wait_time_ms - a.wait_time_ms               AS MsEnElIntervalo,
       d.waiting_tasks_count - a.waiting_tasks_count AS EsperasEnElIntervalo
FROM sys.dm_os_wait_stats AS d
JOIN #Antes AS a ON a.wait_type = d.wait_type
WHERE d.wait_time_ms - a.wait_time_ms > 0
ORDER BY MsEnElIntervalo DESC;
```

`sp_BlitzFirst @Seconds = 300` hace exactamente esto, con las esperas benignas ya filtradas.

</details>

<details>
<summary>Esperas por consulta en SQL Server 2016+</summary>

Desde SQL Server 2016, `sys.dm_exec_session_wait_stats` muestra las esperas acumuladas **por sesión** (no solo del servidor), y desde 2017, Query Store guarda las esperas **por consulta** (`sys.query_store_wait_stats`), agrupadas en categorías. Eso permite responder no solo "en qué espera el servidor" sino "en qué espera esta consulta", que es mucho más accionable.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las DMV son vistas del sistema que muestran información interna de SQL Server, como las consultas que se están ejecutando, las más costosas o los índices que faltan. Las uso para encontrar qué consulta es lenta antes de cambiar algo.

### Respuesta ampliada (semi-senior)

Ante un problema de rendimiento, mido antes de cambiar, con una secuencia fija: `dm_exec_requests` con `dm_exec_sql_text` para ver qué se ejecuta ahora, qué espera y quién bloquea; `dm_exec_query_stats` para las consultas que más consumieron, ordenando por totales y por promedios, y el plan en caché con `dm_exec_query_plan`; `dm_os_wait_stats`, filtrando las esperas benignas y comparando fotos en dos momentos, para saber si el servidor espera disco, bloqueos, el log, paralelismo, CPU o memoria; y `dm_io_virtual_file_stats` para la latencia de los discos. Tengo en cuenta que acumulan desde el reinicio y que solo ven lo que está en caché, así que para la historia uso Query Store y para eventos concretos, Extended Events. En el día a día me apoyo en `sp_WhoIsActive` y en el *First Responder Kit*.

### Preguntas frecuentes de seguimiento

**1. ¿Cómo encuentras la consulta que más CPU consume?**
Con `sys.dm_exec_query_stats` ordenado por `total_worker_time`, unido a `dm_exec_sql_text` para el texto.

**2. ¿Qué te dicen las estadísticas de espera?**
Dónde pasa el tiempo el servidor: si espera disco (`PAGEIOLATCH`), bloqueos (`LCK_M_*`), el log (`WRITELOG`), paralelismo o memoria.

**3. ¿Qué limitación tienen las DMV?**
Acumulan desde el último reinicio o desde que el plan entró en caché, y no conservan lo que salió de la caché.

**4. ¿Qué significa `ASYNC_NETWORK_IO`?**
Que el servidor espera a que el cliente consuma los resultados: el problema suele estar en la aplicación.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que muestre los 5 procedimientos almacenados de la base actual con más lecturas lógicas promedio por ejecución, con su cantidad de ejecuciones y desde cuándo están en caché.

<details>
<summary>Solución</summary>

```sql
SELECT TOP (5)
       OBJECT_SCHEMA_NAME(ps.object_id, ps.database_id) + N'.' + OBJECT_NAME(ps.object_id, ps.database_id) AS Procedimiento,
       ps.execution_count                               AS Ejecuciones,
       ps.total_logical_reads / ps.execution_count      AS LecturasPromedio,
       ps.total_elapsed_time / ps.execution_count / 1000 AS DuracionPromedioMs,
       ps.cached_time                                   AS EnCacheDesde
FROM sys.dm_exec_procedure_stats AS ps
WHERE ps.database_id = DB_ID()
ORDER BY LecturasPromedio DESC;
```

`cached_time` responde "desde cuándo": un procedimiento con muchas lecturas promedio pero en caché desde hace cinco minutos puede reflejar solo un par de ejecuciones atípicas. `OBJECT_NAME` con el segundo parámetro (`database_id`) resuelve bien el nombre aunque se consulte desde otra base.

</details>

**Ejercicio 2.** Las esperas principales de un servidor, en un intervalo de una hora, son: `LCK_M_U` 55 %, `PAGEIOLATCH_SH` 20 %, `CXPACKET` 10 %. ¿Por dónde empezarías y qué consultas ejecutarías?

<details>
<summary>Solución</summary>

Empezaría por **`LCK_M_U`** (más de la mitad del tiempo): sesiones esperando bloqueos de actualización, típico de `UPDATE` que buscan sus filas y chocan con otras transacciones.

1. **`sys.dm_exec_requests`** con `blocking_session_id` (o `sp_WhoIsActive`), para encontrar la cabeza de la cadena y su instrucción.
2. **`sys.dm_tran_active_transactions`**, por si hay transacciones largas abiertas.
3. Revisar los `UPDATE` que esperan: ¿tienen índice sobre las columnas de su `WHERE`? Sin índice, recorren y bloquean filas ajenas (ver [Bloqueos](../05-transacciones-y-concurrencia/03-Bloqueos%2C%20hints%20y%20NOLOCK.md)).

`PAGEIOLATCH_SH` (lecturas de disco) suele estar relacionado: consultas que leen de más ocupan la caché y el disco, y alargan sus transacciones. `dm_exec_query_stats` ordenado por lecturas totales encuentra a las candidatas. `CXPACKET` al 10 % puede ser normal; se revisa después.

</details>

-----

## Siguiente lección

[SQL Profiler y trazas](02-SQL%20Profiler%20y%20trazas.md)
