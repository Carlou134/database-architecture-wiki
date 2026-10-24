# Modificar datos

## En una frase

`INSERT` agrega filas, `UPDATE` cambia valores y `DELETE` borra filas; las tres aceptan subconsultas y joins para decidir **qué** filas tocar, y como un error en ellas no se deshace con Ctrl+Z, se ejecutan con un método: primero un `SELECT` con el mismo filtro, después la modificación dentro de una transacción, y recién al final el `COMMIT`.

-----

## Antes de empezar

Conviene que ya sepas:

* `WHERE`, `JOIN` y subconsultas, de las lecciones anteriores del módulo.
* Restricciones, claves foráneas y `IDENTITY`, de [Fundamentos](../01-fundamentos/README.md).
* Que `F5` ejecuta solo lo seleccionado, de [Entorno de trabajo](00-Entorno%20de%20trabajo.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DML (Data Manipulation Language):** instrucciones que modifican datos: `INSERT`, `UPDATE`, `DELETE`, `MERGE`.
* **`OUTPUT`:** cláusula de SQL Server que devuelve las filas afectadas por una modificación, con sus valores antes (`deleted`) y después (`inserted`).
* **`@@ROWCOUNT`:** cantidad de filas afectadas por la última instrucción.
* **`TRUNCATE`:** vacía una tabla completa, mucho más rápido que un `DELETE`, con varias restricciones.
* **`MERGE`:** instrucción que inserta, actualiza o borra en una sola pasada, comparando dos conjuntos de filas.
* **Upsert:** "actualizar si existe, insertar si no".

-----

## El problema

Un viernes, alguien de soporte necesita marcar como entregado el pedido 2:

```sql
UPDATE dbo.Pedido
SET Estado = 'Entregado'
WHERE PedidoId = 2;
```

Pero en el editor tenía también otras líneas, seleccionó solo las dos primeras y presionó F5:

```sql
UPDATE dbo.Pedido
SET Estado = 'Entregado'
```

```text
(5 rows affected)
```

Los cinco pedidos quedaron entregados, incluido el cancelado. No hay papelera ni "deshacer". Si nadie anotó los estados originales, la única salida es restaurar una copia de seguridad, y perder todo lo que se hizo desde entonces.

Escribir un `UPDATE` es fácil. Lo que hay que aprender es a **ejecutarlo de forma segura**.

-----

## Cómo funciona

### `INSERT`: agregar filas

```sql
-- Una fila, con la lista de columnas
INSERT INTO dbo.Categoria (Nombre) VALUES (N'Deportes');

-- Varias filas en una sola instrucción
INSERT INTO dbo.Categoria (Nombre) VALUES (N'Juguetes'), (N'Jardín');

-- Usando los valores por defecto
INSERT INTO dbo.Pedido (ClienteId) VALUES (5);   -- FechaPedido y Estado salen de sus DEFAULT
```

Reglas:

* **Escribe siempre la lista de columnas.** `INSERT INTO dbo.Categoria VALUES (N'Deportes')` funciona hoy y se rompe (o inserta en la columna equivocada) el día que alguien agrega una columna.
* **No envíes la columna `IDENTITY`:** la genera la base.
* **Las columnas que no envías** toman su `DEFAULT`, o `NULL` si no tienen.

### `INSERT ... SELECT` y `SELECT ... INTO`

Para insertar el resultado de una consulta:

```sql
-- En una tabla que ya existe
INSERT INTO dbo.PedidoArchivado (PedidoId, ClienteId, FechaPedido, Estado)
SELECT PedidoId, ClienteId, FechaPedido, Estado
FROM dbo.Pedido
WHERE Estado = 'Cancelado';

-- Creando la tabla a partir del resultado
SELECT PedidoId, ClienteId, FechaPedido, Estado
INTO dbo.PedidoRespaldo
FROM dbo.Pedido;
```

`SELECT ... INTO` crea la tabla con las columnas y los tipos del resultado, pero **sin** clave primaria, índices, restricciones ni valores por defecto. Es útil para copias rápidas y tablas temporales (`INTO #Temporal`), no para crear tablas definitivas.

### Recuperar lo que se insertó: `OUTPUT`

```sql
INSERT INTO dbo.Categoria (Nombre)
OUTPUT inserted.CategoriaId, inserted.Nombre
VALUES (N'Mascotas'), (N'Oficina');
```

| CategoriaId | Nombre |
| --- | --- |
| 4 | Mascotas |
| 5 | Oficina |

(Los IDs corresponden a ejecutar solo esta instrucción sobre el esquema de ejemplo, que tiene 3 categorías.)

`OUTPUT` devuelve las filas afectadas, con los IDs generados. A diferencia de `SCOPE_IDENTITY()`, funciona con inserciones de varias filas (ver [Claves primarias](../01-fundamentos/04-Claves%20primarias.md)).

### `UPDATE`: cambiar valores

```sql
UPDATE dbo.Producto
SET Precio = Precio * 1.10,
    Stock  = Stock + 5
WHERE CategoriaId = 3;
```

* Se pueden cambiar varias columnas, separadas por comas.
* El lado derecho del `=` ve los valores **anteriores** a la actualización: `SET A = B, B = A` intercambia los dos valores.
* **Sin `WHERE`, actualiza todas las filas.**

`ALTER` y `UPDATE` se confunden a menudo en los apuntes: `ALTER TABLE` cambia la **estructura** (agrega una columna, cambia un tipo; es DDL), mientras que `UPDATE` cambia los **datos** de las filas existentes (es DML).

### `UPDATE` con `JOIN`

Para actualizar usando datos de otra tabla, T-SQL tiene la forma `UPDATE ... FROM`:

```sql
-- Desactivar los productos de la categoría "Hogar" que no tienen stock
UPDATE p
SET p.Activo = 0
FROM dbo.Producto AS p
JOIN dbo.Categoria AS c ON c.CategoriaId = p.CategoriaId
WHERE c.Nombre = N'Hogar'
  AND p.Stock = 0;
```

Después de `UPDATE` va el **alias** de la tabla que se modifica, y el `FROM` con sus joins funciona igual que en un `SELECT`. Eso permite el método más seguro: escribir primero el `SELECT` con el mismo `FROM` y `WHERE`, revisar las filas, y recién después cambiar `SELECT ...` por `UPDATE p SET ...`.

Ojo: si el join empareja una fila con **varias** filas de la otra tabla, SQL Server actualiza la fila con **uno** de los valores, sin error y sin decir cuál. Antes de un `UPDATE` con join, verifica que cada fila a modificar tenga una sola pareja.

### `DELETE`: borrar filas

```sql
DELETE FROM dbo.DetallePedido
WHERE PedidoId = 4 AND ProductoId = 5;
```

Y con un join, con la misma forma que el `UPDATE`:

```sql
-- Borrar los detalles de los pedidos cancelados
DELETE d
FROM dbo.DetallePedido AS d
JOIN dbo.Pedido AS p ON p.PedidoId = d.PedidoId
WHERE p.Estado = 'Cancelado';
```

Las claves foráneas siguen vigentes: no se puede borrar una fila que otra tabla referencia, salvo que la FK tenga `ON DELETE CASCADE` (ver [Claves foráneas](../01-fundamentos/05-Claves%20foráneas%20e%20integridad%20referencial.md)).

Para borrar solo una cantidad de filas (por ejemplo, para procesar una tabla grande en lotes sin bloquearla entera):

```sql
DELETE TOP (1000) FROM dbo.Log WHERE Fecha < '2025-01-01';
```

`TOP` en un `DELETE` no admite `ORDER BY`: borra 1000 filas **cualesquiera** de las que cumplen el filtro. Se repite hasta que `@@ROWCOUNT` sea 0.

### `DELETE` frente a `TRUNCATE`

| | `DELETE FROM Tabla` | `TRUNCATE TABLE Tabla` |
| --- | --- | --- |
| Admite `WHERE` | Sí | No: vacía la tabla completa |
| Velocidad en tablas grandes | Lento: registra cada fila en el log | Muy rápido: libera las páginas de datos |
| Reinicia el `IDENTITY` | No | **Sí**, vuelve al valor inicial |
| Funciona si otra tabla la referencia con una FK | Sí (si no hay filas hijas) | **No**, aunque la tabla hija esté vacía |
| Dispara *triggers* | Sí | No |
| Se puede deshacer con `ROLLBACK` | Sí | Sí, en SQL Server (dentro de una transacción) |
| Tipo de instrucción | DML | DDL (requiere más permisos) |

```text
Msg 4712, Level 16, State 1
Cannot truncate table 'dbo.Cliente' because it is being referenced by a FOREIGN KEY constraint.
```

`TRUNCATE` es para tablas de *staging* o de pruebas que se vacían por completo. En tablas de negocio, casi nunca.

### La transacción como red de seguridad

El método para cualquier modificación que no sea trivial:

```sql
-- 1. Ver exactamente qué se va a tocar
SELECT PedidoId, Estado
FROM dbo.Pedido
WHERE PedidoId = 2;

-- 2. Modificar dentro de una transacción
BEGIN TRANSACTION;

UPDATE dbo.Pedido
SET Estado = 'Entregado'
OUTPUT deleted.Estado AS Antes, inserted.Estado AS Despues
WHERE PedidoId = 2;

SELECT @@ROWCOUNT AS FilasAfectadas;   -- ¿es la cantidad que esperaba?

-- 3. Decidir
-- ROLLBACK;   -- si algo no cuadra
-- COMMIT;     -- si todo está bien
```

| Antes | Despues |
| --- | --- |
| Enviado | Entregado |

* **El `SELECT` previo** usa el mismo `WHERE`: si devuelve 5 filas cuando esperabas 1, el filtro está mal.
* **`OUTPUT deleted.*, inserted.*`** muestra los valores de antes y de después de cada fila modificada. Si hay que deshacer más tarde, ahí están los valores originales.
* **`@@ROWCOUNT`** confirma la cantidad de filas. Se lee **inmediatamente** después de la instrucción, porque cualquier instrucción siguiente lo reemplaza.
* **Hasta el `COMMIT`**, el cambio se puede deshacer con `ROLLBACK`. Mientras la transacción está abierta, las filas modificadas quedan **bloqueadas** para otras sesiones, así que no la dejes abierta mientras vas a buscar un café.

Las transacciones, los bloqueos y los niveles de aislamiento se estudian a fondo en el módulo de transacciones y concurrencia.

### `MERGE`: insertar o actualizar en una pasada

`MERGE` compara una tabla de destino con un origen y decide, fila por fila, qué hacer:

```sql
-- Sincronizar el stock con un archivo del proveedor
MERGE dbo.Producto AS destino
USING (VALUES (1, 20), (2, 35)) AS origen (ProductoId, StockNuevo)
   ON destino.ProductoId = origen.ProductoId
WHEN MATCHED THEN
    UPDATE SET destino.Stock = origen.StockNuevo
WHEN NOT MATCHED BY TARGET THEN
    INSERT (CategoriaId, Nombre, Precio, Stock) VALUES (1, N'Producto nuevo', 0, origen.StockNuevo)
OUTPUT $action, inserted.ProductoId, inserted.Stock;
```

Parece la solución ideal para un *upsert*, pero tiene historial de errores conocidos en SQL Server y, sin las sugerencias de bloqueo correctas, condiciones de carrera con escrituras concurrentes. Muchos equipos prefieren dos instrucciones explícitas:

```sql
BEGIN TRANSACTION;

UPDATE dbo.Producto WITH (UPDLOCK, SERIALIZABLE)
SET Stock = 20
WHERE ProductoId = 1;

IF @@ROWCOUNT = 0
    INSERT INTO dbo.Producto (CategoriaId, Nombre, Precio, Stock) VALUES (1, N'Producto nuevo', 0, 20);

COMMIT;
```

Las sugerencias `UPDLOCK, SERIALIZABLE` impiden que otra sesión inserte la misma fila entre el `UPDATE` y el `INSERT`. Si usas `MERGE`, termínalo con `;` (es obligatorio) y conoce sus limitaciones antes de usarlo en producción.

-----

## Ejemplo completo

Un ajuste de precios: subir un 5 % todos los productos activos de Electrónica que se vendieron al menos una vez en 2026, guardando un registro de los cambios.

```sql
-- Tabla de auditoría (una vez)
CREATE TABLE dbo.CambioPrecio (
    ProductoId    INT           NOT NULL,
    PrecioAntes   DECIMAL(10,2) NOT NULL,
    PrecioDespues DECIMAL(10,2) NOT NULL,
    FechaCambio   DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME()
);

-- 1. Ver qué se va a tocar
SELECT p.ProductoId, p.Nombre, p.Precio, CAST(p.Precio * 1.05 AS DECIMAL(10,2)) AS PrecioNuevo
FROM dbo.Producto AS p
JOIN dbo.Categoria AS c ON c.CategoriaId = p.CategoriaId
WHERE c.Nombre = N'Electrónica'
  AND p.Activo = 1
  AND EXISTS (SELECT 1
              FROM dbo.DetallePedido AS d
              JOIN dbo.Pedido AS pe ON pe.PedidoId = d.PedidoId
              WHERE d.ProductoId = p.ProductoId
                AND pe.FechaPedido >= '2026-01-01' AND pe.FechaPedido < '2027-01-01');
```

| ProductoId | Nombre | Precio | PrecioNuevo |
| --- | --- | --- | --- |
| 1 | Teclado mecánico | 120.00 | 126.00 |
| 2 | Mouse inalámbrico | 45.50 | 47.78 |
| 3 | Monitor 27 pulgadas | 899.90 | 944.90 |

```sql
-- 2. Modificar y auditar en la misma transacción
BEGIN TRANSACTION;

UPDATE p
SET p.Precio = CAST(p.Precio * 1.05 AS DECIMAL(10,2))
OUTPUT inserted.ProductoId, deleted.Precio, inserted.Precio
INTO dbo.CambioPrecio (ProductoId, PrecioAntes, PrecioDespues)
FROM dbo.Producto AS p
JOIN dbo.Categoria AS c ON c.CategoriaId = p.CategoriaId
WHERE c.Nombre = N'Electrónica'
  AND p.Activo = 1
  AND EXISTS (SELECT 1
              FROM dbo.DetallePedido AS d
              JOIN dbo.Pedido AS pe ON pe.PedidoId = d.PedidoId
              WHERE d.ProductoId = p.ProductoId
                AND pe.FechaPedido >= '2026-01-01' AND pe.FechaPedido < '2027-01-01');

SELECT @@ROWCOUNT AS FilasAfectadas;   -- 3

SELECT ProductoId, PrecioAntes, PrecioDespues FROM dbo.CambioPrecio;

-- 3. Si las 3 filas son las esperadas:
COMMIT;
```

| ProductoId | PrecioAntes | PrecioDespues |
| --- | --- | --- |
| 1 | 120.00 | 126.00 |
| 2 | 45.50 | 47.78 |
| 3 | 899.90 | 944.90 |

Qué decisión hay detrás:

* **El `SELECT` y el `UPDATE` comparten el `FROM` y el `WHERE`:** se probó el filtro antes de modificar, y se pasó de uno al otro cambiando solo la primera línea.
* **`EXISTS` en lugar de un `JOIN` con `DetallePedido`:** un producto vendido en varios pedidos generaría varias parejas, y el `UPDATE` con join quedaría ambiguo. `EXISTS` solo pregunta si hay al menos una venta.
* **`CAST(... AS DECIMAL(10,2))`:** el precio con 5 % tiene más decimales; se redondea explícitamente a 2 (45.50 × 1.05 = 47.775 → 47.78).
* **`OUTPUT ... INTO`:** la auditoría se escribe en la misma instrucción, con los valores de antes y de después, sin una consulta aparte que pueda desincronizarse.
* **El `DetallePedido` no cambia:** los pedidos ya hechos conservan el precio pagado, gracias al diseño de [Proceso de diseño](../02-diseno-y-modelado/01-Proceso%20de%20diseño.md).

-----

## Errores comunes

**1. `UPDATE` o `DELETE` sin `WHERE`.**
Qué pasa: se modifican todas las filas.
Por qué: el `WHERE` quedó fuera de la selección, o se olvidó.
Arreglo: primero el `SELECT` con el mismo filtro; después, la modificación dentro de una transacción, verificando `@@ROWCOUNT` antes del `COMMIT`.

**2. `INSERT` sin lista de columnas.**
Qué pasa: falla, o inserta valores en las columnas equivocadas, cuando la tabla cambia.
Por qué: los valores se asignan por posición.
Arreglo: `INSERT INTO Tabla (col1, col2) VALUES (...)`, siempre.

**3. `UPDATE` con un join que empareja varias filas.**
Qué pasa: la fila se actualiza con un valor arbitrario de los posibles, sin error.
Por qué: SQL Server no exige que el join sea uno a uno.
Arreglo: verifica que cada fila tenga una sola pareja, o usa `EXISTS` o una subconsulta agregada.

**4. Usar `TRUNCATE` en una tabla referenciada.**
Qué pasa: `Msg 4712: Cannot truncate table ... because it is being referenced by a FOREIGN KEY constraint`.
Por qué: `TRUNCATE` no comprueba fila por fila, así que no se permite si hay FK que apunten a la tabla.
Arreglo: `DELETE`, o eliminar y recrear las FK (en entornos de prueba).

**5. Dejar una transacción abierta.**
Qué pasa: otras sesiones quedan bloqueadas esperando las filas modificadas, y la aplicación "se cuelga".
Por qué: se ejecutó `BEGIN TRANSACTION` y la modificación, pero ni `COMMIT` ni `ROLLBACK`.
Arreglo: cierra siempre la transacción; `SELECT @@TRANCOUNT;` muestra si tienes una abierta.

**6. Leer `@@ROWCOUNT` tarde.**
Qué pasa: devuelve 1 (o 0) en lugar de las filas modificadas.
Por qué: cualquier instrucción posterior, incluido un `SELECT` o un `IF`, lo reemplaza.
Arreglo: léelo inmediatamente, o guárdalo en una variable (`SET @filas = @@ROWCOUNT;`).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Devolver las filas modificadas | `OUTPUT inserted.*, deleted.*` | `RETURNING *` | `RETURNING` no existe (MariaDB sí lo tiene); usar `LAST_INSERT_ID()` |
| `UPDATE` con join | `UPDATE a SET ... FROM a JOIN b ...` | `UPDATE a SET ... FROM b WHERE a.x = b.x` | `UPDATE a JOIN b ON ... SET ...` |
| `DELETE` con join | `DELETE a FROM a JOIN b ...` | `DELETE FROM a USING b WHERE ...` | `DELETE a FROM a JOIN b ...` |
| Upsert | `MERGE`, o `UPDATE` + `INSERT` | `INSERT ... ON CONFLICT (...) DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` |
| `MERGE` | Sí (con advertencias) | Sí, desde la 15 | No |
| Borrar en lotes | `DELETE TOP (n)` | `DELETE ... WHERE ctid IN (SELECT ctid ... LIMIT n)` | `DELETE ... LIMIT n` |
| `TRUNCATE` dentro de una transacción | Se puede deshacer | Se puede deshacer | **No**: hace un `COMMIT` implícito |
| Crear tabla desde una consulta | `SELECT ... INTO nueva` | `CREATE TABLE nueva AS SELECT ...` | `CREATE TABLE nueva AS SELECT ...` |

El *upsert* es donde más difieren. `INSERT ... ON CONFLICT` de PostgreSQL es atómico y no tiene las condiciones de carrera de `MERGE`; es una de las razones por las que el *upsert* es más simple en PostgreSQL.

### En MongoDB

| SQL | MongoDB |
| --- | --- |
| `INSERT` | `insertOne`, `insertMany` |
| `UPDATE ... WHERE` | `updateOne`, `updateMany` con operadores como `$set` e `$inc` |
| `DELETE ... WHERE` | `deleteOne`, `deleteMany` |
| Upsert | `updateOne(filtro, cambio, { upsert: true })` |
| `OUTPUT` / `RETURNING` | `findOneAndUpdate` (con `returnDocument: "before"` o `"after"`) |

```javascript
db.productos.updateMany(
  { categoriaId: 3 },
  { $mul: { precio: NumberDecimal("1.10") }, $inc: { stock: 5 } }
);
```

El mismo peligro existe: `db.pedidos.updateMany({}, { $set: { estado: "Entregado" } })` (filtro vacío) actualiza todos los documentos.

-----

## Cuándo sí y cuándo no

**Usa una transacción explícita cuando:**

* Modificas datos a mano en un entorno compartido (producción, pruebas con datos reales): siempre.
* Varias modificaciones tienen que aplicarse juntas o no aplicarse.

**Usa `TRUNCATE` cuando:**

* Vacías una tabla de *staging* o de pruebas, completa, y no hay claves foráneas que la referencien.

**Ten en cuenta:**

* Antes de un cambio grande en producción, verifica que haya una copia de seguridad reciente, y si el cambio es crítico, guarda los valores originales (con `OUTPUT ... INTO` o con un `SELECT ... INTO` previo).
* En el código de una aplicación, las modificaciones van parametrizadas (nunca concatenando texto del usuario), para evitar la inyección de SQL.

-----

## Resumen en 5 líneas

1. `INSERT` con la lista de columnas siempre; `INSERT ... SELECT` copia resultados; `OUTPUT inserted.*` devuelve los IDs generados.
2. `UPDATE` y `DELETE` sin `WHERE` afectan todas las filas; con `FROM ... JOIN` usan otras tablas, cuidando que el join sea uno a uno.
3. Método seguro: `SELECT` con el mismo filtro → `BEGIN TRANSACTION` → modificación con `OUTPUT` → `@@ROWCOUNT` → `COMMIT` o `ROLLBACK`.
4. `TRUNCATE` vacía todo, rápido, reinicia el `IDENTITY` y no funciona con tablas referenciadas por una FK.
5. `MERGE` hace *upserts* en una pasada, con advertencias conocidas; la alternativa es `UPDATE` + `INSERT` con `UPDLOCK, SERIALIZABLE`.

-----

## Para profundizar

<details>
<summary>Procesar tablas grandes en lotes</summary>

Un `DELETE` de 50 millones de filas en una sola instrucción genera una transacción enorme: llena el log, bloquea la tabla y, si falla, el `ROLLBACK` tarda tanto como el borrado. La alternativa es un bucle de lotes:

```sql
DECLARE @filas INT = 1;

WHILE @filas > 0
BEGIN
    DELETE TOP (5000) FROM dbo.Log WHERE Fecha < '2025-01-01';
    SET @filas = @@ROWCOUNT;
END;
```

Cada lote es una transacción corta: los bloqueos se liberan entre lotes y el log se puede reutilizar. Un índice sobre `Fecha` hace que cada lote encuentre sus filas sin recorrer la tabla.

</details>

<details>
<summary>Por qué MERGE tiene mala fama</summary>

`MERGE` acumuló a lo largo de los años varios errores documentados en SQL Server (con índices filtrados, vistas indexadas, *triggers* y claves foráneas, entre otros), muchos ya corregidos, y además no es atómico frente a la concurrencia salvo que se use con `HOLDLOCK` (equivalente a `SERIALIZABLE`): dos sesiones pueden comprobar al mismo tiempo que la fila no existe e intentar insertarla las dos. Aaron Bertrand publicó un artículo muy citado, "Use Caution with SQL Server's MERGE Statement", con la lista. Por eso la recomendación habitual es usarlo para cargas por lotes controladas y preferir `UPDATE` + `INSERT` explícitos en código transaccional concurrente.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`INSERT` agrega filas, `UPDATE` modifica valores y `DELETE` borra filas; las dos últimas necesitan un `WHERE`, si no afectan a toda la tabla. `TRUNCATE` vacía la tabla completa más rápido que `DELETE` y reinicia el `IDENTITY`. Antes de modificar datos importantes, uso una transacción para poder deshacer con `ROLLBACK`.

### Respuesta ampliada (semi-senior)

Escribo los `INSERT` con la lista de columnas y uso `OUTPUT` para recuperar los IDs generados, incluso en inserciones de varias filas. Para `UPDATE` y `DELETE` con datos de otras tablas uso la forma `FROM ... JOIN` de T-SQL, verificando que el join sea uno a uno, porque si una fila tiene varias parejas el valor aplicado es arbitrario; si no, uso `EXISTS`. Las modificaciones manuales las hago con un método: el `SELECT` con el mismo filtro, la modificación dentro de una transacción con `OUTPUT deleted/inserted`, la verificación de `@@ROWCOUNT` y recién entonces el `COMMIT`, sin dejar la transacción abierta para no bloquear a otras sesiones. Distingo `DELETE` de `TRUNCATE`: el segundo es DDL, no admite `WHERE`, reinicia el `IDENTITY` y no funciona con tablas referenciadas por una FK. Para *upserts* conozco `MERGE` y sus problemas de concurrencia y errores históricos, y en código concurrente prefiero `UPDATE` + `INSERT` con `UPDLOCK, SERIALIZABLE`. Los borrados masivos los hago en lotes.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `DELETE` y `TRUNCATE`?**
`DELETE` borra fila por fila, admite `WHERE`, dispara *triggers* y no reinicia el `IDENTITY`. `TRUNCATE` vacía la tabla completa liberando páginas, reinicia el `IDENTITY` y no funciona si otra tabla la referencia con una FK.

**2. ¿Qué diferencia hay entre `ALTER` y `UPDATE`?**
`ALTER TABLE` cambia la estructura de la tabla (DDL); `UPDATE` cambia los datos de las filas (DML).

**3. ¿Cómo obtienes los IDs de las filas que insertaste?**
Con `OUTPUT inserted.Id`, que funciona con varias filas; o con `SCOPE_IDENTITY()` para una sola.

**4. ¿Cómo harías un *upsert* en SQL Server?**
Con `MERGE`, sabiendo sus limitaciones, o con un `UPDATE` y, si `@@ROWCOUNT` es 0, un `INSERT`, dentro de una transacción con `UPDLOCK, SERIALIZABLE`.

-----

## Práctica

**Ejercicio 1.** Escribe, con el método seguro, la modificación para cancelar el pedido 5 y devolver al stock las unidades de sus productos.

<details>
<summary>Solución</summary>

```sql
-- 1. Ver qué se va a tocar
SELECT d.ProductoId, d.Cantidad, p.Stock
FROM dbo.DetallePedido AS d
JOIN dbo.Producto AS p ON p.ProductoId = d.ProductoId
WHERE d.PedidoId = 5;

-- 2. Modificar, todo junto
BEGIN TRANSACTION;

UPDATE dbo.Pedido
SET Estado = 'Cancelado'
OUTPUT deleted.Estado AS Antes, inserted.Estado AS Despues
WHERE PedidoId = 5 AND Estado <> 'Cancelado';

UPDATE p
SET p.Stock = p.Stock + d.Cantidad
OUTPUT inserted.ProductoId, deleted.Stock AS StockAntes, inserted.Stock AS StockDespues
FROM dbo.Producto AS p
JOIN dbo.DetallePedido AS d ON d.ProductoId = p.ProductoId
WHERE d.PedidoId = 5;

-- 3. Revisar las salidas de OUTPUT y decidir
-- COMMIT;  o  ROLLBACK;
```

| Antes | Despues |
| --- | --- |
| Pendiente | Cancelado |

| ProductoId | StockAntes | StockDespues |
| --- | --- | --- |
| 2 | 40 | 41 |
| 6 | 12 | 13 |

Las dos instrucciones van en la **misma** transacción: si el pedido se cancela pero el stock no se devuelve (o al revés), los datos quedan inconsistentes. El `AND Estado <> 'Cancelado'` evita cancelar dos veces y devolver el stock dos veces si el script se ejecuta de nuevo. El join del segundo `UPDATE` es uno a uno, porque `(PedidoId, ProductoId)` es la clave de `DetallePedido`.

</details>

**Ejercicio 2.** ¿Qué hace este script, y qué está mal?

```sql
BEGIN TRANSACTION;
DELETE FROM dbo.DetallePedido WHERE PedidoId = 4;
SELECT * FROM dbo.DetallePedido WHERE PedidoId = 4;
SELECT @@ROWCOUNT AS FilasBorradas;
```

<details>
<summary>Solución</summary>

Borra el detalle del pedido 4 (la cafetera), pero tiene dos problemas:

1. **`@@ROWCOUNT` se lee tarde:** devuelve las filas del `SELECT` anterior (0, porque ya no hay detalles), no las del `DELETE` (1). Hay que leerlo inmediatamente después del `DELETE`, o guardarlo en una variable.
2. **La transacción queda abierta:** no hay `COMMIT` ni `ROLLBACK`. La fila borrada queda bloqueada para las demás sesiones hasta que se cierre la transacción (o se cierre la conexión, lo que hace un `ROLLBACK`).

Versión corregida:

```sql
BEGIN TRANSACTION;

DELETE FROM dbo.DetallePedido
OUTPUT deleted.*
WHERE PedidoId = 4;

SELECT @@ROWCOUNT AS FilasBorradas;   -- 1

ROLLBACK;   -- o COMMIT, después de revisar
```

</details>

-----

## Siguiente lección

[SQL con IA](12-SQL%20con%20IA.md)
