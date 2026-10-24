# El modelo relacional

## En una frase

El modelo relacional organiza los datos en **relaciones** (tablas) formadas por **tuplas** (filas) y **atributos** (columnas), donde cada fila se identifica por una **clave** y todos los datos se consultan con operaciones que reciben tablas y devuelven tablas.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un SGBD y qué es SQL, de [Qué es una base de datos y un SGBD](01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md).
* La idea matemática de **conjunto**: una colección de elementos sin orden y sin repetidos.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Relación:** en el modelo formal, una tabla. **No** significa "vínculo entre tablas".
* **Tupla:** una fila de la relación.
* **Atributo:** una columna, con nombre y dominio.
* **Dominio:** el conjunto de valores válidos para un atributo (por ejemplo, enteros positivos o emails).
* **Grado:** cantidad de atributos (columnas) de una relación.
* **Cardinalidad de una relación:** cantidad de tuplas (filas). Otra acepción de cardinalidad aparece en [Cardinalidad y relaciones](../02-diseno-y-modelado/03-Cardinalidad%20y%20relaciones.md).
* **Superclave, clave candidata, clave primaria, clave alternativa:** conjuntos de atributos que identifican filas (se explican abajo).

-----

## El problema

Antes de 1970, las bases de datos eran **jerárquicas** o **en red**: los datos se guardaban como árboles o grafos de registros enlazados por punteros. Para encontrar los pedidos de un cliente, el programa tenía que **navegar** los punteros en un orden concreto:

```text
Cliente "Ana" ──puntero──► Pedido 1 ──puntero──► Pedido 3 ──► fin
```

Eso tenía dos consecuencias graves:

* Cada consulta nueva exigía escribir un programa que conociera la estructura física.
* Cambiar la estructura (por ejemplo, agregar un nivel) rompía los programas existentes.

En 1970, Edgar F. Codd (IBM) propuso separar **qué datos hay** de **cómo están guardados**. Los datos se describen como relaciones matemáticas; el usuario pide lo que quiere y el sistema decide cómo encontrarlo. Esa separación, llamada **independencia de datos**, es la base de SQL Server, PostgreSQL, MySQL y Oracle.

-----

## Cómo funciona

### Vocabulario: tres formas de decir lo mismo

| Modelo formal | SQL | Archivos (lenguaje antiguo) |
| --- | --- | --- |
| Relación | Tabla | Archivo |
| Tupla | Fila (*row*) | Registro |
| Atributo | Columna | Campo |
| Dominio | Tipo de dato + restricciones | Formato |
| Grado | Cantidad de columnas | — |
| Cardinalidad | Cantidad de filas | — |

En el día a día se dice "tabla", "fila" y "columna". El vocabulario formal aparece en libros, en la universidad y en algunas entrevistas.

### "Relación" no significa "relación entre tablas"

Esta es la confusión número uno. El modelo se llama **relacional** porque cada tabla es una **relación** en el sentido matemático: un conjunto de tuplas. **No** se llama así porque las tablas "se relacionan" entre sí.

El vínculo entre tablas (un cliente tiene pedidos) se llama **relación** en español coloquial y *relationship* en inglés, y se implementa con [claves foráneas](05-Claves%20foráneas%20e%20integridad%20referencial.md). Son dos conceptos distintos con nombres parecidos.

### Propiedades de una relación

Una relación, en el modelo formal, cumple cuatro propiedades:

| Propiedad | Qué significa | Consecuencia práctica |
| --- | --- | --- |
| **No hay filas duplicadas** | Es un conjunto: cada tupla aparece una vez | Toda tabla necesita algo que identifique cada fila: una clave |
| **Las filas no tienen orden** | Un conjunto no tiene "primera" ni "última" fila | Sin `ORDER BY`, el orden del resultado no está garantizado |
| **Las columnas no tienen orden** | Se identifican por nombre, no por posición | `SELECT *` y `INSERT` sin lista de columnas son frágiles |
| **Cada valor es atómico** | Una celda contiene un solo valor de su dominio | Nada de listas en una columna (base de la [primera forma normal](../02-diseno-y-modelado/06-Normalización%201FN%202FN%203FN.md)) |

### Las claves

Una **superclave** es cualquier conjunto de columnas cuyos valores no se repiten en dos filas. En la tabla `Cliente`:

* `{ClienteId}` es superclave.
* `{Email}` es superclave.
* `{ClienteId, Nombre}` también lo es, pero le sobra `Nombre`.

Una **clave candidata** es una superclave **mínima**: si le quitas cualquier columna, deja de identificar. `{ClienteId}` y `{Email}` son candidatas; `{ClienteId, Nombre}` no, porque `Nombre` sobra.

De las candidatas se elige una como **clave primaria** (`ClienteId`), y las demás quedan como **claves alternativas** (`Email`), que se protegen con `UNIQUE`.

```text
Superclaves
└── Claves candidatas (mínimas)
    ├── Clave primaria     → la elegida: ClienteId
    └── Claves alternativas → las demás:  Email  (UNIQUE)
```

Cómo elegir la primaria se estudia en [Claves primarias](04-Claves%20primarias.md).

### Operaciones: de tablas a tablas

El modelo define operaciones (el **álgebra relacional**) que reciben relaciones y devuelven relaciones. Las principales tienen equivalente directo en SQL:

| Operación | Qué hace | En SQL |
| --- | --- | --- |
| Selección | Filtra filas | `WHERE` |
| Proyección | Elige columnas | La lista del `SELECT` |
| Producto cartesiano | Combina cada fila con cada fila | `CROSS JOIN` |
| Join (reunión) | Combina filas relacionadas | `JOIN ... ON` |
| Unión, intersección, diferencia | Operaciones de conjuntos | `UNION`, `INTERSECT`, `EXCEPT` |

Como el resultado de una operación es otra relación, las operaciones se pueden **componer**: el resultado de una consulta puede ser la entrada de otra. Esa propiedad (**cierre**) es la que hace posibles las subconsultas y las vistas.

### Donde SQL se aparta de la teoría

SQL se inspira en el modelo relacional, pero no lo cumple al pie de la letra:

* **Permite filas duplicadas.** Una tabla sin clave puede tener dos filas idénticas, y `SELECT` devuelve duplicados salvo que uses `DISTINCT`. Técnicamente, SQL trabaja con multiconjuntos (*bags*), no con conjuntos.
* **Tiene `NULL`.** El modelo original no contemplaba valores ausentes; SQL los añadió junto con una lógica de tres valores (ver [NULL y la lógica de tres valores](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md)).
* **Las columnas tienen posición.** `SELECT *` las devuelve en el orden en que se crearon.

Que SQL lo permita no significa que sea buena idea: diseñar como si el modelo se cumpliera (toda tabla con clave, sin listas en las celdas) evita la mayoría de los problemas.

-----

## Ejemplo completo

La tabla `Cliente` del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md), analizada con el vocabulario del modelo:

| ClienteId | Nombre | Email | Ciudad | FechaRegistro |
| --- | --- | --- | --- | --- |
| 1 | Ana Torres | ana.torres@correo.com | Lima | 2026-01-10 |
| 2 | Luis Ramos | luis.ramos@correo.com | Arequipa | 2026-02-03 |
| 3 | María Quispe | maria.quispe@correo.com | `NULL` | 2026-02-20 |
| 4 | Jorge Díaz | jorge.diaz@correo.com | Lima | 2026-03-15 |
| 5 | Sofía Mendoza | sofia.mendoza@correo.com | Cusco | 2026-04-01 |

* **Grado:** 5 (cinco atributos).
* **Cardinalidad:** 5 (cinco tuplas).
* **Dominios:** `ClienteId` enteros positivos, `Email` textos con formato de email de hasta 254 caracteres, `FechaRegistro` fechas.
* **Claves candidatas:** `{ClienteId}` y `{Email}`.
* **Clave primaria:** `ClienteId`. **Clave alternativa:** `Email` (`UNIQUE`).
* `{Nombre}` **no** es clave: dos personas pueden llamarse igual. `{Ciudad}` tampoco: se repite.

Una consulta con selección y proyección, y su resultado (otra relación):

```sql
SELECT Nombre, Email          -- proyección
FROM dbo.Cliente
WHERE Ciudad = N'Lima';       -- selección
```

| Nombre | Email |
| --- | --- |
| Ana Torres | ana.torres@correo.com |
| Jorge Díaz | jorge.diaz@correo.com |

El resultado tiene grado 2 y cardinalidad 2, y se podría usar como entrada de otra consulta.

-----

## Errores comunes

**1. Asumir que las filas salen en un orden.**
Qué pasa: un `SELECT` sin `ORDER BY` devuelve las filas "en orden" durante meses y un día cambia (tras un índice nuevo, un plan paralelo o más datos).
Por qué: una relación no tiene orden; el motor devuelve las filas como le resulte más barato.
Arreglo: si el orden importa, `ORDER BY` siempre.

**2. Tablas sin clave.**
Qué pasa: filas duplicadas imposibles de distinguir; un `UPDATE` o `DELETE` afecta a más filas de las que querías.
Por qué: sin clave no hay forma de identificar una fila concreta.
Arreglo: toda tabla con clave primaria.

**3. Guardar listas dentro de una celda.**
Qué pasa: `Telefonos = '999111222, 988333444'`. Buscar un teléfono exige `LIKE '%...%'`, no se puede validar cada número y no se puede poner índice.
Por qué: se rompe la atomicidad de los valores.
Arreglo: una tabla aparte con una fila por teléfono (ver [Normalización](../02-diseno-y-modelado/06-Normalización%201FN%202FN%203FN.md)).

**4. Confundir "relación" con "vínculo entre tablas".**
Qué pasa: en una entrevista, decir que la base es "relacional porque las tablas se relacionan".
Por qué: el nombre viene de la relación matemática (un conjunto de tuplas).
Arreglo: relación = tabla; el vínculo entre tablas se implementa con claves foráneas.

-----

## Según el motor

Los tres motores relacionales implementan el mismo modelo, con las mismas desviaciones de SQL frente a la teoría:

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Permite tablas sin clave primaria | Sí | Sí | Sí, aunque InnoDB crea una clave interna oculta |
| Orden sin `ORDER BY` | No garantizado | No garantizado | No garantizado (aunque InnoDB suele devolver por clave primaria) |
| `INTERSECT` / `EXCEPT` | Sí | Sí | Sí, desde MySQL 8.0.31 |
| Columnas con valores compuestos | No nativas (JSON en texto o tipo `json`) | Arreglos y tipos compuestos nativos | JSON |

PostgreSQL es el que más se aparta del valor atómico: permite columnas de tipo arreglo (`text[]`). Son útiles en casos puntuales, pero no reemplazan una tabla relacionada cuando necesitas integridad o consultas sobre cada elemento.

### En MongoDB

MongoDB **no** sigue el modelo relacional. Una colección no es una relación: sus documentos pueden tener campos distintos y contener objetos y arreglos anidados, es decir, rompe a propósito la atomicidad de los valores. Lo que sí comparte es la identificación: todo documento tiene un `_id` único, el equivalente a la clave primaria.

-----

## Cuándo sí y cuándo no

**Piensa en relaciones cuando:**

* Diseñas cualquier tabla: preguntarte "¿cuál es la clave?" y "¿cada celda tiene un solo valor?" evita la mayoría de los problemas de diseño.
* Escribes consultas: pensar en conjuntos ("todos los pedidos que cumplen X") en lugar de en bucles ("para cada pedido...") lleva a SQL más simple y más rápido.

**Ten en cuenta:**

* SQL permite cosas que el modelo prohíbe (duplicados, tablas sin clave). Que se pueda no significa que convenga.
* El vocabulario formal es útil para leer bibliografía; en el trabajo diario se habla de tablas, filas y columnas.

-----

## Resumen en 5 líneas

1. El modelo relacional (Codd, 1970) separa qué datos hay de cómo se guardan: eso es la independencia de datos.
2. Relación = tabla, tupla = fila, atributo = columna, dominio = conjunto de valores válidos.
3. Una relación no tiene filas duplicadas ni orden, y cada valor es atómico.
4. Superclave → clave candidata (mínima) → clave primaria (la elegida) y alternativas (`UNIQUE`).
5. Las operaciones reciben y devuelven tablas, por eso se pueden componer; SQL se aparta de la teoría con duplicados y `NULL`.

-----

## Para profundizar

<details>
<summary>Álgebra relacional con notación formal</summary>

En los libros verás las operaciones con letras griegas:

* **σ** (sigma), selección: `σ Ciudad='Lima' (Cliente)` es `WHERE Ciudad = 'Lima'`.
* **π** (pi), proyección: `π Nombre, Email (Cliente)` es `SELECT Nombre, Email`.
* **⋈** (moño), join natural: `Cliente ⋈ Pedido` une por las columnas con el mismo nombre.

La consulta del ejemplo completo se escribe `π Nombre, Email (σ Ciudad='Lima' (Cliente))`. Leerlo de adentro hacia afuera muestra el orden lógico: primero se filtra y después se proyecta.

</details>

<details>
<summary>Las reglas de Codd</summary>

En 1985 Codd publicó 13 reglas (numeradas del 0 al 12) para decidir si un SGBD era "realmente relacional". Entre ellas: todos los datos se representan como valores en tablas (regla 1), cada valor es accesible por tabla, clave y columna (regla 2), el tratamiento sistemático de valores nulos (regla 3) y la independencia física y lógica de los datos (reglas 8 y 9). Ningún SGBD comercial las cumple todas; sirven como marco para entender qué prometía el modelo.

</details>

<details>
<summary>Lecturas recomendadas</summary>

* *A Relational Model of Data for Large Shared Data Banks*, E. F. Codd (1970): el artículo original, corto y legible.
* *SQL and Relational Theory*, C. J. Date: cómo escribir SQL respetando el modelo.
* *Fundamentos de sistemas de bases de datos*, Elmasri y Navathe: el libro de texto clásico.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El modelo relacional organiza los datos en tablas con filas y columnas. Cada fila se identifica con una clave primaria, y las tablas se vinculan con claves foráneas. Se consulta con SQL, describiendo qué datos quieres y no cómo encontrarlos.

### Respuesta ampliada (semi-senior)

El modelo, propuesto por Codd en 1970, representa los datos como relaciones matemáticas: conjuntos de tuplas sobre atributos con dominios definidos. De ahí salen sus propiedades: no hay tuplas duplicadas (por eso toda relación tiene clave), no hay orden de filas (por eso sin `ORDER BY` no hay orden garantizado) y los valores son atómicos (la base de la primera forma normal). Distingo superclave, clave candidata, primaria y alternativa. Las operaciones del álgebra relacional son cerradas, así que se pueden componer, y eso es lo que permite subconsultas y vistas. Además, el modelo da independencia de datos: el diseño lógico no depende del almacenamiento físico. SQL se aparta del modelo al permitir duplicados y `NULL`, y conviene diseñar como si esas licencias no existieran.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué se llama "relacional"?**
Porque cada tabla es una relación matemática (un conjunto de tuplas), no porque las tablas se relacionen entre sí.

**2. ¿Qué diferencia hay entre una clave candidata y una superclave?**
La candidata es una superclave mínima: si le quitas cualquier columna, deja de identificar las filas de forma única.

**3. ¿Está garantizado el orden de un `SELECT` sin `ORDER BY`?**
No. Las relaciones no tienen orden, y el motor devuelve las filas según el plan que elija.

**4. ¿Qué es la independencia de datos?**
Poder cambiar cómo se guardan los datos (índices, archivos, particiones) sin cambiar las consultas ni las aplicaciones que los usan.

-----

## Práctica

**Ejercicio 1.** Dada esta tabla de `Producto` (resumida), indica su grado, su cardinalidad y sus claves candidatas, suponiendo que no puede haber dos productos con el mismo nombre en la misma categoría:

| ProductoId | CategoriaId | Nombre | Precio |
| --- | --- | --- | --- |
| 1 | 1 | Teclado mecánico | 120.00 |
| 2 | 1 | Mouse inalámbrico | 45.50 |
| 4 | 2 | Lámpara de escritorio | 75.00 |
| 6 | 3 | Clean Code | 95.00 |

<details>
<summary>Solución</summary>

* **Grado:** 4. **Cardinalidad:** 4.
* **Claves candidatas:** `{ProductoId}` y `{CategoriaId, Nombre}`.
* `{Nombre}` sola **no** es candidata según la regla: podría existir un "Clean Code" en dos categorías distintas. Que hoy no se repita en los datos no la convierte en clave: las claves salen de las **reglas de negocio**, no de los datos de un momento dado.
* `{Precio}` no es clave: dos productos pueden costar lo mismo.

La clave alternativa se protegería con `CONSTRAINT UQ_Producto_Categoria_Nombre UNIQUE (CategoriaId, Nombre)`.

</details>

**Ejercicio 2.** Escribe la consulta que corresponde a `π Nombre (σ Precio > 100 (Producto))` y su resultado con los datos del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md).

<details>
<summary>Solución</summary>

```sql
SELECT Nombre
FROM dbo.Producto
WHERE Precio > 100;
```

| Nombre |
| --- |
| Teclado mecánico |
| Monitor 27 pulgadas |
| Cafetera |

El orden puede variar, porque la consulta no tiene `ORDER BY`.

</details>

-----

## Siguiente lección

[Tipos de datos](03-Tipos%20de%20datos.md)
