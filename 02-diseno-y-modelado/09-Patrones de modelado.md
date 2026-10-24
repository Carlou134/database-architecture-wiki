# Patrones de modelado

## En una frase

Muchos problemas de diseño se repiten en casi todos los sistemas (subtipos, auditoría, borrado lógico, catálogos de valores, datos con atributos variables) y tienen soluciones conocidas, con ventajas y costos también conocidos, además de algunos **antipatrones** que conviene reconocer para no caer en ellos.

-----

## Antes de empezar

Conviene que ya sepas:

* Relaciones 1:1, claves foráneas y restricciones, de [Cardinalidad y relaciones](03-Cardinalidad%20y%20relaciones.md) y de [Restricciones](../01-fundamentos/06-Restricciones.md).
* Normalización y desnormalización, de [Normalización](06-Normalización%201FN%202FN%203FN.md) y [Desnormalización](08-Desnormalización.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Supertipo / subtipo:** una entidad general (Persona) y sus especializaciones (Cliente, Empleado).
* **Columna discriminadora:** columna que indica a qué subtipo pertenece cada fila (`TipoCliente`).
* **Borrado lógico (soft delete):** marcar una fila como eliminada en lugar de borrarla.
* **Tabla temporal (system-versioned):** tabla de SQL Server que guarda automáticamente el historial de cada fila.
* **Tabla de catálogo (lookup):** tabla con los valores válidos de un dominio (estados, tipos, países).
* **EAV (entidad-atributo-valor):** diseño que guarda cada atributo como una fila `(entidad, atributo, valor)`. Es un antipatrón.
* **Asociación polimórfica:** una FK que puede apuntar a tablas distintas según otra columna. También es un antipatrón.

-----

## El problema

Mientras la tienda crece, llegan pedidos de cambios que el diseño original no previó:

* "Tenemos clientes personas y clientes empresa. Las empresas tienen RUC y razón social; las personas, DNI."
* "Necesitamos saber quién cambió el precio de un producto y cuándo."
* "No queremos borrar clientes: si se dan de baja, deben poder reactivarse."
* "Cada categoría de producto tiene atributos distintos: las laptops tienen RAM y procesador; los libros, ISBN y autor."
* "Los clientes pueden comentar productos, pedidos y artículos del blog."

Cada uno de estos pedidos tiene una solución "rápida" que funciona la primera semana y se convierte en un problema el primer año. Y cada uno tiene un patrón conocido.

-----

## Cómo funciona

### Patrón 1: herencia (supertipo y subtipos)

Clientes persona y clientes empresa comparten email, ciudad y pedidos, pero cada uno tiene atributos propios. Hay tres formas de modelarlo:

**A. Una sola tabla** (*single table*; en EF Core, **TPH**, *table per hierarchy*):

```sql
CREATE TABLE dbo.Cliente (
    ClienteId   INT IDENTITY(1,1) PRIMARY KEY,
    TipoCliente CHAR(1)       NOT NULL CHECK (TipoCliente IN ('P', 'E')),   -- discriminadora
    Email       NVARCHAR(254) NOT NULL UNIQUE,
    Dni         CHAR(8)       NULL,     -- solo personas
    Nombre      NVARCHAR(100) NULL,     -- solo personas
    Ruc         CHAR(11)      NULL,     -- solo empresas
    RazonSocial NVARCHAR(150) NULL,     -- solo empresas
    CONSTRAINT CK_Cliente_Subtipo CHECK (
        (TipoCliente = 'P' AND Dni IS NOT NULL AND Nombre IS NOT NULL AND Ruc IS NULL AND RazonSocial IS NULL)
     OR (TipoCliente = 'E' AND Ruc IS NOT NULL AND RazonSocial IS NOT NULL AND Dni IS NULL AND Nombre IS NULL)
    )
);
```

**B. Una tabla para el supertipo y una por subtipo** (*class table*; en EF Core, **TPT**, *table per type*):

```sql
CREATE TABLE dbo.Cliente (
    ClienteId   INT IDENTITY(1,1) PRIMARY KEY,
    TipoCliente CHAR(1)       NOT NULL CHECK (TipoCliente IN ('P', 'E')),
    Email       NVARCHAR(254) NOT NULL UNIQUE
);

CREATE TABLE dbo.ClientePersona (
    ClienteId INT           NOT NULL PRIMARY KEY REFERENCES dbo.Cliente (ClienteId),
    Dni       CHAR(8)       NOT NULL UNIQUE,
    Nombre    NVARCHAR(100) NOT NULL
);

CREATE TABLE dbo.ClienteEmpresa (
    ClienteId   INT           NOT NULL PRIMARY KEY REFERENCES dbo.Cliente (ClienteId),
    Ruc         CHAR(11)      NOT NULL UNIQUE,
    RazonSocial NVARCHAR(150) NOT NULL
);
```

**C. Una tabla por subtipo concreto, sin supertipo** (*concrete table*; en EF Core, **TPC**, *table per concrete type*): `ClientePersona` y `ClienteEmpresa`, cada una con todas sus columnas, incluidas las comunes.

| | A. Una tabla (TPH) | B. Supertipo + subtipos (TPT) | C. Tabla por subtipo (TPC) |
| --- | --- | --- | --- |
| Consultas sobre "todos los clientes" | Simples, sin joins | Join con el supertipo | `UNION ALL` de todas las tablas |
| `NOT NULL` en atributos del subtipo | Solo con un `CHECK` complejo | Natural | Natural |
| Columnas vacías | Muchas | Ninguna | Ninguna |
| FK desde `Pedido` hacia "un cliente" | Simple | Simple (al supertipo) | **Imposible** con una sola FK |
| Agregar un subtipo | Más columnas en la misma tabla | Una tabla nueva | Una tabla nueva |
| Cuándo conviene | Pocos atributos propios, consultas sobre el conjunto | Muchos atributos propios, integridad estricta | Subtipos que casi nunca se consultan juntos |

Para la tienda, **B** es la opción equilibrada: `Pedido.ClienteId` apunta a `Cliente`, cada subtipo tiene sus `NOT NULL` y sus `UNIQUE`, y no hay columnas vacías. **A** es razonable si los subtipos difieren en dos o tres columnas.

Una debilidad de B: nada impide que un `ClienteId` aparezca en **las dos** tablas de subtipo, ni que el tipo `'P'` tenga una fila en `ClienteEmpresa`. Se puede blindar con la misma técnica de la FK compuesta de [BCNF](07-BCNF%20y%20formas%20superiores.md): `UNIQUE (ClienteId, TipoCliente)` en el supertipo, y en cada subtipo una columna `TipoCliente` fija con `CHECK` y FK compuesta hacia ese par.

### Patrón 2: auditoría y campos de control

La versión mínima es agregar columnas de control a las tablas importantes:

```sql
CreadoEn      DATETIME2(0)  NOT NULL CONSTRAINT DF_Producto_CreadoEn DEFAULT SYSUTCDATETIME(),
CreadoPor     NVARCHAR(100) NOT NULL,
ModificadoEn  DATETIME2(0)  NULL,
ModificadoPor NVARCHAR(100) NULL,
```

Responde "quién lo creó" y "quién lo tocó por última vez", pero no "qué valores tenía antes". Para el historial completo, SQL Server (desde 2016) tiene **tablas temporales con versionado del sistema**:

```sql
CREATE TABLE dbo.ProductoPrecio (
    ProductoId INT           NOT NULL PRIMARY KEY,
    Precio     DECIMAL(10,2) NOT NULL,
    ValidoDesde DATETIME2(0) GENERATED ALWAYS AS ROW START NOT NULL,
    ValidoHasta DATETIME2(0) GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidoDesde, ValidoHasta)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductoPrecioHistorial));
```

Cada `UPDATE` o `DELETE` mueve la versión anterior a la tabla de historial, automáticamente. Y se puede consultar el pasado:

```sql
SELECT ProductoId, Precio
FROM dbo.ProductoPrecio
FOR SYSTEM_TIME AS OF '2026-03-05T00:00:00';   -- los precios como estaban ese día
```

La tabla temporal guarda **qué** cambió y **cuándo**, pero no **quién**. Para eso, la columna `ModificadoPor` sigue siendo necesaria (y queda también en el historial).

### Patrón 3: borrado lógico (soft delete)

En lugar de `DELETE`, se marca la fila:

```sql
ALTER TABLE dbo.Cliente ADD EliminadoEn DATETIME2(0) NULL;   -- NULL = activo

UPDATE dbo.Cliente SET EliminadoEn = SYSUTCDATETIME() WHERE ClienteId = 5;
```

Ventajas: se conserva el historial, la fila se puede restaurar y las claves foráneas desde `Pedido` siguen siendo válidas. Costos, que hay que asumir de forma explícita:

* **Cada consulta debe filtrar** `WHERE EliminadoEn IS NULL`. Olvidarlo en una sola consulta muestra clientes borrados. (En EF Core se resuelve con un filtro global de consulta, `HasQueryFilter`.)
* **La unicidad se complica:** si se "borra" a Sofía y se registra de nuevo con el mismo email, `UNIQUE (Email)` lo impide. La solución es un índice único filtrado:

  ```sql
  CREATE UNIQUE INDEX UX_Cliente_Email_Activo ON dbo.Cliente (Email) WHERE EliminadoEn IS NULL;
  ```

* **Las tablas crecen** con filas que casi nunca se leen.
* **No sirve para cumplir el "derecho al olvido"** de las leyes de protección de datos: ahí hay que borrar o anonimizar de verdad.

`EliminadoEn DATETIME2 NULL` es preferible a `Eliminado BIT`: dice **cuándo**, y sirve igual para filtrar.

Para la tienda, el producto inactivo (`Activo = 0`) es una variante del mismo patrón con significado de negocio: el producto no se vende, pero sigue existiendo para los pedidos históricos.

### Patrón 4: catálogos de valores (lookup tables) frente a `CHECK`

Los estados de un pedido se pueden restringir de dos formas:

```sql
-- Opción A: CHECK con la lista
CONSTRAINT CK_Pedido_Estado CHECK (Estado IN ('Pendiente', 'Pagado', 'Enviado', 'Entregado', 'Cancelado'))

-- Opción B: tabla de catálogo + FK
CREATE TABLE dbo.EstadoPedido (
    CodigoEstado VARCHAR(20)  NOT NULL PRIMARY KEY,
    Descripcion  NVARCHAR(100) NOT NULL,
    Orden        TINYINT       NOT NULL,
    EsFinal      BIT           NOT NULL
);
-- y en Pedido: Estado VARCHAR(20) NOT NULL REFERENCES dbo.EstadoPedido (CodigoEstado)
```

| | `CHECK` | Tabla de catálogo |
| --- | --- | --- |
| Agregar un valor | `ALTER TABLE` (cambio de esquema) | `INSERT` (cambio de datos) |
| Atributos del valor (descripción, orden, si es final) | No | Sí |
| La aplicación puede listar los valores válidos | Leyendo los metadatos | Con un `SELECT` |
| Simplicidad | Máxima | Una tabla y un join más |

Regla práctica: `CHECK` para listas cortas y estables que solo el equipo técnico cambia; catálogo cuando los valores tienen atributos propios, cambian con el negocio o los administra un usuario. La tienda usa `CHECK` porque los cinco estados son fijos; si aparece "¿qué estados son finales?" o "mostrar los estados en orden", conviene pasar al catálogo.

Con una clave natural legible (`'Pagado'` en lugar de `3`), las consultas se entienden sin el join, y la FK igual protege los valores.

### Antipatrón 1: EAV (entidad-atributo-valor)

Para "cada categoría tiene atributos distintos", la tentación es una tabla genérica:

```sql
CREATE TABLE dbo.ProductoAtributo (
    ProductoId INT           NOT NULL,
    Atributo   NVARCHAR(50)  NOT NULL,   -- 'RAM', 'Procesador', 'ISBN'
    Valor      NVARCHAR(200) NOT NULL,   -- '16 GB', 'i7', '978-0132350884'
    PRIMARY KEY (ProductoId, Atributo)
);
```

Parece flexible. En la práctica:

* **No hay tipos:** `Valor` es texto, así que `'16 GB'`, `'16GB'` y `'dieciséis'` son válidos, y "laptops con más de 8 GB de RAM" es imposible de escribir bien.
* **No hay restricciones:** no se puede exigir que toda laptop tenga RAM, ni validar el formato del ISBN, ni poner una FK sobre un valor.
* **Las consultas son tortuosas:** reconstruir una fila con cinco atributos exige cinco joins (o un `PIVOT`) sobre la misma tabla.
* **Los nombres de los atributos son datos:** un error de tipeo (`'Procesdor'`) crea un atributo nuevo.

Alternativas, de mejor a peor por defecto:

1. **Modelar los subtipos** (patrón 1, opción B): `ProductoLaptop(ProductoId, RamGb, Procesador)`, `ProductoLibro(ProductoId, Isbn, Autor)`. Funciona si las categorías son pocas y conocidas.
2. **Una columna JSON** para los atributos variables, con las columnas comunes como columnas normales. SQL Server permite validar el formato con `ISJSON` en un `CHECK` y consultar con `JSON_VALUE`, e indexar un atributo concreto mediante una columna calculada.
3. **EAV**, solo si los atributos los define el usuario final en tiempo de ejecución y casi nunca se consultan por valor (y aun así, el JSON suele ser mejor).

### Antipatrón 2: asociaciones polimórficas

"Los clientes pueden comentar productos, pedidos y artículos del blog." La tentación:

```sql
CREATE TABLE dbo.Comentario (
    ComentarioId  INT IDENTITY(1,1) PRIMARY KEY,
    EntidadTipo   VARCHAR(20) NOT NULL,   -- 'Producto', 'Pedido', 'Articulo'
    EntidadId     INT         NOT NULL,   -- ¿a qué tabla apunta? depende de EntidadTipo
    Texto         NVARCHAR(1000) NOT NULL
);
```

`EntidadId` no puede tener una clave foránea, porque una FK apunta a **una** tabla. Resultado: comentarios huérfanos, sin integridad referencial y con joins condicionales. Alternativas:

1. **Una tabla de comentarios por destino:** `ComentarioProducto`, `ComentarioPedido`, cada una con su FK. Simple y con integridad.
2. **Varias FK opcionales con un `CHECK` de exclusividad:**

   ```sql
   ProductoId INT NULL REFERENCES dbo.Producto (ProductoId),
   PedidoId   INT NULL REFERENCES dbo.Pedido (PedidoId),
   CONSTRAINT CK_Comentario_UnDestino CHECK (
       (CASE WHEN ProductoId IS NOT NULL THEN 1 ELSE 0 END
      + CASE WHEN PedidoId   IS NOT NULL THEN 1 ELSE 0 END) = 1
   )
   ```

3. **Un supertipo común** ("cosa comentable") del que heredan producto, pedido y artículo, y la FK apunta al supertipo.

-----

## Ejemplo completo

Clientes persona y empresa en la tienda, con el patrón de supertipo y subtipos, borrado lógico y unicidad del email entre los activos:

```sql
CREATE TABLE dbo.ClienteV2 (
    ClienteId   INT IDENTITY(1,1) NOT NULL,
    TipoCliente CHAR(1)           NOT NULL,
    Email       NVARCHAR(254)     NOT NULL,
    Ciudad      NVARCHAR(80)      NULL,
    EliminadoEn DATETIME2(0)      NULL,
    CONSTRAINT PK_ClienteV2 PRIMARY KEY (ClienteId),
    CONSTRAINT CK_ClienteV2_Tipo CHECK (TipoCliente IN ('P', 'E'))
);

CREATE UNIQUE INDEX UX_ClienteV2_Email_Activo ON dbo.ClienteV2 (Email) WHERE EliminadoEn IS NULL;

CREATE TABLE dbo.ClientePersona (
    ClienteId INT           NOT NULL PRIMARY KEY REFERENCES dbo.ClienteV2 (ClienteId),
    Dni       CHAR(8)       NOT NULL UNIQUE,
    Nombre    NVARCHAR(100) NOT NULL
);

CREATE TABLE dbo.ClienteEmpresa (
    ClienteId   INT           NOT NULL PRIMARY KEY REFERENCES dbo.ClienteV2 (ClienteId),
    Ruc         CHAR(11)      NOT NULL UNIQUE,
    RazonSocial NVARCHAR(150) NOT NULL
);

-- Una vista que reúne a todos los clientes activos, con su nombre según el subtipo
CREATE VIEW dbo.vClienteActivo AS
SELECT c.ClienteId, c.TipoCliente, c.Email, c.Ciudad,
       COALESCE(p.Nombre, e.RazonSocial) AS NombreMostrado,
       COALESCE(p.Dni, e.Ruc)            AS Documento
FROM dbo.ClienteV2 AS c
LEFT JOIN dbo.ClientePersona AS p ON p.ClienteId = c.ClienteId
LEFT JOIN dbo.ClienteEmpresa AS e ON e.ClienteId = c.ClienteId
WHERE c.EliminadoEn IS NULL;
```

Con dos clientes de prueba:

```sql
INSERT INTO dbo.ClienteV2 (TipoCliente, Email, Ciudad) VALUES ('P', N'ana.torres@correo.com', N'Lima');      -- 1
INSERT INTO dbo.ClientePersona (ClienteId, Dni, Nombre) VALUES (1, '41234567', N'Ana Torres');

INSERT INTO dbo.ClienteV2 (TipoCliente, Email, Ciudad) VALUES ('E', N'compras@andina.com', N'Arequipa');     -- 2
INSERT INTO dbo.ClienteEmpresa (ClienteId, Ruc, RazonSocial) VALUES (2, '20123456789', N'Distribuidora Andina S.A.C.');

SELECT ClienteId, TipoCliente, NombreMostrado, Documento FROM dbo.vClienteActivo;
```

| ClienteId | TipoCliente | NombreMostrado | Documento |
| --- | --- | --- | --- |
| 1 | P | Ana Torres | 41234567 |
| 2 | E | Distribuidora Andina S.A.C. | 20123456789 |

Qué decisión se tomó y por qué:

* **Supertipo + subtipos:** `Pedido` puede apuntar a `ClienteV2` con una sola FK, y cada subtipo tiene sus `NOT NULL` y su `UNIQUE` (DNI, RUC) sin `CHECK` complejos.
* **`EliminadoEn` con índice único filtrado:** el email es único entre los clientes activos, y un cliente dado de baja puede volver a registrarse.
* **La vista concentra el filtro de borrado lógico:** las consultas que usan `vClienteActivo` no pueden olvidarse del `WHERE EliminadoEn IS NULL`.

-----

## Errores comunes

**1. Una sola tabla con decenas de columnas opcionales por subtipo.**
Qué pasa: una tabla `Cliente` con 40 columnas, la mitad vacías en cada fila, y sin forma de exigir los datos obligatorios de cada tipo.
Por qué: se agregan columnas a medida que aparecen subtipos.
Arreglo: supertipo + subtipos cuando los atributos propios son muchos.

**2. Borrado lógico sin ajustar la unicidad.**
Qué pasa: un cliente dado de baja no puede volver a registrarse con su email.
Por qué: el `UNIQUE` cuenta también las filas "borradas".
Arreglo: índice único filtrado sobre las filas activas.

**3. Borrado lógico sin filtro centralizado.**
Qué pasa: un reporte muestra clientes eliminados porque una consulta olvidó el filtro.
Por qué: el filtro se repite a mano en cada consulta.
Arreglo: vistas que lo apliquen, o filtros globales en el ORM.

**4. EAV para evitar diseñar.**
Qué pasa: una base imposible de consultar y de validar, donde los nombres de los atributos tienen errores de tipeo.
Por qué: se busca flexibilidad total.
Arreglo: subtipos para lo conocido; JSON con validación para lo realmente variable.

**5. FK polimórficas.**
Qué pasa: comentarios que apuntan a productos que no existen.
Por qué: una columna `EntidadId` sin FK posible.
Arreglo: una tabla por destino, FK opcionales exclusivas o un supertipo común.

-----

## Según el motor

| Patrón | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Historial automático | Tablas temporales (`SYSTEM_VERSIONING`, desde 2016) | No nativo: *triggers* o extensiones (`temporal_tables`) | No nativo: *triggers* (MariaDB sí tiene tablas versionadas) |
| Único entre filas activas | Índice único filtrado (`WHERE`) | Índice único parcial (`WHERE`) | No hay índices parciales: columna generada que vale `NULL` para las borradas + `UNIQUE` |
| Herencia nativa | No | `INHERITS` (con limitaciones: las FK y los `UNIQUE` no se heredan; casi no se usa para esto) | No |
| Atributos variables | `json` (2025) o `NVARCHAR(MAX)` + `ISJSON` | `jsonb` con índices GIN | `JSON` con índices sobre columnas generadas |
| Enumeraciones | `CHECK` o catálogo | `CREATE TYPE ... AS ENUM` | Tipo `ENUM` en la columna |

Sobre los `ENUM` de PostgreSQL y MySQL: son cómodos, pero agregar o reordenar valores es un cambio de esquema, y en MySQL el orden interno de los valores afecta al `ORDER BY`. La tabla de catálogo sigue siendo la opción más flexible.

### En MongoDB

Varios de estos patrones son naturales en documental: los subtipos son documentos de la misma colección con campos distintos y un campo discriminador (`tipo: "empresa"`), y los atributos variables son simplemente campos que algunos documentos tienen y otros no. Ahí está justamente el riesgo: sin validación con `$jsonSchema`, cada documento puede tener la estructura que quiera. El borrado lógico y la auditoría se resuelven igual que en relacional (un campo `eliminadoEn`, una colección de historial), y las asociaciones polimórficas son comunes, porque de todos modos no hay claves foráneas.

-----

## Cuándo sí y cuándo no

**Usa estos patrones cuando:**

* El requisito aparece de verdad: subtipos con atributos distintos, historial exigido por el negocio, bajas que deben poder revertirse.

**Ten en cuenta:**

* Cada patrón tiene un costo (más tablas, más filtros, más almacenamiento). No los apliques "por las dudas" a todas las tablas.
* El borrado lógico no es una política de retención de datos: si la ley exige borrar, hay que borrar o anonimizar.
* EAV y las asociaciones polimórficas casi nunca son la respuesta, pero vas a encontrarlos en bases heredadas: reconocerlos es la mitad del trabajo.

**Herramientas que ayudan:**

* En EF Core, las estrategias TPH, TPT y TPC se configuran con `UseTphMappingStrategy()`, `UseTptMappingStrategy()` y `UseTpcMappingStrategy()`; el borrado lógico, con `HasQueryFilter`.
* En SQL Server, `FOR SYSTEM_TIME` consulta el historial de las tablas temporales.

-----

## Resumen en 5 líneas

1. Herencia: una tabla (pocos atributos propios), supertipo + subtipos (muchos atributos, integridad) o tabla por subtipo (casi nunca se consultan juntos).
2. Auditoría: columnas de control para "quién y cuándo"; tablas temporales para el historial completo de valores.
3. Borrado lógico: `EliminadoEn NULL`, con índices únicos filtrados y el filtro centralizado en vistas o en el ORM.
4. Catálogos: `CHECK` para listas cortas y estables; tabla con FK cuando los valores tienen atributos o cambian con el negocio.
5. Evita EAV y las FK polimórficas: subtipos, JSON validado o una tabla por destino.

-----

## Para profundizar

<details>
<summary>Validar JSON en SQL Server</summary>

Para la alternativa JSON a EAV, las columnas comunes van como columnas normales, y solo lo variable va en JSON:

```sql
CREATE TABLE dbo.ProductoV2 (
    ProductoId  INT IDENTITY(1,1) PRIMARY KEY,
    Nombre      NVARCHAR(100) NOT NULL,
    Atributos   NVARCHAR(MAX) NULL CONSTRAINT CK_ProductoV2_Atributos CHECK (ISJSON(Atributos) = 1),
    RamGb       AS CAST(JSON_VALUE(Atributos, '$.ramGb') AS INT)   -- columna calculada para indexar
);

CREATE INDEX IX_ProductoV2_RamGb ON dbo.ProductoV2 (RamGb);
```

La columna calculada extrae el atributo que más se consulta y permite indexarlo; el resto del JSON queda flexible. En SQL Server 2025, el tipo `json` nativo reemplaza al `NVARCHAR(MAX)` con validación incluida.

</details>

<details>
<summary>Lecturas recomendadas</summary>

* *SQL Antipatterns*, Bill Karwin: capítulos dedicados a EAV ("Entity-Attribute-Value"), asociaciones polimórficas y columnas multivaluadas.
* *Patterns of Enterprise Application Architecture*, Martin Fowler: los patrones Single Table Inheritance, Class Table Inheritance y Concrete Table Inheritance.
* La documentación de tablas temporales de SQL Server (*temporal tables*) en Microsoft Learn.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para la herencia hay tres opciones: una tabla con una columna que indica el tipo, una tabla general con tablas por cada subtipo, o una tabla por subtipo. Para no perder datos, uso borrado lógico con una columna que marca la fila como eliminada. Y para los valores fijos, como los estados, uso un `CHECK` o una tabla de catálogo con clave foránea.

### Respuesta ampliada (semi-senior)

Elijo la estrategia de herencia por la cantidad de atributos propios y por cómo se consulta: una tabla única si los subtipos difieren en pocas columnas, supertipo más subtipos si necesito `NOT NULL` y `UNIQUE` propios y una FK simple hacia el supertipo, y tabla por subtipo solo si casi nunca se consultan juntos, porque impide referenciar al conjunto con una FK. Para la auditoría combino columnas de control (quién y cuándo) con tablas temporales de SQL Server para el historial de valores. El borrado lógico lo implemento con `EliminadoEn`, índices únicos filtrados para que la unicidad cuente solo a las filas activas y un filtro centralizado en vistas o en el ORM, sabiendo que no reemplaza el borrado real cuando la ley lo exige. Uso `CHECK` para listas estables y catálogos con FK cuando los valores tienen atributos. Evito EAV, porque pierde tipos, restricciones y legibilidad (prefiero subtipos o JSON validado con columnas calculadas indexables), y evito las asociaciones polimórficas, que no admiten FK.

### Preguntas frecuentes de seguimiento

**1. ¿Qué problemas tiene el borrado lógico?**
Hay que filtrar en todas las consultas, la unicidad se complica (se resuelve con índices únicos filtrados), las tablas crecen y no sirve para cumplir el derecho al olvido.

**2. ¿Qué es EAV y por qué se considera un antipatrón?**
Guardar cada atributo como una fila `(entidad, atributo, valor)`. Pierde los tipos, las restricciones y las FK, y reconstruir un registro exige muchos joins.

**3. ¿TPH, TPT o TPC?**
TPH para pocos atributos propios y consultas sobre el conjunto; TPT para integridad estricta y muchos atributos propios; TPC cuando los subtipos casi nunca se consultan juntos y no hace falta referenciar al conjunto.

**4. ¿Cómo guardas el historial de cambios de una tabla en SQL Server?**
Con una tabla temporal con versionado del sistema: el motor mueve cada versión anterior a una tabla de historial, y se consulta con `FOR SYSTEM_TIME`.

-----

## Práctica

**Ejercicio 1.** Una plataforma de cursos tiene usuarios que pueden ser alumnos, instructores o ambos a la vez. Los instructores tienen biografía y tarifa por hora; los alumnos, nivel y fecha de suscripción. ¿Qué estrategia de herencia usarías y por qué? Escribe las tablas.

<details>
<summary>Solución</summary>

**Supertipo + subtipos**, porque un usuario puede ser alumno **e** instructor a la vez (especialización solapada). Con una columna discriminadora única (`Tipo`) no se puede representar "ambos", y con tabla por subtipo se duplicarían los datos comunes.

```sql
CREATE TABLE dbo.Usuario (
    UsuarioId INT IDENTITY(1,1) PRIMARY KEY,
    Email     NVARCHAR(254) NOT NULL UNIQUE,
    Nombre    NVARCHAR(100) NOT NULL
);

CREATE TABLE dbo.Alumno (
    UsuarioId         INT         NOT NULL PRIMARY KEY REFERENCES dbo.Usuario (UsuarioId),
    Nivel             VARCHAR(20) NOT NULL,
    FechaSuscripcion  DATE        NOT NULL
);

CREATE TABLE dbo.Instructor (
    UsuarioId    INT            NOT NULL PRIMARY KEY REFERENCES dbo.Usuario (UsuarioId),
    Biografia    NVARCHAR(2000) NOT NULL,
    TarifaPorHora DECIMAL(8,2)  NOT NULL CHECK (TarifaPorHora > 0)
);
```

Un usuario que es ambas cosas tiene una fila en `Alumno` y otra en `Instructor`, con el mismo `UsuarioId`. Aquí, a diferencia de los clientes persona y empresa, no hay que impedirlo: es justamente la regla.

</details>

**Ejercicio 2.** Encuentras esta tabla en una base heredada. ¿Qué antipatrón es, qué problemas concretos tiene y cómo la rediseñarías, sabiendo que solo existen las categorías Laptop (RAM, procesador) y Libro (ISBN, autor)?

```sql
CREATE TABLE dbo.Caracteristica (ProductoId INT, Nombre VARCHAR(50), Valor VARCHAR(200));
```

<details>
<summary>Solución</summary>

Es **EAV**. Problemas concretos: no se puede exigir que una laptop tenga RAM, `Valor` acepta cualquier texto (no se puede filtrar "RAM mayor a 8 GB" de forma confiable), no hay clave primaria (se puede repetir el mismo atributo para el mismo producto), no hay FK hacia `Producto`, y los nombres de los atributos pueden tener errores de tipeo.

Como las categorías son pocas y conocidas, se rediseña con subtipos:

```sql
CREATE TABLE dbo.ProductoLaptop (
    ProductoId INT          NOT NULL PRIMARY KEY REFERENCES dbo.Producto (ProductoId),
    RamGb      SMALLINT     NOT NULL CHECK (RamGb > 0),
    Procesador NVARCHAR(50) NOT NULL
);

CREATE TABLE dbo.ProductoLibro (
    ProductoId INT           NOT NULL PRIMARY KEY REFERENCES dbo.Producto (ProductoId),
    Isbn       CHAR(13)      NOT NULL UNIQUE,
    Autor      NVARCHAR(150) NOT NULL
);
```

La migración de datos convierte cada grupo de filas de `Caracteristica` en una fila tipada, y es el momento de descubrir (y limpiar) los valores inválidos acumulados.

</details>

-----

## Siguiente lección

[Convenciones de nombres](10-Convenciones%20de%20nombres.md)
