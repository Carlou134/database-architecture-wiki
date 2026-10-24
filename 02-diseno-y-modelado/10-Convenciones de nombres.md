# Convenciones de nombres

## En una frase

Una convención de nombres es un conjunto de reglas (singular o plural, mayúsculas, prefijos, cómo se llaman las claves y las restricciones) que hace que cualquier persona del equipo pueda **adivinar** el nombre de una tabla o columna sin buscarlo; importa menos cuál elijas que aplicarla siempre igual.

-----

## Antes de empezar

Conviene que ya sepas:

* Claves primarias, foráneas y restricciones, del módulo de [Fundamentos](../01-fundamentos/README.md).
* Esquemas (`dbo`), de [Qué es una base de datos y un SGBD](../01-fundamentos/01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md).
* Los principios de nombres del código (nombres que revelan la intención, sin abreviaturas, un idioma por proyecto): son los mismos que en C#, aplicados a tablas y columnas.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **PascalCase:** cada palabra empieza con mayúscula: `DetallePedido`.
* **snake_case:** palabras en minúscula separadas por guion bajo: `detalle_pedido`.
* **Identificador delimitado:** nombre entre `[corchetes]` (SQL Server) o `"comillas"` (estándar), que permite espacios o palabras reservadas.
* **Palabra reservada:** palabra que el lenguaje SQL usa (`ORDER`, `USER`, `TABLE`) y que no conviene usar como nombre.

-----

## El problema

Una base heredada tiene estas tablas:

```text
tbl_Clientes    Pedido    order_items    PRODUCTOS    tblCat    usr
```

Y estas columnas en `tbl_Clientes`:

```text
ID    cli_nombre    Email_Cliente    fecRegistro    flgActivo    ciudad
```

Para escribir una consulta, hay que abrir el diagrama cada vez: ¿la tabla es `Pedido` o `Pedidos`? ¿La clave de `tblCat` es `ID`, `CatId` o `cat_id`? ¿`order_items` es lo mismo que el detalle del pedido? ¿Qué es `usr`, y por qué `PRODUCTOS` está en mayúsculas?

Ninguno de estos nombres está "mal" por separado. El problema es que **no siguen ninguna regla**, así que no se pueden deducir. Y a diferencia del código, renombrar una tabla en producción es caro: hay que cambiar consultas, procedimientos, vistas, mapeos del ORM, reportes y scripts de ETL.

-----

## Cómo funciona

### La convención de este wiki

| Elemento | Regla | Ejemplo |
| --- | --- | --- |
| Tabla | PascalCase, **singular**, sustantivo del dominio | `Cliente`, `DetallePedido` |
| Columna | PascalCase | `FechaPedido`, `PrecioUnitario` |
| Clave primaria sustituta | `NombreTabla` + `Id` | `ClienteId` |
| Clave foránea | **El mismo nombre** que la clave a la que apunta | `Pedido.ClienteId` → `Cliente.ClienteId` |
| Booleano | Afirmación o pregunta | `Activo`, `EsFinal`, `TieneStock` |
| Fecha / momento | `Fecha...` o `...En` | `FechaPedido`, `EliminadoEn` |
| Esquema | minúsculas o PascalCase, por área | `dbo`, `ventas`, `inventario` |
| Idioma | Uno solo en toda la base | Español en este wiki |

No es la única convención válida. Lo que la hace buena es que cada regla tiene un **porqué**, y que todas se aplican sin excepciones.

### Singular o plural

Las dos opciones tienen defensores:

| | Singular (`Cliente`) | Plural (`Clientes`) |
| --- | --- | --- |
| Argumento | La tabla define **qué es una fila**: "un Cliente" | La tabla es un **conjunto** de clientes |
| Ventaja | Evita plurales irregulares y compuestos raros (`DetallesPedidos`) | Se lee natural en `SELECT * FROM Clientes` |
| Coincide con | Las clases del código (`class Cliente`) | Los `DbSet<Cliente> Clientes` de EF Core |

Este wiki usa **singular**. Lo que no se puede hacer es mezclar: si una tabla es `Pedido` y otra `Clientes`, nadie puede adivinar la siguiente.

### Mayúsculas: PascalCase en SQL Server, snake_case en PostgreSQL

La elección depende del motor, por una razón técnica:

* **SQL Server** conserva el nombre tal como se escribió, y con la intercalación habitual (insensible a mayúsculas) `Cliente` y `cliente` son lo mismo. PascalCase es la convención dominante.
* **PostgreSQL** convierte a **minúsculas** todo identificador que no esté entre comillas. Si creas `CREATE TABLE "DetallePedido"`, después tienes que escribir `"DetallePedido"` con comillas **siempre**; sin comillas, `DetallePedido` se busca como `detallepedido` y falla. Por eso en PostgreSQL se usa `snake_case`: `detalle_pedido`.
* **MySQL** guarda cada tabla como un archivo, y en Linux los nombres de archivo distinguen mayúsculas. Una base desarrollada en Windows (`Cliente` = `cliente`) puede fallar al desplegarse en Linux. La convención habitual es `snake_case` en minúsculas.

### Claves: `ClienteId` en lugar de `Id`

Hay dos escuelas para la clave primaria sustituta:

```sql
-- Opción A: Id en todas las tablas
SELECT p.Id, c.Id FROM Pedido AS p JOIN Cliente AS c ON c.Id = p.ClienteId;

-- Opción B: NombreTabla + Id
SELECT p.PedidoId, c.ClienteId FROM Pedido AS p JOIN Cliente AS c ON c.ClienteId = p.ClienteId;
```

La opción B tiene tres ventajas concretas:

* **La FK se llama igual que la PK:** `Pedido.ClienteId` → `Cliente.ClienteId`. El join se lee solo y es difícil equivocarse.
* **Los resultados no tienen columnas ambiguas:** con la opción A, un `SELECT` con joins devuelve tres columnas llamadas `Id`.
* **Errores más visibles:** `ON c.Id = p.Id` (un error típico de la opción A) compila y devuelve basura; `ON c.ClienteId = p.PedidoId` salta a la vista.

La opción A es la convención por defecto de EF Core y de muchos ORM, y es perfectamente válida si el equipo la aplica de forma consistente. Este wiki usa la B.

### Nombres de restricciones e índices

| Objeto | Patrón | Ejemplo |
| --- | --- | --- |
| Clave primaria | `PK_Tabla` | `PK_Cliente` |
| Clave foránea | `FK_TablaHija_TablaPadre` | `FK_Pedido_Cliente` |
| Única | `UQ_Tabla_Columnas` | `UQ_Cliente_Email` |
| Check | `CK_Tabla_Regla` | `CK_Producto_Precio` |
| Default | `DF_Tabla_Columna` | `DF_Pedido_Estado` |
| Índice | `IX_Tabla_Columnas` | `IX_Pedido_ClienteId` |
| Índice único | `UX_Tabla_Columnas` | `UX_Cliente_Email_Activo` |

Con nombres así, un error como `The INSERT statement conflicted with the CHECK constraint "CK_Producto_Precio"` se entiende sin abrir el script. Y si una tabla tiene dos FK hacia la misma tabla padre, el nombre incluye el rol: `FK_Vuelo_Aeropuerto_Origen`, `FK_Vuelo_Aeropuerto_Destino`.

### Lo que no conviene hacer

| Práctica | Ejemplo | Por qué no |
| --- | --- | --- |
| Prefijos de tipo de objeto | `tbl_Cliente`, `vw_Ventas` | El contexto ya dice qué es; si una tabla se reemplaza por una vista, el nombre miente |
| Prefijos de tipo de dato | `strNombre`, `intEdad` | Si cambia el tipo, el nombre miente (igual que en el código) |
| Abreviaturas | `fecReg`, `cli_nom`, `usr` | Se escriben una vez y se descifran cien |
| Palabras reservadas | `Order`, `User`, `Group`, `Key` | Obligan a escribir `[Order]` en cada consulta |
| Espacios o caracteres especiales | `Detalle Pedido`, `Año` | Obligan a delimitar siempre; algunas herramientas fallan |
| Prefijo `sp_` en procedimientos | `sp_CrearPedido` | SQL Server busca primero los `sp_` en la base `master`; usa `usp_` o un verbo sin prefijo |
| Mezclar idiomas | `Customer.FechaAlta` | Nadie sabe en qué idioma buscar |

Las palabras reservadas son, de hecho, una razón por la que este wiki usa `Pedido` y no `Order`: en una base en inglés, la tabla tendría que llamarse `Orders` o `SalesOrder` para no escribir `[Order]` en cada consulta.

### Esquemas para agrupar por área

En bases grandes, los esquemas organizan las tablas por área del negocio, algo que los prefijos intentaban resolver:

```sql
CREATE SCHEMA ventas;
CREATE SCHEMA inventario;

-- ventas.Pedido, ventas.DetallePedido, ventas.Cliente
-- inventario.Producto, inventario.Categoria
```

Además de organizar, permiten dar permisos por área (`GRANT SELECT ON SCHEMA::ventas TO Analista`). La tienda de este wiki usa solo `dbo` porque es pequeña.

-----

## Ejemplo completo

La base heredada del problema, renombrada con la convención:

**Antes:**

```text
tbl_Clientes (ID, cli_nombre, Email_Cliente, fecRegistro, flgActivo, ciudad)
Pedido       (PedidoID, cliente, fecha)
order_items  (order_id, prod, qty, price)
PRODUCTOS    (ID_PRODUCTO, NOMBRE, CAT)
tblCat       (ID, Descripcion)
```

**Después:**

```text
Cliente       (ClienteId, Nombre, Email, FechaRegistro, Activo, Ciudad)
Pedido        (PedidoId, ClienteId, FechaPedido)
DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
Producto      (ProductoId, Nombre, CategoriaId)
Categoria     (CategoriaId, Nombre)
```

Y la misma consulta, antes y después:

```sql
-- Antes
SELECT c.cli_nombre, p.fecha, oi.qty, pr.NOMBRE, t.Descripcion
FROM tbl_Clientes AS c
JOIN Pedido AS p ON p.cliente = c.ID
JOIN order_items AS oi ON oi.order_id = p.PedidoID
JOIN PRODUCTOS AS pr ON pr.ID_PRODUCTO = oi.prod
JOIN tblCat AS t ON t.ID = pr.CAT;

-- Después
SELECT c.Nombre, p.FechaPedido, d.Cantidad, pr.Nombre AS Producto, cat.Nombre AS Categoria
FROM dbo.Cliente AS c
JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
JOIN dbo.Categoria AS cat ON cat.CategoriaId = pr.CategoriaId;
```

Qué cambió y por qué:

* **Tablas:** singular, PascalCase, sin prefijos, en un solo idioma (`order_items` → `DetallePedido`).
* **Claves:** `NombreTabla + Id`, y cada FK se llama igual que la PK que referencia: cada `ON` compara dos columnas con el mismo nombre.
* **Columnas:** sin abreviaturas (`qty` → `Cantidad`, `fecha` → `FechaPedido`) ni prefijos de tipo (`flgActivo` → `Activo`).
* **Ambigüedades resueltas:** `fecha` podía ser de pedido, de entrega o de pago; `FechaPedido`, no.

En una base en producción, renombrar así se hace en etapas: vistas con los nombres nuevos sobre las tablas viejas, migración gradual de las consultas y, al final, el renombrado real (`sp_rename`).

-----

## Errores comunes

**1. Mezclar convenciones.**
Qué pasa: `Cliente`, `Pedidos`, `order_items` en la misma base.
Por qué: cada persona o cada época aplicó su estilo.
Arreglo: una convención escrita en el repositorio, aplicada a todo objeto nuevo y revisada en el *code review*.

**2. Usar palabras reservadas.**
Qué pasa: una tabla `Order` o una columna `User` que obligan a escribir `[Order]` y que fallan en herramientas que no delimitan.
Por qué: el nombre "natural" en inglés coincide con SQL.
Arreglo: `Pedido`, `SalesOrder`, `AppUser`. En SQL Server, la lista está en la documentación de *Reserved Keywords*.

**3. Identificadores con comillas en PostgreSQL.**
Qué pasa: una tabla creada como `"DetallePedido"` que solo funciona si **siempre** se escribe con comillas y con esas mayúsculas exactas.
Por qué: PostgreSQL pasa a minúsculas los identificadores sin comillas.
Arreglo: `snake_case` en minúsculas y sin comillas.

**4. `Id` en todas las tablas sin disciplina en los alias.**
Qué pasa: `JOIN Pedido AS p ON p.Id = c.Id` compila y une clientes con pedidos que tienen el mismo número.
Por qué: todas las claves se llaman igual.
Arreglo: `NombreTabla + Id`, o al menos alias claros y revisión cuidadosa de cada `ON`.

**5. Restricciones sin nombre.**
Qué pasa: nombres generados como `FK__Pedido__Client__3B75D760`, distintos en cada entorno, que rompen las migraciones.
Por qué: se omite `CONSTRAINT Nombre`.
Arreglo: nombra todas las restricciones con el patrón de la convención.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Convención habitual | `PascalCase` | `snake_case` | `snake_case` |
| Identificadores sin comillas | Se conservan tal cual | Se pasan a **minúsculas** | Se conservan; la sensibilidad a mayúsculas depende del sistema operativo y de `lower_case_table_names` |
| Delimitador | `[Nombre]` (o `"Nombre"`) | `"Nombre"` | `` `Nombre` `` |
| ¿Distingue mayúsculas en nombres de tabla? | Según la intercalación (normalmente no) | Sí, si se usan comillas | Sí en Linux, no en Windows/macOS (por defecto) |
| Longitud máxima de un nombre | 128 caracteres | 63 bytes (se trunca en silencio) | 64 caracteres |
| Esquema por defecto | `dbo` | `public` | — (el esquema es la base) |

PostgreSQL trunca en silencio los nombres de más de 63 bytes: dos índices con nombres largos que solo difieren al final pueden terminar con el mismo nombre truncado y fallar al crearse.

La misma tabla en los tres motores:

```sql
-- SQL Server
CREATE TABLE dbo.DetallePedido (PedidoId INT, ProductoId INT, PrecioUnitario DECIMAL(10,2));

-- PostgreSQL
CREATE TABLE public.detalle_pedido (pedido_id integer, producto_id integer, precio_unitario numeric(10,2));

-- MySQL
CREATE TABLE detalle_pedido (pedido_id INT, producto_id INT, precio_unitario DECIMAL(10,2));
```

### En MongoDB

Las colecciones suelen nombrarse en **plural** y en `camelCase` o minúsculas (`pedidos`, `detallePedidos`), y los campos en `camelCase` (`fechaPedido`, `clienteId`), siguiendo la convención de JavaScript. El campo `_id` es fijo. Como los nombres de los campos se guardan **dentro de cada documento**, los nombres muy largos ocupan espacio en cada uno; no es motivo para abreviar hasta lo ilegible, pero sí para evitar nombres innecesariamente extensos en colecciones enormes.

-----

## Cuándo sí y cuándo no

**Define la convención cuando:**

* Empiezas una base nueva: antes de la primera tabla, y por escrito en el repositorio.

**Ten en cuenta:**

* En una base existente, la consistencia con lo que ya hay vale más que la convención "ideal". Si todas las tablas son plurales, la nueva también.
* Renombrar en producción es caro y riesgoso: hazlo solo con un plan (vistas de transición, migración de consultas, ventana de cambio).
* Si el ORM tiene otra convención por defecto (EF Core usa `Id` y nombres plurales de `DbSet`), configúralo para que siga la de la base, o adopta la del ORM desde el principio. Lo que no funciona es una mezcla.

**Herramientas que ayudan:**

* En EF Core, `ToTable("Cliente")`, `HasKey(c => c.ClienteId)` y `HasConstraintName(...)` alinean el modelo con la convención; el paquete `EFCore.NamingConventions` aplica `snake_case` automáticamente para PostgreSQL.
* Linters de SQL como SQLFluff pueden verificar mayúsculas y estilo en los scripts.

-----

## Resumen en 5 líneas

1. Lo importante de una convención es que permita **adivinar** los nombres; cuál elijas importa menos que aplicarla siempre.
2. Este wiki: tablas singulares en PascalCase, claves `NombreTabla + Id` y FK con el mismo nombre que la PK que referencian.
3. PascalCase en SQL Server; `snake_case` en PostgreSQL (pasa los nombres a minúsculas) y en MySQL (distingue mayúsculas en Linux).
4. Restricciones e índices con prefijos: `PK_`, `FK_`, `UQ_`, `CK_`, `DF_`, `IX_`, `UX_`.
5. Sin prefijos de tipo (`tbl_`, `str`), sin abreviaturas, sin palabras reservadas, sin espacios, sin `sp_` y en un solo idioma.

-----

## Para profundizar

<details>
<summary>Renombrar objetos en SQL Server sin romper nada</summary>

`sp_rename` cambia el nombre de una tabla o columna, pero **no** actualiza las vistas, los procedimientos ni las funciones que la usan: quedan apuntando al nombre viejo y fallan al ejecutarse. Una estrategia segura:

1. Buscar las dependencias: `SELECT * FROM sys.sql_expression_dependencies WHERE referenced_entity_name = 'tbl_Clientes';`.
2. Crear una vista o un sinónimo con el nombre nuevo (`CREATE SYNONYM dbo.Cliente FOR dbo.tbl_Clientes;`).
3. Migrar consultas, procedimientos y código al nombre nuevo.
4. Renombrar la tabla real y eliminar el sinónimo.

</details>

<details>
<summary>Nombres en inglés o en español</summary>

Muchas empresas exigen inglés en el código y en la base, porque el equipo es internacional y porque las palabras clave de SQL ya están en inglés. En un proyecto local, el español acerca la base al lenguaje del negocio (`DetallePedido` es lo que dice el área de ventas). Ambas opciones son válidas; mezclar no lo es. Si eliges inglés, ojo con las palabras reservadas: `Order`, `User`, `Group`, `Key`, `Index` y `Transaction` son nombres de entidad frecuentes que chocan con SQL.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso nombres claros y consistentes: tablas en singular y en PascalCase, la clave primaria como `NombreTablaId` y las claves foráneas con el mismo nombre que la clave a la que apuntan. Evito abreviaturas, prefijos como `tbl_` y palabras reservadas como `Order`. Lo más importante es que todo el equipo siga la misma convención.

### Respuesta ampliada (semi-senior)

La convención busca que cualquier nombre se pueda deducir, porque en una base renombrar es caro: afecta consultas, vistas, procedimientos, ORM y ETL. En SQL Server uso PascalCase, tablas en singular, clave `NombreTablaId` y FK con el mismo nombre que la PK que referencian, lo que hace los joins legibles y evita columnas `Id` ambiguas en los resultados. Nombro todas las restricciones e índices con prefijos (`PK_`, `FK_`, `UQ_`, `CK_`, `DF_`, `IX_`) para tener errores claros y migraciones reproducibles. Evito prefijos de tipo, abreviaturas, palabras reservadas, espacios y el prefijo `sp_`. Adapto la convención al motor: en PostgreSQL uso `snake_case` porque pasa los identificadores sin comillas a minúsculas, y en MySQL cuido la sensibilidad a mayúsculas en Linux. En una base existente priorizo la consistencia con lo que ya hay, y si el ORM tiene otra convención, lo configuro para que siga la de la base.

### Preguntas frecuentes de seguimiento

**1. ¿Tablas en singular o en plural?**
Cualquiera, mientras sea consistente. Singular tiene la ventaja de describir qué es cada fila y de evitar plurales irregulares.

**2. ¿Por qué no usar `snake_case` en SQL Server o `PascalCase` en PostgreSQL?**
En SQL Server es posible, solo que menos habitual. En PostgreSQL, `PascalCase` obliga a usar comillas en cada referencia, porque los identificadores sin comillas se pasan a minúsculas.

**3. ¿Por qué no usar el prefijo `sp_` en los procedimientos?**
Porque SQL Server busca primero los procedimientos `sp_` en la base `master`, lo que agrega una búsqueda innecesaria y puede ejecutar uno del sistema con el mismo nombre.

**4. ¿Qué haces si heredas una base sin convención?**
Para los objetos nuevos, sigo la convención predominante de la base; si se decide renombrar, lo hago por etapas, con sinónimos o vistas de transición.

-----

## Práctica

**Ejercicio 1.** Renombra estos objetos siguiendo la convención de este wiki y explica cada cambio:

```text
tblUsuarios (UserID, usr_name, pwd, fecAlta, isAdmin)
Order       (id, user, total, dt)
```

<details>
<summary>Solución</summary>

```text
Usuario (UsuarioId, Nombre, HashContrasena, FechaAlta, EsAdministrador)
Pedido  (PedidoId, UsuarioId, Total, FechaPedido)
```

* `tblUsuarios` → `Usuario`: sin prefijo, singular, en español como el resto.
* `UserID` → `UsuarioId`: un solo idioma, patrón `NombreTabla + Id`.
* `usr_name` → `Nombre`: sin abreviatura ni prefijo de tabla.
* `pwd` → `HashContrasena`: sin abreviatura, y además deja claro que **no** es la contraseña en texto plano (que nunca se guarda).
* `fecAlta` → `FechaAlta`; `isAdmin` → `EsAdministrador`: sin abreviaturas, un idioma, booleano como afirmación.
* `Order` → `Pedido`: evita la palabra reservada.
* `id` → `PedidoId`; `user` → `UsuarioId` (otra palabra reservada, y además es una FK: se llama igual que la PK de `Usuario`).
* `dt` → `FechaPedido`: dice qué fecha es.
* `Total` se conserva, aunque en el diseño lógico es un derivado: si existe, debería ser una snapshot o una desnormalización justificada (ver [Desnormalización](08-Desnormalización.md)).

</details>

**Ejercicio 2.** Escribe la tabla `Vuelo` (origen y destino son aeropuertos) en SQL Server, con nombres explícitos para todas las restricciones.

<details>
<summary>Solución</summary>

```sql
CREATE TABLE dbo.Aeropuerto (
    CodigoAeropuerto CHAR(3)      NOT NULL,   -- código IATA: LIM, CUZ
    Nombre           NVARCHAR(100) NOT NULL,
    CONSTRAINT PK_Aeropuerto PRIMARY KEY (CodigoAeropuerto)
);

CREATE TABLE dbo.Vuelo (
    VueloId                 INT IDENTITY(1,1) NOT NULL,
    CodigoAeropuertoOrigen  CHAR(3)           NOT NULL,
    CodigoAeropuertoDestino CHAR(3)           NOT NULL,
    FechaSalida             DATETIME2(0)      NOT NULL,
    CONSTRAINT PK_Vuelo PRIMARY KEY (VueloId),
    CONSTRAINT FK_Vuelo_Aeropuerto_Origen FOREIGN KEY (CodigoAeropuertoOrigen)
        REFERENCES dbo.Aeropuerto (CodigoAeropuerto),
    CONSTRAINT FK_Vuelo_Aeropuerto_Destino FOREIGN KEY (CodigoAeropuertoDestino)
        REFERENCES dbo.Aeropuerto (CodigoAeropuerto),
    CONSTRAINT CK_Vuelo_OrigenDistintoDestino CHECK (CodigoAeropuertoOrigen <> CodigoAeropuertoDestino)
);

CREATE INDEX IX_Vuelo_CodigoAeropuertoOrigen ON dbo.Vuelo (CodigoAeropuertoOrigen);
CREATE INDEX IX_Vuelo_CodigoAeropuertoDestino ON dbo.Vuelo (CodigoAeropuertoDestino);
```

Cuando hay dos FK hacia la misma tabla, la regla "la FK se llama igual que la PK" no alcanza: se agrega el **rol** (`Origen`, `Destino`) al nombre de la columna y al de la restricción.

</details>

-----

## Siguiente lección

Con esto termina el módulo de diseño. El resultado de todo el módulo es el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md), que usan los módulos siguientes. El próximo módulo, **Consultas**, está planificado.
