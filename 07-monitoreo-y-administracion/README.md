# 07. Monitoreo y administración

Cómo saber qué está pasando dentro de SQL Server y cómo mantenerlo funcionando sin depender de nadie: las DMVs para el diagnóstico en el momento, las trazas y Extended Events para capturar eventos, Query Store para la historia de las consultas y sus planes, el SQL Server Agent para automatizar, la auditoría para responder quién hizo qué, y un workshop final que integra todo en un método de tuning.

Las lecciones usan `TiendaGrande` para el diagnóstico y el rendimiento, y `Tienda` para los ejemplos de auditoría con resultados exactos.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 01 | [DMVs para diagnóstico](01-DMVs%20para%20diagnóstico.md) | Sesiones y peticiones en curso, bloqueos, consultas en caché, estadísticas de espera, latencia de E/S |
| 02 | [SQL Profiler y trazas](02-SQL%20Profiler%20y%20trazas.md) | Eventos y columnas, trazas del lado del servidor, unidades de `Duration`, la traza por defecto, por qué está obsoleto |
| 03 | [Extended Events](03-Extended%20Events.md) | Sesiones, eventos, acciones, predicados y destinos; `system_health`; consultas lentas, errores y *deadlocks* |
| 04 | [Query Store](04-Query%20Store.md) | Configuración, reportes, regresiones, forzar planes, corrección automática, Query Store hints |
| 05 | [SQL Server Agent](05-SQL%20Server%20Agent.md) | Trabajos, pasos, horarios, operadores, alertas, Database Mail, historial, buenas prácticas |
| 06 | [Auditoría](06-Auditoría.md) | SQL Server Audit, triggers fila por fila, `SESSION_CONTEXT`, tablas temporales, CDC, Change Tracking, retención |
| 07 | [Workshop de tuning de punta a punta](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md) | Identificar, analizar, diagnosticar, optimizar y validar, con una lista de verificación final |

-----

## Mapa del tema

```text
                       ¿Qué está pasando en el servidor?
                                      │
        ┌─────────────────────────────┼──────────────────────────────┐
        ▼                             ▼                              ▼
     AHORA                       EVENTOS CONCRETOS                 HISTORIA
  DMVs (01)                  Profiler / SQL Trace (02)           Query Store (04)
  sesiones, esperas,         Extended Events (03)                consultas, planes,
  caché de planes            consultas lentas, errores,          regresiones
                             deadlocks
        │                             │                              │
        └─────────────────────────────┼──────────────────────────────┘
                                      ▼
                    Workshop: el método de tuning (07)

                    ¿Quién hizo qué? ──► Auditoría (06)
                    ¿Y que corra solo? ──► SQL Server Agent (05)
```

-----

## Sobre los apuntes de origen

Este módulo integra las sesiones 5, 6, 14, 15 y 16 del curso **SQL Server Intermedio 2026** (carpeta `SQL SERVER FRACTAL`), más la parte de Query Store y de *plan guides* de la sesión 8. Correcciones y precisiones sobre los cheat sheets:

* `Duration` está en **microsegundos** en las trazas guardadas a archivo y en Extended Events; Profiler la muestra en milisegundos por defecto, y la columna `CPU` de las trazas está en milisegundos ([lecciones 02](02-SQL%20Profiler%20y%20trazas.md) y [03](03-Extended%20Events.md)).
* Las *plan guides* siguen existiendo, pero para cambiar el plan sin tocar el código hoy se usan los planes forzados y los hints de Query Store ([lección 04](04-Query%20Store.md)).
* El SQL Server Agent no existe en la edición Express y en Docker está deshabilitado por defecto (`MSSQL_AGENT_ENABLED=true`) ([lección 05](05-SQL%20Server%20Agent.md)).
* La plantilla de trigger de auditoría clasifica como **borrado** un `UPDATE` que no afectó filas, usa una sola variable para toda la sentencia y, con varias filas, `FOR JSON PATH, WITHOUT_ARRAY_WRAPPER` produce varios objetos pegados que no forman un JSON válido. Hay que salir si `inserted` y `deleted` están vacías y auditar fila por fila con un `FULL JOIN` ([lección 06](06-Auditoría.md)).
* `SYSTEM_USER` y `SUSER_SNAME()` devuelven el login de la aplicación en un sistema con pool de conexiones; el usuario real se pasa con `SESSION_CONTEXT`. `APP_NAME()` y `HOST_NAME()` los informa el cliente y se pueden falsificar ([lección 06](06-Auditoría.md)).
* Las tablas temporales guardan los períodos en **UTC** y no registran quién hizo el cambio; CDC limpia sus tablas a los tres días por defecto, así que no es un historial ([lección 06](06-Auditoría.md)).
* `EXISTS` no es más eficiente que `IN` en SQL Server: producen el mismo plan (*semi join*). La diferencia que importa es `NOT IN` con `NULL`, que no devuelve filas ([lección 07](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md)).
* La reescritura de `OR` con `UNION ALL ... AND Col1 <> 'A'` pierde las filas donde `Col1` es `NULL` ([lección 07](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md)).
* La consulta de "índices no usados" con `LEFT JOIN` y `WHERE ius.user_seeks = 0` excluye justamente los índices que nunca se usaron (sus columnas son `NULL`) ([lección 07](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md)).
* El patrón de lotes `UPDATE TOP (@n) ... WHERE Condicion` solo termina si la condición deja de cumplirse para las filas ya actualizadas; si no, el bucle no termina nunca (ver [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md)).
* El "resumen del curso" de la sesión 16 no coincide con el contenido real de las sesiones (por ejemplo, lista la 5 como procedimientos almacenados y la 7 como CTEs, cuando son Profiler y planes de ejecución). El mapa real de las sesiones a los módulos de esta wiki es: 1, 2 y 10 → [05](../05-transacciones-y-concurrencia/README.md); 3, 4, 7, 8 y 9 → [04](../04-indices-y-rendimiento/README.md); 11, 12 y 13 → [06](../06-programacion-t-sql/README.md); 5, 6, 14, 15 y 16 → este módulo.

-----

## Requisitos

Los módulos [04. Índices y rendimiento](../04-indices-y-rendimiento/README.md), [05. Transacciones y concurrencia](../05-transacciones-y-concurrencia/README.md) y [06. Programación en T-SQL](../06-programacion-t-sql/README.md).

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
