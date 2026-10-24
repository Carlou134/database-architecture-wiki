# Extended Events

## En una frase

Extended Events (XEvents) es el sistema moderno de captura de eventos de SQL Server: una **sesión** define qué **eventos** escuchar (consultas que terminan, *deadlocks*, errores, esperas), qué datos extra agregar (**acciones**), qué filtrar antes de recolectar (**predicados**) y dónde guardarlos (**destinos**), con un costo mucho menor que SQL Trace y cientos de eventos más.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué capturaba SQL Trace y por qué se reemplazó, de [SQL Profiler y trazas](02-SQL%20Profiler%20y%20trazas.md).
* La sesión `system_health` y el gráfico de *deadlock*, de [Deadlocks](../05-transacciones-y-concurrencia/05-Deadlocks.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Sesión de eventos:** el contenedor que agrupa eventos, acciones, predicados y destinos; se crea, se inicia y se detiene.
* **Evento:** un punto del código de SQL Server que puede registrarse (`sql_statement_completed`, `xml_deadlock_report`, `error_reported`).
* **Acción (action):** dato extra que se agrega al evento cuando ocurre (el texto de la consulta, el usuario, el nombre de la aplicación).
* **Predicado:** filtro que decide si el evento se registra, evaluado antes de recolectar datos.
* **Destino (target):** dónde se guardan los eventos: memoria (`ring_buffer`), archivo (`event_file`), histograma...
* **Paquete (package):** el agrupamiento de los objetos de XEvents (`sqlserver`, `sqlos`, `package0`).

-----

## El problema

El equipo quiere saber qué consultas de `TiendaGrande` tardan más de un segundo, durante todo el día de mañana, sin afectar el rendimiento. Con lo visto hasta ahora:

* **Las DMV** muestran lo acumulado en la caché, pero no cada ejecución, ni las consultas que se recompilan, ni con qué parámetros.
* **Profiler** captura cada ejecución, pero en vivo contra producción puede frenar el servidor.
* **Una traza del lado del servidor** funciona, con números crípticos y una tecnología obsoleta.

Extended Events resuelve esto con una sesión que filtra **antes** de recolectar: el servidor solo hace trabajo extra con las consultas que de verdad tardan más de un segundo, y las guarda en un archivo que se lee después.

-----

## Cómo funciona

### Las piezas de una sesión

```text
SESIÓN  "ConsultasLentas"
 ├── EVENTO   sqlserver.sql_statement_completed
 │     ├── PREDICADO   duration > 1000000  AND  database_name = 'TiendaGrande'
 │     └── ACCIONES    sql_text, client_app_name, username
 ├── EVENTO   sqlserver.rpc_completed  (mismos predicados y acciones)
 └── DESTINO  package0.event_file  (archivo .xel con rotación)
```

| Pieza | Pregunta que responde | Ejemplos |
| --- | --- | --- |
| **Evento** | ¿Qué quiero capturar? | `sql_statement_completed`, `rpc_completed`, `sp_statement_completed`, `xml_deadlock_report`, `error_reported`, `lock_escalation`, `wait_completed` |
| **Predicado** | ¿Cuándo vale la pena registrarlo? | `duration > 1000000`, `database_name = N'TiendaGrande'`, `severity >= 16` |
| **Acción** | ¿Qué datos extra necesito? | `sql_text`, `database_name`, `username`, `client_app_name`, `client_hostname`, `query_hash`, `plan_handle` |
| **Destino** | ¿Dónde lo guardo? | `event_file` (archivo), `ring_buffer` (memoria), `histogram` (cuenta por valor), `event_counter` (solo cuenta) |

### Por qué cuesta menos

El orden de trabajo es la clave:

1. Ocurre el evento, con sus **campos propios** (por ejemplo, `duration`, `cpu_time`, `logical_reads` de una consulta). Recolectarlos es casi gratis.
2. Se evalúa el **predicado**, sobre esos campos. Si no se cumple, el evento se descarta **sin** hacer nada más.
3. Solo si se cumple, se ejecutan las **acciones** (obtener el texto de la consulta, el usuario...), que son más caras.
4. El evento va a un búfer en memoria y, de forma asíncrona, al destino.

Por eso un predicado sobre `duration` hace que el costo de la sesión dependa de cuántas consultas son lentas, no de cuántas consultas hay. La regla práctica: **predicados sobre campos del evento siempre que se pueda**, y acciones solo las necesarias.

### Unidades

Como en SQL Trace, `duration` está en **microsegundos** en los eventos de consultas: 1 segundo = 1.000.000. `cpu_time` también está en microsegundos en los eventos de XEvents (a diferencia de la columna `CPU` de las trazas, en milisegundos).

### Crear una sesión con T-SQL

La plantilla de consultas lentas de tus apuntes de la sesión 6, adaptada:

```sql
CREATE EVENT SESSION ConsultasLentas ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.client_app_name, sqlserver.username, sqlserver.query_hash)
    WHERE duration > 1000000
      AND sqlserver.database_name = N'TiendaGrande'
),
ADD EVENT sqlserver.rpc_completed (
    ACTION (sqlserver.sql_text, sqlserver.client_app_name, sqlserver.username, sqlserver.query_hash)
    WHERE duration > 1000000
      AND sqlserver.database_name = N'TiendaGrande'
)
ADD TARGET package0.event_file (
    SET filename = N'C:\XEvents\ConsultasLentas.xel',
        max_file_size = 50,              -- MB por archivo
        max_rollover_files = 5           -- conserva los 5 últimos
)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 5 SECONDS,
    STARTUP_STATE = OFF                  -- no arranca sola al reiniciar el servidor
);

ALTER EVENT SESSION ConsultasLentas ON SERVER STATE = START;
-- ... más tarde ...
ALTER EVENT SESSION ConsultasLentas ON SERVER STATE = STOP;
```

* `sqlserver.database_name` en el `WHERE` es un **predicado global** (un dato del contexto, no un campo del evento); también se evalúa antes de las acciones.
* `EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS` permite perder algún evento si los búferes se llenan, en lugar de frenar al servidor. `NO_EVENT_LOSS` garantiza no perder nada **a costa de bloquear** las tareas que generan eventos: solo para eventos raros e importantes (como los *deadlocks*), nunca para consultas.
* `STARTUP_STATE = ON` hace que la sesión arranque con el servidor; para una sesión de diagnóstico temporal, mejor `OFF`.

### Destinos

| Destino | Qué hace | Cuándo |
| --- | --- | --- |
| `event_file` | Escribe archivos `.xel` en el disco del servidor | Capturas que se analizan después; producción |
| `ring_buffer` | Guarda los últimos eventos en memoria (se pierden al detener la sesión) | Diagnósticos rápidos, en vivo |
| `histogram` | Cuenta los eventos agrupados por un campo o acción | "¿Qué base genera más errores?" sin guardar cada evento |
| `event_counter` | Solo cuenta | Medir frecuencia |
| `pair_matching` | Empareja eventos de inicio y fin, y muestra los que no terminaron | Encontrar consultas que empiezan y nunca terminan (cancelaciones, *timeouts*) |

### Leer los datos

Desde SSMS: en el Object Explorer, *Management* → *Extended Events* → *Sessions* → clic derecho en la sesión → *Watch Live Data* (en vivo), o doble clic en el archivo del destino. La vista permite agrupar, filtrar y ordenar los eventos como una tabla.

Con T-SQL, el archivo se lee con `sys.fn_xe_file_target_read_file` y el XML de cada evento se descompone:

```sql
SELECT x.e.value('(@timestamp)[1]', 'datetime2')                                 AS Momento,
       x.e.value('(data[@name="duration"]/value)[1]', 'bigint') / 1000           AS DuracionMs,
       x.e.value('(data[@name="logical_reads"]/value)[1]', 'bigint')             AS Lecturas,
       x.e.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(256)')  AS Aplicacion,
       x.e.value('(action[@name="sql_text"]/value)[1]', 'nvarchar(max)')         AS Consulta
FROM sys.fn_xe_file_target_read_file(N'C:\XEvents\ConsultasLentas*.xel', NULL, NULL, NULL) AS f
CROSS APPLY (SELECT CAST(f.event_data AS XML)) AS d (x_)
CROSS APPLY d.x_.nodes('/event') AS x (e)
ORDER BY DuracionMs DESC;
```

El patrón `*.xel` lee todos los archivos de la rotación. Leer XML es costoso con muchos eventos: para análisis grandes, conviene cargar el resultado en una tabla una vez y consultarla desde ahí.

### Sesiones que ya existen

SQL Server incluye sesiones activas por defecto:

* **`system_health`:** siempre activa. Registra *deadlocks* (`xml_deadlock_report`), errores graves, esperas largas de bloqueos y de recursos, y problemas de memoria. Es lo primero que hay que mirar después de un incidente (ver [Deadlocks](../05-transacciones-y-concurrencia/05-Deadlocks.md)).
* **`telemetry_xevents`**, **`AlwaysOn_health`** (si hay grupos de disponibilidad) y otras, según la configuración.

### XEvent Profiler

SSMS incluye, en el Object Explorer, el nodo **XEvent Profiler** con dos sesiones listas: *Standard* y *TSQL*. Con doble clic, abren una vista en vivo de las consultas que se ejecutan, igual que Profiler, pero con Extended Events por debajo. Es el reemplazo directo de "abrir Profiler para ver qué está pasando", con mucho menos impacto (aunque, en un servidor con mucha carga, igual conviene agregar filtros).

### Eventos útiles más allá de las consultas

| Para responder | Evento | Predicado típico |
| --- | --- | --- |
| ¿Qué errores recibe la aplicación? | `error_reported` | `severity >= 11` |
| ¿Qué consultas se cancelan por *timeout*? | `attention` | — |
| ¿Hay escalamientos de bloqueos? | `lock_escalation` | Base de datos |
| ¿Qué divisiones de página son caras? | `transaction_log` | `operation = LOP_DELETE_SPLIT` (ver [Mantenimiento de índices](../04-indices-y-rendimiento/07-Mantenimiento%20de%20índices.md)) |
| ¿Qué planes se compilan? | `sql_statement_recompile` | `recompile_cause` |
| ¿Quién falla al iniciar sesión? | `error_reported` con el error 18456 | `error_number = 18456` |

-----

## Ejemplo completo

Capturar los errores que recibe una aplicación durante una prueba, contados por número de error, sin guardar cada evento:

```sql
CREATE EVENT SESSION ErroresAplicacion ON SERVER
ADD EVENT sqlserver.error_reported (
    ACTION (sqlserver.client_app_name, sqlserver.sql_text)
    WHERE severity >= 11
      AND sqlserver.database_name = N'Tienda'
)
ADD TARGET package0.histogram (
    SET filtering_event_name = N'sqlserver.error_reported',
        source = N'error_number',
        source_type = 0                 -- 0 = campo del evento
),
ADD TARGET package0.ring_buffer (SET max_memory = 4096)
WITH (STARTUP_STATE = OFF);

ALTER EVENT SESSION ErroresAplicacion ON SERVER STATE = START;

-- Generar errores de prueba (cada uno falla, a propósito)
USE Tienda;
INSERT INTO dbo.Cliente (Nombre, Email) VALUES (N'Duplicado', N'ana.torres@correo.com');  -- 2627
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (999, 'Pendiente');                     -- 547
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (999, 'Pendiente');                     -- 547

-- Leer el histograma
SELECT x.s.value('(value)[1]', 'int')    AS NumeroError,
       x.s.value('(@count)[1]', 'int')   AS Veces
FROM (
    SELECT CAST(t.target_data AS XML) AS d
    FROM sys.dm_xe_session_targets AS t
    JOIN sys.dm_xe_sessions AS s ON s.address = t.event_session_address
    WHERE s.name = N'ErroresAplicacion' AND t.target_name = N'histogram'
) AS h
CROSS APPLY h.d.nodes('/HistogramTarget/Slot') AS x (s)
ORDER BY Veces DESC;

ALTER EVENT SESSION ErroresAplicacion ON SERVER STATE = STOP;
DROP EVENT SESSION ErroresAplicacion ON SERVER;
```

| NumeroError | Veces |
| --- | --- |
| 547 | 2 |
| 2627 | 1 |

Qué decisión hay detrás:

* **El histograma cuenta sin guardar cada evento:** para saber "qué errores y cuántos", no hace falta conservar miles de eventos.
* **El `ring_buffer` como segundo destino** guarda los últimos eventos completos, con el texto de la consulta, para ver el detalle de alguno.
* **Predicados en el evento** (`severity`, base de datos): los mensajes informativos y los errores de otras bases no generan trabajo.
* **Se detiene y se borra al terminar:** una sesión de diagnóstico no debería quedar activa.

-----

## Errores comunes

**1. Sesiones sin predicados.**
Qué pasa: millones de eventos, archivos enormes y un costo innecesario.
Por qué: se capturan todos los eventos "por las dudas".
Arreglo: predicados sobre campos del evento (duración, base, severidad).

**2. Filtrar por acciones en lugar de campos.**
Qué pasa: más costo del necesario, porque la acción se calcula para todos los eventos antes de descartarlos.
Por qué: se filtra por `sql_text` u otra acción cara cuando había un campo equivalente.
Arreglo: preferir los campos del evento; usar predicados globales baratos (base de datos) antes que acciones caras.

**3. `NO_EVENT_LOSS` en eventos frecuentes.**
Qué pasa: el servidor se frena esperando a que los eventos se escriban.
Por qué: con esa opción, las tareas que generan eventos esperan si los búferes están llenos.
Arreglo: `ALLOW_SINGLE_EVENT_LOSS` para todo lo que no sea raro y crítico.

**4. Confundir las unidades.**
Qué pasa: un filtro de "más de 1000" que captura todo lo que tarda más de un milisegundo.
Por qué: `duration` está en microsegundos.
Arreglo: 1 segundo = 1.000.000.

**5. Sesiones olvidadas.**
Qué pasa: archivos que rotan durante meses y un costo permanente.
Por qué: se creó para un diagnóstico y nadie la detuvo.
Arreglo: `STARTUP_STATE = OFF` para las temporales, y revisar `sys.dm_xe_sessions` periódicamente.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Sistema de eventos | Extended Events | No hay un equivalente directo; se combinan el registro del servidor, `pg_stat_*` y extensiones | Performance Schema (instrumentos, consumidores, filtros) |
| Consultas lentas | Sesión con predicado sobre `duration` | `log_min_duration_statement` | *Slow query log*, `events_statements_history_long` |
| Planes de las consultas lentas | Evento `query_post_execution_showplan` (caro) | `auto_explain` | `EXPLAIN ANALYZE` manual |
| Deadlocks | `xml_deadlock_report` (en `system_health`) | Registro del servidor (`log_lock_waits`, mensajes de *deadlock*) | `SHOW ENGINE INNODB STATUS`, `innodb_print_all_deadlocks` |
| Rastreo dinámico | XEvents | `eBPF` / herramientas del sistema operativo | Performance Schema |

El Performance Schema de MySQL es el más parecido en espíritu: instrumentos que se activan y desactivan en tiempo de ejecución, con consumidores que deciden dónde se guardan.

### En MongoDB

No hay un sistema de eventos configurable como XEvents. El *Database Profiler* y el registro del servidor cubren las operaciones lentas; los *Change Streams* permiten reaccionar a cambios de datos; y en MongoDB Atlas, el *Performance Advisor* y el *Query Profiler* ofrecen vistas de las operaciones lentas con recomendaciones de índices.

-----

## Cuándo sí y cuándo no

**Usa Extended Events cuando:**

* Necesitas capturar ejecuciones concretas (consultas lentas con sus parámetros, errores, *timeouts*, *deadlocks*) que las DMV no conservan.
* Diagnosticas un problema intermitente: una sesión con buenos predicados puede quedar activa días con poco costo.

**Usa otra herramienta cuando:**

* Te alcanza con lo acumulado: las DMV.
* Necesitas el historial de rendimiento y planes por consulta a lo largo del tiempo: Query Store.

**Ten en cuenta:**

* El evento `query_post_execution_showplan` (el plan real de cada consulta) es de los más caros: úsalo solo con predicados muy restrictivos y por poco tiempo.

-----

## Resumen en 5 líneas

1. Una sesión de XEvents tiene eventos (qué), predicados (cuándo), acciones (qué datos extra) y destinos (dónde).
2. El predicado se evalúa antes de las acciones: filtrar por campos del evento hace que el costo dependa solo de lo que se captura.
3. `event_file` para capturas que se analizan después; `ring_buffer` para diagnósticos en vivo; `histogram` para contar.
4. `duration` está en microsegundos, y `NO_EVENT_LOSS` solo se usa para eventos raros e importantes.
5. `system_health` ya registra *deadlocks* y errores graves; el XEvent Profiler de SSMS reemplaza a Profiler.

-----

## Para profundizar

<details>
<summary>Correlacionar eventos con TRACK_CAUSALITY</summary>

Con `WITH (TRACK_CAUSALITY = ON)`, cada evento recibe un identificador de actividad (`attach_activity_id`): un GUID y un número de secuencia. Todos los eventos de una misma tarea comparten el GUID, en orden. Permite, por ejemplo, capturar `sql_statement_starting`, las esperas (`wait_completed`) y `sql_statement_completed`, y reconstruir en qué esperó **esa** instrucción concreta, en lugar de ver esperas sueltas del servidor.

</details>

<details>
<summary>Sesiones con alcance de base de datos en Azure SQL</summary>

En Azure SQL Database no hay acceso al servidor, así que las sesiones se crean con `ON DATABASE` en lugar de `ON SERVER`, y el destino de archivo es un contenedor de Azure Blob Storage (con una credencial). La sintaxis de eventos, acciones y predicados es la misma. En Azure SQL Managed Instance, en cambio, las sesiones son de servidor, como en SQL Server.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Extended Events es la herramienta de SQL Server para capturar eventos, como consultas lentas o *deadlocks*. Reemplaza a SQL Profiler porque consume menos recursos. Se crea una sesión con los eventos que interesan, filtros y un destino, como un archivo.

### Respuesta ampliada (semi-senior)

Uso Extended Events para capturar lo que las DMV no conservan: ejecuciones concretas de consultas lentas con sus parámetros, errores que recibe la aplicación, cancelaciones por *timeout*, escalamientos o divisiones de página caras. Diseño las sesiones pensando en el costo: predicados sobre campos del evento (duración en microsegundos, severidad) y predicados globales baratos como la base de datos, que se evalúan antes de las acciones; solo las acciones necesarias; `ALLOW_SINGLE_EVENT_LOSS` salvo para eventos raros como los *deadlocks*; y `event_file` con rotación para producción, `ring_buffer` para diagnósticos en vivo o `histogram` cuando solo necesito contar. Leo los archivos con `fn_xe_file_target_read_file` y descomponiendo el XML, o desde SSMS. Sé que `system_health` ya registra *deadlocks* y errores graves, uso el XEvent Profiler en lugar de Profiler, y detengo y borro las sesiones de diagnóstico al terminar.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre una acción y un predicado?**
El predicado decide si el evento se registra; la acción agrega datos extra (como el texto de la consulta) a los eventos que pasan el predicado.

**2. ¿Por qué Extended Events tiene menos impacto que Profiler?**
Porque filtra antes de recolectar los datos caros, y entrega los eventos de forma asíncrona a destinos en el servidor, sin depender de una aplicación cliente.

**3. ¿Dónde buscarías un deadlock que ya ocurrió?**
En la sesión `system_health`, que está siempre activa y registra el evento `xml_deadlock_report`.

**4. ¿En qué unidad está la duración de los eventos de consultas?**
En microsegundos.

-----

## Práctica

**Ejercicio 1.** Crea una sesión que capture, en un archivo, las consultas de `TiendaGrande` que hagan más de 10.000 lecturas lógicas, con el texto de la consulta y la aplicación.

<details>
<summary>Solución</summary>

```sql
CREATE EVENT SESSION ConsultasPesadas ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.client_app_name)
    WHERE logical_reads > 10000
      AND sqlserver.database_name = N'TiendaGrande'
),
ADD EVENT sqlserver.rpc_completed (
    ACTION (sqlserver.sql_text, sqlserver.client_app_name)
    WHERE logical_reads > 10000
      AND sqlserver.database_name = N'TiendaGrande'
)
ADD TARGET package0.event_file (
    SET filename = N'C:\XEvents\ConsultasPesadas.xel', max_file_size = 50, max_rollover_files = 5
)
WITH (STARTUP_STATE = OFF);

ALTER EVENT SESSION ConsultasPesadas ON SERVER STATE = START;
```

Filtrar por `logical_reads` (un campo del evento) en lugar de por duración encuentra las consultas que **trabajan** de más aunque hoy sean rápidas (porque sus páginas están en caché): son las que se vuelven lentas cuando la caché se llena o el servidor se carga. Con `rpc_completed` también se capturan los procedimientos y las consultas parametrizadas que envían las aplicaciones.

</details>

**Ejercicio 2.** ¿Qué problemas tiene esta sesión, pensada para dejarla activa en producción?

```sql
CREATE EVENT SESSION TodoLoQuePasa ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.plan_handle, sqlserver.username)
),
ADD EVENT sqlserver.query_post_execution_showplan
ADD TARGET package0.ring_buffer
WITH (EVENT_RETENTION_MODE = NO_EVENT_LOSS, STARTUP_STATE = ON);
```

<details>
<summary>Solución</summary>

1. **Sin predicados:** captura **todas** las instrucciones del servidor.
2. **`query_post_execution_showplan` sin filtro:** el plan real de cada consulta, uno de los eventos más caros que existen.
3. **`NO_EVENT_LOSS`:** con este volumen, los búferes se llenan y las consultas del servidor esperan a que se escriban los eventos.
4. **`ring_buffer` como único destino:** solo conserva los últimos eventos en memoria; el historial se pierde y no se puede analizar después.
5. **`STARTUP_STATE = ON`:** arranca sola con cada reinicio, así que el problema se vuelve permanente.

Una versión razonable filtra por duración o lecturas y por base, quita el evento de planes (o lo filtra muchísimo, por poco tiempo), usa `ALLOW_SINGLE_EVENT_LOSS`, un `event_file` con rotación, y `STARTUP_STATE = OFF` mientras sea un diagnóstico.

</details>

-----

## Siguiente lección

[Query Store](04-Query%20Store.md)
