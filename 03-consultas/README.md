# 03. Consultas

Cómo se le pregunta a una base de datos: leer, filtrar, ordenar, calcular, agrupar, combinar tablas, escribir consultas por pasos y modificar datos de forma segura, todo en T-SQL sobre la base `Tienda` del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md). El módulo termina con el uso de la IA para escribir SQL y, sobre todo, para revisarlo.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 00 | [Entorno de trabajo](00-Entorno%20de%20trabajo.md) | Servicio, conexión, SSMS y sus atajos, `GO`, VS Code, `sqlcmd` |
| 01 | [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md) | `SELECT`, alias, `DISTINCT` y el orden lógico de procesamiento |
| 02 | [Filtrar con WHERE](02-Filtrar%20con%20WHERE.md) | Comparaciones, precedencia de `AND`/`OR`, `BETWEEN` y rangos de fechas, `IN`, `LIKE`, `IS NULL`, filtros SARGables |
| 03 | [Ordenar y paginar](03-Ordenar%20y%20paginar.md) | `ORDER BY`, `TOP`, `WITH TIES`, `OFFSET FETCH`, orden determinista, paginación por clave |
| 04 | [Expresiones, CASE y funciones](04-Expresiones%2C%20CASE%20y%20funciones.md) | División entera, concatenación, `CASE`, funciones de texto, fecha y conversión; de SQLite a T-SQL |
| 05 | [Agregados y GROUP BY](05-Agregados%20y%20GROUP%20BY.md) | `COUNT`, `SUM`, `AVG`, `GROUP BY`, `HAVING`, `STRING_AGG`, agregación condicional, `ROLLUP`, embudos |
| 06 | [JOIN](06-JOIN.md) | `INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`, self join, anti join, `ON` frente a `WHERE`, multiplicación de filas |
| 07 | [Operaciones de conjuntos](07-Operaciones%20de%20conjuntos.md) | `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT` |
| 08 | [Subconsultas](08-Subconsultas.md) | Escalares, `IN`, `EXISTS`/`NOT EXISTS`, correlacionadas, tablas derivadas |
| 09 | [CTE](09-CTE.md) | `WITH`, CTE encadenadas, CTE recursivas, jerarquías y series |
| 10 | [Funciones de ventana](10-Funciones%20de%20ventana.md) | `OVER`, `PARTITION BY`, rankings, top N por grupo, `LAG`/`LEAD`, acumulados, `ROWS` frente a `RANGE` |
| 11 | [Modificar datos](11-Modificar%20datos.md) | `INSERT`, `UPDATE` y `DELETE` con joins, `OUTPUT`, `TRUNCATE`, `MERGE`, el método seguro con transacciones |
| 12 | [SQL con IA](12-SQL%20con%20IA.md) | Contexto para los prompts, lista de control para verificar, riesgos |

-----

## Mapa del tema

```text
                    SELECT ... FROM (01)
                           │
       ┌───────────────────┼────────────────────┐
       ▼                   ▼                    ▼
 Filtrar (02)        Calcular (04)        Ordenar y paginar (03)
 WHERE, LIKE,        CASE, funciones      ORDER BY, TOP, OFFSET
 IS NULL
       │                   │
       └─────────┬─────────┘
                 ▼
         Agregar (05)  ◄──────────────  Funciones de ventana (10)
         GROUP BY, HAVING               el agregado sin colapsar filas
                 │
     ┌───────────┼──────────────┐
     ▼           ▼              ▼
 JOIN (06)   Conjuntos (07)  Subconsultas (08) ──► CTE (09)
 horizontal  vertical        consultas dentro      por pasos y
                             de consultas          recursivas
                 │
                 ▼
       Modificar datos (11) ──► SQL con IA (12)
       INSERT, UPDATE, DELETE    escribir rápido, verificar siempre
```

-----

## Requisitos

Los módulos [01. Fundamentos](../01-fundamentos/README.md) y [02. Diseño y modelado](../02-diseno-y-modelado/README.md), y la base `Tienda` levantada con el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md): todos los resultados de las lecciones se calcularon sobre esos datos.

Sobre los apuntes de origen: este módulo integra los apuntes del curso de SQL de Codecademy, que usa **SQLite**. Las diferencias de dialecto (`LIMIT`, `strftime`, `GROUP_CONCAT`, `||`) están traducidas a T-SQL en cada lección, con una tabla de equivalencias en [Expresiones, CASE y funciones](04-Expresiones%2C%20CASE%20y%20funciones.md).

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
