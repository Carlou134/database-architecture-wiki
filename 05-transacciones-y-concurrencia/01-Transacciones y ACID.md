# Transacciones y ACID

## En una frase

Una transacción agrupa varias operaciones en una **unidad indivisible**: o se aplican todas o no se aplica ninguna, y el motor garantiza cuatro propiedades (atomicidad, consistencia, aislamiento y durabilidad, ACID) que son la razón principal por la que los datos de negocio viven en una base de datos y no en archivos.

-----

## Antes de empezar

Conviene que ya sepas:

* El método seguro para modificar datos con `BEGIN TRANSACTION`, `COMMIT` y `ROLLBACK`, de [Modificar datos](../03-consultas/11-Modificar%20datos.md).
* Que el SGBD escribe primero en el log, de [Qué es una base de datos y un SGBD](../01-fundamentos/01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md).
* Restricciones y claves foráneas, de [Fundamentos](../01-fundamentos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Transacción:** conjunto de operaciones que se confirman o se deshacen juntas.
* **ACID:** atomicidad, consistencia, aislamiento y durabilidad.
* **`COMMIT` / `ROLLBACK`:** confirmar / deshacer una transacción.
* **Modo de confirmación automática (autocommit):** cada instrucción es su propia transacción; el modo por defecto de SQL Server.
* **Transacción explícita:** la que se abre con `BEGIN TRANSACTION` y se cierra con `COMMIT` o `ROLLBACK`.
* **Punto de guardado (savepoint):** marca dentro de una transacción a la que se puede volver sin deshacer todo.
* **`@@TRANCOUNT`:** cuántos `BEGIN TRANSACTION` están abiertos en la sesión.

-----

## El problema

Registrar un pedido en la tienda son tres operaciones:

```sql
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (5, 'Pagado');                       -- 1. el pedido
INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
VALUES (SCOPE_IDENTITY(), 3, 1, 899.90);                                                -- 2. sus líneas
UPDATE dbo.Producto SET Stock = Stock - 1 WHERE ProductoId = 3;                         -- 3. el stock
```

Sin una transacción, cada línea se confirma por separado. Si la segunda falla (un producto que no existe, un corte de conexión, un error en la aplicación entre una instrucción y la otra), queda un **pedido sin líneas**: la clienta pagó y no compró nada. Si falla la tercera, el pedido existe pero el stock no bajó, y mañana se vende un monitor que ya no está.

Ningún `CHECK` ni clave foránea puede impedir esto: cada instrucción, sola, es válida. El problema es que las tres **juntas** son un solo hecho de negocio, y tienen que aplicarse como tal.

-----

## Cómo funciona

### Las propiedades ACID

| Propiedad | Qué garantiza | En el ejemplo |
| --- | --- | --- |
| **Atomicidad** | Todo o nada: si una parte falla, se deshace todo | No queda un pedido sin líneas |
| **Consistencia** | La base pasa de un estado válido a otro: se cumplen todas las restricciones | El stock nunca queda negativo (`CK_Producto_Stock`) |
| **Aislamiento** | Las transacciones concurrentes no ven los cambios a medias de las demás | Otro usuario no ve el pedido "a medio armar" |
| **Durabilidad** | Lo confirmado sobrevive a un fallo, incluso un corte de luz | Después del `COMMIT`, el pedido existe aunque el servidor se reinicie un segundo después |

La consistencia es, en parte, responsabilidad tuya: el motor hace cumplir las restricciones declaradas, pero las reglas que no declaraste (que el total del pedido coincida con las líneas, por ejemplo) solo se cumplen si las transacciones están bien escritas.

El aislamiento tiene **niveles**, y cada nivel permite o impide distintos problemas de concurrencia: es el tema de la [próxima lección](02-Niveles%20de%20aislamiento%20y%20anomalías.md).

### Cómo se logra: el log de transacciones

La atomicidad y la durabilidad se apoyan en el **log de transacciones** (el archivo `.ldf`), con una regla llamada *write-ahead logging*: antes de modificar una página de datos, el cambio se escribe en el log.

```text
BEGIN TRANSACTION
  INSERT Pedido     → log: "insertar fila X en Pedido"       (en memoria y en el log)
  INSERT Detalle    → log: "insertar fila Y en DetallePedido"
  UPDATE Producto   → log: "Stock de 3: 8 → 7"
COMMIT              → log: "transacción T confirmada"         ← se escribe al disco ANTES de responder
                    → las páginas de datos se escriben más tarde, en segundo plano
```

* **Durabilidad:** el `COMMIT` no termina hasta que el registro del log está en el disco. Las páginas de datos pueden quedar solo en memoria; si el servidor se cae, al reiniciar rehace desde el log lo confirmado.
* **Atomicidad:** si la transacción no llegó al `COMMIT` (un `ROLLBACK`, un error, una caída), el log tiene lo necesario para deshacer cada cambio.

Por eso una transacción muy grande (un `UPDATE` de 50 millones de filas) hace crecer el log: todo lo que hizo tiene que poder deshacerse hasta que termine.

### Tres modos de transacción

| Modo | Cómo funciona | En SQL Server |
| --- | --- | --- |
| **Confirmación automática** | Cada instrucción es una transacción: se confirma sola si termina bien | **El modo por defecto** |
| **Explícita** | `BEGIN TRANSACTION` ... `COMMIT` / `ROLLBACK` | El que usas para agrupar operaciones |
| **Implícita** | La primera instrucción abre una transacción, que queda abierta hasta un `COMMIT` explícito | Se activa con `SET IMPLICIT_TRANSACTIONS ON` (habitual en Oracle, y en algunos drivers) |

El modo implícito es una fuente clásica de sorpresas: una herramienta o un driver lo activa, alguien ejecuta un `UPDATE`, cierra la ventana sin `COMMIT` y la transacción queda abierta, bloqueando filas, o se deshace al desconectar.

Incluso en modo automático, **una sola instrucción ya es atómica**: un `UPDATE` que modifica 1.000 filas y falla en la 600 no deja 599 filas modificadas. Se deshace entero.

### Una transacción explícita

```sql
BEGIN TRANSACTION;

INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (5, 'Pagado');
DECLARE @PedidoId INT = SCOPE_IDENTITY();

INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
VALUES (@PedidoId, 3, 1, 899.90);

UPDATE dbo.Producto SET Stock = Stock - 1 WHERE ProductoId = 3;

COMMIT TRANSACTION;
```

Hay una trampa importante: **sin manejo de errores, una transacción explícita no es atómica por sí sola**. En SQL Server, muchos errores (una violación de clave foránea, un `CHECK`) terminan **solo la instrucción** que falló, y la ejecución **sigue** con la siguiente. El script de arriba, si la segunda instrucción falla, igual ejecuta el `UPDATE` y el `COMMIT`: confirma un pedido sin líneas.

La forma correcta combina dos cosas: `SET XACT_ABORT ON` (cualquier error deshace la transacción entera) y `TRY...CATCH` (para controlar qué hacer con el error). Se estudia en [Manejo de errores](06-Manejo%20de%20errores.md); el patrón mínimo es:

```sql
SET XACT_ABORT ON;

BEGIN TRY
    BEGIN TRANSACTION;
        -- operaciones
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    THROW;   -- relanzar el error para que quien llamó se entere
END CATCH;
```

### `@@TRANCOUNT` y el mito de las transacciones anidadas

SQL Server permite escribir `BEGIN TRANSACTION` dentro de otra transacción, pero **no** son transacciones anidadas de verdad:

| Instrucción | Efecto sobre `@@TRANCOUNT` | Efecto real |
| --- | --- | --- |
| `BEGIN TRANSACTION` | +1 | Solo el primero abre una transacción |
| `COMMIT` | −1 | Solo el último (el que deja el contador en 0) confirma algo |
| `ROLLBACK` | **A 0** | Deshace **todo**, desde el primer `BEGIN` |

```sql
BEGIN TRANSACTION;                 -- @@TRANCOUNT = 1
    INSERT ...;
    BEGIN TRANSACTION;             -- @@TRANCOUNT = 2
        INSERT ...;
    COMMIT;                        -- @@TRANCOUNT = 1: NO confirmó nada
ROLLBACK;                          -- @@TRANCOUNT = 0: deshace los DOS insert
```

La consecuencia práctica: un procedimiento que hace `BEGIN TRANSACTION` y `COMMIT` puede llamarse desde dentro de otra transacción sin problema, pero un `ROLLBACK` dentro del procedimiento deshace también el trabajo de quien lo llamó, y al volver, el llamador encuentra `@@TRANCOUNT` en 0 y un error 266 (*Transaction count after EXECUTE indicates a mismatching number of BEGIN and COMMIT statements*).

### Puntos de guardado

Para deshacer **una parte** de una transacción sin perder el resto, se usa un punto de guardado:

```sql
BEGIN TRANSACTION;
    INSERT INTO dbo.Pedido ...;
    SAVE TRANSACTION AntesDeLasLineas;
    INSERT INTO dbo.DetallePedido ...;     -- algo sale mal aquí
    ROLLBACK TRANSACTION AntesDeLasLineas; -- deshace solo las líneas
    -- el pedido sigue en la transacción, @@TRANCOUNT sigue en 1
COMMIT;
```

Son útiles en procesos que reintentan una parte, pero no funcionan si la transacción quedó en estado "condenado" (ver [Manejo de errores](06-Manejo%20de%20errores.md)), y no se usan en transacciones distribuidas.

### Transacciones cortas

Mientras una transacción está abierta, las filas que modificó quedan **bloqueadas** para las demás sesiones (lo que se estudia en [Bloqueos](03-Bloqueos%2C%20hints%20y%20NOLOCK.md)). Por eso una transacción debe durar lo menos posible:

* Validar los datos, calcular y buscar lo necesario **antes** del `BEGIN TRANSACTION`.
* Dentro, solo las escrituras que tienen que ser atómicas.
* **Nunca** esperar algo externo con la transacción abierta: una respuesta del usuario, una llamada a una API, el envío de un correo.

-----

## Ejemplo completo

El registro de un pedido en la tienda, como procedimiento almacenado atómico:

```sql
USE Tienda;
GO

CREATE OR ALTER PROCEDURE dbo.RegistrarPedido
    @ClienteId  INT,
    @ProductoId INT,
    @Cantidad   INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    -- Fuera de la transacción: leer lo necesario
    DECLARE @Precio DECIMAL(10,2) =
        (SELECT Precio FROM dbo.Producto WHERE ProductoId = @ProductoId AND Activo = 1);

    IF @Precio IS NULL
        THROW 50001, N'El producto no existe o no está activo.', 1;

    BEGIN TRY
        BEGIN TRANSACTION;

        INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (@ClienteId, 'Pagado');
        DECLARE @PedidoId INT = SCOPE_IDENTITY();

        INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
        VALUES (@PedidoId, @ProductoId, @Cantidad, @Precio);

        UPDATE dbo.Producto
        SET Stock = Stock - @Cantidad
        WHERE ProductoId = @ProductoId;   -- si el stock queda negativo, CK_Producto_Stock falla

        COMMIT TRANSACTION;

        SELECT @PedidoId AS PedidoId;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO
```

Probándolo dentro de una transacción de prueba para no modificar los datos de referencia:

```sql
BEGIN TRANSACTION;   -- envoltorio de prueba

EXEC dbo.RegistrarPedido @ClienteId = 5, @ProductoId = 3, @Cantidad = 1;   -- ✅ PedidoId 6, stock del monitor: 8 → 7
EXEC dbo.RegistrarPedido @ClienteId = 5, @ProductoId = 3, @Cantidad = 50;  -- ❌ CK_Producto_Stock

SELECT PedidoId, ClienteId, Estado FROM dbo.Pedido WHERE ClienteId = 5;
SELECT Stock FROM dbo.Producto WHERE ProductoId = 3;

ROLLBACK;            -- deshacer la prueba completa
```

El segundo llamado falla con:

```text
Msg 547, Level 16, State 0
The UPDATE statement conflicted with the CHECK constraint "CK_Producto_Stock".
```

El `CATCH` del procedimiento ejecuta `ROLLBACK`, y ese `ROLLBACK` deshace **toda** la transacción, incluido el envoltorio de prueba y el primer pedido. Además, el `THROW` relanza el error y detiene el lote, así que los `SELECT` y el `ROLLBACK` finales no llegan a ejecutarse; junto al error original puede aparecer `Msg 266` (*Transaction count after EXECUTE indicates a mismatching number of BEGIN and COMMIT statements*), porque el procedimiento empezó con `@@TRANCOUNT = 1` (la transacción del envoltorio) y terminó con 0. Un `SELECT @@TRANCOUNT;` en un lote nuevo devuelve 0. Es exactamente el comportamiento de las "transacciones anidadas" descrito arriba: un `ROLLBACK` deshace todo.

Qué decisión hay detrás:

* **El precio se lee antes de la transacción:** la transacción solo contiene las escrituras, y dura lo mínimo.
* **El stock negativo lo detecta el `CHECK`:** la base hace cumplir la regla aunque la aplicación olvide validarla, y la atomicidad garantiza que el pedido y la línea también se deshagan.
* **`XACT_ABORT ON` + `TRY...CATCH`:** cualquier error deshace todo, y el `THROW` relanza el error original para que la aplicación lo reciba.
* **Lo que falta:** dos clientes comprando el último monitor **al mismo tiempo**. Las dos lecturas de precio y los dos `UPDATE` se pueden cruzar. La atomicidad no alcanza: hace falta el **aislamiento**, que se ve en las próximas lecciones.

-----

## Errores comunes

**1. Creer que `BEGIN TRANSACTION` alcanza para la atomicidad.**
Qué pasa: un error en la segunda instrucción, y la tercera y el `COMMIT` se ejecutan igual.
Por qué: muchos errores de SQL Server terminan solo la instrucción, no el lote.
Arreglo: `SET XACT_ABORT ON` y `TRY...CATCH` con `ROLLBACK`.

**2. Transacciones largas.**
Qué pasa: bloqueos que frenan a otros usuarios, log que crece, `ROLLBACK` que tardan tanto como el trabajo.
Por qué: validaciones, cálculos o llamadas externas dentro de la transacción.
Arreglo: todo lo que no es escritura, antes del `BEGIN`; nada externo dentro.

**3. Confiar en las transacciones "anidadas".**
Qué pasa: un `COMMIT` interno no confirma nada, y un `ROLLBACK` interno deshace todo, incluido el trabajo del llamador.
Por qué: SQL Server solo tiene una transacción real; los `BEGIN` internos solo incrementan un contador.
Arreglo: los procedimientos que pueden llamarse dentro de otra transacción solo hacen `ROLLBACK` si ellos la abrieron, o usan puntos de guardado.

**4. Transacciones que quedan abiertas.**
Qué pasa: filas bloqueadas durante horas, y una sesión que, al desconectarse, deshace todo lo que hizo.
Por qué: un `BEGIN` sin `COMMIT`, o el modo implícito activado por una herramienta.
Arreglo: cerrar siempre con `COMMIT` o `ROLLBACK`; revisar `SELECT @@TRANCOUNT;` y `DBCC OPENTRAN;` para encontrar transacciones abiertas.

**5. `COMMIT` sin `BEGIN`.**
Qué pasa: `Msg 3902: The COMMIT TRANSACTION request has no corresponding BEGIN TRANSACTION`.
Por qué: la transacción ya se había deshecho (por un error con `XACT_ABORT ON`, o por un `ROLLBACK` en otro procedimiento).
Arreglo: verificar `@@TRANCOUNT` o `XACT_STATE()` antes de confirmar.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Modo por defecto | Confirmación automática | Confirmación automática | Confirmación automática (`autocommit = 1`) |
| Abrir una transacción | `BEGIN TRANSACTION` | `BEGIN` o `START TRANSACTION` | `START TRANSACTION` o `BEGIN` |
| Un error dentro de la transacción | Según el error, termina la instrucción o el lote (`XACT_ABORT` cambia esto) | **Aborta la transacción**: toda instrucción siguiente falla hasta el `ROLLBACK` | Termina la instrucción; la transacción sigue abierta |
| DDL dentro de una transacción | Sí, con `ROLLBACK` | Sí, con `ROLLBACK` (casi todo el DDL) | **No**: el DDL hace un `COMMIT` implícito |
| Puntos de guardado | `SAVE TRANSACTION nombre` | `SAVEPOINT nombre` | `SAVEPOINT nombre` |
| Transacciones anidadas | Simuladas con `@@TRANCOUNT` | No (usar *savepoints*) | No (un `BEGIN` dentro de otro confirma el anterior) |

PostgreSQL es el más estricto: después de cualquier error, la transacción queda inválida (`current transaction is aborted, commands ignored until end of transaction block`), lo que elimina la trampa de "el error terminó solo la instrucción" que existe en SQL Server y MySQL. En MySQL, un `CREATE TABLE` en medio de una transacción confirma todo lo anterior sin aviso.

### En MongoDB

Una operación sobre **un** documento siempre es atómica, aunque modifique varios campos o subdocumentos; por eso embeber los detalles dentro del pedido evita muchas necesidades de transacciones. Para varios documentos o colecciones, MongoDB tiene transacciones multidocumento desde la versión 4.0 (y en clústeres fragmentados desde la 4.2), con `session.startTransaction()`, `commitTransaction()` y `abortTransaction()`. Tienen límites (por defecto, 60 segundos de duración) y un costo mayor, así que el diseño documental intenta que las operaciones frecuentes toquen un solo documento.

-----

## Cuándo sí y cuándo no

**Usa una transacción explícita cuando:**

* Varias escrituras representan un solo hecho de negocio (un pedido, una transferencia, un traspaso de stock).
* Una modificación manual debe poder revisarse antes de confirmarse.

**No hace falta cuando:**

* Es una sola instrucción: ya es atómica.
* Solo lees datos y no necesitas que varias lecturas vean el mismo estado (si lo necesitas, eso es aislamiento: próximas lecciones).

**Ten en cuenta:**

* La transacción garantiza atomicidad dentro de **una** base. Coordinar una base y una cola de mensajes, o dos servicios, necesita otros patrones (outbox, sagas), que se ven en el wiki de arquitectura.

-----

## Resumen en 5 líneas

1. Una transacción es todo o nada; ACID es atomicidad, consistencia, aislamiento y durabilidad.
2. El log de transacciones lo hace posible: el `COMMIT` espera a que el log esté en disco, y el log permite deshacer.
3. SQL Server confirma cada instrucción sola por defecto; `BEGIN TRANSACTION` agrupa, pero sin `XACT_ABORT ON` y `TRY...CATCH` un error no deshace todo.
4. No hay transacciones anidadas reales: `COMMIT` interno no confirma, `ROLLBACK` deshace todo; para deshacer una parte, `SAVE TRANSACTION`.
5. Transacciones cortas: validar y calcular antes, nada externo dentro.

-----

## Para profundizar

<details>
<summary>Durabilidad diferida</summary>

Desde SQL Server 2014 existe la *delayed durability*: el `COMMIT` responde **antes** de que el log llegue al disco, y el registro se escribe un instante después, en bloques. Mejora mucho el rendimiento de cargas con muchísimas transacciones pequeñas, a cambio de poder perder las últimas transacciones confirmadas si el servidor se cae en ese instante. Es decir, renuncia a la "D" de ACID de forma controlada. Se activa por base (`ALTER DATABASE ... SET DELAYED_DURABILITY = ALLOWED`) y por transacción (`COMMIT WITH (DELAYED_DURABILITY = ON)`). Solo tiene sentido para datos que se pueden reconstruir.

</details>

<details>
<summary>Recuperación acelerada de bases de datos (ADR)</summary>

Tradicionalmente, deshacer una transacción enorme (o recuperar una base después de una caída en medio de una) tardaba tanto como la transacción misma, porque había que recorrer el log hacia atrás. Desde SQL Server 2019, la *Accelerated Database Recovery* guarda versiones de las filas en la propia base y permite que el `ROLLBACK` y la recuperación sean casi instantáneos, independientemente del tamaño de la transacción. Además, permite truncar el log aunque haya transacciones largas abiertas. Se activa por base (`ALTER DATABASE ... SET ACCELERATED_DATABASE_RECOVERY = ON`).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una transacción agrupa varias operaciones para que se apliquen todas juntas o ninguna. Se abre con `BEGIN TRANSACTION` y se termina con `COMMIT` para confirmar o `ROLLBACK` para deshacer. ACID significa atomicidad, consistencia, aislamiento y durabilidad.

### Respuesta ampliada (semi-senior)

La transacción es la unidad de trabajo atómica, y ACID se apoya en el log con *write-ahead logging*: el `COMMIT` espera a que el log esté en disco (durabilidad) y el log permite deshacer lo no confirmado (atomicidad). En SQL Server, el modo por defecto confirma cada instrucción sola; con `BEGIN TRANSACTION` agrupo, pero sé que muchos errores terminan solo la instrucción, así que uso siempre `SET XACT_ABORT ON` con `TRY...CATCH` y `ROLLBACK`. No hay transacciones anidadas reales: `@@TRANCOUNT` es un contador, un `COMMIT` interno no confirma nada y un `ROLLBACK` deshace todo, lo que tengo en cuenta en procedimientos que pueden llamarse dentro de otra transacción; para deshacer solo una parte uso `SAVE TRANSACTION`. Mantengo las transacciones cortas (validaciones y lecturas antes, nada externo dentro), porque mientras están abiertas bloquean filas y retienen el log. Y tengo claro que la atomicidad sola no resuelve la concurrencia: para eso está el aislamiento.

### Preguntas frecuentes de seguimiento

**1. ¿Qué significa cada letra de ACID?**
Atomicidad (todo o nada), consistencia (de un estado válido a otro), aislamiento (las transacciones concurrentes no se ven a medias) y durabilidad (lo confirmado sobrevive a un fallo).

**2. ¿Un `ROLLBACK` dentro de un `TRY` pasa el control al `CATCH`?**
No. El `ROLLBACK` deshace la transacción y la ejecución continúa en la instrucción siguiente. Lo que pasa el control al `CATCH` es un **error**.

**3. ¿SQL Server tiene transacciones anidadas?**
No realmente: los `BEGIN TRANSACTION` internos solo incrementan `@@TRANCOUNT`; el `COMMIT` que confirma es el último, y cualquier `ROLLBACK` deshace todo.

**4. ¿Por qué deben ser cortas las transacciones?**
Porque mientras están abiertas mantienen bloqueos sobre las filas modificadas y retienen el log de transacciones.

-----

## Práctica

**Ejercicio 1.** ¿Qué queda en la tabla después de ejecutar este script, y por qué?

```sql
CREATE TABLE dbo.Prueba (Id INT PRIMARY KEY);

BEGIN TRANSACTION;
    INSERT INTO dbo.Prueba VALUES (1);
    INSERT INTO dbo.Prueba VALUES (1);   -- clave duplicada
    INSERT INTO dbo.Prueba VALUES (2);
COMMIT;

SELECT Id FROM dbo.Prueba;
```

<details>
<summary>Solución</summary>

| Id |
| --- |
| 1 |
| 2 |

El segundo `INSERT` falla con `Msg 2627` (violación de clave primaria), pero ese error termina **solo esa instrucción**: la ejecución continúa, el tercer `INSERT` funciona y el `COMMIT` confirma los dos que sí entraron. La transacción no fue atómica.

Con `SET XACT_ABORT ON;` antes del `BEGIN TRANSACTION`, el error deshace toda la transacción y aborta el lote: la tabla queda vacía y el `SELECT` no se ejecuta (porque está en el mismo lote).

</details>

**Ejercicio 2.** Indica el valor de `@@TRANCOUNT` después de cada línea y qué filas quedan al final.

```sql
TRUNCATE TABLE dbo.Prueba;
BEGIN TRANSACTION;                      -- a
    INSERT INTO dbo.Prueba VALUES (10);
    SAVE TRANSACTION Punto;             -- b
    INSERT INTO dbo.Prueba VALUES (20);
    ROLLBACK TRANSACTION Punto;         -- c
    BEGIN TRANSACTION;                  -- d
        INSERT INTO dbo.Prueba VALUES (30);
    COMMIT;                             -- e
COMMIT;                                 -- f
SELECT Id FROM dbo.Prueba;
```

<details>
<summary>Solución</summary>

| Línea | `@@TRANCOUNT` | Qué pasó |
| --- | --- | --- |
| a | 1 | Se abre la transacción |
| b | 1 | El punto de guardado no cambia el contador |
| c | 1 | Se deshace solo el 20; el 10 sigue en la transacción |
| d | 2 | Solo incrementa el contador |
| e | 1 | No confirma nada todavía |
| f | 0 | Confirma todo |

Resultado: **10** y **30**. El 20 se deshizo con el `ROLLBACK` al punto de guardado, que, a diferencia de un `ROLLBACK` sin nombre, no deshace toda la transacción ni pone el contador en 0.

</details>

-----

## Siguiente lección

[Niveles de aislamiento y anomalías](02-Niveles%20de%20aislamiento%20y%20anomalías.md)
