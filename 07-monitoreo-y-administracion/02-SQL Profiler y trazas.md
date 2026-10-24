# SQL Profiler y trazas

## En una frase

SQL Trace es el mecanismo antiguo de SQL Server para capturar eventos (cada consulta que termina, cada *deadlock*, cada inicio de sesión), y SQL Profiler es su interfaz gráfica; los dos están **obsoletos** desde SQL Server 2012 en favor de Extended Events, pero siguen presentes en sistemas heredados, en guías viejas y en la traza por defecto, así que hay que saber leerlos y, sobre todo, cómo reemplazarlos.

-----

## Antes de empezar

Conviene que ya sepas:

* Para qué sirve capturar consultas y *deadlocks*, de [DMVs para diagnóstico](01-DMVs%20para%20diagnóstico.md) y [Deadlocks](../05-transacciones-y-concurrencia/05-Deadlocks.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **SQL Trace:** el sistema de captura de eventos anterior a Extended Events.
* **SQL Profiler:** la herramienta gráfica para crear y ver trazas.
* **Traza del lado del servidor (server-side trace):** una traza creada con procedimientos T-SQL (`sp_trace_*`) que escribe en un archivo, sin la interfaz gráfica.
* **Clase de evento (EventClass):** el tipo de evento que se captura (`SQL:BatchCompleted`, `RPC:Completed`, `Deadlock graph`...).
* **Columna de datos:** cada dato que se guarda de un evento (`TextData`, `Duration`, `Reads`, `CPU`...).
* **Traza por defecto (default trace):** una traza liviana que SQL Server mantiene activa siempre, con cambios de configuración y de esquema.

-----

## El problema

Un documento interno de diagnóstico, escrito hace diez años, dice: "Si el sistema está lento, abre SQL Profiler contra el servidor de producción, usa la plantilla *Standard* y mira qué consultas tardan". Alguien lo hace un martes a las 11 de la mañana:

* La plantilla captura **todos** los lotes y llamadas de **todas** las bases, sin filtro.
* Profiler recibe cada evento por la red y lo muestra en pantalla, en tiempo real.
* El servidor, que ya estaba lento, empieza a esperar a que Profiler consuma los eventos.

La herramienta de diagnóstico empeora el problema que vino a diagnosticar. Es la razón principal por la que Profiler está obsoleto, y por la que conviene conocerlo: para entender las guías viejas, mantener lo que ya existe y saber qué usar en su lugar.

-----

## Cómo funciona

### Dos formas de usar SQL Trace

| | SQL Profiler (gráfico) | Traza del lado del servidor |
| --- | --- | --- |
| Cómo se crea | Desde la interfaz, con plantillas | Con `sp_trace_create`, `sp_trace_setevent`, `sp_trace_setfilter` |
| Dónde van los eventos | A la aplicación Profiler, por la red (o a un archivo/tabla desde Profiler) | A un archivo `.trc` en el servidor |
| Impacto en el servidor | **Alto** (puede frenar el servidor si la aplicación no consume los eventos a tiempo) | Bajo a moderado, con buenos filtros |
| Uso razonable hoy | Desarrollo, en una máquina propia | Sistemas heredados que todavía dependen de trazas |

Tu cheat sheet de la sesión 5 lo resume bien: Profiler para desarrollo, trazas del lado del servidor si no queda otra en producción, y Extended Events como reemplazo.

### Eventos y columnas más usados

| EventClass | Evento | Cuándo se registra |
| --- | --- | --- |
| 10 | `RPC:Completed` | Termina una llamada RPC: así llegan los procedimientos y las consultas parametrizadas desde las aplicaciones |
| 12 | `SQL:BatchCompleted` | Termina un lote de T-SQL (consultas ad hoc, SSMS) |
| 45 | `SP:StmtCompleted` | Termina una instrucción dentro de un procedimiento |
| 148 | `Deadlock graph` | Ocurre un *deadlock*, con su XML |

| Columna | Dato | Unidad |
| --- | --- | --- |
| `TextData` | El texto de la consulta | — |
| `Duration` | Duración | **Microsegundos** en las trazas guardadas a archivo (Profiler la muestra en milisegundos por defecto) |
| `CPU` | Tiempo de CPU | Milisegundos |
| `Reads` / `Writes` | Lecturas lógicas / escrituras | Páginas |
| `ApplicationName`, `HostName`, `LoginName` | Quién ejecutó | — |

La diferencia de unidades de `Duration` es una fuente clásica de confusión: un filtro "duración mayor a 1000" significa un milisegundo en una traza de archivo, no un segundo.

### Una traza del lado del servidor

La plantilla de "consultas lentas" de tus apuntes, comentada:

```sql
DECLARE @TraceID INT, @maxfilesize BIGINT = 100;

-- Crear la traza: archivo con rollover (opción 2), 100 MB por archivo
EXEC sp_trace_create @traceid = @TraceID OUTPUT, @options = 2,
     @tracefile = N'C:\Trazas\ConsultasLentas',   -- sin la extensión .trc
     @maxfilesize = @maxfilesize;

-- Eventos y columnas: RPC:Completed (10) y SQL:BatchCompleted (12)
EXEC sp_trace_setevent @TraceID, 10, 1, 1;    -- TextData
EXEC sp_trace_setevent @TraceID, 10, 13, 1;   -- Duration
EXEC sp_trace_setevent @TraceID, 10, 16, 1;   -- Reads
EXEC sp_trace_setevent @TraceID, 12, 1, 1;
EXEC sp_trace_setevent @TraceID, 12, 13, 1;
EXEC sp_trace_setevent @TraceID, 12, 16, 1;

-- Filtro: duración >= 1 segundo (1.000.000 de microsegundos)
DECLARE @Duracion BIGINT = 1000000;
EXEC sp_trace_setfilter @TraceID, 13, 0, 4, @Duracion;   -- columna 13, AND, >=

-- Iniciar
EXEC sp_trace_setstatus @TraceID, 1;
SELECT @TraceID AS TraceID;   -- guardarlo para detenerla después
```

Para detenerla y borrar su definición: `sp_trace_setstatus @TraceID, 0` (detener) y `sp_trace_setstatus @TraceID, 2` (cerrar). Para leerla:

```sql
SELECT TextData, Duration / 1000 AS DuracionMs, Reads, CPU, StartTime
FROM sys.fn_trace_gettable(N'C:\Trazas\ConsultasLentas.trc', DEFAULT)
ORDER BY Duration DESC;
```

Los números de evento, columna y operador son crípticos (`13, 0, 4`), y por eso estas trazas se suelen generar desde Profiler (*File → Export → Script Trace Definition*) en lugar de escribirlas a mano.

### La traza por defecto

SQL Server mantiene activa, desde siempre, una traza liviana con eventos de configuración y de esquema: creación y borrado de objetos, cambios de configuración, crecimientos automáticos de archivos, errores de memoria. Sigue siendo útil para preguntas como "¿quién borró esta tabla anoche?" o "¿cuándo creció el log?":

```sql
DECLARE @Ruta NVARCHAR(260) = (SELECT path FROM sys.traces WHERE is_default = 1);

SELECT t.StartTime, e.name AS Evento, t.DatabaseName, t.ObjectName, t.LoginName, t.HostName
FROM sys.fn_trace_gettable(@Ruta, DEFAULT) AS t
JOIN sys.trace_events AS e ON e.trace_event_id = t.EventClass
WHERE e.name IN ('Object:Deleted', 'Object:Created', 'Object:Altered')
ORDER BY t.StartTime DESC;
```

Guarda solo unos pocos archivos de 20 MB que rotan, así que en servidores con mucha actividad cubre apenas horas o días.

### Por qué está obsoleto y qué usar en su lugar

| | SQL Trace / Profiler | Extended Events |
| --- | --- | --- |
| Estado | Obsoleto desde 2012 (para el motor de base de datos) | El mecanismo actual |
| Impacto | Mayor: filtra después de recolectar muchos datos | Menor: filtra antes, con predicados |
| Eventos disponibles | Unos 180, sin eventos nuevos desde 2008 | Más de mil, y crecen en cada versión |
| Azure SQL Database | No disponible | Disponible |
| Interfaz | Profiler | Integrada en SSMS (incluido el *XEvent Profiler*, con la misma comodidad que Profiler) |

Profiler sigue siendo la herramienta para **Analysis Services** (donde no está obsoleto). Para el motor de base de datos, la migración es directa: cada plantilla de Profiler tiene un equivalente en Extended Events, y SSMS incluye el *XEvent Profiler* (en el Object Explorer), que ofrece la misma vista en vivo de las consultas con mucho menos impacto. Se estudia en la [próxima lección](03-Extended%20Events.md).

-----

## Ejemplo completo

Responder "¿quién borró la tabla `PedidoArchivado` y cuándo?" con la traza por defecto, en `TiendaGrande`:

```sql
-- Simular el incidente
CREATE TABLE dbo.PedidoArchivadoPrueba (Id INT);
DROP TABLE dbo.PedidoArchivadoPrueba;

-- Investigar
DECLARE @Ruta NVARCHAR(260) = (SELECT path FROM sys.traces WHERE is_default = 1);

SELECT t.StartTime, e.name AS Evento, t.DatabaseName, t.ObjectName,
       t.LoginName, t.HostName, t.ApplicationName
FROM sys.fn_trace_gettable(@Ruta, DEFAULT) AS t
JOIN sys.trace_events AS e ON e.trace_event_id = t.EventClass
WHERE t.ObjectName = N'PedidoArchivadoPrueba'
ORDER BY t.StartTime;
```

| StartTime | Evento | DatabaseName | ObjectName | LoginName | HostName | ApplicationName |
| --- | --- | --- | --- | --- | --- | --- |
| 2026-10-24 10:15:02 | Object:Created | TiendaGrande | PedidoArchivadoPrueba | sa | MI-EQUIPO | Microsoft SQL Server Management Studio - Query |
| 2026-10-24 10:15:02 | Object:Deleted | TiendaGrande | PedidoArchivadoPrueba | sa | MI-EQUIPO | Microsoft SQL Server Management Studio - Query |

(Los valores de fecha, usuario y equipo dependen de tu entorno.)

Qué muestra el ejemplo:

* **La traza por defecto ya estaba activa:** no hubo que prepararla antes del incidente.
* **Responde quién, desde dónde y con qué aplicación:** suficiente para una investigación rápida.
* **Su alcance es limitado:** si el incidente fue hace una semana en un servidor con mucha actividad, los archivos probablemente ya rotaron. Para una auditoría formal de cambios de esquema, existen los *triggers* DDL y SQL Server Audit (ver [Auditoría](06-Auditoría.md)).

-----

## Errores comunes

**1. Profiler contra producción, sin filtros.**
Qué pasa: el servidor se vuelve más lento mientras Profiler está abierto.
Por qué: cada evento viaja a la aplicación, y si no los consume a tiempo, el servidor espera.
Arreglo: Extended Events (o, como mínimo, una traza del lado del servidor con filtros).

**2. Confundir las unidades de `Duration`.**
Qué pasa: un filtro que captura todo (o nada).
Por qué: microsegundos en el archivo, milisegundos en la pantalla de Profiler.
Arreglo: verificar la unidad: 1 segundo = 1.000.000 en una traza de archivo.

**3. Capturar `SQL:BatchCompleted` y olvidar `RPC:Completed`.**
Qué pasa: la traza no muestra las consultas de la aplicación.
Por qué: las aplicaciones envían sus consultas parametrizadas y procedimientos como llamadas RPC.
Arreglo: capturar los dos eventos.

**4. Dejar trazas olvidadas.**
Qué pasa: una traza que lleva meses escribiendo archivos y consumiendo recursos.
Por qué: nadie la detuvo.
Arreglo: revisar `SELECT * FROM sys.traces;` y detener las que no tienen dueño.

**5. Seguir creando trazas nuevas.**
Qué pasa: se mantiene una tecnología obsoleta, sin los eventos nuevos.
Por qué: costumbre, o documentación vieja.
Arreglo: Extended Events para todo lo nuevo.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Captura de eventos (moderno) | Extended Events | `log_min_duration_statement`, `auto_explain` | Performance Schema, *slow query log* |
| Captura heredada | SQL Trace / Profiler | — | *General query log* |
| Registro de consultas lentas | XEvents con filtro de duración | `log_min_duration_statement = 1000` (ms) en la configuración | `slow_query_log = ON`, `long_query_time = 1` |
| Registro de cambios de esquema | Traza por defecto, *triggers* DDL | `log_statement = 'ddl'` | *General query log* o auditoría |

En PostgreSQL y MySQL, la forma habitual de capturar consultas lentas es el **registro del servidor** con un umbral de duración, analizado después con herramientas como `pgBadger` (PostgreSQL) o `pt-query-digest` (MySQL).

### En MongoDB

El equivalente es el *Database Profiler* (`db.setProfilingLevel(1, { slowms: 100 })`), que guarda las operaciones lentas en la colección `system.profile`, y el registro del servidor, que anota por defecto las operaciones de más de 100 ms. Igual que con Profiler, el nivel 2 (capturar todo) tiene un costo alto y no se usa en producción.

-----

## Cuándo sí y cuándo no

**Usa SQL Trace o Profiler cuando:**

* Mantienes un sistema heredado cuyas herramientas dependen de trazas.
* Necesitas la traza por defecto para investigar un cambio de esquema reciente.
* Trabajas con Analysis Services.

**No lo uses cuando:**

* Empiezas cualquier captura nueva sobre el motor de base de datos: Extended Events.
* Es producción y la alternativa es Profiler en vivo y sin filtros.

-----

## Resumen en 5 líneas

1. SQL Trace captura eventos y Profiler es su interfaz; los dos están obsoletos para el motor desde 2012.
2. Profiler en vivo contra producción puede frenar el servidor; las trazas del lado del servidor, con filtros, impactan menos.
3. Capturar `RPC:Completed` (aplicaciones) y `SQL:BatchCompleted` (lotes); `Duration` está en microsegundos en los archivos.
4. La traza por defecto sigue activa y responde rápido "quién creó o borró qué".
5. Para todo lo nuevo, Extended Events: menos impacto, más eventos, disponible en Azure.

-----

## Para profundizar

<details>
<summary>Convertir una traza existente a Extended Events</summary>

SQL Server incluye un mapeo entre los eventos de SQL Trace y los de Extended Events (`sys.trace_xe_event_map` y `sys.trace_xe_action_map`), y la comunidad tiene scripts que leen la definición de una traza existente y generan la sesión de Extended Events equivalente. SSMS, además, tiene plantillas de Extended Events con los mismos nombres que las de Profiler (*Standard*, *TSQL*, *TSQL_Duration*), lo que facilita la migración sin escribir nada.

</details>

<details>
<summary>Distributed Replay y Database Experimentation Assistant</summary>

Las trazas también se usaban para **repetir** una carga real contra otro servidor (para probar una migración o un cambio de configuración), con *Distributed Replay*. Esa herramienta también quedó obsoleta; hoy se usan Query Store (para comparar el rendimiento antes y después de un cambio de compatibilidad), y herramientas de terceros o capturas con Extended Events para reproducir cargas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

SQL Profiler es una herramienta para capturar las consultas que se ejecutan en SQL Server. Está obsoleta y se reemplaza por Extended Events, que tiene menos impacto en el servidor. No conviene usar Profiler contra producción porque puede hacerlo más lento.

### Respuesta ampliada (semi-senior)

SQL Trace y Profiler están obsoletos para el motor desde 2012. Profiler en vivo contra producción puede degradar el servidor, porque los eventos viajan a la aplicación y el servidor espera si no se consumen; si hay que usar trazas, son del lado del servidor, a archivo, con filtros, capturando `RPC:Completed` y `SQL:BatchCompleted`, y recordando que `Duration` está en microsegundos en los archivos. Sigo usando la traza por defecto para preguntas rápidas sobre cambios de esquema o crecimientos de archivos, sabiendo que solo cubre un período corto. Para cualquier captura nueva uso Extended Events, que filtra antes de recolectar, tiene muchos más eventos, funciona en Azure y, con el *XEvent Profiler* de SSMS, ofrece la misma comodidad que Profiler.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué Profiler está obsoleto?**
Por su impacto en el servidor y porque Extended Events lo reemplaza con menos costo, más eventos y soporte en Azure.

**2. ¿Qué es la traza por defecto?**
Una traza liviana siempre activa que registra cambios de esquema, de configuración y crecimientos de archivos.

**3. ¿Qué eventos capturarías para ver las consultas de una aplicación?**
`RPC:Completed` (consultas parametrizadas y procedimientos) y `SQL:BatchCompleted` (lotes ad hoc).

-----

## Práctica

**Ejercicio 1.** Escribe la consulta que lista las trazas activas del servidor, indicando cuál es la traza por defecto, su ruta y la cantidad de eventos capturados.

<details>
<summary>Solución</summary>

```sql
SELECT id, is_default, status, path, max_size, event_count, start_time
FROM sys.traces;
```

`is_default = 1` marca la traza por defecto; `status = 1` indica que está en ejecución. Cualquier otra traza activa que nadie reconozca es candidata a detenerse (`sp_trace_setstatus @id, 0` y después `2`).

</details>

**Ejercicio 2.** En la traza por defecto, busca los crecimientos automáticos de archivos (`Data File Auto Grow` y `Log File Auto Grow`) de las últimas 24 horas. ¿Qué indicaría ver muchos?

<details>
<summary>Solución</summary>

```sql
DECLARE @Ruta NVARCHAR(260) = (SELECT path FROM sys.traces WHERE is_default = 1);

SELECT t.StartTime, e.name AS Evento, t.DatabaseName, t.FileName,
       t.IntegerData * 8 / 1024 AS CrecimientoMB, t.Duration / 1000 AS DuracionMs
FROM sys.fn_trace_gettable(@Ruta, DEFAULT) AS t
JOIN sys.trace_events AS e ON e.trace_event_id = t.EventClass
WHERE e.name IN ('Data File Auto Grow', 'Log File Auto Grow')
  AND t.StartTime >= DATEADD(hour, -24, GETDATE())
ORDER BY t.StartTime DESC;
```

`IntegerData` es la cantidad de páginas de 8 KB que creció el archivo. Muchos crecimientos chicos indican que el archivo está configurado para crecer de a poco (por ejemplo, 1 MB o un porcentaje pequeño), lo que fragmenta el archivo en el disco y, en el caso del log, crea muchos archivos virtuales. La solución es dimensionar los archivos de antemano y configurar un crecimiento fijo en MB razonable. Que el log crezca mucho durante una carga es, además, la señal de que hace falta procesarla en lotes (ver [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md)).

</details>

-----

## Siguiente lección

[Extended Events](03-Extended%20Events.md)
