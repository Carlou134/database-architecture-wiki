# Query Store

## En una frase

Query Store es la "caja negra" de rendimiento de cada base de datos: guarda, de forma persistente y a lo largo del tiempo, cada consulta, cada plan que usó y cuánto costó cada uno, así que permite responder "¿qué consulta empeoró desde el martes, y con qué plan?" y, si hace falta, **forzar** el plan bueno sin tocar el código.

-----

## Antes de empezar

Conviene que ya sepas:

* Caché de planes, *parameter sniffing* y hints, de [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md).
* Las DMV de consultas y sus límites, de [DMVs para diagnóstico](01-DMVs%20para%20diagnóstico.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Query Store:** función de cada base que registra consultas, planes y estadísticas de ejecución de forma persistente.
* **Regresión de rendimiento:** una consulta que empieza a tardar más que antes, normalmente porque cambió su plan.
* **Forzar un plan (plan forcing):** obligar a una consulta a usar siempre un plan concreto registrado en Query Store.
* **Intervalo de agregación:** el período (por ejemplo, una hora) en el que Query Store acumula las estadísticas de cada plan.
* **Corrección automática de planes:** función que detecta regresiones y fuerza automáticamente el último plan bueno.
* **Query Store hints:** hints que se aplican a una consulta desde Query Store, sin modificar su código (SQL Server 2022).

-----

## El problema

El lunes, el procedimiento `dbo.LineasDeCliente` tardaba 2 ms. El jueves tarda 900 ms. Entre medio hubo un reinicio del servidor y una actualización de estadísticas. Las preguntas son:

* ¿Desde **cuándo** está lenta?
* ¿**Cambió** el plan? ¿Cómo era el plan que funcionaba bien?
* ¿Se puede volver al plan bueno **ya**, mientras se investiga la causa?

Las DMV no ayudan mucho: el reinicio vació la caché, así que el plan del lunes ya no existe en ningún lado, y `dm_exec_query_stats` solo sabe lo que pasó desde el jueves. Profiler o Extended Events tendrían que haber estado capturando desde antes del problema.

Query Store resuelve exactamente esto: guarda la historia de planes y rendimiento **dentro de la base**, sobrevive a los reinicios, y permite forzar un plan con un clic.

-----

## Cómo funciona

### Qué guarda

| Dato | Dónde | Para qué |
| --- | --- | --- |
| El texto de cada consulta | `sys.query_store_query_text` | Identificarla |
| Cada consulta (con su contexto) | `sys.query_store_query` | Agrupar sus planes |
| Cada plan que usó | `sys.query_store_plan` | Comparar planes, forzar uno |
| Estadísticas por plan e intervalo: ejecuciones, duración, CPU, lecturas, memoria | `sys.query_store_runtime_stats` | Ver el rendimiento en el tiempo |
| Esperas por plan e intervalo (desde 2017) | `sys.query_store_wait_stats` | Saber en qué esperó cada consulta |

Todo se guarda en la **propia base**: se incluye en las copias de seguridad y sigue disponible después de un reinicio.

### Activarlo y configurarlo

Desde SQL Server 2022, Query Store está **activado por defecto** en las bases nuevas. En versiones anteriores (o en bases migradas), se activa por base:

```sql
ALTER DATABASE TiendaGrande SET QUERY_STORE = ON;

ALTER DATABASE TiendaGrande SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = AUTO,                         -- no registrar consultas triviales
    MAX_STORAGE_SIZE_MB = 1024,
    INTERVAL_LENGTH_MINUTES = 60,                      -- agregación por hora
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    SIZE_BASED_CLEANUP_MODE = AUTO
);
```

| Opción | Recomendación | Por qué |
| --- | --- | --- |
| `QUERY_CAPTURE_MODE` | `AUTO` | Con `ALL`, registra también las consultas triviales y de una sola ejecución, y llena el espacio rápido |
| `MAX_STORAGE_SIZE_MB` | Suficiente para el período que se quiere conservar | Si se llena, Query Store pasa a `READ_ONLY` y **deja de registrar** |
| `SIZE_BASED_CLEANUP_MODE` | `AUTO` | Borra lo más viejo antes de llegar al límite |
| `INTERVAL_LENGTH_MINUTES` | 60 (o 15 para más detalle) | Intervalos más cortos dan más detalle y ocupan más |

Para verificar el estado:

```sql
SELECT actual_state_desc, desired_state_desc, current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;
```

Si `actual_state_desc` es `READ_ONLY` cuando se configuró `READ_WRITE`, Query Store se llenó y dejó de registrar: hay que aumentar el tamaño o limpiar.

### Los informes de SSMS

En el Object Explorer, cada base tiene una carpeta **Query Store** con informes listos:

| Informe | Para qué |
| --- | --- |
| **Regressed Queries** | Consultas que empeoraron en el período reciente comparado con uno anterior |
| **Top Resource Consuming Queries** | Las que más consumen (duración, CPU, lecturas...) en un período |
| **Tracked Queries** | La historia de una consulta concreta, con todos sus planes en un gráfico |
| **Queries With Forced Plans** | Las consultas que tienen un plan forzado |
| **Queries With High Variation** | Consultas cuyo rendimiento varía mucho (candidatas a *parameter sniffing*) |
| **Query Wait Statistics** | En qué esperan las consultas, por categoría |

En el informe *Tracked Queries*, cada plan aparece como un punto de un color en un gráfico de duración en el tiempo: una regresión se ve como el momento en que aparece un plan nuevo (otro color) con duraciones más altas. Seleccionando el plan bueno, el botón **Force Plan** lo fija.

### Forzar un plan con T-SQL

```sql
-- Encontrar la consulta y sus planes
SELECT q.query_id, p.plan_id, qt.query_sql_text,
       rs.count_executions, rs.avg_duration / 1000 AS DuracionPromedioMs,
       rs.avg_logical_io_reads AS LecturasPromedio, p.is_forced_plan
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs ON rs.plan_id = p.plan_id
WHERE qt.query_sql_text LIKE N'%DetallePedido AS d ON d.PedidoId = p.PedidoId%'
ORDER BY q.query_id, rs.avg_duration;

-- Forzar el plan bueno
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 7;

-- Quitar el forzado (cuando se resuelva la causa de fondo)
EXEC sp_query_store_unforce_plan @query_id = 42, @plan_id = 7;
```

(`avg_duration` está en microsegundos.)

Forzar un plan es un **parche controlado**: estabiliza el rendimiento sin tocar el código, pero congela una decisión. Si los datos cambian mucho, el plan forzado puede dejar de ser bueno, y si el plan deja de ser válido (por ejemplo, se borró un índice que usaba), el forzado falla y el motor vuelve a optimizar normalmente (`sys.query_store_plan` registra el motivo en `last_force_failure_reason_desc`).

### Corrección automática de planes

Desde SQL Server 2017 (Enterprise y Developer, y siempre en Azure SQL), el motor puede detectar regresiones y forzar el último plan bueno solo:

```sql
ALTER DATABASE TiendaGrande SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON);
```

Con esta opción, si una consulta cambia de plan y empeora de forma significativa, el motor vuelve al plan anterior; y si después el plan forzado resulta peor, lo quita. Sin activarla, igual se pueden ver sus recomendaciones en `sys.dm_db_tuning_recommendations`.

### Query Store hints

En SQL Server 2022, se pueden **agregar hints** a una consulta desde Query Store, sin cambiar su código (útil cuando la consulta viene de una aplicación que no se puede modificar):

```sql
EXEC sys.sp_query_store_set_hints @query_id = 42, @query_hints = N'OPTION (RECOMPILE)';
EXEC sys.sp_query_store_clear_hints @query_id = 42;
```

Es el reemplazo moderno de las *plan guides* de tus apuntes de la sesión 8: el mismo objetivo (cambiar el comportamiento sin tocar el código), mucho más fácil de administrar.

### Comparar antes y después de un cambio

Un uso muy valioso: antes de cambiar el nivel de compatibilidad de una base (por ejemplo, al migrar de SQL Server 2016 a 2022), se deja Query Store registrando con el nivel viejo, se cambia, y se comparan las consultas con el informe *Regressed Queries*. Las que empeoraron se pueden forzar a su plan anterior mientras se analizan. Microsoft recomienda exactamente este procedimiento para las actualizaciones de versión.

-----

## Ejemplo completo

El *parameter sniffing* de `dbo.LineasDeCliente`, detectado y estabilizado con Query Store en `TiendaGrande`:

```sql
USE TiendaGrande;
ALTER DATABASE TiendaGrande SET QUERY_STORE = ON;
ALTER DATABASE TiendaGrande SET QUERY_STORE (OPERATION_MODE = READ_WRITE, QUERY_CAPTURE_MODE = ALL);   -- ALL solo para la prueba
GO

-- (el procedimiento original, sin OPTION (RECOMPILE), de Planes de ejecución II)

-- 1. Un período "bueno": plan compilado para un cliente común, muchas ejecuciones de clientes comunes
EXEC sp_recompile 'dbo.LineasDeCliente';
EXEC dbo.LineasDeCliente @ClienteId = 700;
EXEC dbo.LineasDeCliente @ClienteId = 701;
EXEC dbo.LineasDeCliente @ClienteId = 702;

-- 2. Un "reinicio": el plan se descarta y el primero en llegar es el mayorista
EXEC sp_recompile 'dbo.LineasDeCliente';
EXEC dbo.LineasDeCliente @ClienteId = 1;
EXEC dbo.LineasDeCliente @ClienteId = 700;      -- ahora un cliente común usa el plan del mayorista
EXEC dbo.LineasDeCliente @ClienteId = 701;

-- 3. Ver los planes registrados para esa consulta
SELECT q.query_id, p.plan_id, rs.count_executions,
       rs.avg_duration / 1000 AS DuracionPromedioMs,
       rs.avg_logical_io_reads AS LecturasPromedio
FROM sys.query_store_query AS q
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs ON rs.plan_id = p.plan_id
WHERE q.object_id = OBJECT_ID('dbo.LineasDeCliente')
ORDER BY p.plan_id;
```

| query_id | plan_id | count_executions | DuracionPromedioMs | LecturasPromedio |
| --- | --- | --- | --- | --- |
| 15 | 3 | 3 | ~1 | ~40 |
| 15 | 4 | 3 | ~300 | ~9.000 |

(Los números de `query_id`, `plan_id` y las métricas dependen de tu entorno; los de esta tabla son ilustrativos. Si el procedimiento se ejecuta pocas veces, Query Store puede tardar unos segundos en mostrar los datos, porque los escribe de forma asíncrona; `EXEC sp_query_store_flush_db;` fuerza la escritura.)

```sql
-- 4. Forzar el plan bueno para los clientes comunes
EXEC sp_query_store_force_plan @query_id = 15, @plan_id = 3;
```

Qué muestra el ejemplo:

* **Los dos planes quedaron registrados**, con sus métricas, aunque el primero ya no esté en la caché: Query Store guarda la historia que la caché pierde.
* **Forzar el plan 3** estabiliza el rendimiento para el 99,99 % de los clientes sin cambiar ni una línea del procedimiento. El mayorista queda con un plan que no es ideal para él (los *Nested Loops*), lo que es un compromiso aceptable si su pantalla se usa poco.
* **No es la solución definitiva:** la causa sigue siendo el *sniffing*. La corrección de fondo (`OPTION (RECOMPILE)`, separar el caso del mayorista, o PSP en SQL Server 2022) se aplica después, con calma, y entonces se quita el forzado.
* **`QUERY_CAPTURE_MODE = ALL`** se usó solo para que la prueba registre consultas con pocas ejecuciones; en producción, `AUTO`.

-----

## Errores comunes

**1. Query Store lleno y en `READ_ONLY`.**
Qué pasa: deja de registrar justo cuando hace falta.
Por qué: el tamaño máximo es chico y la limpieza automática está desactivada.
Arreglo: `SIZE_BASED_CLEANUP_MODE = AUTO`, un tamaño adecuado y revisar `actual_state_desc`.

**2. `QUERY_CAPTURE_MODE = ALL` en producción.**
Qué pasa: miles de consultas triviales o de una sola ejecución que ocupan el espacio.
Por qué: se registra todo.
Arreglo: `AUTO` (o `CUSTOM` con umbrales, desde 2019).

**3. Forzar planes y olvidarlos.**
Qué pasa: meses después, con otros datos, el plan forzado es el peor plan posible.
Por qué: el forzado congela una decisión.
Arreglo: documentar cada forzado con su motivo, resolver la causa de fondo y quitarlo; revisar el informe *Queries With Forced Plans*.

**4. Esperar que Query Store vea lo que no registra.**
Qué pasa: no aparecen las consultas de otras bases, ni las que corrieron antes de activarlo.
Por qué: es por base y empieza a registrar al activarse.
Arreglo: activarlo antes de necesitarlo, en todas las bases importantes.

**5. Confundir el texto de la consulta.**
Qué pasa: no se encuentra la consulta buscando el texto del procedimiento.
Por qué: Query Store registra **cada instrucción** por separado, con su propio texto, no el procedimiento completo.
Arreglo: buscar por `object_id` (para procedimientos) o por un fragmento de la instrucción.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Historial de consultas y planes | Query Store | `pg_stat_statements` (estadísticas, sin historia de planes); extensiones como `pg_store_plans` | Performance Schema (resúmenes por *digest*, sin historia de planes) |
| Persistente tras reinicios | Sí (en la base) | `pg_stat_statements` sí (en un archivo); se puede reiniciar | No (en memoria) |
| Forzar un plan | `sp_query_store_force_plan` | No nativo (`pg_hint_plan`) | No (hints en el texto de la consulta) |
| Corrección automática | `FORCE_LAST_GOOD_PLAN` | No | No |
| Hints sin cambiar el código | Query Store hints (2022) | `pg_hint_plan` con una tabla de hints | Reescritura de consultas (*query rewrite plugin*) |

Query Store es una de las funciones de SQL Server sin un equivalente nativo completo en los otros motores; en PostgreSQL, la combinación de `pg_stat_statements`, `auto_explain` y `pg_hint_plan` cubre una parte.

### En MongoDB

MongoDB no guarda una historia de planes. El *Database Profiler* registra operaciones lentas con su plan elegido, y en MongoDB Atlas el *Query Profiler* y el *Performance Advisor* muestran tendencias y recomiendan índices. La caché de planes por forma de consulta (`db.collection.getPlanCache()`) se puede inspeccionar y limpiar, y `planCacheSetFilter` (o `setIndexFilter`) permite limitar los índices que el optimizador considera para una forma de consulta, lo más parecido a forzar un plan.

-----

## Cuándo sí y cuándo no

**Activa Query Store cuando:**

* La base es de producción: siempre (en SQL Server 2022 ya viene activo).

**Fuerza un plan cuando:**

* Hay una regresión que afecta al negocio y necesitas estabilizar **ya**, mientras investigas la causa.

**No fuerces un plan cuando:**

* La causa es fácil de corregir (un índice que falta, una consulta no SARGable): corrige la causa.
* El rendimiento varía porque los datos son muy distintos según el parámetro: un único plan forzado no sirve para todos; usa `RECOMPILE` o separa los casos.

-----

## Resumen en 5 líneas

1. Query Store guarda en cada base la historia de consultas, planes y rendimiento; sobrevive a los reinicios.
2. Desde SQL Server 2022 viene activo; configúralo con `QUERY_CAPTURE_MODE = AUTO`, tamaño suficiente y limpieza automática, y vigila que no pase a `READ_ONLY`.
3. Los informes de SSMS (*Regressed Queries*, *Top Resource Consuming*, *Tracked Queries*) muestran qué empeoró y con qué plan.
4. `sp_query_store_force_plan` fija el plan bueno sin tocar el código; es un parche que se documenta y se quita al resolver la causa.
5. `FORCE_LAST_GOOD_PLAN` corrige regresiones solo, y los Query Store hints (2022) reemplazan a las *plan guides*.

-----

## Para profundizar

<details>
<summary>Las esperas por consulta</summary>

`sys.query_store_wait_stats` (desde 2017) agrupa las esperas de cada plan en categorías (CPU, bloqueos, E/S de búfer, red, paralelismo...). Responde una pregunta que las DMV del servidor no responden: "¿en qué espera **esta** consulta?". Una consulta lenta que espera sobre todo *Lock* tiene un problema de concurrencia, no de plan; forzarle otro plan no la ayudaría.

```sql
SELECT ws.wait_category_desc, SUM(ws.total_query_wait_time_ms) AS EsperaTotalMs
FROM sys.query_store_wait_stats AS ws
JOIN sys.query_store_plan AS p ON p.plan_id = ws.plan_id
WHERE p.query_id = 15
GROUP BY ws.wait_category_desc
ORDER BY EsperaTotalMs DESC;
```

</details>

<details>
<summary>Query Store en réplicas secundarias</summary>

Hasta SQL Server 2019, Query Store solo registraba la carga de la réplica principal de un grupo de disponibilidad: las consultas de solo lectura que se envían a las réplicas secundarias no quedaban registradas. SQL Server 2022 agregó Query Store para réplicas secundarias, que envía los datos de rendimiento de las secundarias a la principal, para tener la visión completa.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Query Store guarda en la base de datos el historial de las consultas, sus planes y cuánto tardaron. Sirve para encontrar las consultas que empeoraron y, si cambió el plan, forzar el plan que funcionaba bien.

### Respuesta ampliada (semi-senior)

Query Store es la caja negra de rendimiento por base: registra el texto, los planes y las estadísticas de ejecución y de esperas por intervalo, de forma persistente, así que resuelve lo que las DMV no pueden (la historia antes de un reinicio o de que el plan saliera de la caché). Lo configuro con captura `AUTO`, un tamaño adecuado y limpieza por tamaño, vigilando que no pase a solo lectura. Para diagnosticar uso los informes de consultas con regresión, las de más consumo y las de alta variación (candidatas a *parameter sniffing*), y el gráfico de *Tracked Queries* para ver qué plan cambió y cuándo. Fuerzo planes como parche para estabilizar una regresión mientras corrijo la causa, documentando cada forzado y quitándolo después, porque congela una decisión. Conozco `FORCE_LAST_GOOD_PLAN` para la corrección automática y los Query Store hints de 2022 para cambiar el comportamiento de consultas de aplicaciones que no puedo modificar, y lo uso para comparar antes y después de cambiar el nivel de compatibilidad.

### Preguntas frecuentes de seguimiento

**1. ¿Qué ventaja tiene Query Store sobre las DMV?**
Que es persistente: conserva la historia de consultas y planes después de un reinicio o de que el plan salga de la caché.

**2. ¿Qué pasa si Query Store se llena?**
Pasa a `READ_ONLY` y deja de registrar, salvo que esté activa la limpieza automática por tamaño.

**3. ¿Cuándo forzarías un plan?**
Ante una regresión por un cambio de plan, para estabilizar mientras se corrige la causa; nunca como solución permanente sin revisarla.

**4. ¿Qué es `FORCE_LAST_GOOD_PLAN`?**
La corrección automática de planes: el motor detecta una regresión por cambio de plan y fuerza el último plan bueno.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta sobre las vistas de Query Store que muestre las 5 consultas con más lecturas lógicas totales en las últimas 24 horas, con su texto y su cantidad de planes distintos.

<details>
<summary>Solución</summary>

```sql
SELECT TOP (5)
       q.query_id,
       MAX(qt.query_sql_text)                                AS Consulta,
       COUNT(DISTINCT p.plan_id)                             AS Planes,
       SUM(rs.count_executions)                              AS Ejecuciones,
       SUM(rs.avg_logical_io_reads * rs.count_executions)    AS LecturasTotales
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval AS i ON i.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE i.start_time >= DATEADD(hour, -24, SYSUTCDATETIME())
GROUP BY q.query_id
ORDER BY LecturasTotales DESC;
```

`runtime_stats` guarda promedios por intervalo, así que el total se calcula como promedio × ejecuciones. Los intervalos están en UTC (`start_time` es un `DATETIMEOFFSET`). Una consulta con varios planes en 24 horas es candidata a revisar: puede estar cambiando de plan según el parámetro.

</details>

**Ejercicio 2.** Una consulta tiene un plan forzado desde hace seis meses. Hoy, el informe muestra que con ese plan tarda el triple que hace seis meses. ¿Qué harías?

<details>
<summary>Solución</summary>

1. **Revisar por qué se forzó:** la documentación del forzado (si existe) dice qué problema resolvía.
2. **Comparar con los otros planes registrados:** en *Tracked Queries*, ver si el optimizador generaría hoy un plan mejor (los datos crecieron, hay índices nuevos).
3. **Probar sin el forzado en un entorno de prueba** (o en producción con monitoreo), con `sp_query_store_unforce_plan`, y comparar las métricas durante un período representativo.
4. **Corregir la causa original** (si era *sniffing*, `OPTION (RECOMPILE)`, PSP o separar casos), para no tener que volver a forzar.

El caso muestra el riesgo de los forzados: lo que era el mejor plan con los datos de hace seis meses dejó de serlo, y el optimizador no pudo adaptarse porque se le había quitado la decisión.

</details>

-----

## Siguiente lección

[SQL Server Agent](05-SQL%20Server%20Agent.md)
