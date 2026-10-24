# 06. Programación en T-SQL

Cómo se programa dentro de SQL Server: vistas, procedimientos almacenados, funciones y *triggers*; tablas temporales, variables de tabla y parámetros con valor de tabla; por qué el procesamiento fila por fila es la excepción; y cómo armar SQL dinámico sin abrirle la puerta a la inyección de SQL.

Las lecciones usan la base `Tienda` para los ejemplos con resultados exactos, y `TiendaGrande` para las comparaciones de rendimiento.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 01 | [Vistas](01-Vistas.md) | Expansión, usos, `SCHEMABINDING`, `WITH CHECK OPTION`, seguridad, el problema de `SELECT *` y de las vistas apiladas |
| 02 | [Procedimientos almacenados](02-Procedimientos%20almacenados.md) | Parámetros y `OUTPUT`, `NOCOUNT`, seguridad con `EXECUTE`, lógica en la base o en la aplicación |
| 03 | [Funciones](03-Funciones.md) | Escalares, de tabla en línea y de varias instrucciones; *inlining*; qué no pueden hacer |
| 04 | [Triggers](04-Triggers.md) | `AFTER` e `INSTEAD OF`, `inserted` y `deleted`, una ejecución por instrucción, auditoría, costos |
| 05 | [Tablas temporales y variables de tabla](05-Tablas%20temporales%20y%20variables%20de%20tabla.md) | `#Tabla`, `##Tabla`, `@Tabla`, estadísticas, alcance, `ROLLBACK`, TVP, `tempdb` |
| 06 | [Cursores frente a operaciones sobre conjuntos](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md) | Por qué el fila por fila es lento, cómo traducir a conjuntos, cuándo un cursor es legítimo |
| 07 | [SQL dinámico seguro](07-SQL%20dinámico%20seguro.md) | `sp_executesql`, parámetros, `QUOTENAME`, listas blancas, filtros opcionales, pivot dinámico, permisos |

-----

## Mapa del tema

```text
                     Encapsular lógica en la base
                                 │
      ┌──────────────┬───────────┼───────────────┬──────────────┐
      ▼              ▼           ▼               ▼              ▼
  Vistas (01)   Procedimientos  Funciones (03)  Triggers (04)  SQL dinámico (07)
  consultas     (02)            valores o       automáticos    texto armado en
  con nombre    operaciones     tablas dentro   ante cambios   tiempo de ejecución
                completas       de consultas
                     │
                     ▼
        Resultados intermedios: #Tabla, @Tabla, TVP (05)
                     │
                     ▼
        Pensar en conjuntos, no en filas (06)
```

-----

## Sobre los apuntes de origen

Este módulo integra las sesiones 11, 12 y 13 del curso **SQL Server Intermedio 2026** (carpeta `SQL SERVER FRACTAL`). Las lecciones 01 a 04 cubren lo que el README principal tenía como "planificado" (vistas, procedimientos, funciones y *triggers*). Correcciones sobre los cheat sheets:

* Las variables de tabla **no** tienen estadísticas desde 2019 (lo nuevo es la compilación diferida) y **no** viven en memoria: están en `tempdb` ([lección 05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md)).
* Reemplazar un cursor por un `WHILE` sobre una tabla numerada sigue siendo procesamiento fila por fila ([lección 06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md)).
* `QUOTENAME` devuelve `NULL` con más de 128 caracteres, y el SQL dinámico rompe el encadenamiento de propiedad ([lección 07](07-SQL%20dinámico%20seguro.md)).

-----

## Requisitos

Los módulos [03. Consultas](../03-consultas/README.md), [04. Índices y rendimiento](../04-indices-y-rendimiento/README.md) y [05. Transacciones y concurrencia](../05-transacciones-y-concurrencia/README.md), sobre todo el manejo de errores.

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
