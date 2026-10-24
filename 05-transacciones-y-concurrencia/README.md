# 05. Transacciones y concurrencia

Qué pasa cuando muchas sesiones leen y escriben los mismos datos a la vez: transacciones y ACID, niveles de aislamiento y anomalías, bloqueos, versionado de filas, *deadlocks*, manejo de errores en T-SQL y cómo ejecutar operaciones masivas sin detener la base.

Las lecciones 01 a 06 usan la base `Tienda`, con ejemplos de **dos ventanas de SSMS** que se cruzan en el tiempo; la lección 07 usa `TiendaGrande`.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 01 | [Transacciones y ACID](01-Transacciones%20y%20ACID.md) | ACID, el log, modos de transacción, `@@TRANCOUNT`, el mito de las anidadas, puntos de guardado |
| 02 | [Niveles de aislamiento y anomalías](02-Niveles%20de%20aislamiento%20y%20anomalías.md) | Lecturas sucias, no repetibles, fantasma, actualización perdida; `UPDATE` atómico, `UPDLOCK` y `ROWVERSION` |
| 03 | [Bloqueos, hints y NOLOCK](03-Bloqueos%2C%20hints%20y%20NOLOCK.md) | Modos S, U, X, compatibilidad, escalamiento, cadenas de bloqueo, hints, colas con `READPAST`, los riesgos de `NOLOCK` |
| 04 | [Versionado con RCSI y SNAPSHOT](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md) | El almacén de versiones, RCSI, `SNAPSHOT`, el error 3960, riesgos de activar RCSI |
| 05 | [Deadlocks](05-Deadlocks.md) | Detección, víctima, patrones típicos, el gráfico de `system_health`, prevención y reintento |
| 06 | [Manejo de errores](06-Manejo%20de%20errores.md) | Alcance de los errores, `XACT_ABORT`, `TRY...CATCH`, `XACT_STATE`, `THROW`, transitorios, registro |
| 07 | [Operaciones masivas en lotes](07-Operaciones%20masivas%20en%20lotes.md) | Lotes por clave, progreso reanudable, regulación, staging, registro mínimo, `MERGE` |

-----

## Mapa del tema

```text
               Transacción (01): todo o nada
                          │
          muchas transacciones a la vez
                          │
                          ▼
         Anomalías y niveles de aislamiento (02)
                          │
          ┌───────────────┴────────────────┐
          ▼                                ▼
   Bloqueos (03)                   Versiones (04)
   S, U, X, escalamiento           RCSI, SNAPSHOT
   (las lecturas esperan)          (las lecturas no esperan)
          │
          ▼
   Deadlocks (05): ciclos de espera
          │
          ▼
   Manejo de errores (06): XACT_ABORT, TRY...CATCH, reintentos
          │
          ▼
   Operaciones masivas (07): muchas transacciones cortas
```

-----

## Sobre los apuntes de origen

Este módulo integra las sesiones 1, 2 y 10 del curso **SQL Server Intermedio 2026** (carpeta `SQL SERVER FRACTAL`) y la nota sobre `NOLOCK` de los apuntes anteriores. Las lecciones corrigen algunas afirmaciones de los cheat sheets:

* Un `ROLLBACK` dentro del `TRY` no pasa el control al `CATCH`: lo que lo pasa es un error ([lección 01](01-Transacciones%20y%20ACID.md) y [06](06-Manejo%20de%20errores.md)).
* Registrar el error en un procedimiento separado no lo protege del `ROLLBACK`: hay que registrarlo **después** del `ROLLBACK` ([lección 06](06-Manejo%20de%20errores.md)).
* `ROWLOCK` no es una herramienta general contra los *deadlocks*: SQL Server ya bloquea por fila ([lección 03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md)).
* `NOT BETWEEN '22:00' AND '06:00'` no define una ventana nocturna, y `TRY_CONVERT(DECIMAL, ...)` sin precisión redondea a enteros ([lección 07](07-Operaciones%20masivas%20en%20lotes.md)).

-----

## Requisitos

Los módulos [03. Consultas](../03-consultas/README.md) y [04. Índices y rendimiento](../04-indices-y-rendimiento/README.md). Para los ejemplos de concurrencia, abre dos ventanas de consulta en SSMS sobre la misma base.

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
