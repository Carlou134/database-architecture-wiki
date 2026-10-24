# 🗄️ Database Architecture Wiki

Repositorio personal de apuntes y guías rápidas (*cheatsheets*) sobre bases de datos relacionales (**SQL Server**, **PostgreSQL**, **MySQL**) y NoSQL (**MongoDB**): modelado de datos, índices, optimización de consultas, transacciones y concurrencia. Cada lección sigue el mismo formato de estudio, con consultas de ejemplo, errores comunes, preguntas de entrevista y práctica con la solución plegada.

Motor de referencia: **SQL Server (T-SQL)**. PostgreSQL y MySQL aparecen como comparación; MongoDB tiene su propio módulo.

-----

## 📂 Estructura

Todas las lecciones relacionales usan el mismo conjunto de datos, definido en [`00-Esquema de ejemplo.md`](00-Esquema%20de%20ejemplo.md). Los términos nuevos de cada lección se reúnen en el `Glosario.md` de su módulo.

| Módulo | Temas |
| --- | --- |
| 01. Fundamentos | Modelo relacional, tablas, tipos de datos, claves primarias y foráneas, restricciones |
| 02. Consultas | `SELECT`, filtros, ordenamiento, `JOIN`, `GROUP BY` y `HAVING`, subconsultas, CTE, funciones de ventana, `INSERT`/`UPDATE`/`DELETE` |
| 03. Diseño y modelado | Modelo entidad-relación, cardinalidad, normalización y desnormalización |
| 04. Índices y rendimiento | Índices *clustered* y *non-clustered*, planes de ejecución, consultas SARGables, estadísticas |
| 05. Transacciones y concurrencia | ACID, niveles de aislamiento, bloqueos, *deadlocks*, MVCC |
| 06. NoSQL con MongoDB | Modelo documental, embeber frente a referenciar, índices, *aggregation pipeline*, transacciones |

Planificado: vistas, procedimientos almacenados, funciones y *triggers*; seguridad y permisos; respaldos y recuperación.

-----

## 📖 Formato de las lecciones

Cada módulo tiene un `README.md` (orden de lectura y mapa del tema) y un `Glosario.md`.

Cada lección tiene las mismas 13 secciones:

1. **En una frase**
2. **Antes de empezar** (requisitos y palabras nuevas)
3. **El problema**
4. **Cómo funciona** (tabla de entrada, consulta y tabla de resultado)
5. **Ejemplo completo** (sobre el esquema de ejemplo, con un antes y un después)
6. **Errores comunes** (qué pasa, por qué y cómo se arregla)
7. **Según el motor** (SQL Server, PostgreSQL y MySQL; MongoDB cuando el concepto tiene un equivalente real)
8. **Cuándo sí y cuándo no**
9. **Resumen en 5 líneas**
10. **Para profundizar** (plegado)
11. **En entrevista** (respuesta junior, respuesta semi-senior y preguntas de seguimiento)
12. **Práctica** (solución plegada, con la consulta y el resultado esperado)
13. **Siguiente lección**

-----

## 📄 Licencia

[MIT](LICENSE)
