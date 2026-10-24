# 02. Diseño y modelado

Cómo se diseña una base de datos relacional desde cero: de las reglas de negocio al diagrama entidad-relación, del diagrama a las tablas, y de las tablas a un esquema sin redundancias, con las decisiones de desnormalización, los patrones de modelado y las convenciones de nombres que lo completan.

El hilo conductor del módulo es la base `Tienda`: cada lección construye una parte, y el resultado final es el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md) que usan los módulos siguientes.

-----

## Orden de lectura

| # | Lección | Qué aprendes |
| --- | --- | --- |
| 01 | [Proceso de diseño](01-Proceso%20de%20diseño.md) | Niveles conceptual, lógico y físico; reglas de negocio; las preguntas que más diseño ahorran |
| 02 | [Modelo entidad-relación](02-Modelo%20entidad-relación.md) | Entidades, tipos de atributos, relaciones con atributos, entidades débiles, notación pata de gallo |
| 03 | [Cardinalidad y relaciones](03-Cardinalidad%20y%20relaciones.md) | Máximos y mínimos, 1:N, N:M y tablas intermedias, 1:1, relaciones recursivas |
| 04 | [Del modelo ER a tablas](04-Del%20modelo%20ER%20a%20tablas.md) | Las reglas de traducción: compuestos, multivaluados, derivados, entidades débiles, columnas calculadas |
| 05 | [Dependencias funcionales](05-Dependencias%20funcionales.md) | `X → Y`, dependencias parciales y transitivas, anomalías, cierre y claves candidatas |
| 06 | [Normalización 1FN 2FN 3FN](06-Normalización%201FN%202FN%203FN.md) | De una planilla al esquema de la tienda, paso a paso; descomposición sin pérdida |
| 07 | [BCNF y formas superiores](07-BCNF%20y%20formas%20superiores.md) | BCNF y las dependencias que se pierden, 4FN, 5FN, redundancia controlada con FK compuestas |
| 08 | [Desnormalización](08-Desnormalización.md) | Snapshots frente a copias, columnas calculadas, vistas indexadas, esquema estrella |
| 09 | [Patrones de modelado](09-Patrones%20de%20modelado.md) | Herencia (TPH/TPT/TPC), auditoría y tablas temporales, borrado lógico, catálogos, EAV y FK polimórficas |
| 10 | [Convenciones de nombres](10-Convenciones%20de%20nombres.md) | Singular o plural, PascalCase y `snake_case`, nombres de claves y restricciones, palabras reservadas |

-----

## Mapa del tema

```text
Reglas de negocio ──► Modelo conceptual ──► Modelo lógico ──► Modelo físico
   (01)                 (02, 03)              (04)              (08, 09, 10)
                        entidades,            tablas, PK,       tipos, índices,
                        relaciones,           FK, tablas        desnormalización,
                        cardinalidad          intermedias       patrones, nombres
                                                 │
                                                 ▼
                                      Validación: normalización
                                      dependencias funcionales (05)
                                      1FN → 2FN → 3FN (06)
                                      BCNF → 4FN → 5FN (07)
```

-----

## Requisitos

El módulo [01. Fundamentos](../01-fundamentos/README.md) completo: tipos, claves primarias y foráneas, restricciones y `NULL` aparecen en todas las lecciones.

Las palabras nuevas de cada lección están en el [Glosario](Glosario.md).
