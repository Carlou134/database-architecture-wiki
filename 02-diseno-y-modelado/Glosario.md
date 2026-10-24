# Glosario: Diseño y modelado

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**1FN (primera forma normal):** cada celda tiene un solo valor, no hay grupos repetidos de columnas y la tabla tiene clave. ([06](06-Normalización%201FN%202FN%203FN.md))

**2FN (segunda forma normal):** 1FN y ningún atributo no clave depende de una parte de una clave candidata. ([06](06-Normalización%201FN%202FN%203FN.md))

**3FN (tercera forma normal):** 2FN y ningún atributo no clave depende de otro atributo no clave. ([06](06-Normalización%201FN%202FN%203FN.md))

**4FN (cuarta forma normal):** BCNF y sin dependencias multivaluadas no triviales que no partan de una superclave. ([07](07-BCNF%20y%20formas%20superiores.md))

**5FN (quinta forma normal):** sin dependencias de reunión que no se deriven de las claves candidatas. ([07](07-BCNF%20y%20formas%20superiores.md))

**Anomalía:** problema de actualización, inserción o borrado causado por la redundancia. ([05](05-Dependencias%20funcionales.md))

**Asociación polimórfica:** columna que referencia a tablas distintas según el valor de otra columna; no admite clave foránea. Antipatrón. ([09](09-Patrones%20de%20modelado.md))

**Atributo compuesto:** atributo divisible en partes con significado propio (dirección = calle + número + ciudad). ([02](02-Modelo%20entidad-relación.md))

**Atributo derivado:** atributo que se calcula a partir de otros (el total de un pedido, la edad). ([02](02-Modelo%20entidad-relación.md))

**Atributo multivaluado:** atributo que puede tener varios valores para una misma instancia (los teléfonos de un cliente). ([02](02-Modelo%20entidad-relación.md))

**Atributo primo:** atributo que forma parte de alguna clave candidata. ([06](06-Normalización%201FN%202FN%203FN.md))

**BCNF (forma normal de Boyce-Codd):** para toda dependencia no trivial `X → Y`, `X` es superclave. ([07](07-BCNF%20y%20formas%20superiores.md))

**Borrado lógico (soft delete):** marcar una fila como eliminada en lugar de borrarla. ([09](09-Patrones%20de%20modelado.md))

**Cardinalidad máxima:** cuántas instancias, como máximo, se pueden vincular con una instancia de la otra entidad: uno o muchos. ([03](03-Cardinalidad%20y%20relaciones.md))

**Cardinalidad mínima (participación):** si el vínculo es opcional (cero) u obligatorio (al menos uno). ([03](03-Cardinalidad%20y%20relaciones.md))

**Cierre (X⁺):** todos los atributos que se pueden determinar a partir de `X` aplicando las dependencias. ([05](05-Dependencias%20funcionales.md))

**Columna calculada (computed column):** columna cuyo valor calcula el motor a partir de otras de la misma fila; con `PERSISTED`, se guarda y se puede indexar. ([04](04-Del%20modelo%20ER%20a%20tablas.md))

**Columna discriminadora:** columna que indica el subtipo de cada fila en una tabla con herencia. ([09](09-Patrones%20de%20modelado.md))

**Dependencia de reunión (join dependency):** la tabla se reconstruye exactamente uniendo varias proyecciones suyas. ([07](07-BCNF%20y%20formas%20superiores.md))

**Dependencia funcional (`X → Y`):** dos filas con el mismo valor de `X` tienen el mismo valor de `Y`. ([05](05-Dependencias%20funcionales.md))

**Dependencia multivaluada (`X ↠ Y`):** `X` determina un conjunto de valores de `Y`, independiente del resto de las columnas. ([07](07-BCNF%20y%20formas%20superiores.md))

**Dependencia parcial:** un atributo depende de una parte de una clave compuesta. ([05](05-Dependencias%20funcionales.md))

**Dependencia transitiva:** `X → Y → Z`, donde `Y` no es clave. ([05](05-Dependencias%20funcionales.md))

**Descomposición sin pérdida:** división de una tabla que permite reconstruir la original exacta con un join. ([06](06-Normalización%201FN%202FN%203FN.md))

**Desnormalización:** redundancia introducida a propósito para mejorar las lecturas. ([08](08-Desnormalización.md))

**Determinante:** el lado izquierdo de una dependencia funcional. ([05](05-Dependencias%20funcionales.md))

**EAV (entidad-atributo-valor):** diseño que guarda cada atributo como una fila `(entidad, atributo, valor)`. Antipatrón. ([09](09-Patrones%20de%20modelado.md))

**Entidad:** algo del negocio de lo que se guarda información y que tiene varias instancias distinguibles. ([02](02-Modelo%20entidad-relación.md))

**Entidad débil:** entidad que no se identifica sin la clave de otra, su propietaria (el aula 101 del edificio A). ([02](02-Modelo%20entidad-relación.md))

**Esquema estrella:** diseño analítico con una tabla de hechos central y dimensiones desnormalizadas. ([08](08-Desnormalización.md))

**Fuente de verdad:** el lugar donde vive el valor original; las copias se derivan de él. ([08](08-Desnormalización.md))

**Identificador:** atributo o conjunto de atributos que distingue cada instancia de una entidad. ([02](02-Modelo%20entidad-relación.md))

**Modelo conceptual:** entidades del negocio y sus vínculos, sin tecnología. ([01](01-Proceso%20de%20diseño.md))

**Modelo físico:** implementación en un motor concreto: tipos, índices, esquemas, opciones de almacenamiento. ([01](01-Proceso%20de%20diseño.md))

**Modelo lógico:** tablas, columnas, claves y relaciones, independiente del motor. ([01](01-Proceso%20de%20diseño.md))

**Normalización:** proceso de dividir tablas para que cada hecho se guarde en un solo lugar. ([06](06-Normalización%201FN%202FN%203FN.md))

**Notación pata de gallo (crow's foot):** notación de diagramas ER que dibuja la cardinalidad con símbolos en los extremos de cada línea. ([02](02-Modelo%20entidad-relación.md))

**PascalCase / snake_case:** `DetallePedido` frente a `detalle_pedido`; el primero es habitual en SQL Server, el segundo en PostgreSQL y MySQL. ([10](10-Convenciones%20de%20nombres.md))

**Patrón de acceso:** cómo se leen y escriben los datos (qué consultas, con qué frecuencia). ([01](01-Proceso%20de%20diseño.md))

**Palabra reservada:** palabra que usa SQL (`ORDER`, `USER`) y que no conviene usar como nombre. ([10](10-Convenciones%20de%20nombres.md))

**Regla de negocio:** afirmación sobre el negocio que los datos deben respetar. ([01](01-Proceso%20de%20diseño.md))

**Relación (vínculo, relationship):** en el modelo ER, asociación entre entidades ("un cliente realiza pedidos"). ([02](02-Modelo%20entidad-relación.md))

**Relación recursiva (reflexiva):** relación de una entidad consigo misma (empleado y su jefe). ([03](03-Cardinalidad%20y%20relaciones.md))

**Relación ternaria:** relación entre tres entidades a la vez. ([04](04-Del%20modelo%20ER%20a%20tablas.md))

**Snapshot histórica:** copia de un dato tal como era en el momento de un hecho, que no debe cambiar después (el precio pagado). ([08](08-Desnormalización.md))

**Supertipo / subtipo:** entidad general y sus especializaciones (Cliente; ClientePersona y ClienteEmpresa). ([09](09-Patrones%20de%20modelado.md))

**Tabla de catálogo (lookup):** tabla con los valores válidos de un dominio, referenciada con una clave foránea. ([09](09-Patrones%20de%20modelado.md))

**Tabla intermedia (asociativa, de unión):** tabla con dos claves foráneas que implementa una relación N:M. ([03](03-Cardinalidad%20y%20relaciones.md))

**Tabla temporal (system-versioned):** tabla de SQL Server que guarda automáticamente el historial de cada fila. ([09](09-Patrones%20de%20modelado.md))

**TPH / TPT / TPC:** estrategias de herencia de EF Core: una tabla para la jerarquía, una tabla por tipo, una tabla por tipo concreto. ([09](09-Patrones%20de%20modelado.md))

**Vista indexada / vista materializada:** vista cuyo resultado se guarda físicamente; en SQL Server se mantiene sincrónicamente, en PostgreSQL con `REFRESH`. ([08](08-Desnormalización.md))
