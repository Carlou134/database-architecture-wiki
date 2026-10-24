# Niveles de aislamiento y anomalías

## En una frase

Cuando varias transacciones trabajan a la vez, pueden aparecer **anomalías** (leer datos que nunca se confirmaron, ver un valor cambiar entre dos lecturas, ver aparecer filas nuevas, pisar el cambio de otro), y el **nivel de aislamiento** decide cuáles se permiten a cambio de más concurrencia: cuanto más aislamiento, menos anomalías y más esperas.

-----

## Antes de empezar

Conviene que ya sepas:

* Transacciones y ACID, de [Transacciones y ACID](01-Transacciones%20y%20ACID.md).
* El procedimiento `dbo.RegistrarPedido` de esa lección, que quedó con un problema de concurrencia pendiente.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Concurrencia:** varias sesiones leyendo y escribiendo los mismos datos al mismo tiempo.
* **Anomalía de concurrencia:** resultado incorrecto que solo aparece cuando dos transacciones se cruzan.
* **Lectura sucia (dirty read):** leer un cambio que otra transacción todavía no confirmó.
* **Lectura no repetible (non-repeatable read):** leer dos veces la misma fila y obtener valores distintos.
* **Lectura fantasma (phantom read):** repetir una consulta y obtener filas que antes no estaban.
* **Actualización perdida (lost update):** dos transacciones leen el mismo valor, lo modifican y la segunda escritura pisa a la primera.
* **Nivel de aislamiento:** configuración de la sesión que decide qué anomalías se permiten.

-----

## El problema

Quedan 8 monitores. Dos clientes compran uno cada uno, al mismo tiempo, con una versión del procedimiento que lee el stock, calcula y escribe:

| Momento | Sesión A (Ana) | Sesión B (Luis) |
| --- | --- | --- |
| t1 | `SELECT @s = Stock ... WHERE ProductoId = 3;` → 8 | |
| t2 | | `SELECT @s = Stock ... WHERE ProductoId = 3;` → 8 |
| t3 | `UPDATE ... SET Stock = @s - 1;` → 7 | |
| t4 | `COMMIT;` | |
| t5 | | `UPDATE ... SET Stock = @s - 1;` → 7 |
| t6 | | `COMMIT;` |

Se vendieron dos monitores y el stock dice 7. Cada transacción, sola, es correcta y atómica. Juntas, una **pisó** el cambio de la otra. No hubo ningún error, ningún bloqueo visible, ninguna advertencia.

Es una **actualización perdida**, y es solo una de las anomalías que aparecen cuando las transacciones se cruzan. El nivel de aislamiento por defecto de SQL Server no la impide.

-----

## Cómo funciona

### Las cuatro anomalías

**Lectura sucia:** leer algo que después se deshace.

| Momento | Sesión A | Sesión B |
| --- | --- | --- |
| t1 | `UPDATE Producto SET Precio = 1.00 WHERE ProductoId = 3;` (error de tipeo, sin confirmar) | |
| t2 | | `SELECT Precio ... WHERE ProductoId = 3;` → **1.00** |
| t3 | `ROLLBACK;` (el precio vuelve a 899.90) | |
| t4 | | Vende un monitor a 1.00 |

**Lectura no repetible:** la misma fila cambia entre dos lecturas de la misma transacción.

| Momento | Sesión A | Sesión B |
| --- | --- | --- |
| t1 | `SELECT Precio ... WHERE ProductoId = 3;` → 899.90 | |
| t2 | | `UPDATE ... SET Precio = 949.90 ...; COMMIT;` |
| t3 | `SELECT Precio ... WHERE ProductoId = 3;` → **949.90** | |

**Lectura fantasma:** aparecen filas nuevas que cumplen el filtro.

| Momento | Sesión A | Sesión B |
| --- | --- | --- |
| t1 | `SELECT COUNT(*) FROM Pedido WHERE Estado = 'Pendiente';` → 1 | |
| t2 | | `INSERT INTO Pedido (..., 'Pendiente'); COMMIT;` |
| t3 | `SELECT COUNT(*) FROM Pedido WHERE Estado = 'Pendiente';` → **2** | |

**Actualización perdida:** el problema del principio. Dos transacciones leen el mismo valor, calculan con él y escriben; la segunda escritura borra la primera.

### Los niveles de aislamiento

| Nivel | Lectura sucia | No repetible | Fantasma | Actualización perdida | Cómo lo logra |
| --- | --- | --- | --- | --- | --- |
| `READ UNCOMMITTED` | ⚠️ Posible | ⚠️ Posible | ⚠️ Posible | ⚠️ Posible | No pide bloqueos para leer |
| **`READ COMMITTED`** (por defecto) | ✅ Evitada | ⚠️ Posible | ⚠️ Posible | ⚠️ Posible | Bloqueo compartido mientras **lee** cada fila, y lo suelta enseguida |
| `REPEATABLE READ` | ✅ | ✅ Evitada | ⚠️ Posible | ✅ (se convierte en un *deadlock* en lugar de pisarse) | Mantiene los bloqueos de lectura **hasta el final** de la transacción |
| `SERIALIZABLE` | ✅ | ✅ | ✅ Evitada | ✅ | Además, bloquea **rangos** de claves, para que nadie inserte en ellos |
| `SNAPSHOT` | ✅ | ✅ | ✅ | ✅ (error de conflicto) | Versiones de filas: lee la foto del inicio de la transacción |
| `READ COMMITTED SNAPSHOT` (RCSI) | ✅ | ⚠️ Posible | ⚠️ Posible | ⚠️ Posible | Versiones: lee la foto del inicio de **cada instrucción** |

Los dos últimos, basados en versiones en lugar de bloqueos, se estudian en [Versionado con RCSI y SNAPSHOT](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md).

El nivel se define **por sesión**, y vale hasta que se cambie:

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Ver el nivel actual de la sesión
SELECT CASE transaction_isolation_level
           WHEN 1 THEN 'READ UNCOMMITTED' WHEN 2 THEN 'READ COMMITTED'
           WHEN 3 THEN 'REPEATABLE READ'  WHEN 4 THEN 'SERIALIZABLE'
           WHEN 5 THEN 'SNAPSHOT' END AS Nivel
FROM sys.dm_exec_sessions
WHERE session_id = @@SPID;
```

Y también se puede pedir para **una tabla** dentro de una consulta, con un hint: `WITH (NOLOCK)` equivale a `READ UNCOMMITTED`, `WITH (HOLDLOCK)` a `SERIALIZABLE` (ver [Bloqueos](03-Bloqueos%2C%20hints%20y%20NOLOCK.md)).

### Más aislamiento, más esperas

Cada nivel más estricto mantiene más bloqueos durante más tiempo. Con `SERIALIZABLE`, una transacción que leyó "los pedidos pendientes" impide que **cualquier** otra inserte un pedido pendiente hasta que termine. Eso evita los fantasmas, y también frena a todos los demás usuarios y aumenta los *deadlocks*.

No existe "el nivel correcto" para toda la base: el nivel por defecto (`READ COMMITTED`) es razonable para la mayoría de las consultas, y las operaciones que necesitan más garantías las piden **puntualmente**, en la transacción o en la fila que lo necesita.

### `READ COMMITTED` no es "consistente"

Una sorpresa frecuente: con el nivel por defecto, una consulta larga **no** ve una foto consistente de la tabla. Cada fila se lee con un bloqueo breve que se suelta enseguida, así que mientras la consulta recorre la tabla, otras transacciones pueden modificar filas que ya leyó o que todavía no leyó. Un `SUM` sobre una tabla que se actualiza mientras se calcula puede mezclar valores de antes y de después de un cambio.

Si un reporte necesita una foto consistente, el nivel adecuado es `SNAPSHOT` (o la consulta con RCSI activado, que da consistencia a nivel de instrucción).

### Cómo se evita la actualización perdida

La actualización perdida no se arregla subiendo el nivel de aislamiento de toda la base, sino escribiendo la operación de forma que **no** haya una ventana entre leer y escribir. Tres formas, de la más simple a la más general:

**1. Leer y escribir en una sola instrucción atómica.** El cálculo lo hace el `UPDATE`, sobre el valor actual de la fila, que la instrucción bloquea mientras la modifica:

```sql
UPDATE dbo.Producto
SET Stock = Stock - @Cantidad
WHERE ProductoId = @ProductoId
  AND Stock >= @Cantidad;           -- la condición de negocio, en la misma instrucción

IF @@ROWCOUNT = 0
    THROW 50002, N'Stock insuficiente.', 1;
```

Es la mejor opción cuando el cálculo cabe en un `UPDATE`.

**2. Bloquear al leer (bloqueo pesimista).** Si hace falta leer, decidir en el código y después escribir, la lectura pide un bloqueo de actualización que impide que otro lea con la misma intención hasta que la transacción termine:

```sql
BEGIN TRANSACTION;
    SELECT @Stock = Stock FROM dbo.Producto WITH (UPDLOCK, ROWLOCK) WHERE ProductoId = @ProductoId;
    -- lógica con @Stock
    UPDATE dbo.Producto SET Stock = @Stock - @Cantidad WHERE ProductoId = @ProductoId;
COMMIT;
```

La segunda sesión **espera** en el `SELECT` hasta que la primera confirme, y después lee el valor ya actualizado. Es el patrón bancario de tu cheat sheet de la sesión 1 (`SELECT SALDO ... WITH (UPDLOCK)`).

**3. Detectar el conflicto al escribir (concurrencia optimista).** No se bloquea nada al leer; al escribir, se verifica que la fila no haya cambiado desde que se leyó, usando una columna de versión:

```sql
ALTER TABLE dbo.Producto ADD Version ROWVERSION;   -- cambia sola en cada UPDATE

-- Al leer: SELECT Stock, Version ...  (la aplicación guarda la versión)
-- Al escribir:
UPDATE dbo.Producto
SET Stock = @StockNuevo
WHERE ProductoId = @ProductoId
  AND Version = @VersionLeida;       -- solo si nadie la modificó

IF @@ROWCOUNT = 0
    THROW 50003, N'El producto fue modificado por otro usuario. Vuelva a intentarlo.', 1;
```

Es el modelo que usan los ORM (en EF Core, una propiedad `[Timestamp]` o `IsRowVersion()`): ideal cuando los conflictos son raros y la lectura y la escritura están separadas por tiempo de usuario (un formulario que se edita durante minutos), donde mantener un bloqueo sería inaceptable.

| Estrategia | Bloquea al leer | Qué pasa en un conflicto | Ideal para |
| --- | --- | --- | --- |
| `UPDATE` atómico | No (la instrucción bloquea la fila mientras la modifica) | La condición del `WHERE` falla: 0 filas | Contadores, stock, saldos |
| Pesimista (`UPDLOCK`) | Sí, hasta el `COMMIT` | La segunda sesión espera | Lógica entre la lectura y la escritura, dentro de una transacción corta |
| Optimista (`ROWVERSION`) | No | La segunda escritura falla y se reintenta | Edición con tiempo de usuario de por medio |

-----

## Ejemplo completo

La compra concurrente del último monitor, y el procedimiento corregido.

**Antes** (lee, calcula y escribe):

```sql
CREATE OR ALTER PROCEDURE dbo.Comprar @ProductoId INT, @Cantidad INT
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        DECLARE @Stock INT = (SELECT Stock FROM dbo.Producto WHERE ProductoId = @ProductoId);
        IF @Stock < @Cantidad THROW 50002, N'Stock insuficiente.', 1;
        WAITFOR DELAY '00:00:05';   -- simula la lógica de la aplicación, para poder cruzar dos sesiones
        UPDATE dbo.Producto SET Stock = @Stock - @Cantidad WHERE ProductoId = @ProductoId;
    COMMIT;
END;
```

Con el stock del monitor en 1, se ejecuta `EXEC dbo.Comprar 3, 1;` en dos ventanas de SSMS, con un segundo de diferencia:

| Momento | Ventana A | Ventana B |
| --- | --- | --- |
| t0 | Lee `Stock = 1`, pasa la validación, espera | |
| t1 | | Lee `Stock = 1` (`READ COMMITTED` no lo impide), pasa la validación, espera |
| t5 | Escribe `Stock = 0`, confirma | |
| t6 | | Escribe `Stock = 0`, confirma |

Resultado: **dos ventas** del único monitor, y el stock en 0. Ningún error.

**Después** (la condición de negocio dentro del `UPDATE`):

```sql
CREATE OR ALTER PROCEDURE dbo.Comprar @ProductoId INT, @Cantidad INT
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        UPDATE dbo.Producto
        SET Stock = Stock - @Cantidad
        WHERE ProductoId = @ProductoId
          AND Stock >= @Cantidad;

        IF @@ROWCOUNT = 0
            THROW 50002, N'Stock insuficiente.', 1;

        -- ... insertar el pedido y sus líneas ...
    COMMIT;
END;
```

| Momento | Ventana A | Ventana B |
| --- | --- | --- |
| t0 | `UPDATE`: bloquea la fila, `Stock` 1 → 0 | |
| t1 | | `UPDATE`: **espera** el bloqueo de A |
| t2 | `COMMIT` | |
| t3 | | Reevalúa `Stock >= 1` con el valor confirmado (0): **0 filas**, `THROW` |

Una venta, un error claro para la segunda, y el stock correcto.

Qué cambió y por qué:

* **No hay ventana entre leer y escribir:** el `UPDATE` lee y modifica la fila en la misma instrucción, con un bloqueo exclusivo que la otra sesión respeta.
* **La validación de negocio va en el `WHERE`:** al obtener el bloqueo, la segunda sesión vuelve a evaluar la condición con el valor actual.
* **`@@ROWCOUNT` convierte "0 filas" en un error explícito.**
* **No hizo falta cambiar el nivel de aislamiento:** con el nivel por defecto, la instrucción atómica alcanza. Subir a `SERIALIZABLE` habría frenado a todos para resolver un problema que era de cómo estaba escrita la operación.

El procedimiento `dbo.RegistrarPedido` de la lección anterior tiene la misma corrección pendiente: se resuelve igual, con el `WHERE Stock >= @Cantidad` y el `@@ROWCOUNT`.

-----

## Errores comunes

**1. Leer, calcular en una variable y escribir.**
Qué pasa: actualizaciones perdidas (stock, saldos, contadores incorrectos) sin ningún error.
Por qué: entre la lectura y la escritura, otra transacción puede leer el mismo valor.
Arreglo: `UPDATE` atómico con la condición en el `WHERE`, `UPDLOCK` al leer, o concurrencia optimista.

**2. Subir el nivel de aislamiento de toda la aplicación "por seguridad".**
Qué pasa: bloqueos, esperas y *deadlocks* por todas partes.
Por qué: `SERIALIZABLE` en todo frena todo.
Arreglo: el nivel por defecto (o RCSI) para casi todo, y mayor aislamiento puntualmente donde se necesita.

**3. Asumir que un reporte largo ve una foto consistente.**
Qué pasa: totales que no cuadran entre sí cuando la tabla se modifica durante el reporte.
Por qué: `READ COMMITTED` suelta cada bloqueo después de leer la fila.
Arreglo: `SNAPSHOT` para los reportes que necesitan consistencia.

**4. Confundir actualización perdida con lectura sucia.**
Qué pasa: se agrega `NOLOCK` o se quita, creyendo que eso resuelve el stock negativo.
Por qué: son anomalías distintas; `READ COMMITTED` ya evita las lecturas sucias y aun así permite la actualización perdida.
Arreglo: identificar la anomalía concreta antes de elegir la solución.

**5. Cambiar el nivel en una conexión que vuelve al pool.**
Qué pasa: otra parte de la aplicación hereda un `SERIALIZABLE` o un `READ UNCOMMITTED` que no pidió.
Por qué: el nivel es de la sesión, y el pool de conexiones reutiliza sesiones.
Arreglo: volver a `READ COMMITTED` al terminar, o pedir el nivel por transacción desde la aplicación (en .NET, `TransactionScope` o `BeginTransaction(IsolationLevel...)`).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Nivel por defecto | `READ COMMITTED` con bloqueos (RCSI en Azure SQL) | `READ COMMITTED` con versiones (MVCC) | **`REPEATABLE READ`** con versiones |
| Lecturas que bloquean escrituras (nivel por defecto) | Sí | No | No |
| `READ UNCOMMITTED` | Lecturas sucias reales | Se comporta como `READ COMMITTED` (no hay lecturas sucias) | Lecturas sucias reales |
| `REPEATABLE READ` | Con bloqueos | Snapshot de la transacción (también evita fantasmas) | Snapshot + bloqueos de huecos (*gap locks*) en escrituras |
| `SERIALIZABLE` | Bloqueos de rango | *Serializable Snapshot Isolation*: detecta conflictos y aborta una transacción | Convierte las lecturas en `LOCK IN SHARE MODE` |
| Concurrencia optimista | `ROWVERSION` | `xmin` (columna del sistema) o una versión propia | Una columna de versión propia |
| Bloqueo pesimista al leer | `WITH (UPDLOCK)` | `SELECT ... FOR UPDATE` | `SELECT ... FOR UPDATE` |

La diferencia más importante en la práctica: en PostgreSQL y MySQL, las lecturas normales **no** bloquean a las escrituras (usan versiones), mientras que en SQL Server, con la configuración por defecto, sí. Por eso los problemas de bloqueos de lectura son tan típicos de SQL Server, y por eso RCSI (la próxima lección) es tan recomendado.

### En MongoDB

Las operaciones sobre un documento son atómicas, así que la actualización perdida se evita con operadores atómicos, igual que el `UPDATE` atómico de SQL: `db.productos.updateOne({ _id: 3, stock: { $gte: 1 } }, { $inc: { stock: -1 } })`, y verificando `modifiedCount`. Para lecturas, el *read concern* (`local`, `majority`, `snapshot`) define qué datos puede ver una consulta respecto de la replicación, y las transacciones multidocumento usan aislamiento de tipo *snapshot*.

-----

## Cuándo sí y cuándo no

**Mantén el nivel por defecto (o RCSI) cuando:**

* Es una consulta o una escritura normal de una aplicación OLTP: casi siempre.

**Pide más aislamiento puntualmente cuando:**

* Una transacción lee y después escribe en función de lo que leyó: `UPDLOCK` en la lectura, o el `UPDATE` atómico.
* Un reporte necesita una foto consistente: `SNAPSHOT`.
* Una regla depende de que no aparezcan filas nuevas en un rango durante la transacción: `SERIALIZABLE`, en esa transacción y nada más.

**No uses `READ UNCOMMITTED` cuando:**

* Los números importan: saldos, stock, facturación, auditoría.

-----

## Resumen en 5 líneas

1. Las anomalías son lectura sucia, no repetible, fantasma y actualización perdida; solo aparecen cuando las transacciones se cruzan.
2. El nivel por defecto de SQL Server, `READ COMMITTED`, evita las lecturas sucias y permite las demás.
3. Cada nivel más estricto bloquea más y por más tiempo: más garantías, más esperas y más *deadlocks*.
4. La actualización perdida se evita escribiendo bien la operación: `UPDATE` atómico con la condición en el `WHERE`, `UPDLOCK` o `ROWVERSION`.
5. Pide aislamiento puntualmente donde se necesita, no para toda la aplicación.

-----

## Para profundizar

<details>
<summary>Write skew: la anomalía que SNAPSHOT no evita</summary>

Una regla de negocio dice "siempre tiene que haber al menos un médico de guardia". Hay dos de guardia. Cada uno, en su transacción, verifica que hay dos (lo ve en su foto) y se da de baja. Las dos transacciones modificaron filas **distintas**, así que no hay conflicto de escritura, y las dos confirman: quedaron cero médicos de guardia. Esto se llama *write skew*, y `SNAPSHOT` no lo evita, porque cada transacción ve una foto consistente pero desactualizada. Lo evitan `SERIALIZABLE` (con bloqueos de rango sobre la consulta de verificación) o bloquear explícitamente las filas que se leyeron para decidir (`UPDLOCK` sobre las dos filas de guardia).

</details>

<details>
<summary>Lecturas que se saltan o duplican filas con READ COMMITTED</summary>

Con `READ COMMITTED` por bloqueos, un *scan* que recorre un índice puede, en casos poco frecuentes, leer una fila dos veces o no leerla nunca: si otra transacción actualiza la clave de esa fila y la "mueve" a una posición del índice que el *scan* todavía no recorrió (se lee dos veces) o que ya recorrió (se pierde). Con `NOLOCK`, además, se suma el riesgo de las divisiones de página durante la lectura. Las lecturas basadas en versiones (RCSI y `SNAPSHOT`) no tienen este problema, porque leen una versión fija de cada fila.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los niveles de aislamiento definen qué puede ver una transacción de las demás. `READ UNCOMMITTED` permite lecturas sucias, `READ COMMITTED` (el de SQL Server por defecto) las evita, `REPEATABLE READ` evita que cambien las filas leídas y `SERIALIZABLE` evita también las filas fantasma. Más aislamiento significa más bloqueos.

### Respuesta ampliada (semi-senior)

Distingo cuatro anomalías: lectura sucia, no repetible, fantasma y actualización perdida, y cada nivel de aislamiento permite o impide algunas a cambio de concurrencia. SQL Server usa por defecto `READ COMMITTED` con bloqueos, que evita las lecturas sucias pero no da una foto consistente ni evita las actualizaciones perdidas; `REPEATABLE READ` y `SERIALIZABLE` mantienen los bloqueos hasta el final (y de rangos, en el segundo), con el costo de más esperas y *deadlocks*; `SNAPSHOT` y RCSI usan versiones en lugar de bloqueos. No subo el aislamiento de toda la aplicación: la actualización perdida la resuelvo en la operación, con un `UPDATE` atómico que lleva la condición de negocio en el `WHERE` y verifica `@@ROWCOUNT`, con `UPDLOCK` cuando hay lógica entre la lectura y la escritura, o con concurrencia optimista con `ROWVERSION` cuando hay tiempo de usuario de por medio. Para reportes que necesitan consistencia, uso `SNAPSHOT`, y tengo en cuenta el *write skew*, que solo evita `SERIALIZABLE` o un bloqueo explícito.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es el nivel de aislamiento por defecto en SQL Server?**
`READ COMMITTED`, implementado con bloqueos (salvo que la base tenga RCSI activado, como en Azure SQL por defecto).

**2. ¿Qué diferencia hay entre una lectura no repetible y una fantasma?**
En la no repetible cambia el valor de una fila que ya se leyó; en la fantasma aparecen filas nuevas que cumplen el filtro.

**3. ¿Cómo evitas que dos usuarios vendan el último producto?**
Con un `UPDATE` atómico que descuenta el stock solo si alcanza (`WHERE Stock >= @Cantidad`) y verificando `@@ROWCOUNT`; o leyendo con `UPDLOCK` dentro de la transacción.

**4. ¿Qué es la concurrencia optimista?**
No bloquear al leer y, al escribir, verificar con una columna de versión que nadie modificó la fila; si alguien lo hizo, la escritura falla y se reintenta.

-----

## Práctica

**Ejercicio 1.** Identifica la anomalía en cada caso:

1. Un reporte suma los pedidos del día dos veces en la misma transacción y obtiene totales distintos, porque entraron pedidos nuevos entre una suma y la otra.
2. Una pantalla muestra un precio que alguien estaba cambiando, y que después se deshizo.
3. Dos administradores editan la descripción de un producto en dos pantallas; el que guarda segundo borra los cambios del primero.
4. Una transacción lee el precio del producto 3, otra lo cambia y confirma, y la primera vuelve a leerlo y ve el precio nuevo.

<details>
<summary>Solución</summary>

1. **Lectura fantasma:** aparecieron filas nuevas que cumplen el filtro "pedidos del día".
2. **Lectura sucia:** se leyó un cambio no confirmado que luego se deshizo.
3. **Actualización perdida:** la segunda escritura pisó la primera. Se resuelve con concurrencia optimista (`ROWVERSION`), porque hay minutos de edición de por medio.
4. **Lectura no repetible:** la misma fila cambió entre dos lecturas de la misma transacción.

</details>

**Ejercicio 2.** Escribe un `UPDATE` atómico que reserve un asiento en una tabla `Asiento(AsientoId, FuncionId, ReservadoPor NULL)`, de modo que dos clientes no puedan reservar el mismo asiento, e informe si la reserva no fue posible.

<details>
<summary>Solución</summary>

```sql
UPDATE dbo.Asiento
SET ReservadoPor = @ClienteId
WHERE AsientoId = @AsientoId
  AND ReservadoPor IS NULL;          -- solo si sigue libre

IF @@ROWCOUNT = 0
    THROW 50010, N'El asiento ya fue reservado.', 1;
```

Si dos sesiones intentan reservar el mismo asiento a la vez, la segunda espera el bloqueo de la primera; cuando la primera confirma, la segunda reevalúa `ReservadoPor IS NULL`, que ya es falso, y actualiza 0 filas. No hace falta leer antes ni cambiar el nivel de aislamiento.

</details>

-----

## Siguiente lección

[Bloqueos, hints y NOLOCK](03-Bloqueos%2C%20hints%20y%20NOLOCK.md)
