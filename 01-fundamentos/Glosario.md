# Glosario: Fundamentos

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Acción referencial:** lo que hace el motor con las filas hijas cuando se borra o actualiza el padre: `NO ACTION`, `CASCADE`, `SET NULL` o `SET DEFAULT`. ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**Álgebra relacional:** conjunto de operaciones (selección, proyección, join, unión...) que reciben tablas y devuelven tablas. ([02](02-El%20modelo%20relacional.md))

**Atributo:** columna de una relación, con nombre y dominio. ([02](02-El%20modelo%20relacional.md))

**Autorreferencia:** clave foránea que apunta a la misma tabla, como `Empleado.JefeId`. ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**Base de datos:** conjunto organizado de datos persistentes sobre un dominio. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Cardinalidad (de una relación):** cantidad de filas de una tabla. No confundir con la cardinalidad de un vínculo (1:N), que se ve en el módulo de diseño. ([02](02-El%20modelo%20relacional.md))

**`CHECK`:** restricción con una condición que cada fila debe cumplir; rechaza solo cuando la condición es falsa. ([06](06-Restricciones.md))

**Clave alternativa:** clave candidata que no se eligió como primaria; se protege con `UNIQUE`. ([02](02-El%20modelo%20relacional.md))

**Clave candidata:** superclave mínima: si se le quita cualquier columna, deja de identificar las filas. ([02](02-El%20modelo%20relacional.md))

**Clave compuesta:** clave formada por más de una columna. ([04](04-Claves%20primarias.md))

**Clave foránea (FK):** columna cuyos valores deben existir en la clave primaria o `UNIQUE` de otra tabla. ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**Clave natural:** clave que existe en el mundo real (email, código ISO). ([04](04-Claves%20primarias.md))

**Clave primaria (PK):** clave candidata elegida para identificar las filas; implica `UNIQUE` y `NOT NULL`. ([04](04-Claves%20primarias.md))

**Clave sustituta (surrogate):** identificador sin significado de negocio, generado por la base. ([04](04-Claves%20primarias.md))

**`COALESCE`:** función estándar que devuelve el primer argumento no nulo. ([07](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md))

**Conversión implícita:** conversión de tipo que el motor hace por su cuenta para comparar valores; puede impedir el uso de índices. ([03](03-Tipos%20de%20datos.md))

**DCL:** sublenguaje de permisos: `GRANT`, `REVOKE`, `DENY`. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**DDL:** sublenguaje de definición de estructura: `CREATE`, `ALTER`, `DROP`. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**`DEFAULT`:** valor que toma una columna cuando el `INSERT` no la menciona. ([06](06-Restricciones.md))

**DML:** sublenguaje de manipulación de datos: `SELECT`, `INSERT`, `UPDATE`, `DELETE`. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Dominio:** conjunto de valores válidos para un atributo. ([02](02-El%20modelo%20relacional.md))

**Esquema (schema):** contenedor con nombre que agrupa objetos dentro de una base de datos, como `dbo`. También se usa para referirse al diseño completo de la base. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Fila huérfana:** fila hija cuya clave foránea apunta a un padre inexistente. ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**Grado:** cantidad de columnas de una tabla. ([02](02-El%20modelo%20relacional.md))

**GUID / UUID:** identificador de 128 bits prácticamente irrepetible; en SQL Server, `UNIQUEIDENTIFIER`. ([04](04-Claves%20primarias.md))

**`IDENTITY`:** propiedad de una columna de SQL Server que genera números crecientes al insertar; puede tener huecos. ([04](04-Claves%20primarias.md))

**Independencia de datos:** poder cambiar cómo se guardan los datos sin cambiar las consultas ni las aplicaciones. ([02](02-El%20modelo%20relacional.md))

**Índice agrupado (clustered):** índice que define el orden físico de las filas de la tabla; en SQL Server, por defecto es la clave primaria. ([04](04-Claves%20primarias.md))

**Índice filtrado:** índice que solo incluye las filas que cumplen un `WHERE`; permite unicidad solo cuando hay valor. ([06](06-Restricciones.md))

**Instancia:** servidor de base de datos en ejecución, con su memoria y sus procesos. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Integridad referencial:** garantía de que toda clave foránea apunta a una fila que existe. ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**Intercalación (collation):** reglas de comparación y orden del texto (mayúsculas, acentos). ([03](03-Tipos%20de%20datos.md))

**Lógica de tres valores (3VL):** lógica en la que una condición puede ser `TRUE`, `FALSE` o `UNKNOWN`. ([07](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md))

**`NOT NULL`:** restricción que hace obligatoria una columna. ([06](06-Restricciones.md))

**`NULL`:** marcador de ausencia de valor ("no se sabe" o "no aplica"). ([07](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md))

**OLAP:** carga analítica: pocas consultas grandes, casi solo de lectura. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**OLTP:** carga transaccional: muchas operaciones pequeñas de lectura y escritura. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Precisión y escala:** en `DECIMAL(p, s)`, `p` es el total de dígitos y `s` los decimales. ([03](03-Tipos%20de%20datos.md))

**Relación:** en el modelo formal, una tabla (un conjunto de tuplas). No significa "vínculo entre tablas". ([02](02-El%20modelo%20relacional.md))

**Restricción (constraint):** regla declarativa que el motor verifica en cada escritura. ([06](06-Restricciones.md))

**Restricción no confiable:** clave foránea o `CHECK` creada o reactivada sin verificar los datos existentes (`is_not_trusted = 1`). ([05](05-Claves%20foráneas%20e%20integridad%20referencial.md))

**`SEQUENCE`:** objeto que genera números de forma independiente de una tabla. ([04](04-Claves%20primarias.md))

**SGBD (DBMS):** software que administra bases de datos: SQL Server, PostgreSQL, MySQL, MongoDB. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**SQL:** lenguaje declarativo estándar para definir, consultar y modificar datos relacionales. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Superclave:** conjunto de columnas cuyos valores no se repiten entre filas. ([02](02-El%20modelo%20relacional.md))

**TCL:** sublenguaje de transacciones: `BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`. ([01](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md))

**Tipo exacto / aproximado:** `DECIMAL` guarda el valor exacto; `FLOAT` guarda una aproximación binaria. ([03](03-Tipos%20de%20datos.md))

**Tupla:** fila de una relación. ([02](02-El%20modelo%20relacional.md))

**Unicode:** estándar que representa los caracteres de todos los idiomas; en SQL Server, `NVARCHAR`. ([03](03-Tipos%20de%20datos.md))

**`UNIQUE`:** restricción que impide valores repetidos; en SQL Server admite un solo `NULL`. ([06](06-Restricciones.md))

**`UNKNOWN`:** tercer valor de verdad; resultado de comparar con `NULL`. ([07](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md))
