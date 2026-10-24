# Glosario: Monitoreo y administración

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Acción (action, XEvents):** dato extra que se agrega al evento cuando ocurre (el texto de la consulta, el usuario, el nombre de la aplicación). ([03](03-Extended%20Events.md))

**Alerta:** respuesta automática del SQL Server Agent a un error, una severidad o una condición de rendimiento. ([05](05-SQL%20Server%20Agent.md))

**Cambio mínimo:** la modificación más pequeña que corrige la causa, para poder atribuirle la mejora. ([07](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md))

**Change Data Capture (CDC):** lee el log de transacciones y copia cada cambio, con sus valores, a tablas de cambios; necesita el SQL Server Agent. ([06](06-Auditoría.md))

**Change Tracking:** registra qué filas cambiaron (solo la clave), sin los valores, para sincronizar. ([06](06-Auditoría.md))

**Clase de evento (EventClass):** el tipo de evento que captura una traza (`SQL:BatchCompleted`, `RPC:Completed`, `Deadlock graph`...). ([02](02-SQL%20Profiler%20y%20trazas.md))

**Columna de datos (trazas):** cada dato que se guarda de un evento (`TextData`, `Duration`, `Reads`, `CPU`...). ([02](02-SQL%20Profiler%20y%20trazas.md))

**Corrección automática de planes:** función que detecta regresiones y fuerza automáticamente el último plan bueno (`FORCE_LAST_GOOD_PLAN`). ([04](04-Query%20Store.md))

**Database Mail:** el componente que envía correos desde SQL Server. ([05](05-SQL%20Server%20Agent.md))

**Destino (target, XEvents):** dónde se guardan los eventos: memoria (`ring_buffer`), archivo (`event_file`), histograma... ([03](03-Extended%20Events.md))

**DMV / DMF:** vista / función de administración dinámica; empiezan con `sys.dm_`. ([01](01-DMVs%20para%20diagnóstico.md))

**Especificación de auditoría:** la lista de acciones que registra SQL Server Audit, a nivel de servidor o de base. ([06](06-Auditoría.md))

**Espera (wait):** tiempo en que una tarea no avanza porque necesita algo (un bloqueo, una página del disco, memoria, CPU). ([01](01-DMVs%20para%20diagnóstico.md))

**Estadísticas de espera (wait stats):** el acumulado de todas las esperas del servidor desde el último reinicio, por tipo. ([01](01-DMVs%20para%20diagnóstico.md))

**Evento (XEvents):** un punto del código de SQL Server que puede registrarse (`sql_statement_completed`, `xml_deadlock_report`, `error_reported`). ([03](03-Extended%20Events.md))

**Forzar un plan (plan forcing):** obligar a una consulta a usar siempre un plan concreto registrado en Query Store. ([04](04-Query%20Store.md))

**Horario (schedule):** cuándo se ejecuta un trabajo del Agent; se puede compartir entre trabajos. ([05](05-SQL%20Server%20Agent.md))

**Intervalo de agregación:** el período (por ejemplo, una hora) en el que Query Store acumula las estadísticas de cada plan. ([04](04-Query%20Store.md))

**Latencia de E/S:** cuánto tarda, en promedio, una lectura o escritura en un archivo de la base. ([01](01-DMVs%20para%20diagnóstico.md))

**Línea base (baseline):** las mediciones antes de cambiar nada, contra las que se compara. ([07](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md))

**`msdb`:** la base del sistema donde el Agent guarda trabajos, horarios, historial y operadores. ([05](05-SQL%20Server%20Agent.md))

**Operador:** destinatario de las notificaciones del Agent (un correo). ([05](05-SQL%20Server%20Agent.md))

**Paquete (package, XEvents):** el agrupamiento de los objetos de Extended Events (`sqlserver`, `sqlos`, `package0`). ([03](03-Extended%20Events.md))

**Paso (step):** una acción de un trabajo del Agent: un script T-SQL, un comando del sistema, PowerShell, un paquete SSIS. ([05](05-SQL%20Server%20Agent.md))

**Predicado (XEvents):** filtro que decide si el evento se registra, evaluado antes de recolectar datos. ([03](03-Extended%20Events.md))

**Query Store:** función de cada base que registra consultas, planes y estadísticas de ejecución de forma persistente. ([04](04-Query%20Store.md))

**Query Store hints:** hints que se aplican a una consulta desde Query Store, sin modificar su código (SQL Server 2022). ([04](04-Query%20Store.md))

**Regresión de rendimiento:** una consulta que empieza a tardar más que antes, normalmente porque cambió su plan. ([04](04-Query%20Store.md))

**`SESSION_CONTEXT`:** pares clave-valor asociados a la conexión; sirve para pasar el usuario de la aplicación a la base. ([06](06-Auditoría.md))

**Sesión de eventos:** el contenedor de Extended Events que agrupa eventos, acciones, predicados y destinos. ([03](03-Extended%20Events.md))

**Sesión / petición (session / request):** una conexión / una instrucción que se está ejecutando en ella. ([01](01-DMVs%20para%20diagnóstico.md))

**SQL Profiler:** la herramienta gráfica para crear y ver trazas. ([02](02-SQL%20Profiler%20y%20trazas.md))

**SQL Server Agent:** servicio que ejecuta tareas programadas; no existe en la edición Express. ([05](05-SQL%20Server%20Agent.md))

**SQL Server Audit:** la auditoría nativa del motor; registra acciones (inicios de sesión, cambios de permisos, lecturas y escrituras sobre objetos) en un archivo o en el registro de eventos de Windows. ([06](06-Auditoría.md))

**SQL Trace:** el sistema de captura de eventos anterior a Extended Events. ([02](02-SQL%20Profiler%20y%20trazas.md))

**`sql_handle` / `plan_handle`:** identificadores con los que se obtiene el texto y el plan de una consulta en caché. ([01](01-DMVs%20para%20diagnóstico.md))

**Tabla temporal (system-versioned):** tabla que guarda automáticamente cada versión anterior de sus filas en una tabla de historial, con los períodos en UTC. No confundir con las tablas temporales `#Tabla` de [Programación T-SQL](../06-programacion-t-sql/05-Tablas%20temporales%20y%20variables%20de%20tabla.md). ([06](06-Auditoría.md))

**Trabajo (job):** una tarea automatizada del Agent, con uno o varios pasos. ([05](05-SQL%20Server%20Agent.md))

**Traza del lado del servidor (server-side trace):** una traza creada con procedimientos T-SQL (`sp_trace_*`) que escribe en un archivo, sin la interfaz gráfica. ([02](02-SQL%20Profiler%20y%20trazas.md))

**Traza por defecto (default trace):** traza liviana que SQL Server mantiene activa siempre, con cambios de configuración y de esquema. ([02](02-SQL%20Profiler%20y%20trazas.md))
