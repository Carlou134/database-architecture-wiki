# SQL Server Agent

## En una frase

SQL Server Agent es el servicio que **automatiza** las tareas del servidor: ejecuta **trabajos** (jobs) formados por **pasos**, según **horarios**, y avisa a los **operadores** cuando algo falla o cuando ocurre un error grave (**alertas**); es donde viven el mantenimiento de índices, las copias de seguridad, los procesos nocturnos y los archivados.

-----

## Antes de empezar

Conviene que ya sepas:

* Mantenimiento de índices y estadísticas, de [Mantenimiento de índices](../04-indices-y-rendimiento/07-Mantenimiento%20de%20índices.md).
* Procesos por lotes reanudables, de [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md).
* Procedimientos y manejo de errores, de los módulos anteriores.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **SQL Server Agent:** servicio de Windows (o proceso en Linux) que ejecuta tareas programadas.
* **Trabajo (job):** una tarea automatizada, con uno o varios pasos.
* **Paso (step):** una acción del trabajo: un script T-SQL, un comando del sistema, PowerShell, un paquete SSIS.
* **Horario (schedule):** cuándo se ejecuta un trabajo; se puede compartir entre trabajos.
* **Operador:** destinatario de las notificaciones (un correo).
* **Alerta:** respuesta automática a un error, una severidad o una condición de rendimiento.
* **Database Mail:** el componente que envía correos desde SQL Server.
* **`msdb`:** la base del sistema donde el Agent guarda trabajos, horarios, historial y operadores.

-----

## El problema

El mantenimiento de la tienda depende de una persona: cada domingo a la noche se conecta, ejecuta el script de estadísticas, el de índices y el de archivado de pedidos cancelados, y revisa que todo haya salido bien. El domingo que se enfermó, no se ejecutó nada. El domingo que el script de archivado falló a la mitad, nadie se enteró hasta el miércoles. Y el día que esa persona cambió de trabajo, se fue con el conocimiento de qué había que ejecutar y en qué orden.

Las tareas recurrentes de una base de datos tienen que estar **automatizadas**, **programadas**, **registradas** y **vigiladas**: que corran solas, que quede historia de cada ejecución y que alguien se entere cuando fallan. Eso es el SQL Server Agent.

-----

## Cómo funciona

### Dónde está, y dónde no

* **SQL Server Agent no existe en la edición Express.** En Express, la automatización se hace con el Programador de tareas de Windows y `sqlcmd` (ver [Entorno de trabajo](../03-consultas/00-Entorno%20de%20trabajo.md)).
* **En Docker o Linux**, el Agent está instalado pero deshabilitado: se habilita con la variable `MSSQL_AGENT_ENABLED=true` al crear el contenedor, o con `mssql-conf set sqlagent.enabled true`.
* **En Azure SQL Database no hay Agent:** se usan *Elastic Jobs* o Azure Automation. En Azure SQL Managed Instance sí está.

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=TuClaveSegura123!" -e "MSSQL_AGENT_ENABLED=true" \
  -p 1433:1433 --name sqlserver -d mcr.microsoft.com/mssql/server:2022-latest
```

En SSMS, el Agent aparece al final del Object Explorer; si tiene una flecha roja, el servicio está detenido.

### La arquitectura

```text
SQL Server Agent  (todo se guarda en msdb)
 ├── Jobs
 │    ├── Steps       (T-SQL, CmdExec, PowerShell, SSIS...)  con acción en éxito y en fallo
 │    └── Schedules   (diario, semanal, mensual, cada n minutos, al iniciar el Agent)
 ├── Operators        (quién recibe los avisos)
 ├── Alerts           (errores, severidades, contadores de rendimiento)
 └── Proxies          (credenciales para pasos que no son T-SQL)
```

### Crear un trabajo con T-SQL

Los trabajos se crean con la interfaz de SSMS o con los procedimientos de `msdb`. La versión T-SQL se puede versionar en el repositorio y desplegar igual en todos los entornos:

```sql
USE msdb;

-- 1. El trabajo
EXEC dbo.sp_add_job
     @job_name = N'Tienda - Mantenimiento semanal',
     @description = N'Estadísticas e índices de la base Tienda. Responsable: equipo de datos.',
     @owner_login_name = N'sa',
     @notify_level_email = 2,                       -- 2 = avisar si falla
     @notify_email_operator_name = N'EquipoDatos';

-- 2. Los pasos
EXEC dbo.sp_add_jobstep
     @job_name = N'Tienda - Mantenimiento semanal',
     @step_name = N'Actualizar estadísticas',
     @subsystem = N'TSQL',
     @database_name = N'Tienda',
     @command = N'EXEC sp_updatestats;',
     @on_success_action = 3,                        -- 3 = ir al paso siguiente
     @on_fail_action = 2,                           -- 2 = terminar con error
     @retry_attempts = 1, @retry_interval = 5;      -- reintentar una vez a los 5 minutos

EXEC dbo.sp_add_jobstep
     @job_name = N'Tienda - Mantenimiento semanal',
     @step_name = N'Mantenimiento de índices',
     @subsystem = N'TSQL',
     @database_name = N'Tienda',
     @command = N'EXEC dbo.MantenerIndices;',       -- procedimiento propio (o la solución de Ola Hallengren)
     @on_success_action = 1,                        -- 1 = terminar con éxito
     @on_fail_action = 2;

-- 3. El horario: domingos a las 3:00
EXEC dbo.sp_add_jobschedule
     @job_name = N'Tienda - Mantenimiento semanal',
     @name = N'Domingos 03:00',
     @freq_type = 8,                 -- semanal
     @freq_interval = 1,             -- domingo
     @freq_recurrence_factor = 1,    -- cada 1 semana
     @active_start_time = 30000;     -- 03:00:00 (formato HHMMSS)

-- 4. Asignarlo a este servidor
EXEC dbo.sp_add_jobserver @job_name = N'Tienda - Mantenimiento semanal', @server_name = N'(LOCAL)';
```

### Los códigos de los horarios y las acciones

Los parámetros numéricos son el detalle que más se consulta (tu cheat sheet de la sesión 14 tiene las tablas completas):

| `@freq_type` | Significa | `@freq_interval` |
| --- | --- | --- |
| 1 | Una vez | — |
| 4 | Diario | Cada cuántos días |
| 8 | Semanal | Suma de días: domingo 1, lunes 2, martes 4, miércoles 8, jueves 16, viernes 32, sábado 64 (lunes a viernes = 62) |
| 16 | Mensual, un día del mes | El día (1 a 31) |
| 32 | Mensual relativo ("el primer lunes") | El día de la semana (1 = domingo ... 7 = sábado), con `@freq_relative_interval` (1 primero, 2 segundo, 4 tercero, 8 cuarto, 16 último) |
| 64 | Al iniciar el Agent | — |

| Acción en éxito / fallo | Significa |
| --- | --- |
| 1 | Terminar el trabajo con éxito |
| 2 | Terminar el trabajo con error |
| 3 | Ir al paso siguiente |
| 4 | Ir a un paso concreto (`@on_success_step_id` / `@on_fail_step_id`) |

Para repetir dentro del día (cada 15 minutos entre las 8 y las 18), se agregan `@freq_subday_type = 4` (minutos), `@freq_subday_interval = 15`, `@active_start_time = 80000` y `@active_end_time = 180000`.

Con las acciones de los pasos se arman flujos con manejo de errores: un último paso "Notificar fallo" al que van todos los pasos con `@on_fail_action = 4`.

### Operadores, Database Mail y alertas

Para que los avisos lleguen, primero hay que configurar **Database Mail** (un perfil con una cuenta SMTP, desde *Management → Database Mail* en SSMS) y habilitarlo para el Agent (*Propiedades del Agent → Alert System*). Después:

```sql
EXEC msdb.dbo.sp_add_operator @name = N'EquipoDatos', @email_address = N'datos@tienda.com';

-- Alerta por severidad: errores graves de recursos y fatales
EXEC msdb.dbo.sp_add_alert @name = N'Severidad 19+', @severity = 19, @include_event_description_in = 1;
EXEC msdb.dbo.sp_add_notification @alert_name = N'Severidad 19+', @operator_name = N'EquipoDatos',
     @notification_method = 1;   -- 1 = correo

-- Alerta por error concreto: el log de transacciones se llenó
EXEC msdb.dbo.sp_add_alert @name = N'Log lleno (9002)', @message_id = 9002;
EXEC msdb.dbo.sp_add_notification @alert_name = N'Log lleno (9002)', @operator_name = N'EquipoDatos',
     @notification_method = 1;
```

Las alertas recomendadas en cualquier servidor son las de severidades 19 a 25 (errores fatales) y los errores **823, 824 y 825**, que indican problemas de lectura del disco o corrupción: detectarlos pronto es la diferencia entre restaurar una página y perder una base.

### Ver qué pasó

```sql
-- Últimas ejecuciones de cada trabajo (resultado del trabajo completo: step_id = 0)
SELECT j.name AS Trabajo,
       msdb.dbo.agent_datetime(h.run_date, h.run_time) AS Inicio,
       CASE h.run_status WHEN 0 THEN 'Falló' WHEN 1 THEN 'Éxito' WHEN 2 THEN 'Reintento'
                         WHEN 3 THEN 'Cancelado' WHEN 4 THEN 'En curso' END AS Resultado,
       h.run_duration AS DuracionHHMMSS,
       h.message
FROM msdb.dbo.sysjobhistory AS h
JOIN msdb.dbo.sysjobs AS j ON j.job_id = h.job_id
WHERE h.step_id = 0
ORDER BY Inicio DESC;
```

* `step_id = 0` es la fila con el resultado del trabajo completo; las demás, el de cada paso, con el mensaje de error del paso que falló.
* `run_date`, `run_time` y `run_duration` son enteros con formato (`20261024`, `30000`, `130` = 1 minuto 30 segundos); `agent_datetime` convierte fecha y hora.
* El historial tiene un límite (por defecto, 1.000 filas en total y 100 por trabajo); en servidores con muchos trabajos frecuentes, conviene ampliarlo en las propiedades del Agent para no perder la historia de los fallos.

### Buenas prácticas

| Práctica | Por qué |
| --- | --- |
| Un dueño que no sea una persona (`sa` o una cuenta de servicio) | Si el dueño es el login de alguien que deja la empresa, el trabajo falla al deshabilitarse ese login |
| Nombres y descripciones claros (`Tienda - Mantenimiento semanal`) | Que cualquiera sepa qué hace y de quién es |
| Notificación de fallo en todo trabajo importante | Que alguien se entere el mismo día |
| La lógica en procedimientos, el paso solo los llama | El código se versiona y se prueba como el resto; el paso queda en una línea |
| Pasos idempotentes y reanudables | Si un paso falla y se reintenta, no duplica el trabajo (ver el proceso con tabla de progreso de [Operaciones masivas](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md)) |
| Horarios que no se pisen | El mantenimiento de índices y las copias de seguridad compiten por disco y CPU |
| Los scripts de creación en el repositorio | Recrear los trabajos en otro servidor sin depender de la memoria de nadie |

-----

## Ejemplo completo

El archivado de pedidos cancelados del módulo de transacciones, convertido en un trabajo nocturno con dos pasos y aviso de fallo:

```sql
USE msdb;

EXEC dbo.sp_add_job
     @job_name = N'TiendaGrande - Archivar cancelados',
     @description = N'Archiva y borra pedidos cancelados de más de un año. Reanudable (dbo.ProcesoLote).',
     @owner_login_name = N'sa',
     @notify_level_email = 2,
     @notify_email_operator_name = N'EquipoDatos';

-- Paso 1: el proceso por lotes (dentro de un procedimiento)
EXEC dbo.sp_add_jobstep
     @job_name = N'TiendaGrande - Archivar cancelados',
     @step_name = N'Archivar por lotes',
     @subsystem = N'TSQL',
     @database_name = N'TiendaGrande',
     @command = N'EXEC dbo.ArchivarCancelados @AntiguedadDias = 365, @TamanoLote = 4000;',
     @on_success_action = 3,
     @on_fail_action = 2,
     @retry_attempts = 2, @retry_interval = 10;   -- un deadlock o un timeout: reintentar

-- Paso 2: estadísticas de la tabla afectada
EXEC dbo.sp_add_jobstep
     @job_name = N'TiendaGrande - Archivar cancelados',
     @step_name = N'Actualizar estadísticas de Pedido',
     @subsystem = N'TSQL',
     @database_name = N'TiendaGrande',
     @command = N'UPDATE STATISTICS dbo.Pedido; UPDATE STATISTICS dbo.DetallePedido;',
     @on_success_action = 1,
     @on_fail_action = 2;

-- Todos los días a las 02:30
EXEC dbo.sp_add_jobschedule
     @job_name = N'TiendaGrande - Archivar cancelados',
     @name = N'Diario 02:30',
     @freq_type = 4, @freq_interval = 1,
     @active_start_time = 23000;

EXEC dbo.sp_add_jobserver @job_name = N'TiendaGrande - Archivar cancelados', @server_name = N'(LOCAL)';

-- Probarlo sin esperar al horario
EXEC dbo.sp_start_job @job_name = N'TiendaGrande - Archivar cancelados';
```

(`dbo.ArchivarCancelados` es el bucle por lotes de [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md), empaquetado en un procedimiento con los parámetros de antigüedad y tamaño de lote.)

Qué decisión hay detrás:

* **La lógica en un procedimiento:** el paso del trabajo es una línea; el código se versiona y se prueba aparte.
* **Reintentos en el paso 1:** un *deadlock* o un *timeout* de bloqueo son transitorios; como el proceso es reanudable (lleva su progreso en `dbo.ProcesoLote`), reintentar no repite lo ya hecho.
* **Estadísticas después de borrar muchas filas:** sin esperar a que la actualización automática se dispare.
* **Aviso de fallo al operador:** el miércoles ya no es el día en que alguien se entera.
* **`sp_start_job`** permite probarlo de inmediato; el resultado aparece en el historial.

-----

## Errores comunes

**1. Trabajos cuyo dueño es una persona.**
Qué pasa: el día que deshabilitan el login de esa persona, sus trabajos dejan de ejecutarse.
Por qué: el Agent ejecuta los pasos T-SQL en el contexto del dueño.
Arreglo: dueño `sa` (o una cuenta de servicio dedicada).

**2. Trabajos sin notificación de fallo.**
Qué pasa: un proceso falla durante semanas y nadie se entera.
Por qué: el Agent registra el fallo en el historial, pero no avisa si no se le pide.
Arreglo: operador, Database Mail y `@notify_level_email = 2`.

**3. Lógica larga dentro del paso.**
Qué pasa: 300 líneas de T-SQL en el `@command`, sin control de versiones ni pruebas.
Por qué: es lo más rápido al crear el trabajo.
Arreglo: un procedimiento, y el paso solo lo llama.

**4. Pasos que no se pueden reintentar.**
Qué pasa: un reintento duplica el trabajo hecho (por ejemplo, inserta dos veces).
Por qué: el paso no es idempotente.
Arreglo: procesos reanudables o idempotentes antes de configurar reintentos.

**5. Esperar el Agent en Express o en Docker sin habilitarlo.**
Qué pasa: el nodo del Agent no aparece, o aparece detenido.
Por qué: Express no lo incluye; en contenedores está deshabilitado por defecto.
Arreglo: otra edición, o `MSSQL_AGENT_ENABLED=true`; en Express, el Programador de tareas con `sqlcmd`.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Programador integrado | SQL Server Agent (no en Express) | **No nativo**: extensión `pg_cron`, o `cron` del sistema operativo | **Event Scheduler** (`CREATE EVENT ... ON SCHEDULE EVERY 1 DAY`) |
| Pasos con flujo de control | Sí (acciones en éxito y fallo) | No (una sentencia o función por tarea) | No (un evento ejecuta un bloque) |
| Notificaciones | Operadores, Database Mail, alertas | Externas (monitoreo del sistema) | Externas |
| Historial | `msdb.dbo.sysjobhistory` | `cron.job_run_details` (con `pg_cron`) | `performance_schema` / registro del servidor |

En PostgreSQL y MySQL, lo habitual en producción es programar las tareas desde fuera de la base (el `cron` del sistema operativo, Kubernetes *CronJobs* u orquestadores como Airflow), y el mismo patrón es válido también para SQL Server cuando la empresa ya tiene una herramienta central de programación.

### En MongoDB

No hay un programador de tareas en el servidor. Las tareas recurrentes se ejecutan desde fuera (`cron`, orquestadores, funciones programadas), y en MongoDB Atlas existen los *Scheduled Triggers*, que ejecutan una función según un horario. Para el borrado por antigüedad, el índice TTL hace el trabajo solo, sin ninguna tarea programada.

-----

## Cuándo sí y cuándo no

**Usa el Agent cuando:**

* Hay tareas recurrentes de la base: mantenimiento, copias de seguridad, archivados, cargas nocturnas, verificaciones de integridad.
* Quieres que el servidor avise ante errores graves (alertas de severidad y de errores 823/824/825).

**Considera otra herramienta cuando:**

* La empresa ya centraliza la programación en un orquestador (Airflow, Control-M, Azure Data Factory), y conviene que los procesos de la base estén ahí también, con sus dependencias.
* Estás en Azure SQL Database (Elastic Jobs) o en Express (Programador de tareas).

-----

## Resumen en 5 líneas

1. El Agent ejecuta trabajos con pasos según horarios, guarda todo en `msdb` y avisa a operadores; no existe en Express y en Docker hay que habilitarlo.
2. Cada paso define qué hacer en éxito y en fallo, con reintentos: así se arman flujos con manejo de errores.
3. Los horarios usan códigos (`freq_type`, días como suma de potencias de 2, horas en formato `HHMMSS`).
4. Configura Database Mail, operadores, notificaciones de fallo y alertas de severidad 19+ y de los errores 823/824/825.
5. Dueño que no sea una persona, lógica en procedimientos, pasos reanudables y los scripts en el repositorio.

-----

## Para profundizar

<details>
<summary>La solución de mantenimiento de Ola Hallengren</summary>

Casi todos los equipos que administran SQL Server usan la *Maintenance Solution* de Ola Hallengren (gratuita, en ola.hallengren.com): un conjunto de procedimientos (`DatabaseBackup`, `DatabaseIntegrityCheck`, `IndexOptimize`) que se instalan en una base y crean trabajos del Agent listos para usar. `IndexOptimize`, por ejemplo, decide por cada índice si reorganizar, reconstruir o no hacer nada según umbrales configurables, actualiza las estadísticas que lo necesitan y registra cada operación en una tabla. Antes de escribir un mantenimiento propio, conviene conocerla.

</details>

<details>
<summary>Proxies para pasos que no son T-SQL</summary>

Los pasos de tipo CmdExec, PowerShell o SSIS se ejecutan, por defecto, con la cuenta del servicio del Agent, que suele tener muchos permisos. Un **proxy** asocia una credencial (una cuenta de Windows concreta) a un subsistema, para que esos pasos se ejecuten con permisos mínimos: por ejemplo, una cuenta que solo puede escribir en la carpeta de exportaciones. Es la forma recomendada de ejecutar pasos fuera de T-SQL.

</details>

-----

## En entrevista

### Respuesta corta (junior)

SQL Server Agent sirve para programar tareas automáticas, como copias de seguridad o mantenimiento de índices. Un trabajo tiene pasos y horarios, y se pueden configurar avisos por correo cuando falla.

### Respuesta ampliada (semi-senior)

Uso el Agent para automatizar las tareas recurrentes de la base (mantenimiento de índices y estadísticas, copias de seguridad, verificaciones de integridad, archivados y cargas), sabiendo que no existe en Express, que en contenedores hay que habilitarlo y que en Azure SQL Database se reemplaza por Elastic Jobs. Diseño los trabajos con pasos cortos que llaman a procedimientos versionados, con acciones de éxito y fallo para el flujo, reintentos solo en pasos idempotentes o reanudables, y un dueño que no sea una persona. Configuro Database Mail, operadores y notificaciones de fallo en todos los trabajos importantes, más alertas por severidad 19 a 25 y por los errores 823, 824 y 825. Reviso el historial en `msdb` y amplío su retención, evito que los horarios pesados se pisen, guardo los scripts de creación en el repositorio y, para el mantenimiento, parto de la solución de Ola Hallengren.

### Preguntas frecuentes de seguimiento

**1. ¿Dónde guarda el Agent su información?**
En la base del sistema `msdb`: trabajos, pasos, horarios, operadores, alertas e historial.

**2. ¿Por qué no conviene que el dueño de un trabajo sea un usuario personal?**
Porque si ese login se deshabilita (por ejemplo, cuando la persona deja la empresa), el trabajo falla.

**3. ¿Qué alertas configurarías en cualquier servidor?**
Las de severidades 19 a 25 y las de los errores 823, 824 y 825, que indican problemas de disco o corrupción.

**4. ¿Cómo programas tareas si no tienes Agent?**
Con el Programador de tareas del sistema operativo y `sqlcmd`, o con un orquestador externo.

-----

## Práctica

**Ejercicio 1.** Escribe el horario de un trabajo que debe ejecutarse cada 30 minutos, de lunes a viernes, entre las 8:00 y las 20:00.

<details>
<summary>Solución</summary>

```sql
EXEC msdb.dbo.sp_add_jobschedule
     @job_name = N'Tienda - Sincronizar stock',
     @name = N'Cada 30 min, L-V, 08 a 20',
     @freq_type = 8,                  -- semanal
     @freq_interval = 62,             -- lunes (2) + martes (4) + miércoles (8) + jueves (16) + viernes (32)
     @freq_recurrence_factor = 1,     -- cada semana
     @freq_subday_type = 4,           -- repetir cada n minutos
     @freq_subday_interval = 30,
     @active_start_time = 80000,      -- 08:00:00
     @active_end_time = 200000;       -- 20:00:00
```

Los días se combinan sumando sus valores (cada día es una potencia de 2, así que la suma identifica la combinación sin ambigüedad). La última ejecución del día es a las 20:00 si cae dentro del rango.

</details>

**Ejercicio 2.** Escribe una consulta que muestre los trabajos que fallaron en las últimas 24 horas, con el paso y el mensaje de error.

<details>
<summary>Solución</summary>

```sql
SELECT j.name AS Trabajo,
       h.step_id, h.step_name,
       msdb.dbo.agent_datetime(h.run_date, h.run_time) AS Momento,
       h.message
FROM msdb.dbo.sysjobhistory AS h
JOIN msdb.dbo.sysjobs AS j ON j.job_id = h.job_id
WHERE h.run_status = 0                -- falló
  AND h.step_id > 0                   -- los pasos (el 0 es el resumen del trabajo)
  AND msdb.dbo.agent_datetime(h.run_date, h.run_time) >= DATEADD(hour, -24, GETDATE())
ORDER BY Momento DESC;
```

Filtrar por `step_id > 0` muestra **qué paso** falló y su mensaje, que es lo que hace falta para corregirlo; la fila con `step_id = 0` solo dice que el trabajo completo terminó con error. Las horas del historial están en la hora local del servidor, por eso se compara con `GETDATE()` y no con la hora UTC.

</details>

-----

## Siguiente lección

[Auditoría](06-Auditoría.md)
