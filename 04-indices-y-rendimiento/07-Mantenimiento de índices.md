# Mantenimiento de índices

## En una frase

Los índices se degradan con las escrituras (páginas partidas, desordenadas o medio vacías) y se acumulan con el tiempo (índices que nadie usa, duplicados, sugerencias aplicadas sin criterio); mantenerlos es medir la fragmentación **y la densidad** de las páginas, reconstruir o reorganizar solo lo que lo necesita, mantener las estadísticas al día y revisar periódicamente qué índices sobran y cuáles faltan.

-----

## Antes de empezar

Conviene que ya sepas:

* Páginas, árbol B y densidad, de [Cómo guarda los datos SQL Server](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md).
* El costo de escritura de cada índice, de [Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md).
* Las estadísticas, de [Planes de ejecución II](05-Planes%20de%20ejecución%20II.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **División de página (page split):** cuando una fila no entra en la página donde le corresponde, SQL Server la divide en dos, moviendo la mitad de las filas a una página nueva.
* **Fragmentación lógica (externa):** el orden lógico de las páginas no coincide con su orden físico en el archivo.
* **Densidad de página (fragmentación interna):** qué tan llenas están las páginas; páginas medio vacías significan más páginas para los mismos datos.
* **`REORGANIZE`:** reordena y compacta las páginas hoja del índice, en pasos pequeños.
* **`REBUILD`:** reconstruye el índice completo desde cero.
* **Fill factor:** el porcentaje de cada página hoja que se llena al crear o reconstruir un índice, dejando espacio libre para futuras inserciones.

-----

## El problema

Una tabla de sesiones de usuario con clave `UNIQUEIDENTIFIER DEFAULT NEWID()` recibe miles de inserciones por hora. Seis meses después:

* El índice agrupado ocupa el doble de lo que debería.
* Los reportes que recorren la tabla son cada vez más lentos.
* El monitoreo muestra una "fragmentación del 99 %".

Alguien programa un `REBUILD` de **todos** los índices de **todas** las tablas, todas las noches. La base tarda horas en el mantenimiento, el log crece, las consultas nocturnas se bloquean... y a los tres días la fragmentación vuelve al 99 %.

El mantenimiento sin diagnóstico es caro y no resuelve la causa. Para hacerlo bien, hay que entender qué se degrada, cómo medirlo y qué importa de verdad.

-----

## Cómo funciona

### Cómo se fragmenta un índice

Cuando se inserta una fila en un índice, va a la página que le corresponde según su clave. Si esa página está llena, SQL Server la **divide**: crea una página nueva, mueve aproximadamente la mitad de las filas y deja las dos páginas al 50 %.

```text
Antes:       [pág 10: A B C D E F G H] (llena)  → [pág 11: I J ...]
Insertar D2:
Después:     [pág 10: A B C D D2 ·  ·  ·]       → [pág 87: E F G H ·  ·  ·] → [pág 11: I J ...]
                       50 % lleno                   50 % lleno, al final del archivo
```

El resultado tiene dos problemas distintos:

| Tipo | Qué es | Cómo se mide | Qué afecta |
| --- | --- | --- | --- |
| **Fragmentación lógica** | La página 87 está lógicamente entre la 10 y la 11, pero físicamente lejos | `avg_fragmentation_in_percent` | Las lecturas secuenciales grandes (*scans*), sobre todo en discos mecánicos |
| **Baja densidad de página** | Las páginas quedaron medio vacías | `avg_page_space_used_in_percent` | **Todas** las lecturas: más páginas para los mismos datos, más memoria ocupada |

Las causas típicas: claves aleatorias (GUID con `NEWID()`), actualizaciones que agrandan filas (un `NVARCHAR` que pasa de 10 a 200 caracteres) y borrados que dejan huecos. Una clave creciente (`IDENTITY`) casi no fragmenta: las filas nuevas siempre van al final.

### Medir

```sql
SELECT OBJECT_NAME(ps.object_id)            AS Tabla,
       i.name                                AS Indice,
       ps.index_level,
       ps.avg_fragmentation_in_percent       AS FragmentacionLogica,
       ps.avg_page_space_used_in_percent     AS DensidadPagina,
       ps.page_count                         AS Paginas
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') AS ps
JOIN sys.indexes AS i ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE ps.page_count > 1000
ORDER BY ps.page_count DESC;
```

| Modo | Qué lee | Densidad de página |
| --- | --- | --- |
| `LIMITED` | Solo los niveles superiores del árbol (rápido) | **No la calcula** (devuelve `NULL`) |
| `SAMPLED` | Una muestra de las hojas | Sí, aproximada |
| `DETAILED` | Todas las páginas (lento en tablas grandes) | Sí, exacta |

`WHERE page_count > 1000` descarta los índices chicos: en un índice de 50 páginas, la fragmentación puede ser del 80 % y no tener ningún impacto.

### `REORGANIZE` frente a `REBUILD`

| | `REORGANIZE` | `REBUILD` |
| --- | --- | --- |
| Qué hace | Reordena y compacta las páginas hoja existentes | Crea el índice de nuevo y descarta el viejo |
| Siempre en línea | Sí: bloqueos breves, no impide el acceso | No: por defecto bloquea la tabla; `ONLINE = ON` evita casi todo el bloqueo (Enterprise, Developer y Azure) |
| Si se interrumpe | Conserva el trabajo hecho (trabaja en transacciones chicas) | Se deshace entero (es atómico), salvo con `RESUMABLE = ON` (desde 2017) |
| Actualiza las estadísticas | **No** | **Sí**, con lectura completa (solo las del índice) |
| Cambia el fill factor | No | Sí |
| Uso de recursos | Bajo, pero puede ser lento con mucha fragmentación | Alto (CPU, log, `tempdb` con `SORT_IN_TEMPDB`), pero rápido |

```sql
ALTER INDEX IX_Pedido_Fecha ON dbo.Pedido REORGANIZE;

ALTER INDEX IX_Pedido_Fecha ON dbo.Pedido REBUILD
    WITH (ONLINE = ON, FILLFACTOR = 90, SORT_IN_TEMPDB = ON);

ALTER INDEX ALL ON dbo.Pedido REBUILD;    -- todos los índices de la tabla
```

Un detalle que tu cheat sheet de la sesión 9 resume en la fila "Transaccional": un `REBUILD` **sí** es transaccional (todo o nada: si se cancela, se deshace entero), y justamente por eso no es interrumpible sin perder el trabajo. `REORGANIZE` confirma en pasos pequeños, así que se puede detener a mitad sin perder lo avanzado.

### ¿Cuándo hacer qué? Los umbrales clásicos y su revisión

La regla tradicional, que aparece en tus apuntes, es:

| Fragmentación lógica | Acción |
| --- | --- |
| < 5–10 % | Nada |
| 5–10 % a 30 % | `REORGANIZE` |
| > 30 % | `REBUILD` |

Es un buen punto de partida, pero en 2021 Microsoft revisó su documentación para dejar claro que estos números **no son reglas**. Las razones:

* En almacenamiento **SSD** y en la nube, leer páginas "fuera de orden" casi no cuesta más que leerlas en orden: la fragmentación lógica importa mucho menos que con discos mecánicos.
* Lo que sigue importando siempre es la **densidad**: un índice con páginas al 50 % ocupa el doble, lee el doble y usa el doble de memoria, en cualquier disco.
* Un `REBUILD` tiene costos reales: CPU, log de transacciones (que se replica a las réplicas y se incluye en las copias de seguridad del log) y bloqueos.

La recomendación actual: medir **si el mantenimiento mejora algo concreto** (menos páginas, menos lecturas, consultas más rápidas) para cada carga de trabajo, priorizar la densidad de página sobre el porcentaje de fragmentación, y empezar por mantener las **estadísticas**, que suele ser lo que más impacto tiene.

### Fill factor

El fill factor deja espacio libre en cada página hoja al reconstruir, para que las inserciones futuras no provoquen divisiones inmediatamente:

| Patrón de escritura | Fill factor razonable |
| --- | --- |
| Solo lectura, o inserciones siempre al final (`IDENTITY`, fechas crecientes) | 100 (el valor por defecto, `0` = 100) |
| Inserciones aleatorias en la clave (GUID aleatorio, emails) | 80–90, si las divisiones se miden como un problema |
| Actualizaciones que agrandan filas con frecuencia | 80–90 |

El fill factor es un **intercambio**: con 80, la tabla ocupa un 25 % más desde el primer día y cada *scan* lee un 25 % más de páginas, a cambio de menos divisiones. Bajar el fill factor de **todo** el servidor (una opción de configuración) es un error frecuente: penaliza a todos los índices crecientes, que nunca lo necesitaron. Se ajusta índice por índice. La opción `PAD_INDEX = ON` aplica el mismo espacio libre a los niveles intermedios.

La solución de fondo para un índice que se fragmenta muy rápido suele ser cambiar la **clave** (una clave creciente en lugar de un GUID aleatorio), no el fill factor.

### Mantener las estadísticas

Las estadísticas desactualizadas causan más planes malos que la fragmentación. Además de la actualización automática:

* Después de cargas o borrados masivos: `UPDATE STATISTICS` sobre las tablas afectadas.
* En el mantenimiento periódico: actualizar las estadísticas que cambiaron (`sp_updatestats` lo hace solo para las que tienen modificaciones).
* Después de un `REORGANIZE`: actualizar las estadísticas del índice, porque `REORGANIZE` no lo hace.
* Después de un `REBUILD`: las estadísticas **del índice** ya quedaron con lectura completa; no las actualices otra vez con muestreo, porque empeorarías su calidad. Las estadísticas de columnas (las que no son de un índice) no se tocan con el `REBUILD`.

### Índices que sobran: sin uso y duplicados

`sys.dm_db_index_usage_stats` cuenta cuántas veces se usó cada índice desde el último reinicio:

```sql
SELECT OBJECT_NAME(i.object_id)                         AS Tabla,
       i.name                                           AS Indice,
       COALESCE(us.user_seeks, 0)                       AS Seeks,
       COALESCE(us.user_scans, 0)                       AS Scans,
       COALESCE(us.user_lookups, 0)                     AS Lookups,
       COALESCE(us.user_updates, 0)                     AS Escrituras
FROM sys.indexes AS i
LEFT JOIN sys.dm_db_index_usage_stats AS us
       ON us.object_id = i.object_id AND us.index_id = i.index_id AND us.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.type_desc = 'NONCLUSTERED'
  AND i.is_primary_key = 0 AND i.is_unique_constraint = 0
ORDER BY Seeks + Scans + Lookups, Escrituras DESC;
```

* Un índice con **muchas escrituras y ninguna lectura** es puro costo: candidato a eliminar.
* **Cuidado:** los contadores se reinician con cada reinicio del servidor (y cuando la base se pone fuera de línea). Un índice que solo usa el cierre contable de fin de mes parece "sin uso" el día 20. Observa durante un ciclo de negocio completo antes de borrar.
* Las restricciones `UNIQUE` y las claves primarias se excluyen: aunque no se lean, hacen cumplir una regla.

Los **duplicados** se buscan comparando las columnas clave: dos índices con las mismas columnas en el mismo orden son redundantes, y uno cuyas claves son el prefijo de otro casi siempre también (ver [Índices compuestos](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md)).

### Índices que faltan: las sugerencias del motor

El optimizador registra los índices que **habría querido** tener:

```sql
SELECT TOP (10)
       d.statement                                     AS Tabla,
       d.equality_columns, d.inequality_columns, d.included_columns,
       s.user_seeks, s.avg_user_impact
FROM sys.dm_db_missing_index_details AS d
JOIN sys.dm_db_missing_index_groups AS g ON g.index_handle = d.index_handle
JOIN sys.dm_db_missing_index_group_stats AS s ON s.group_handle = g.index_group_handle
WHERE d.database_id = DB_ID()
ORDER BY s.user_seeks * s.avg_user_impact DESC;
```

Son **sugerencias**, con limitaciones importantes:

* No consideran el costo de escritura del índice sugerido.
* Sugieren índices casi idénticos para consultas parecidas, y a veces redundantes con los existentes.
* No ordenan bien las columnas de igualdad (las listan en orden de la tabla, no de selectividad ni de reutilización) y nunca sugieren índices filtrados ni columnstore.
* Se reinician con el servidor.

Úsalas como **pista** de qué consultas sufren, y diseña el índice con los criterios de este módulo.

### Una estrategia de mantenimiento

| Frecuencia | Tarea |
| --- | --- |
| Diaria (fuera del horario de mayor uso) | Actualizar las estadísticas modificadas de las tablas críticas |
| Semanal | Medir fragmentación y densidad de los índices grandes; reorganizar o reconstruir solo los que lo justifican |
| Mensual o por ciclo de negocio | Revisar índices sin uso, duplicados y sugerencias; ajustar el fill factor de los índices que se fragmentan rápido |

En la práctica, casi nadie escribe estos scripts desde cero: la solución de mantenimiento de **Ola Hallengren** (gratuita, ampliamente usada) implementa el mantenimiento de índices y estadísticas con umbrales configurables, y se programa con el SQL Server Agent (ver [SQL Server Agent](../07-monitoreo-y-administracion/05-SQL%20Server%20Agent.md)).

-----

## Ejemplo completo

Reproducir la fragmentación de un GUID aleatorio y medir el efecto del mantenimiento, en `TiendaGrande`:

```sql
USE TiendaGrande;

CREATE TABLE dbo.Sesion (
    SesionId UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID() PRIMARY KEY,
    ClienteId INT NOT NULL,
    Inicio    DATETIME2(0) NOT NULL
);

-- 200.000 inserciones en orden aleatorio de clave
INSERT INTO dbo.Sesion (ClienteId, Inicio)
SELECT 1 + n % 100000, DATEADD(second, n, '2026-01-01')
FROM dbo.Numero WHERE n <= 200000;

-- 1. Medir
SELECT avg_fragmentation_in_percent, avg_page_space_used_in_percent, page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Sesion'), 1, NULL, 'DETAILED')
WHERE index_level = 0;
```

| avg_fragmentation_in_percent | avg_page_space_used_in_percent | page_count |
| --- | --- | --- |
| ~99 | ~70 | ~1.250 |

```sql
-- 2. Reconstruir y medir de nuevo
ALTER INDEX ALL ON dbo.Sesion REBUILD;

SELECT avg_fragmentation_in_percent, avg_page_space_used_in_percent, page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Sesion'), 1, NULL, 'DETAILED')
WHERE index_level = 0;
```

| avg_fragmentation_in_percent | avg_page_space_used_in_percent | page_count |
| --- | --- | --- |
| < 1 | ~99 | ~880 |

```sql
-- 3. Insertar otras 20.000 sesiones y medir
INSERT INTO dbo.Sesion (ClienteId, Inicio)
SELECT 1 + n % 100000, DATEADD(second, n, '2026-06-01')
FROM dbo.Numero WHERE n <= 20000;
-- La fragmentación vuelve a subir rápidamente: con las páginas al 100 %,
-- casi cada inserción aleatoria parte una página.
```

(Las cifras son aproximadas.) Qué muestra el ejemplo:

* **El GUID aleatorio fragmenta casi al 100 % desde el principio**, y deja las páginas al ~70 %: la tabla ocupa ~40 % más de lo necesario.
* **El `REBUILD` lo arregla... por poco tiempo**: con páginas llenas, las inserciones aleatorias siguientes vuelven a partirlas enseguida.
* **La solución no es reconstruir todas las noches.** Las opciones, de mejor a peor: cambiar la clave del agrupado (un `BIGINT IDENTITY` agrupado, con el GUID como columna `UNIQUE` no agrupada, o `NEWSEQUENTIALID()`); o, si la clave no se puede cambiar, un fill factor de 80–90 en ese índice, aceptando su costo en espacio.

```sql
DROP TABLE dbo.Sesion;   -- limpieza
```

-----

## Errores comunes

**1. Reconstruir todo, todas las noches.**
Qué pasa: horas de mantenimiento, log enorme, bloqueos, y la fragmentación vuelve en días.
Por qué: se trata el síntoma en lugar de la causa (claves aleatorias, actualizaciones que agrandan filas).
Arreglo: mantener solo lo que lo necesita, según la densidad y el impacto medido, y corregir la causa.

**2. Bajar el fill factor de todo el servidor.**
Qué pasa: todas las tablas ocupan más y cada *scan* lee más páginas.
Por qué: se aplica a índices crecientes que nunca lo necesitaban.
Arreglo: fill factor índice por índice.

**3. Actualizar estadísticas con muestreo después de un `REBUILD`.**
Qué pasa: las estadísticas del índice pierden la calidad de la lectura completa que acababan de obtener.
Por qué: un script de mantenimiento ejecuta las dos tareas sin coordinación.
Arreglo: después de un `REBUILD`, actualizar solo las estadísticas que no son de índices.

**4. Borrar índices "sin uso" a mitad de mes.**
Qué pasa: el cierre contable de fin de mes pasa de minutos a horas.
Por qué: los contadores de uso se reinician con el servidor y no cubrían un ciclo completo.
Arreglo: observar durante un ciclo de negocio completo, y deshabilitar antes de borrar (`ALTER INDEX ... DISABLE` conserva la definición).

**5. Crear todos los índices sugeridos.**
Qué pasa: decenas de índices casi iguales, escrituras cada vez más lentas.
Por qué: las sugerencias no consideran el costo de escritura ni la redundancia.
Arreglo: usar las sugerencias como pista y diseñar el índice consolidado.

**6. `REBUILD` sin `ONLINE` en horario productivo.**
Qué pasa: la tabla queda bloqueada durante la reconstrucción.
Por qué: el `REBUILD` por defecto es fuera de línea.
Arreglo: `ONLINE = ON` (si la edición lo permite), `REORGANIZE`, o una ventana de mantenimiento.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Causa principal de degradación | Divisiones de página, borrados | **Bloat**: MVCC deja versiones muertas de las filas | Divisiones de página, borrados |
| Limpieza rutinaria | No hace falta para las versiones (el versionado va en `tempdb`) | `VACUUM` (automático con *autovacuum*): imprescindible | Purga automática del *undo log* |
| Reconstruir un índice | `ALTER INDEX ... REBUILD` | `REINDEX` (`REINDEX CONCURRENTLY` desde la 12, sin bloquear) | `OPTIMIZE TABLE` (reconstruye la tabla y sus índices) |
| Reorganizar | `ALTER INDEX ... REORGANIZE` | No hay equivalente directo (`pg_repack` como extensión) | No |
| Fill factor | `FILLFACTOR` por índice | `fillfactor` por tabla e índice | `innodb_fill_factor` (global, para la creación de índices) |
| Uso de índices | `sys.dm_db_index_usage_stats` | `pg_stat_user_indexes` (`idx_scan`) | `sys.schema_unused_indexes` (esquema `sys`) |
| Índice invisible (probar antes de borrar) | `DISABLE` (deja de mantenerse y de usarse) | No nativo | `ALTER INDEX ... INVISIBLE` (sigue manteniéndose, pero el optimizador lo ignora) |

En PostgreSQL, el mantenimiento más importante no es de índices sino `VACUUM`: sin él, las filas viejas que deja cada `UPDATE` y `DELETE` se acumulan y la tabla crece sin límite. Es la contracara de su modelo de versiones, que se ve en el módulo de transacciones.

### En MongoDB

WiredTiger reutiliza el espacio libre de los documentos borrados, pero no lo devuelve al sistema operativo. El comando `compact` reorganiza una colección y sus índices para recuperar espacio, y `db.coleccion.aggregate([{ $indexStats: {} }])` muestra cuántas veces se usó cada índice desde el último reinicio, con la misma advertencia que en SQL Server. Para probar si un índice es prescindible sin borrarlo, MongoDB permite ocultarlo (`hideIndex`).

-----

## Cuándo sí y cuándo no

**Reconstruye o reorganiza cuando:**

* La densidad de página de un índice grande es baja y eso se traduce en más lecturas.
* Un índice se usa en *scans* grandes y la medición muestra una mejora después del mantenimiento.

**No lo hagas cuando:**

* El índice es chico (menos de unas mil páginas).
* La fragmentación vuelve en días: corrige la causa (la clave o el patrón de escritura).

**Ten en cuenta:**

* El mantenimiento de estadísticas suele tener más impacto que el de índices, y es mucho más barato.

-----

## Resumen en 5 líneas

1. Las divisiones de página causan fragmentación lógica (desorden) y baja densidad (páginas medio vacías); la densidad importa siempre, el desorden mucho menos en SSD.
2. Mide con `sys.dm_db_index_physical_stats` en modo `SAMPLED` o `DETAILED` (`LIMITED` no mide la densidad) y solo en índices grandes.
3. `REORGANIZE` es en línea e interrumpible pero no actualiza estadísticas; `REBUILD` es completo, atómico y actualiza las estadísticas del índice.
4. Los umbrales 5/30 % son un punto de partida, no una regla; el fill factor se ajusta índice por índice, y la solución de fondo suele ser cambiar la clave.
5. Revisa periódicamente índices sin uso (durante un ciclo completo) y sugerencias (como pistas, no como órdenes).

-----

## Para profundizar

<details>
<summary>Medir las divisiones de página</summary>

No todas las divisiones son iguales: agregar una página al final de un índice creciente también cuenta como "división" en los contadores generales, pero es barata. Las divisiones caras son las que ocurren en medio del índice. El contador `Page Splits/sec` (en `sys.dm_os_performance_counters`) las mezcla; para distinguirlas, Extended Events tiene el evento `transaction_log` filtrado por la operación `LOP_DELETE_SPLIT`, que solo registra las divisiones "malas". Se ve en [Extended Events](../07-monitoreo-y-administracion/03-Extended%20Events.md).

</details>

<details>
<summary>Deshabilitar un índice antes de borrarlo</summary>

`ALTER INDEX IX_Nombre ON dbo.Tabla DISABLE;` hace que el índice deje de usarse y de mantenerse, pero conserva su definición en los metadatos. Si después de un tiempo nadie se quejó, se borra con `DROP INDEX`; si algo se volvió lento, se vuelve a crear con `ALTER INDEX IX_Nombre ON dbo.Tabla REBUILD;`. Ojo: deshabilitar el índice **agrupado** deja la tabla entera inaccesible, porque el agrupado **es** la tabla.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Con el tiempo, los índices se fragmentan por las inserciones y actualizaciones. Se miden con `sys.dm_db_index_physical_stats`: con poca fragmentación se usa `REORGANIZE`, y con mucha `REBUILD`. También hay que actualizar las estadísticas y revisar si hay índices que no se usan.

### Respuesta ampliada (semi-senior)

Distingo la fragmentación lógica, que afecta sobre todo a los *scans* en discos mecánicos, de la baja densidad de página, que en cualquier almacenamiento significa más páginas, más lecturas y más memoria. Las mido con `sys.dm_db_index_physical_stats` en modo `SAMPLED`, solo en índices grandes, y uso los umbrales clásicos como punto de partida, no como regla: en SSD priorizo la densidad y verifico que el mantenimiento mejore algo medible. `REORGANIZE` es en línea, interrumpible sin perder trabajo y no actualiza estadísticas; `REBUILD` es atómico, permite cambiar el fill factor, actualiza con lectura completa las estadísticas del índice y, en línea o reanudable, reduce el impacto. El fill factor lo ajusto índice por índice, y si un índice se fragmenta en días, cambio la causa (una clave aleatoria) en lugar de reconstruirlo cada noche. Le doy más prioridad a las estadísticas que a la fragmentación. Reviso índices sin uso durante un ciclo de negocio completo, porque los contadores se reinician, y trato las sugerencias de índices faltantes como pistas.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `REORGANIZE` y `REBUILD`?**
`REORGANIZE` reordena las hojas en línea y en pasos pequeños, sin actualizar estadísticas; `REBUILD` recrea el índice completo, puede cambiar el fill factor y actualiza las estadísticas del índice.

**2. ¿Qué es el fill factor y cuándo lo bajarías?**
El porcentaje de cada página que se llena al reconstruir. Lo bajaría en índices con inserciones aleatorias o actualizaciones que agrandan filas, y solo si las divisiones de página son un problema medido.

**3. ¿Borrarías un índice que `dm_db_index_usage_stats` muestra sin uso?**
No sin verificar durante un ciclo de negocio completo, porque los contadores se reinician; primero lo deshabilitaría.

**4. ¿Siguen siendo válidos los umbrales de 5 % y 30 %?**
Como punto de partida; Microsoft aclaró que no son reglas, y en SSD la densidad de página importa más que la fragmentación lógica.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta que liste los índices de más de 1.000 páginas de `TiendaGrande` con una densidad de página menor al 75 %, y el comando que usarías para cada uno.

<details>
<summary>Solución</summary>

```sql
SELECT OBJECT_NAME(ps.object_id)        AS Tabla,
       i.name                            AS Indice,
       ps.avg_page_space_used_in_percent AS Densidad,
       ps.avg_fragmentation_in_percent   AS Fragmentacion,
       ps.page_count,
       CONCAT('ALTER INDEX ', QUOTENAME(i.name), ' ON ',
              QUOTENAME(OBJECT_SCHEMA_NAME(ps.object_id)), '.', QUOTENAME(OBJECT_NAME(ps.object_id)),
              ' REBUILD WITH (ONLINE = ON);') AS Comando
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') AS ps
JOIN sys.indexes AS i ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE ps.index_level = 0
  AND ps.page_count > 1000
  AND ps.avg_page_space_used_in_percent < 75
ORDER BY ps.page_count DESC;
```

Se usa `REBUILD` y no `REORGANIZE` porque el objetivo es recuperar la **densidad** (compactar al fill factor), que `REBUILD` hace completamente. `QUOTENAME` arma nombres seguros para copiar y ejecutar (y es la base del SQL dinámico seguro del módulo de programación). `ONLINE = ON` solo funciona en las ediciones que lo admiten; en Standard, se quita y se ejecuta en una ventana de mantenimiento.

</details>

**Ejercicio 2.** Una tabla de 50 millones de filas tiene 14 índices no agrupados. `dm_db_index_usage_stats`, con el servidor encendido desde hace 40 días, muestra que 5 de ellos tienen 0 lecturas y millones de escrituras. ¿Qué harías, en qué orden?

<details>
<summary>Solución</summary>

1. **Confirmar el período:** 40 días cubren un ciclo mensual; verificar si hay procesos trimestrales o anuales (cierres, auditorías) que podrían usarlos.
2. **Revisar si alguno sostiene algo:** un índice único (aunque no se lea, impide duplicados), o un índice nombrado en un hint (`WITH (INDEX(...))`) en algún procedimiento: si se borra, esa consulta falla. Buscar el nombre en `sys.sql_modules`.
3. **Guardar el script** de creación de cada uno (clic derecho → *Script Index as → CREATE To*).
4. **Deshabilitarlos** (`ALTER INDEX ... DISABLE`) en lugar de borrarlos, y observar unas semanas.
5. **Borrarlos** si nadie notó nada. Se ganan 5 escrituras menos por cada fila insertada, menos espacio y copias de seguridad más chicas.

Lo que **no** haría: borrarlos el mismo día "porque la DMV lo dice".

</details>

-----

## Siguiente lección

Con esto termina el módulo de índices y rendimiento. El siguiente módulo trata lo que pasa cuando muchas consultas trabajan a la vez:

[Transacciones y ACID](../05-transacciones-y-concurrencia/01-Transacciones%20y%20ACID.md)
