# 01. Fundamentos

Qué es una base de datos relacional y con qué piezas se construye: tablas, tipos, claves, restricciones y `NULL`. Es la base de todo lo demás: el [módulo de diseño](../02-diseno-y-modelado/README.md) da por sabido cada concepto de aquí.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 01 | [Qué es una base de datos y un SGBD](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md) | Qué resuelve un SGBD, cliente-servidor, instancia/base/esquema, familias de bases, DDL/DML/DCL/TCL, OLTP frente a OLAP |
| 02 | [El modelo relacional](02-El%20modelo%20relacional.md) | Relación, tupla, atributo, dominio; propiedades de una relación; superclave, candidata, primaria y alternativa; álgebra relacional |
| 03 | [Tipos de datos](03-Tipos%20de%20datos.md) | Enteros, `DECIMAL` frente a `FLOAT`, `NVARCHAR` frente a `VARCHAR`, `DATETIME2`, criterios de elección, conversiones implícitas |
| 04 | [Claves primarias](04-Claves%20primarias.md) | Natural frente a sustituta, `IDENTITY`, `SEQUENCE`, GUID, claves compuestas, índice agrupado |
| 05 | [Claves foráneas e integridad referencial](05-Claves%20foráneas%20e%20integridad%20referencial.md) | Huérfanas, `ON DELETE`/`ON UPDATE`, FK opcionales, autorreferencia, índices en las FK, restricciones no confiables |
| 06 | [Restricciones](06-Restricciones.md) | `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`, nombres de restricciones, defensa en profundidad |
| 07 | [NULL y la lógica de tres valores](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md) | `UNKNOWN`, `IS NULL`, la trampa de `NOT IN`, agregados, `COALESCE` frente a `ISNULL` |

-----

## Mapa del tema

```text
                       SGBD (01)
                          │
                 Modelo relacional (02)
            relación = tabla, tupla = fila
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
  Dominio de cada     Identidad de       Vínculos entre
     columna           cada fila             tablas
        │                 │                  │
  Tipos de datos (03)  Claves primarias  Claves foráneas (05)
  Restricciones (06)       (04)          integridad referencial
        │                 │                  │
        └────────── NULL (07) ───────────────┘
            atraviesa todo: WHERE, JOIN, CHECK,
            UNIQUE, agregados
```

-----

## Requisitos

Ninguno más allá de saber qué es un archivo y una variable. Para ejecutar los ejemplos, levanta el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md).

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
