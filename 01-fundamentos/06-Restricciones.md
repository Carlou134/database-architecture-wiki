# Restricciones

## En una frase

Las restricciones (`NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`, junto con la clave primaria y las foráneas) son reglas de negocio declaradas en la tabla, que el motor hace cumplir en cada escritura, venga de donde venga.

-----

## Antes de empezar

Conviene que ya sepas:

* Claves primarias y foráneas, de [Claves primarias](04-Claves%20primarias.md) y [Claves foráneas](05-Claves%20foráneas%20e%20integridad%20referencial.md).
* Que el tipo de dato ya es una primera restricción, de [Tipos de datos](03-Tipos%20de%20datos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Restricción (constraint):** regla declarativa que el motor verifica en cada `INSERT` y `UPDATE`.
* **`NOT NULL`:** la columna es obligatoria.
* **`UNIQUE`:** los valores de la columna (o combinación de columnas) no se repiten.
* **`CHECK`:** una condición que cada fila debe cumplir.
* **`DEFAULT`:** el valor que toma la columna cuando el `INSERT` no la menciona.
* **Índice filtrado:** índice que solo incluye las filas que cumplen un `WHERE`.

-----

## El problema

La tienda tiene reglas claras: el precio no puede ser negativo, el email es obligatorio y único, la cantidad de un pedido es al menos 1, el estado es uno de cinco valores. Si esas reglas viven solo en la aplicación:

```csharp
if (producto.Precio < 0) throw new ArgumentException("Precio inválido");
```

entonces viven en **un** lugar de **un** cliente. El script de importación, el panel de administración hecho en otro lenguaje y el `UPDATE` manual de soporte no pasan por ese `if`. Un día aparece un producto con precio `-120.00` y un pedido en estado `'pagado '` (con espacio), y nadie sabe de dónde salieron.

La regla tiene que vivir donde están los datos. Una restricción se escribe una vez y protege todas las escrituras, para siempre.

-----

## Cómo funciona

### `NOT NULL`: columnas obligatorias

```sql
Nombre NVARCHAR(100) NOT NULL,
Ciudad NVARCHAR(80)  NULL,
```

Cada columna debe decir explícitamente si admite `NULL`. Si no lo dices, SQL Server usa la configuración de la sesión (normalmente `NULL`), y es mejor no depender de eso.

La pregunta de diseño es: **¿puede existir una fila válida sin este dato?** Un cliente sin nombre, no. Un cliente sin ciudad, sí (María no la informó). Hacer `NOT NULL` todo lo que sea obligatorio simplifica todas las consultas posteriores, porque el `NULL` trae su propia lógica (ver [NULL y la lógica de tres valores](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md)).

```text
Msg 515, Level 16, State 2
Cannot insert the value NULL into column 'Nombre', table 'Tienda.dbo.Producto';
column does not allow nulls. INSERT fails.
```

### `UNIQUE`: claves alternativas

```sql
CONSTRAINT UQ_Cliente_Email UNIQUE (Email),
CONSTRAINT UQ_Producto_Categoria_Nombre UNIQUE (CategoriaId, Nombre),   -- compuesta
```

`UNIQUE` protege las claves naturales que no son la clave primaria. Por dentro, SQL Server crea un índice único para hacerla cumplir, así que de paso acelera las búsquedas por esa columna.

**La trampa del `NULL`:** en SQL Server, una restricción `UNIQUE` admite **un solo** `NULL`, porque trata dos `NULL` como iguales. Si quieres "único cuando hay valor, y que puedan faltar muchos" (por ejemplo, un documento opcional), usa un índice único filtrado:

```sql
CREATE UNIQUE INDEX UX_Cliente_Documento
    ON dbo.Cliente (NumeroDocumento)
    WHERE NumeroDocumento IS NOT NULL;
```

### `CHECK`: reglas sobre los valores

```sql
CONSTRAINT CK_Producto_Precio CHECK (Precio >= 0),
CONSTRAINT CK_Pedido_Estado CHECK (Estado IN ('Pendiente', 'Pagado', 'Enviado', 'Entregado', 'Cancelado')),
CONSTRAINT CK_Promocion_Fechas CHECK (FechaFin >= FechaInicio),   -- compara columnas de la misma fila
```

Un `CHECK` es una condición que se evalúa en cada fila que se inserta o modifica. Puede comparar columnas de la **misma fila**, pero no puede consultar otras filas ni otras tablas.

```text
Msg 547, Level 16, State 0
The INSERT statement conflicted with the CHECK constraint "CK_Producto_Precio".
The conflict occurred in database "Tienda", table "dbo.Producto", column 'Precio'.
```

**La trampa del `NULL`, otra vez:** un `CHECK` rechaza la fila solo si la condición es **falsa**. Si es **desconocida** (porque hay un `NULL`), la fila pasa:

```sql
-- Descuento NULL: la condición Descuento BETWEEN 0 AND 50 es UNKNOWN → la fila se acepta
CONSTRAINT CK_Producto_Descuento CHECK (Descuento BETWEEN 0 AND 50)
```

Si la columna no debe ser nula, el `CHECK` no te protege: necesitas también `NOT NULL`.

### `DEFAULT`: valores por defecto

```sql
FechaPedido DATETIME2(0) NOT NULL CONSTRAINT DF_Pedido_FechaPedido DEFAULT SYSUTCDATETIME(),
Estado      VARCHAR(20)  NOT NULL CONSTRAINT DF_Pedido_Estado DEFAULT 'Pendiente',
Activo      BIT          NOT NULL CONSTRAINT DF_Producto_Activo DEFAULT 1,
```

El valor por defecto se aplica **solo cuando el `INSERT` no menciona la columna** (o usa la palabra `DEFAULT`). Si envías `NULL` explícitamente, se guarda `NULL`, o falla si la columna es `NOT NULL`:

```sql
INSERT INTO dbo.Pedido (ClienteId) VALUES (1);                       -- Estado = 'Pendiente'
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (1, DEFAULT);      -- Estado = 'Pendiente'
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (1, NULL);         -- falla: Estado es NOT NULL
```

`DEFAULT` + `NOT NULL` es una combinación muy útil: la columna siempre tiene valor, y quien inserta no tiene que acordarse de enviarlo.

### Ponerle nombre a cada restricción

Si no le das nombre, SQL Server inventa uno como `CK__Producto__Precio__3B75D760`. Funciona, pero:

* El mensaje de error no dice nada útil.
* El nombre es distinto en cada base (desarrollo, pruebas, producción), y los scripts de migración que la borran por nombre fallan.

Convención usada en este wiki (más en [Convenciones de nombres](../02-diseno-y-modelado/10-Convenciones%20de%20nombres.md)):

| Restricción | Prefijo | Ejemplo |
| --- | --- | --- |
| Clave primaria | `PK_` | `PK_Cliente` |
| Clave foránea | `FK_` | `FK_Pedido_Cliente` |
| Única | `UQ_` | `UQ_Cliente_Email` |
| Check | `CK_` | `CK_Producto_Precio` |
| Default | `DF_` | `DF_Pedido_Estado` |

### Agregar restricciones a una tabla con datos

```sql
ALTER TABLE dbo.Producto ADD CONSTRAINT CK_Producto_Stock CHECK (Stock >= 0);
```

Si alguna fila existente la incumple, el `ALTER` falla. Las opciones son corregir los datos primero (lo correcto) o crearla con `WITH NOCHECK`, lo que valida solo las filas nuevas y deja la restricción marcada como no confiable, igual que las claves foráneas.

Lo mismo con `NOT NULL`: no se puede cambiar una columna a `NOT NULL` si tiene nulos. Primero se completan (`UPDATE ... SET Col = valor WHERE Col IS NULL`) y después se altera la columna.

### Defensa en profundidad

Validar en la aplicación **y** en la base no es duplicar esfuerzo. Cada capa cumple un papel distinto:

| Capa | Para qué |
| --- | --- |
| Interfaz | Respuesta inmediata al usuario ("el email no es válido") |
| Aplicación / dominio | Reglas de negocio complejas, mensajes claros, lógica que depende de varias tablas |
| Base de datos | La garantía final: ningún cliente puede saltársela |

La base no reemplaza a la aplicación (un `Msg 547` no es un buen mensaje para el usuario), y la aplicación no reemplaza a la base.

-----

## Ejemplo completo

La tabla `Producto` sin restricciones y con restricciones:

**Antes:**

```sql
CREATE TABLE dbo.Producto (
    ProductoId  INT IDENTITY(1,1) PRIMARY KEY,
    CategoriaId INT,
    Nombre      NVARCHAR(100),
    Precio      DECIMAL(10, 2),
    Stock       INT,
    Activo      BIT
);
```

**Después** (es la definición que usa el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md)):

```sql
CREATE TABLE dbo.Producto (
    ProductoId  INT IDENTITY(1,1) NOT NULL,
    CategoriaId INT               NOT NULL,
    Nombre      NVARCHAR(100)     NOT NULL,
    Precio      DECIMAL(10, 2)    NOT NULL,
    Stock       INT               NOT NULL CONSTRAINT DF_Producto_Stock DEFAULT 0,
    Activo      BIT               NOT NULL CONSTRAINT DF_Producto_Activo DEFAULT 1,
    CONSTRAINT PK_Producto PRIMARY KEY (ProductoId),
    CONSTRAINT FK_Producto_Categoria FOREIGN KEY (CategoriaId) REFERENCES dbo.Categoria (CategoriaId),
    CONSTRAINT UQ_Producto_Categoria_Nombre UNIQUE (CategoriaId, Nombre),
    CONSTRAINT CK_Producto_Precio CHECK (Precio >= 0),
    CONSTRAINT CK_Producto_Stock CHECK (Stock >= 0)
);
```

Inserciones de prueba:

```sql
INSERT INTO dbo.Producto (CategoriaId, Nombre, Precio) VALUES (1, N'Webcam HD', 150.00);   -- ✅ Stock 0, Activo 1
INSERT INTO dbo.Producto (CategoriaId, Nombre, Precio) VALUES (1, N'Parlante', -20.00);   -- ❌ CK_Producto_Precio
INSERT INTO dbo.Producto (CategoriaId, Nombre, Precio) VALUES (1, N'Webcam HD', 160.00);   -- ❌ UQ_Producto_Categoria_Nombre
INSERT INTO dbo.Producto (CategoriaId, Precio) VALUES (2, 30.00);                          -- ❌ Nombre NOT NULL
```

Resultado sobre la tabla de la tienda, que ya tenía 6 productos: solo la primera fila entró.

| ProductoId | CategoriaId | Nombre | Precio | Stock | Activo |
| --- | --- | --- | --- | --- | --- |
| 7 | 1 | Webcam HD | 150.00 | 0 | 1 |

Qué cambió y por qué:

* **`NOT NULL` en todo lo obligatorio:** un producto sin nombre o sin precio no es un producto válido.
* **`FK` + `NOT NULL` en `CategoriaId`:** todo producto pertenece a una categoría que existe.
* **`UQ` compuesta:** no hay dos productos con el mismo nombre en la misma categoría.
* **`CK` de precio y stock:** nunca negativos.
* **`DF` en stock y activo:** quien inserta no tiene que acordarse de ellos.
* **Nombres en todas las restricciones:** cada error dice exactamente qué regla se rompió.

-----

## Errores comunes

**1. Confiar en que un `CHECK` impide los `NULL`.**
Qué pasa: `CHECK (Precio >= 0)` acepta `Precio = NULL`.
Por qué: la condición es desconocida, no falsa, y el `CHECK` solo rechaza lo falso.
Arreglo: `NOT NULL` además del `CHECK`.

**2. Esperar que `DEFAULT` reemplace un `NULL` enviado.**
Qué pasa: un ORM envía todas las columnas, incluidas las nulas, y el `DEFAULT` nunca se aplica.
Por qué: el `DEFAULT` solo actúa si la columna no aparece en el `INSERT`.
Arreglo: no envíes la columna, envía `DEFAULT` o configura el ORM para que deje generar el valor a la base (en EF Core, `HasDefaultValueSql`).

**3. Un `UNIQUE` sobre una columna opcional.**
Qué pasa: el segundo cliente sin documento falla con una violación de unicidad.
Por qué: en SQL Server, `UNIQUE` admite un solo `NULL`.
Arreglo: un índice único filtrado `WHERE Col IS NOT NULL`.

**4. Restricciones sin nombre.**
Qué pasa: errores con nombres como `CK__Producto__Precio__3B75D760` y migraciones que fallan en otro entorno.
Por qué: el nombre generado es distinto en cada base.
Arreglo: nombra todas las restricciones con una convención.

**5. Usar funciones en un `CHECK` para validar contra otras tablas.**
Qué pasa: un `CHECK (dbo.fnClienteActivo(ClienteId) = 1)` que se salta casos en actualizaciones concurrentes y degrada el rendimiento.
Por qué: un `CHECK` está pensado para la fila que se escribe; con una función que lee otras tablas, el motor no revalida cuando cambian esas otras tablas.
Arreglo: claves foráneas para la existencia; la lógica entre tablas, en la aplicación o en una transacción explícita.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| `NULL` en `UNIQUE` | Uno solo | Varios (desde la 15, `NULLS NOT DISTINCT` permite uno solo) | Varios |
| Único "solo si hay valor" | Índice único filtrado | Comportamiento por defecto | Comportamiento por defecto |
| `CHECK` | Se cumple siempre | Se cumple siempre | Se cumple desde la 8.0.16; **antes se aceptaba la sintaxis y se ignoraba** |
| `DEFAULT` con expresiones | Sí (`SYSUTCDATETIME()`, `NEWID()`) | Sí, cualquier expresión | Sí, desde la 8.0.13 (entre paréntesis) |
| Restricciones extra | — | `EXCLUDE` (por ejemplo, impedir reservas con fechas solapadas) | — |
| Validar después | `WITH NOCHECK` + `WITH CHECK CHECK CONSTRAINT` | `NOT VALID` + `VALIDATE CONSTRAINT` | No tiene equivalente |

Si heredas una base MySQL anterior a 8.0.16, revisa los `CHECK`: estaban escritos, pero nunca se aplicaron, así que los datos pueden incumplirlos.

### En MongoDB

Como no hay esquema fijo, las reglas se declaran con un **validador** en la colección, usando `$jsonSchema`:

```javascript
db.createCollection("productos", {
  validator: {
    $jsonSchema: {
      required: ["nombre", "precio"],
      properties: {
        precio: { bsonType: "decimal", minimum: 0 }
      }
    }
  }
});
```

La unicidad se consigue con un índice único (`db.clientes.createIndex({ email: 1 }, { unique: true })`). No hay `DEFAULT`: los valores por defecto los pone la aplicación o el ODM (Mongoose, por ejemplo).

-----

## Cuándo sí y cuándo no

**Declara la restricción cuando:**

* La regla se puede expresar sobre una fila (rangos, formatos simples, listas de valores, comparaciones entre columnas).
* La regla nunca debe romperse, venga de donde venga el dato.

**Ten en cuenta:**

* Las reglas que dependen de varias filas o tablas ("un cliente no puede tener más de 3 pedidos pendientes") no caben en un `CHECK`: van en la aplicación, en un procedimiento almacenado o en una transacción.
* Un `CHECK` con una lista de valores (`Estado IN (...)`) es simple, pero para agregar un valor hay que alterar la tabla. Si la lista cambia a menudo o necesita atributos propios, conviene una tabla de catálogo con clave foránea (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)).

**Herramientas que ayudan:**

* `sp_helpconstraint 'dbo.Producto'` lista todas las restricciones de una tabla.
* `sys.check_constraints` y `sys.foreign_keys` tienen la columna `is_not_trusted`.

-----

## Resumen en 5 líneas

1. Las restricciones son reglas de negocio que el motor hace cumplir en cada escritura, venga de donde venga.
2. `NOT NULL` para lo obligatorio; `UNIQUE` para las claves alternativas; `CHECK` para reglas sobre la fila; `DEFAULT` para valores implícitos.
3. `CHECK` deja pasar los `NULL`, y `DEFAULT` solo actúa si la columna no se menciona.
4. En SQL Server, `UNIQUE` admite un solo `NULL`; para columnas opcionales, índice único filtrado.
5. Nombra todas las restricciones (`PK_`, `FK_`, `UQ_`, `CK_`, `DF_`) y valida en la aplicación **y** en la base.

-----

## Para profundizar

<details>
<summary>Las restricciones también ayudan al optimizador</summary>

Un `CHECK` confiable le da información al optimizador. Si `CK_Pedido_Estado` garantiza que `Estado` solo toma cinco valores, una consulta `WHERE Estado = 'Devuelto'` puede resolverse sin leer la tabla, porque el resultado es necesariamente vacío. Algo parecido pasa con las vistas particionadas: si cada tabla tiene un `CHECK` sobre el rango de fechas, el optimizador solo lee las tablas que pueden tener datos del rango pedido.

</details>

<details>
<summary>Cuándo se verifica una restricción</summary>

En SQL Server, las restricciones se verifican al final de **cada instrucción**, no fila por fila ni al final de la transacción. Por eso un `UPDATE dbo.Producto SET Orden = Orden + 1` puede reordenar todos los valores de una columna `UNIQUE` sin fallar a mitad de camino: lo que se verifica es el estado final de la instrucción. PostgreSQL, en cambio, verifica las `UNIQUE` fila por fila salvo que se declaren `DEFERRABLE`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las restricciones son reglas que la base de datos hace cumplir: `NOT NULL` para columnas obligatorias, `UNIQUE` para valores que no se pueden repetir, `CHECK` para condiciones como "el precio no puede ser negativo" y `DEFAULT` para valores por defecto. Junto con la clave primaria y las foráneas, garantizan que los datos sean válidos aunque la aplicación tenga un error.

### Respuesta ampliada (semi-senior)

Uso las restricciones como la última línea de defensa: todo lo que se puede expresar sobre una fila (obligatoriedad, unicidad, rangos, listas de valores, comparaciones entre columnas) va declarado en la tabla, porque la aplicación no es el único cliente de la base. Conozco sus trampas con `NULL`: el `CHECK` solo rechaza lo falso, así que una columna obligatoria necesita `NOT NULL`; y en SQL Server `UNIQUE` admite un solo nulo, por lo que en columnas opcionales uso un índice único filtrado. El `DEFAULT` solo actúa si la columna se omite, algo que hay que configurar en los ORM. Nombro todas las restricciones para tener errores claros y migraciones reproducibles, y al agregarlas a tablas con datos corrijo los datos antes de crearlas, para que queden confiables. Las reglas entre filas o tablas no van en un `CHECK`, sino en la aplicación o en transacciones explícitas.

### Preguntas frecuentes de seguimiento

**1. ¿Un `CHECK (Precio > 0)` impide que el precio sea `NULL`?**
No. Con `NULL`, la condición es desconocida, y el `CHECK` solo rechaza las condiciones falsas. Hace falta `NOT NULL`.

**2. ¿Cuántos `NULL` admite una restricción `UNIQUE` en SQL Server?**
Uno solo. Para permitir varios, se usa un índice único filtrado con `WHERE Col IS NOT NULL`.

**3. ¿Por qué validar en la base si ya valido en la aplicación?**
Porque la base tiene más clientes que la aplicación (scripts, ETL, otros servicios, cambios manuales) y es el único lugar donde la regla no se puede saltar.

**4. ¿Cuándo se aplica un `DEFAULT`?**
Cuando el `INSERT` no menciona la columna o usa la palabra clave `DEFAULT`. Si se envía `NULL` explícitamente, no se aplica.

-----

## Práctica

**Ejercicio 1.** Escribe las restricciones para esta tabla de promociones, con estas reglas: el nombre es obligatorio y único; el descuento es obligatorio y está entre 1 y 90; la fecha de fin no puede ser anterior a la de inicio; si no se indica, la promoción está activa.

```sql
CREATE TABLE dbo.Promocion (
    PromocionId INT IDENTITY(1,1),
    Nombre      NVARCHAR(80),
    Descuento   TINYINT,
    FechaInicio DATE,
    FechaFin    DATE,
    Activa      BIT
);
```

<details>
<summary>Solución</summary>

```sql
CREATE TABLE dbo.Promocion (
    PromocionId INT IDENTITY(1,1) NOT NULL,
    Nombre      NVARCHAR(80)      NOT NULL,
    Descuento   TINYINT           NOT NULL,
    FechaInicio DATE              NOT NULL,
    FechaFin    DATE              NOT NULL,
    Activa      BIT               NOT NULL CONSTRAINT DF_Promocion_Activa DEFAULT 1,
    CONSTRAINT PK_Promocion PRIMARY KEY (PromocionId),
    CONSTRAINT UQ_Promocion_Nombre UNIQUE (Nombre),
    CONSTRAINT CK_Promocion_Descuento CHECK (Descuento BETWEEN 1 AND 90),
    CONSTRAINT CK_Promocion_Fechas CHECK (FechaFin >= FechaInicio)
);
```

`NOT NULL` en `Descuento` y en las fechas es lo que impide que los `CHECK` dejen pasar valores nulos.

</details>

**Ejercicio 2.** ¿Qué inserciones fallan y con qué restricción?

```sql
INSERT INTO dbo.Promocion (Nombre, Descuento, FechaInicio, FechaFin) VALUES (N'Cyber', 30, '2026-11-01', '2026-11-03');
INSERT INTO dbo.Promocion (Nombre, Descuento, FechaInicio, FechaFin) VALUES (N'Navidad', 95, '2026-12-20', '2026-12-25');
INSERT INTO dbo.Promocion (Nombre, Descuento, FechaInicio, FechaFin) VALUES (N'Verano', 20, '2027-02-01', '2027-01-01');
INSERT INTO dbo.Promocion (Nombre, Descuento, FechaInicio, FechaFin, Activa) VALUES (N'Cyber', 10, '2027-11-01', '2027-11-02', 0);
INSERT INTO dbo.Promocion (Nombre, Descuento, FechaInicio, FechaFin, Activa) VALUES (N'Invierno', 15, '2027-07-01', '2027-07-15', NULL);
```

<details>
<summary>Solución</summary>

| Fila | Resultado | Motivo |
| --- | --- | --- |
| Cyber (2026) | ✅ Entra, con `Activa = 1` por el `DEFAULT` | — |
| Navidad | ❌ | `CK_Promocion_Descuento`: 95 > 90 |
| Verano | ❌ | `CK_Promocion_Fechas`: fin anterior al inicio |
| Cyber (2027) | ❌ | `UQ_Promocion_Nombre`: el nombre ya existe |
| Invierno | ❌ | `Activa` es `NOT NULL`, y el `NULL` explícito no activa el `DEFAULT` |

</details>

-----

## Siguiente lección

[NULL y la lógica de tres valores](07-NULL%20y%20la%20lógica%20de%20tres%20valores.md)
