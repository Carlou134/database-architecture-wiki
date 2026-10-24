# Claves primarias

## En una frase

La clave primaria es la columna (o el conjunto de columnas) que identifica cada fila de forma **única, obligatoria y estable**; elegirla bien significa decidir entre una clave **natural** que viene del negocio y una **sustituta** que genera la base, y casi siempre gana la sustituta.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué son las claves candidatas y alternativas, de [El modelo relacional](02-El%20modelo%20relacional.md).
* Los tipos `INT`, `BIGINT` y `UNIQUEIDENTIFIER`, de [Tipos de datos](03-Tipos%20de%20datos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Clave primaria (PK):** la clave candidata elegida para identificar las filas de una tabla.
* **Clave natural:** una clave que existe en el mundo real (DNI, email, código ISO).
* **Clave sustituta (surrogate):** un identificador sin significado de negocio, generado por la base (`1`, `2`, `3`...).
* **Clave compuesta:** clave formada por más de una columna.
* **`IDENTITY`:** propiedad de una columna de SQL Server que genera números consecutivos automáticamente.
* **GUID / UUID:** identificador de 128 bits, prácticamente imposible de repetir.
* **Índice agrupado (clustered):** el índice que define el orden físico en que se guardan las filas de la tabla.

-----

## El problema

Una tabla de clientes usa el DNI como clave primaria. Parece perfecto: es único y todos lo tienen. Hasta que:

* Llega un cliente extranjero con pasaporte y sin DNI. ¿Qué va en la clave?
* Alguien registró mal un DNI. Corregirlo exige actualizar la clave en `Cliente` **y** en cada `Pedido`, `Factura` y `Pago` que lo referencia.
* El DNI aparece en las URLs (`/clientes/45871236`), en los logs y en los correos de soporte. Es un dato personal filtrado por todas partes.
* Una empresa compra a otra y hay que unir dos bases con clientes que se repiten.

Ninguno de estos problemas es raro. La clave primaria se copia en cada tabla que se relaciona con ella, así que **cualquier cambio en su valor se propaga por toda la base**. Por eso necesita una propiedad que los datos de negocio casi nunca tienen: no cambiar jamás.

-----

## Cómo funciona

### Las cuatro propiedades de una buena clave primaria

| Propiedad | Qué significa | ¿El DNI la cumple? |
| --- | --- | --- |
| **Única** | No se repite en dos filas | Sí |
| **Obligatoria** | Nunca es `NULL` | No: hay clientes sin DNI |
| **Estable** | No cambia en la vida de la fila | No del todo: hay errores de carga y correcciones |
| **Mínima** | No tiene columnas de más | Sí |

El motor hace cumplir las dos primeras: una `PRIMARY KEY` implica `UNIQUE` y `NOT NULL`. Las otras dos dependen de tu diseño.

### Natural frente a sustituta

| | Clave natural | Clave sustituta |
| --- | --- | --- |
| Ejemplo | `Email`, `DNI`, `CodigoPais` | `ClienteId INT IDENTITY` |
| Significado | Tiene significado de negocio | No significa nada |
| Estabilidad | Cambia si cambia el negocio | Nunca cambia |
| Tamaño | Variable (un email ocupa hasta ~500 bytes) | Fijo y pequeño (4 u 8 bytes) |
| Evita duplicados de negocio | Sí, por sí sola | **No**: hace falta un `UNIQUE` aparte |

La práctica habitual: **clave sustituta como primaria + la clave natural como `UNIQUE`**. Así tienes un identificador estable y pequeño para las relaciones, y sigues impidiendo dos clientes con el mismo email:

```sql
CREATE TABLE dbo.Cliente (
    ClienteId INT IDENTITY(1,1) NOT NULL,
    Email     NVARCHAR(254)     NOT NULL,
    Nombre    NVARCHAR(100)     NOT NULL,
    CONSTRAINT PK_Cliente PRIMARY KEY (ClienteId),
    CONSTRAINT UQ_Cliente_Email UNIQUE (Email)
);
```

Si olvidas el `UNIQUE`, la clave sustituta te deja insertar al mismo cliente diez veces con IDs distintos. Una clave sustituta **sin** su clave natural protegida es la causa más frecuente de duplicados.

La clave natural sí es buena primaria cuando es corta, estable por definición y controlada por un estándar: `CodigoPais CHAR(2)` (ISO 3166), `CodigoMoneda CHAR(3)` (ISO 4217).

### `IDENTITY`: números generados por la base

```sql
ClienteId INT IDENTITY(1, 1)   -- empieza en 1, incrementa de a 1
```

Al insertar no se envía el valor; la base lo asigna:

```sql
INSERT INTO dbo.Cliente (Email, Nombre) VALUES (N'ana.torres@correo.com', N'Ana Torres');
```

Para recuperar el ID generado, la forma más segura es `OUTPUT`:

```sql
INSERT INTO dbo.Cliente (Email, Nombre)
OUTPUT inserted.ClienteId
VALUES (N'luis.ramos@correo.com', N'Luis Ramos');
```

| ClienteId |
| --- |
| 2 |

Otras opciones y sus diferencias:

| Función | Devuelve | Riesgo |
| --- | --- | --- |
| `OUTPUT inserted.Col` | Los IDs de todas las filas insertadas | Ninguno; funciona con inserciones de varias filas |
| `SCOPE_IDENTITY()` | El último ID generado en tu ámbito (tu lote o procedimiento) | Solo devuelve uno |
| `@@IDENTITY` | El último ID generado en tu sesión, **en cualquier tabla** | Si un *trigger* inserta en otra tabla, devuelve el ID equivocado |

### `IDENTITY` tiene huecos, y está bien

Los valores de `IDENTITY` **no** son consecutivos garantizados:

* Una transacción que inserta y hace `ROLLBACK` consume el número igual.
* Tras un reinicio del servidor, SQL Server puede saltar valores (guarda en caché un bloque de números; desde SQL Server 2017 se puede desactivar con `ALTER DATABASE SCOPED CONFIGURATION SET IDENTITY_CACHE = OFF`, a costa de rendimiento).
* Borrar filas deja huecos.

Una clave sustituta solo tiene que ser **única**. Si el negocio necesita números correlativos sin huecos (como la numeración de facturas exigida por ley), eso es otra columna con su propia lógica, no la clave primaria.

### `SEQUENCE`: el generador independiente

Una `SEQUENCE` (desde SQL Server 2012) genera números fuera de una tabla. Sirve para compartir una numeración entre varias tablas o para conocer el ID antes de insertar:

```sql
CREATE SEQUENCE dbo.SecuenciaDocumento AS INT START WITH 1 INCREMENT BY 1;

DECLARE @id INT = NEXT VALUE FOR dbo.SecuenciaDocumento;
```

### GUID como clave primaria

Un `UNIQUEIDENTIFIER` (GUID) se puede generar en cualquier lugar (la aplicación, otro servidor, un dispositivo sin conexión) sin coordinarse con la base, y no revela cuántos registros hay. Pero tiene costos:

* **Tamaño:** 16 bytes frente a 4 de un `INT`, y se copia en cada clave foránea y en cada índice.
* **Fragmentación:** `NEWID()` genera valores aleatorios. Si la clave es el índice agrupado, cada inserción cae en un lugar aleatorio de la tabla y parte páginas que ya estaban llenas.

`NEWSEQUENTIALID()` genera GUIDs crecientes en un mismo servidor y reduce mucho la fragmentación, pero solo se puede usar como `DEFAULT` de una columna:

```sql
PedidoId UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_Pedido_PedidoId DEFAULT NEWSEQUENTIALID()
```

| | `INT` / `BIGINT IDENTITY` | `UNIQUEIDENTIFIER` |
| --- | --- | --- |
| Tamaño | 4 / 8 bytes | 16 bytes |
| Generado | Por la base, al insertar | En cualquier lugar |
| Orden de inserción | Creciente: inserta al final | Aleatorio con `NEWID()`, creciente con `NEWSEQUENTIALID()` |
| Expone volumen y es adivinable | Sí (`/pedidos/1001` → existe el 1002) | No |
| Legible para depurar | Sí | No |

Para la mayoría de las tablas OLTP, `INT` o `BIGINT IDENTITY` es la opción por defecto. Los GUID se justifican en sistemas distribuidos, en replicación con varios orígenes o cuando el ID se genera fuera de la base.

Si lo que te preocupa es exponer IDs adivinables en una API, otra opción es mantener la clave `INT` interna y exponer una columna aparte (`CodigoPublico UNIQUEIDENTIFIER UNIQUE`).

### Claves compuestas

Una clave compuesta tiene más de una columna. Es la elección natural en las **tablas intermedias**, donde la combinación es lo que identifica a la fila:

```sql
CREATE TABLE dbo.DetallePedido (
    PedidoId       INT           NOT NULL,
    ProductoId     INT           NOT NULL,
    Cantidad       INT           NOT NULL,
    PrecioUnitario DECIMAL(10,2) NOT NULL,
    CONSTRAINT PK_DetallePedido PRIMARY KEY (PedidoId, ProductoId)
);
```

`(PedidoId, ProductoId)` impide que el mismo producto aparezca dos veces en un pedido, que es una regla de negocio. El orden de las columnas importa para los índices: con `PedidoId` primero, buscar "los detalles del pedido 3" es eficiente; buscar "en qué pedidos aparece el producto 6" necesita otro índice.

En las tablas de entidades (clientes, productos), las claves compuestas se vuelven incómodas: cada tabla que las referencia tiene que copiar todas sus columnas. Ahí conviene una sustituta.

### La clave primaria y el índice agrupado

En SQL Server, `PRIMARY KEY` crea por defecto un **índice agrupado**: las filas se guardan físicamente ordenadas por la clave. Por eso una clave creciente (`IDENTITY`) es tan eficiente para insertar: cada fila nueva va al final.

Son dos conceptos separables: la clave primaria es una regla **lógica** (unicidad) y el índice agrupado es una decisión **física** (orden de almacenamiento). Se puede tener la primaria como `NONCLUSTERED` y agrupar por otra columna. Se estudia en el módulo de índices.

-----

## Ejemplo completo

Elegir la clave primaria de tres tablas de la tienda:

```sql
-- 1. Entidad con clave natural inestable → sustituta + UNIQUE
CREATE TABLE dbo.Cliente (
    ClienteId INT IDENTITY(1,1) NOT NULL,
    Email     NVARCHAR(254)     NOT NULL,
    Nombre    NVARCHAR(100)     NOT NULL,
    CONSTRAINT PK_Cliente PRIMARY KEY (ClienteId),
    CONSTRAINT UQ_Cliente_Email UNIQUE (Email)
);

-- 2. Catálogo con estándar estable → natural
CREATE TABLE dbo.Pais (
    CodigoPais CHAR(2)      NOT NULL,   -- ISO 3166-1 alfa-2
    Nombre     NVARCHAR(60) NOT NULL,
    CONSTRAINT PK_Pais PRIMARY KEY (CodigoPais)
);

-- 3. Tabla intermedia → compuesta
CREATE TABLE dbo.DetallePedido (
    PedidoId   INT NOT NULL,
    ProductoId INT NOT NULL,
    Cantidad   INT NOT NULL,
    CONSTRAINT PK_DetallePedido PRIMARY KEY (PedidoId, ProductoId)
);

-- Probar las reglas
INSERT INTO dbo.Cliente (Email, Nombre) VALUES (N'ana.torres@correo.com', N'Ana Torres');
INSERT INTO dbo.Cliente (Email, Nombre) VALUES (N'ana.torres@correo.com', N'Ana T.');
```

La segunda inserción falla, y no por la clave primaria (el `IDENTITY` habría generado un 2), sino por la clave alternativa:

```text
Msg 2627, Level 14, State 1
Violation of UNIQUE KEY constraint 'UQ_Cliente_Email'. Cannot insert duplicate key in object 'dbo.Cliente'.
The duplicate key value is (ana.torres@correo.com).
```

Qué decisión se tomó en cada tabla y por qué:

* **`Cliente`:** el email puede cambiar (la persona cambia de proveedor), así que se usa una sustituta estable y el email queda protegido como `UNIQUE`.
* **`Pais`:** el código ISO es corto, estable y controlado por un estándar: la clave natural funciona.
* **`DetallePedido`:** la combinación pedido-producto es la identidad de la fila y, además, hace cumplir una regla.

-----

## Errores comunes

**1. Clave sustituta sin la natural protegida.**
Qué pasa: el mismo cliente registrado tres veces con IDs 7, 19 y 32.
Por qué: `IDENTITY` solo garantiza que el **ID** no se repite, no que el **cliente** no se repita.
Arreglo: `UNIQUE` sobre la clave natural (email, documento, código).

**2. Usar datos de negocio cambiantes como clave primaria.**
Qué pasa: corregir un DNI o un email exige actualizarlo en cascada en muchas tablas.
Por qué: la clave primaria se replica en cada clave foránea.
Arreglo: sustituta como primaria; el dato de negocio como `UNIQUE`.

**3. Esperar que `IDENTITY` no tenga huecos.**
Qué pasa: alguien reporta "se perdieron los pedidos 1005 a 1010".
Por qué: los `ROLLBACK`, los borrados y los reinicios consumen valores.
Arreglo: no le des significado a la clave sustituta. Si necesitas una numeración correlativa legal, es otra columna.

**4. Usar `@@IDENTITY` para leer el ID generado.**
Qué pasa: con un *trigger* de auditoría que inserta en otra tabla, obtienes el ID de la tabla de auditoría.
Por qué: `@@IDENTITY` no se limita a tu ámbito.
Arreglo: `OUTPUT inserted.Id` o `SCOPE_IDENTITY()`.

**5. GUID aleatorio como índice agrupado en una tabla con muchas inserciones.**
Qué pasa: las inserciones se vuelven lentas y los índices se fragmentan.
Por qué: cada fila nueva cae en una posición aleatoria y parte páginas llenas.
Arreglo: `NEWSEQUENTIALID()`, una clave `BIGINT IDENTITY` agrupada con el GUID como columna `UNIQUE`, o la primaria GUID declarada `NONCLUSTERED`.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Autoincremento | `INT IDENTITY(1,1)` | `integer GENERATED ALWAYS AS IDENTITY` (desde la 10) | `INT AUTO_INCREMENT` |
| Forma heredada | — | `SERIAL` / `BIGSERIAL` (todavía muy común) | — |
| Secuencias | `CREATE SEQUENCE` | `CREATE SEQUENCE` (lo que usa `IDENTITY` por dentro) | No tiene objetos secuencia |
| Recuperar el ID | `OUTPUT inserted.Id` | `RETURNING id` | `LAST_INSERT_ID()` |
| Tipo UUID | `UNIQUEIDENTIFIER` | `uuid` | `BINARY(16)` o `CHAR(36)` |
| Generar UUID | `NEWID()`, `NEWSEQUENTIALID()` | `gen_random_uuid()` (desde la 13); `uuidv7()` ordenado por tiempo (desde la 18) | `UUID()` (versión 1); `UUID_TO_BIN(UUID(), 1)` reordena los bytes para que sea creciente |
| ¿La PK es el índice agrupado? | Por defecto sí, pero es configurable | No hay índice agrupado: la tabla es un *heap* y la PK es un índice aparte | Siempre: InnoDB guarda las filas ordenadas por la PK |

En **MySQL InnoDB** la elección es todavía más importante: si la tabla no tiene clave primaria, InnoDB usa la primera columna `UNIQUE NOT NULL` o crea una clave oculta de 6 bytes. Una PK aleatoria fragmenta la tabla entera, porque la tabla **es** el índice.

### En MongoDB

Todo documento tiene un campo `_id`, que es su clave primaria: único, obligatorio e inmutable (no se puede cambiar después de insertar). Si no lo envías, el driver genera un `ObjectId` de 12 bytes: 4 de marca de tiempo, 5 aleatorios y 3 de contador. Por eso los `ObjectId` crecen aproximadamente en el tiempo y se puede extraer de ellos la fecha de creación. Se puede usar cualquier valor como `_id` (un email, un número), con el mismo análisis natural frente a sustituta de esta lección.

-----

## Cuándo sí y cuándo no

**Usa una clave sustituta (`INT`/`BIGINT IDENTITY`) cuando:**

* La tabla representa una entidad del negocio (clientes, productos, pedidos).
* La clave natural es larga, puede cambiar o puede faltar.

**Usa una clave natural cuando:**

* Es corta, estable por definición y controlada por un estándar externo (códigos ISO).

**Usa una clave compuesta cuando:**

* La tabla es intermedia y la combinación de columnas es la identidad de la fila.

**Usa GUID cuando:**

* Los IDs se generan fuera de la base, hay varios orígenes de escritura o necesitas IDs no adivinables (y aun así, considera exponer un GUID público y mantener el `INT` interno).

-----

## Resumen en 5 líneas

1. La clave primaria debe ser única, obligatoria, estable y mínima; el motor solo garantiza las dos primeras.
2. Por defecto: clave sustituta (`INT`/`BIGINT IDENTITY`) como primaria + la clave natural como `UNIQUE`.
3. `IDENTITY` tiene huecos y está bien: la clave sustituta no significa nada.
4. Recupera el ID generado con `OUTPUT inserted.Id` o `SCOPE_IDENTITY()`, nunca con `@@IDENTITY`.
5. GUID: generable en cualquier lugar, pero más grande y, si es aleatorio, fragmenta el índice agrupado.

-----

## Para profundizar

<details>
<summary>¿Y los UUID versión 7?</summary>

Un UUIDv7 empieza con una marca de tiempo, así que los valores generados uno tras otro son crecientes. Eso resuelve la fragmentación en motores que ordenan los UUID byte a byte, como PostgreSQL. En .NET 9 o superior se generan con `Guid.CreateVersion7()`.

En SQL Server hay una trampa: `UNIQUEIDENTIFIER` no se ordena byte a byte de izquierda a derecha, sino que compara primero los **últimos 6 bytes**. Un UUIDv7 generado en la aplicación, que es creciente al principio, no queda creciente según el orden de SQL Server. Para obtener inserciones secuenciales en SQL Server, `NEWSEQUENTIALID()` sigue siendo la opción nativa, o hay que generar el GUID con los bytes reordenados.

</details>

<details>
<summary>El debate natural frente a sustituta</summary>

Hay defensores serios de las claves naturales: evitan joins (el código de país ya dice qué país es), hacen cumplir la regla de negocio sin un `UNIQUE` aparte y tienen significado al leer los datos. El argumento a favor de las sustitutas es la estabilidad y el tamaño. En la práctica, la mayoría de los equipos usan sustitutas para las entidades y naturales para los catálogos estándar, y lo importante es que **la clave natural siempre quede protegida**, sea primaria o `UNIQUE`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La clave primaria identifica cada fila de forma única y no puede ser nula. Normalmente uso un entero autoincremental (`IDENTITY`) como clave sustituta, y si hay un dato de negocio único, como el email, le pongo una restricción `UNIQUE`. En las tablas intermedias uso una clave compuesta con las dos claves foráneas.

### Respuesta ampliada (semi-senior)

Una buena clave primaria es única, no nula, estable y mínima. Prefiero una sustituta `INT` o `BIGINT IDENTITY` para las entidades, porque es pequeña, creciente (lo que favorece al índice agrupado, que en SQL Server es la PK por defecto) y nunca cambia, mientras que las claves naturales suelen cambiar o faltar. La clave natural siempre queda como `UNIQUE`, porque una sustituta sola no evita duplicados de negocio. Uso claves naturales para catálogos estándar como los códigos ISO y claves compuestas en las tablas intermedias. Los GUID los reservo para IDs generados fuera de la base o en sistemas distribuidos, cuidando la fragmentación con `NEWSEQUENTIALID()` o con un índice agrupado distinto. No le doy significado a la secuencia: `IDENTITY` tiene huecos, y una numeración legal correlativa es otra columna. Para leer el ID generado uso `OUTPUT` o `SCOPE_IDENTITY()`.

### Preguntas frecuentes de seguimiento

**1. ¿Clave natural o sustituta?**
Sustituta para entidades con claves naturales inestables, más un `UNIQUE` sobre la natural. Natural para catálogos cortos y estables por estándar.

**2. ¿Por qué `IDENTITY` tiene huecos?**
Porque los valores se reservan al insertar y no se devuelven si la transacción hace `ROLLBACK`, porque los borrados no se reutilizan y porque el servidor guarda en caché bloques de valores que se pierden al reiniciar.

**3. ¿Qué diferencia hay entre `SCOPE_IDENTITY()` y `@@IDENTITY`?**
`SCOPE_IDENTITY()` se limita a tu ámbito; `@@IDENTITY` devuelve el último ID de la sesión en cualquier tabla, incluido el que inserte un *trigger*.

**4. ¿Qué problema tiene un GUID como clave primaria?**
Ocupa 16 bytes, que se copian en cada índice y clave foránea, y si es aleatorio (`NEWID()`) fragmenta el índice agrupado, porque cada inserción cae en un lugar aleatorio.

**5. ¿La clave primaria es lo mismo que el índice agrupado?**
No. La clave primaria es una regla lógica de unicidad; el índice agrupado define el orden físico. En SQL Server coinciden por defecto, pero se pueden separar.

-----

## Práctica

**Ejercicio 1.** Elige la clave primaria de cada tabla y di qué otras restricciones `UNIQUE` agregarías:

1. `Empleado`: tiene número de documento, email corporativo y código de empleado que asigna RR. HH. (puede reasignarse si alguien se va y vuelve).
2. `Moneda`: soles, dólares, euros.
3. `AlumnoCurso`: qué alumno está inscrito en qué curso.
4. `EventoDeLog`: millones de eventos por día.

<details>
<summary>Solución</summary>

1. **`EmpleadoId INT IDENTITY`** como primaria. `UNIQUE` sobre el documento (tipo + número, si hay varios tipos) y sobre el email corporativo. El código de RR. HH. puede ser `UNIQUE` solo si nunca se reasigna; como sí se reasigna, no sirve como identidad.
2. **`CodigoMoneda CHAR(3)`** (ISO 4217: `PEN`, `USD`, `EUR`) como clave natural primaria.
3. **Compuesta `(AlumnoId, CursoId)`**: impide inscribir dos veces al mismo alumno. Si el alumno puede repetir el curso en otro periodo, la clave pasa a ser `(AlumnoId, CursoId, PeriodoId)`.
4. **`EventoId BIGINT IDENTITY`**: a millones de filas por día, `INT` se agota en pocos años, y una clave creciente mantiene las inserciones al final del índice.

</details>

**Ejercicio 2.** ¿Qué devuelve la última consulta?

```sql
CREATE TABLE dbo.Prueba (Id INT IDENTITY(1,1) PRIMARY KEY, Texto NVARCHAR(10) NOT NULL);

INSERT INTO dbo.Prueba (Texto) VALUES (N'uno');

BEGIN TRANSACTION;
    INSERT INTO dbo.Prueba (Texto) VALUES (N'dos');
ROLLBACK;

INSERT INTO dbo.Prueba (Texto) VALUES (N'tres');

SELECT Id, Texto FROM dbo.Prueba ORDER BY Id;
```

<details>
<summary>Solución</summary>

| Id | Texto |
| --- | --- |
| 1 | uno |
| 3 | tres |

El `ROLLBACK` deshizo la fila "dos", pero el valor 2 del `IDENTITY` ya se había consumido y no se devuelve. Es el comportamiento esperado: la clave solo garantiza unicidad, no continuidad.

</details>

-----

## Siguiente lección

[Claves foráneas e integridad referencial](05-Claves%20foráneas%20e%20integridad%20referencial.md)
