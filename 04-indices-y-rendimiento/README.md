# 04. Índices y rendimiento

Cómo guarda los datos SQL Server, cómo los encuentra y cómo hacer que los encuentre rápido: páginas y árboles B, índices agrupados, no agrupados, compuestos, cubrientes y filtrados, lectura de planes de ejecución, estadísticas y *parameter sniffing*, consultas SARGables y mantenimiento.

Este módulo usa `TiendaGrande`, una copia del esquema de la tienda con un millón de pedidos, generada en la lección 00: con las 5 filas de `Tienda`, ningún índice se nota.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 00 | [Datos de volumen](00-Datos%20de%20volumen.md) | El script de `TiendaGrande`: tabla de números, datos reproducibles y distribuciones sesgadas |
| 01 | [Cómo guarda los datos SQL Server](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md) | Páginas de 8 KB, tamaño de fila, heap frente a árbol B, lecturas lógicas y físicas |
| 02 | [Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md) | Seek, scan, Key Lookup, punto de inflexión, cómo elegir la clave del agrupado |
| 03 | [Índices compuestos, cubrientes y filtrados](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md) | Prefijo izquierdo, regla ESR, `INCLUDE`, índices filtrados, redundancias |
| 04 | [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md) | Plan estimado y real, operadores, predicados, advertencias, `STATISTICS IO/TIME` |
| 05 | [Planes de ejecución II](05-Planes%20de%20ejecución%20II.md) | Estadísticas e histogramas, caché de planes, *parameter sniffing*, hints |
| 06 | [Consultas SARGables y antipatrones](06-Consultas%20SARGables%20y%20antipatrones.md) | Funciones sobre columnas, conversiones implícitas, `OR`, consultas "para todo", N+1 |
| 07 | [Mantenimiento de índices](07-Mantenimiento%20de%20índices.md) | Fragmentación y densidad, `REORGANIZE` y `REBUILD`, fill factor, índices sin uso y sugeridos |

-----

## Mapa del tema

```text
                Páginas de 8 KB (01)
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
      Heap (scan)            Árbol B (seek)
                                     │
               ┌─────────────────────┼──────────────────────┐
               ▼                     ▼                      ▼
     Agrupado = la tabla     No agrupado + Key Lookup   Compuesto, cubriente,
     (02)                    (02)                       filtrado (03)
                                     │
                                     ▼
                   ¿Qué eligió el optimizador y por qué?
               Plan de ejecución (04) ← Estadísticas (05)
                                     │
                     ┌───────────────┴────────────────┐
                     ▼                                ▼
          ¿La consulta deja usar        ¿El índice sigue sano?
          el índice? SARGable (06)      Mantenimiento (07)
```

-----

## Sobre los apuntes de origen

Este módulo integra las sesiones 3, 4, 7, 8 y 9 del curso **SQL Server Intermedio 2026** (carpeta `SQL SERVER FRACTAL`). Las lecciones corrigen algunas afirmaciones de los cheat sheets:

* `ISNULL(Col, 0) = 5` equivale a `Col = 5`, no a `Col = 5 OR Col IS NULL` ([lección 06](06-Consultas%20SARGables%20y%20antipatrones.md)).
* `WHERE PrecioDecimal = 99.99` y `WHERE FechaDatetime = '2025-03-28'` no provocan conversiones problemáticas: se convierte el literal, no la columna ([lección 06](06-Consultas%20SARGables%20y%20antipatrones.md)).
* El orden recomendado de las columnas de un índice es igualdad → orden → rango (ESR), no igualdad → rango → orden ([lección 03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md)).
* Los umbrales de fragmentación de 5 % y 30 % son un punto de partida, no una regla, y `LIMITED` no mide la densidad de página ([lección 07](07-Mantenimiento%20de%20índices.md)).

-----

## Requisitos

El módulo [03. Consultas](../03-consultas/README.md) completo, y una instancia de SQL Server con unos 500 MB libres para `TiendaGrande`.

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
