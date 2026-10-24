# Manejo de errores

## En una frase

En T-SQL, un error puede terminar solo la instrucción, todo el lote o la conexión, según su tipo, y por eso el manejo de errores serio combina `SET XACT_ABORT ON` (cualquier error deshace la transacción), `TRY...CATCH` (para decidir qué hacer), `XACT_STATE()` (para saber si la transacción todavía sirve), `THROW` (para avisarle a quien llamó) y una clasificación entre errores **transitorios**, que se reintentan, y **permanentes**, que no.

-----

## Antes de empezar

Conviene que ya sepas:

* Transacciones, `@@TRANCOUNT` y el mito de las transacciones anidadas, de [Transacciones y ACID](01-Transacciones%20y%20ACID.md).
* El error 1205 y el reintento, de [Deadlocks](05-Deadlocks.md).
* Los números de error de las restricciones (547, 2627, 515), de [Restricciones](../01-fundamentos/06-Restricciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`TRY...CATCH`:** bloque que captura los errores del código del `TRY` y pasa el control al `CATCH`.
* **Severidad:** número de 0 a 25 que indica la gravedad de un error.
* **`XACT_STATE()`:** función que indica si hay una transacción y si todavía puede confirmarse: 1 (activa), 0 (ninguna), -1 (condenada).
* **Transacción condenada (uncommittable / doomed):** transacción que sigue abierta pero solo admite `ROLLBACK`.
* **`SET XACT_ABORT ON`:** configuración que hace que cualquier error en tiempo de ejecución deshaga la transacción entera.
* **`THROW` / `RAISERROR`:** instrucciones para lanzar un error propio o relanzar el capturado.
* **Error transitorio:** error que probablemente no se repita si se reintenta (un *deadlock*, un *timeout* de bloqueo).

-----

## El problema

Un procedimiento que registra pagos tiene este "manejo de errores":

```sql
BEGIN TRANSACTION;
    INSERT INTO dbo.Pago (...) VALUES (...);
    IF @@ERROR <> 0 GOTO Error;
    UPDATE dbo.Pedido SET Estado = 'Pagado' WHERE PedidoId = @PedidoId;
    IF @@ERROR <> 0 GOTO Error;
COMMIT;
RETURN;
Error:
    ROLLBACK;
```

Y tiene, como mínimo, cuatro problemas:

* **`@@ERROR` se reinicia en cada instrucción:** si se olvida un `IF` en una línea, ese error pasa inadvertido.
* **Algunos errores nunca llegan al `IF`:** un error de conversión, por ejemplo, aborta el lote completo y el `GOTO` no se ejecuta; la transacción queda **abierta**.
* **Quien llamó no se entera:** el procedimiento termina "bien" (sin error), así que la aplicación muestra "pago registrado".
* **No queda registro** de qué falló, ni de con qué datos.

`TRY...CATCH` existe desde SQL Server 2005, `THROW` desde 2012. Y aun así, el patrón de arriba sigue apareciendo en código heredado.

-----

## Cómo funciona

### Qué termina un error

No todos los errores tienen el mismo alcance:

| Alcance | Qué termina | Ejemplos |
| --- | --- | --- |
| **Instrucción** | Solo la instrucción que falló; la ejecución sigue con la siguiente | Violación de `CHECK`, FK o clave primaria (547, 2627), `NULL` en columna obligatoria (515) |
| **Lote** | Todo el lote; la transacción puede quedar abierta | Error de conversión (245), la mayoría de los errores de un `THROW` |
| **Conexión** | La conexión (severidad 20 o más) | Errores graves del servidor, corrupción |
| **Compilación** | El lote no llega a ejecutarse | Errores de sintaxis, objetos inexistentes en algunos casos |

La primera fila es la que sorprende: con la configuración por defecto, una violación de clave foránea **no** detiene el procedimiento, que sigue ejecutando las instrucciones siguientes (lo que se vio en el ejercicio 1 de [Transacciones y ACID](01-Transacciones%20y%20ACID.md)).

### Severidades

| Severidad | Significado | ¿Lo captura `CATCH`? |
| --- | --- | --- |
| 0 a 10 | Mensajes informativos (no son errores) | No (se muestran como mensajes) |
| 11 a 16 | Errores que el usuario puede corregir: datos, permisos, restricciones | Sí |
| 17 a 19 | Errores de recursos del servidor (memoria, disco) | Sí |
| 20 a 25 | Errores fatales: cierran la conexión | Solo si la conexión sobrevive |

### `SET XACT_ABORT ON`

Con `XACT_ABORT ON`, **cualquier** error en tiempo de ejecución deshace la transacción entera y aborta el lote, sin importar si era un error "de instrucción". Con `TRY...CATCH`, en lugar de abortar el lote, el control pasa al `CATCH` con la transacción **condenada**.

| | `XACT_ABORT OFF` (por defecto) | `XACT_ABORT ON` |
| --- | --- | --- |
| Violación de restricción sin `TRY` | Termina la instrucción; la transacción sigue abierta y "sana" | Deshace la transacción y aborta el lote |
| Violación de restricción con `TRY` | Va al `CATCH`; la transacción puede estar activa (1) | Va al `CATCH` con la transacción condenada (-1) |
| *Timeout* del cliente (la aplicación cancela) | **La transacción queda abierta** en el servidor | Se deshace la transacción |

Esa última fila es la razón más importante para usarlo siempre: cuando la aplicación cancela una consulta por *timeout*, el servidor recibe una "atención" que detiene la ejecución **sin pasar por el `CATCH`**. Sin `XACT_ABORT ON`, la transacción queda abierta, con sus bloqueos, hasta que la conexión se cierre o se reutilice desde el pool. Con `XACT_ABORT ON`, se deshace.

La recomendación habitual: **`SET XACT_ABORT ON` al principio de todo procedimiento que abra transacciones.** Es el "fail fast" que menciona tu cheat sheet para los sistemas bancarios, y vale para cualquier sistema.

### `TRY...CATCH`

```sql
BEGIN TRY
    -- código que puede fallar
END TRY
BEGIN CATCH
    -- se ejecuta si hubo un error en el TRY
END CATCH;
```

Dentro del `CATCH`, estas funciones describen el error (fuera del `CATCH` devuelven `NULL`):

| Función | Devuelve |
| --- | --- |
| `ERROR_NUMBER()` | El número (547, 2627, 1205...) |
| `ERROR_MESSAGE()` | El texto completo |
| `ERROR_SEVERITY()` | La severidad |
| `ERROR_STATE()` | El estado (un número que distingue variantes del mismo error) |
| `ERROR_LINE()` | La línea donde ocurrió |
| `ERROR_PROCEDURE()` | El procedimiento o *trigger* (`NULL` en una consulta suelta) |

Lo que `TRY...CATCH` **no** captura:

* **Errores de compilación en el mismo nivel:** un error de sintaxis o un objeto inexistente en el mismo lote. Sí se capturan si ocurren en un nivel inferior (dentro de un procedimiento llamado desde el `TRY`, o en SQL dinámico).
* **Errores de severidad 20 o más** que cierran la conexión.
* **Las atenciones del cliente** (*timeouts*, cancelaciones): la ejecución se detiene sin pasar por el `CATCH`.
* **`KILL`** de la sesión.

Otra corrección sobre tus apuntes: un `ROLLBACK` dentro del `TRY` **no** pasa el control al `CATCH`. El `CATCH` se ejecuta por un **error**; un `ROLLBACK` deshace la transacción y la ejecución sigue en la línea siguiente.

### `XACT_STATE()` y `@@TRANCOUNT`

| `XACT_STATE()` | Significa | ¿`COMMIT`? | ¿`ROLLBACK`? |
| --- | --- | --- | --- |
| 1 | Hay una transacción activa y sana | Sí | Sí |
| 0 | No hay transacción | No | No (daría error) |
| -1 | Hay una transacción **condenada** | **No** | Sí (es lo único posible) |

`@@TRANCOUNT` solo cuenta cuántos `BEGIN TRANSACTION` hay abiertos: puede valer 1 con la transacción condenada. Por eso, en el `CATCH`, `XACT_STATE()` es más precisa:

```sql
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;   -- hay algo que deshacer (activa o condenada)
    THROW;
END CATCH;
```

Con `XACT_ABORT ON`, la transacción en el `CATCH` siempre está condenada (-1), así que la única decisión es deshacer. Con `XACT_ABORT OFF`, puede estar activa (1), y en teoría se podría confirmar lo hecho hasta el error; en la práctica, casi nunca es lo que se quiere.

### `THROW` frente a `RAISERROR`

| | `THROW` | `RAISERROR` |
| --- | --- | --- |
| Disponible desde | SQL Server 2012 | Siempre |
| Relanzar el error capturado | `THROW;` (sin parámetros), conserva número, mensaje y línea originales | No puede: hay que armar un error nuevo con los datos |
| Error propio | `THROW 50001, N'Mensaje', 1;` (número ≥ 50000, severidad siempre 16) | `RAISERROR(N'Mensaje %s', 16, 1, @valor);` (severidad elegible, formato tipo `printf`) |
| Respeta `XACT_ABORT` | **Sí**: deshace la transacción | No |
| Termina el lote fuera de un `TRY` | Sí | No (con severidad 11–19, sigue) |
| Punto y coma | La instrucción anterior debe terminar en `;` | No |

Para código nuevo, `THROW`. `RAISERROR` sigue siendo útil para un caso: mensajes de progreso que se muestran **mientras** se ejecuta un proceso largo, con severidad 0 y `WITH NOWAIT` (`PRINT` espera a que se llene el búfer):

```sql
RAISERROR(N'Procesado el lote %d', 0, 1, @Lote) WITH NOWAIT;
```

### Errores transitorios y permanentes

| Transitorios (reintentar) | Permanentes (no reintentar) |
| --- | --- |
| **1205**: víctima de un *deadlock* | **547**: violación de `CHECK` o de clave foránea |
| **1222**: *timeout* de bloqueo (`LOCK_TIMEOUT`) | **2627** / **2601**: clave duplicada |
| **3960**: conflicto de actualización en `SNAPSHOT` | **515**: `NULL` en columna obligatoria |
| **8645**: *timeout* esperando memoria | **2628** / **8152**: texto truncado |
| Errores de conexión y de conmutación por error (en Azure SQL: 40613, 40197, 40501, entre otros) | **245**: error de conversión |
| | Errores propios de negocio (50000 en adelante) |

Reintentar un error permanente solo produce el mismo error otra vez, con más carga. Reintentar uno transitorio, con una espera, casi siempre funciona.

El *timeout* de conexión o de comando (`-2` en ADO.NET, que aparece en tu cheat sheet) es transitorio, pero ocurre **del lado del cliente**: el servidor nunca lo ve como un error, así que solo se puede reintentar desde la aplicación.

### Registrar el error: después del `ROLLBACK`

Otra corrección sobre tus apuntes de la sesión 2: el registro del error **no** se conserva por hacerlo en un procedimiento separado. Si el `INSERT` en la tabla de log se ejecuta **dentro** de la transacción, el `ROLLBACK` lo deshace, sin importar en qué procedimiento esté escrito. El orden correcto es:

```sql
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;   -- 1. deshacer primero
    EXEC dbo.RegistrarError @Contexto = N'...';  -- 2. registrar, ya fuera de la transacción
    THROW;                                       -- 3. avisar a quien llamó
END CATCH;
```

Las funciones `ERROR_*()` siguen disponibles después del `ROLLBACK`, mientras se esté dentro del `CATCH`. Si hace falta guardar datos generados **dentro** de la transacción (por ejemplo, los pasos de un proceso), un truco útil es acumularlos en una **variable de tabla**, que no se ve afectada por el `ROLLBACK`, e insertarlos en la tabla de log después.

-----

## Ejemplo completo

Una plantilla de procedimiento con el patrón completo, aplicada al registro de un pago en `Tienda`.

**La tabla y el procedimiento de log** (una vez):

```sql
CREATE TABLE dbo.LogError (
    LogErrorId    INT IDENTITY(1,1) PRIMARY KEY,
    FechaHora     DATETIME2(3)   NOT NULL DEFAULT SYSUTCDATETIME(),
    Numero        INT            NOT NULL,
    Severidad     INT            NOT NULL,
    Estado        INT            NOT NULL,
    Procedimiento NVARCHAR(128)  NULL,
    Linea         INT            NULL,
    Mensaje       NVARCHAR(4000) NOT NULL,
    Usuario       NVARCHAR(128)  NOT NULL DEFAULT SUSER_SNAME(),
    Contexto      NVARCHAR(1000) NULL
);
GO

CREATE OR ALTER PROCEDURE dbo.RegistrarError @Contexto NVARCHAR(1000) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    -- Llamarlo solo desde un CATCH y después del ROLLBACK
    INSERT INTO dbo.LogError (Numero, Severidad, Estado, Procedimiento, Linea, Mensaje, Contexto)
    VALUES (ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(), ERROR_PROCEDURE(),
            ERROR_LINE(), ERROR_MESSAGE(), @Contexto);
END;
GO
```

**El procedimiento de negocio:**

```sql
CREATE OR ALTER PROCEDURE dbo.MarcarPedidoPagado @PedidoId INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    -- Validaciones antes de la transacción: errores de negocio claros
    IF NOT EXISTS (SELECT 1 FROM dbo.Pedido WHERE PedidoId = @PedidoId)
        THROW 50101, N'El pedido no existe.', 1;

    DECLARE @Intento INT = 1;

    WHILE 1 = 1
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;

            UPDATE dbo.Pedido
            SET Estado = 'Pagado'
            WHERE PedidoId = @PedidoId
              AND Estado = 'Pendiente';             -- regla de negocio en el WHERE

            IF @@ROWCOUNT = 0
                THROW 50102, N'Solo se pueden pagar pedidos pendientes.', 1;

            -- ... registrar el pago, descontar stock, etc. ...

            COMMIT TRANSACTION;
            RETURN 0;
        END TRY
        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;

            -- Transitorio: reintentar con espera exponencial (200 ms, 400 ms)
            IF ERROR_NUMBER() IN (1205, 1222, 3960) AND @Intento < 3
            BEGIN
                DECLARE @Espera DATETIME = DATEADD(millisecond, 200 * POWER(2, @Intento - 1), 0);
                SET @Intento += 1;
                WAITFOR DELAY @Espera;
                CONTINUE;
            END;

            -- Permanente, o se agotaron los intentos: registrar y avisar
            DECLARE @Contexto NVARCHAR(1000) = CONCAT(N'PedidoId=', @PedidoId, N'; Intento=', @Intento);
            EXEC dbo.RegistrarError @Contexto = @Contexto;
            THROW;
        END CATCH;
    END;
END;
GO
```

Probándolo:

```sql
EXEC dbo.MarcarPedidoPagado @PedidoId = 99;   -- Msg 50101: El pedido no existe.
EXEC dbo.MarcarPedidoPagado @PedidoId = 1;    -- Msg 50102: el pedido 1 está Entregado

SELECT Numero, Procedimiento, Mensaje, Contexto FROM dbo.LogError;
```

| Numero | Procedimiento | Mensaje | Contexto |
| --- | --- | --- | --- |
| 50102 | dbo.MarcarPedidoPagado | Solo se pueden pagar pedidos pendientes. | PedidoId=1; Intento=1 |

Qué decisión hay detrás:

* **`XACT_ABORT ON`:** cualquier error deshace todo, también ante un *timeout* del cliente que no pasa por el `CATCH`.
* **Validaciones fuera de la transacción:** el pedido inexistente da un error de negocio claro, sin abrir una transacción ni registrarse como fallo técnico (el 50101 no aparece en el log, porque se lanzó fuera del `TRY`).
* **`XACT_STATE() <> 0`** antes del `ROLLBACK`: deshace tanto una transacción activa como una condenada, y no falla si no hay ninguna.
* **El reintento solo para los transitorios,** con espera creciente, y la transacción completa desde el `BEGIN`.
* **El registro después del `ROLLBACK`,** con el contexto (los parámetros) que permite reproducir el problema.
* **`THROW;` al final:** la aplicación recibe el error original (número, mensaje y línea), y puede mostrar un mensaje adecuado según el número.

-----

## Errores comunes

**1. Manejar errores con `@@ERROR`.**
Qué pasa: errores que pasan inadvertidos, porque `@@ERROR` se reinicia en cada instrucción, y errores de lote que nunca llegan al `IF`.
Por qué: código anterior a 2005, o copiado de él.
Arreglo: `TRY...CATCH`.

**2. `CATCH` que se "traga" el error.**
Qué pasa: el procedimiento termina sin error, y la aplicación cree que todo salió bien.
Por qué: un `CATCH` que hace `ROLLBACK` y nada más, o que devuelve un `SELECT 'ERROR'`.
Arreglo: `THROW;` al final del `CATCH`, para que quien llamó se entere.

**3. Registrar el error dentro de la transacción.**
Qué pasa: el log de errores queda vacío justo cuando más se necesita.
Por qué: el `ROLLBACK` deshace también el `INSERT` del log.
Arreglo: `ROLLBACK` primero, registrar después.

**4. No usar `XACT_ABORT ON`.**
Qué pasa: transacciones abiertas después de un *timeout* de la aplicación, con bloqueos que nadie libera.
Por qué: la atención del cliente no pasa por el `CATCH`.
Arreglo: `SET XACT_ABORT ON` en todos los procedimientos que abren transacciones.

**5. Reintentar cualquier error.**
Qué pasa: una violación de clave foránea que se reintenta tres veces, con el mismo resultado.
Por qué: no se distinguen los transitorios de los permanentes.
Arreglo: reintentar solo una lista explícita de errores transitorios.

**6. `THROW` sin punto y coma en la instrucción anterior.**
Qué pasa: error de sintaxis, o el `THROW` se interpreta como alias de la instrucción anterior.
Por qué: `THROW` exige que la instrucción previa termine en `;`.
Arreglo: terminar todas las instrucciones con `;` (o escribir `;THROW;`).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Capturar errores | `BEGIN TRY ... END TRY BEGIN CATCH ... END CATCH` | `BEGIN ... EXCEPTION WHEN ... THEN ... END` (en PL/pgSQL) | `DECLARE ... HANDLER FOR SQLEXCEPTION` (en procedimientos) |
| Lanzar un error propio | `THROW 50001, N'...', 1;` | `RAISE EXCEPTION '...' USING ERRCODE = 'P0001';` | `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...';` |
| Relanzar | `THROW;` | `RAISE;` | `RESIGNAL;` |
| Información del error | `ERROR_NUMBER()`, `ERROR_MESSAGE()`... | `SQLSTATE`, `SQLERRM`, `GET STACKED DIAGNOSTICS` | `GET DIAGNOSTICS CONDITION 1 ...` |
| Un error dentro de una transacción | Según el error (y `XACT_ABORT`) | La transacción queda abortada siempre | Termina la instrucción |
| Códigos estándar | Números propios | `SQLSTATE` estándar (`23505` = clave duplicada, `40P01` = *deadlock*) | Números propios + `SQLSTATE` |

En PostgreSQL, un bloque `EXCEPTION` crea internamente un punto de guardado: si se captura el error, solo se deshace lo hecho dentro de ese bloque. Es una forma de manejo de errores más estructurada, a cambio de un costo de rendimiento si se abusa de ella.

### En MongoDB

El manejo de errores ocurre en la aplicación, a través del driver: cada operación devuelve un error con un código (`11000` = clave duplicada, `112` = `WriteConflict`) y, en las transacciones, una etiqueta (`TransientTransactionError` o `UnknownTransactionCommitResult`) que indica si se puede reintentar. Los drivers implementan reintentos automáticos de lecturas y escrituras simples (*retryable reads/writes*), y `withTransaction` reintenta la transacción completa ante errores transitorios.

-----

## Cuándo sí y cuándo no

**Usa el patrón completo cuando:**

* El procedimiento abre una transacción o modifica datos de negocio.

**Reintenta en la base cuando:**

* La transacción es autocontenida en el procedimiento; si la operación de negocio incluye pasos fuera de la base (una API, un mensaje), el reintento va en la aplicación.

**Ten en cuenta:**

* Los errores de negocio (50000 en adelante) son parte del contrato del procedimiento: documenta sus números para que la aplicación los traduzca a mensajes para el usuario.

-----

## Resumen en 5 líneas

1. Un error puede terminar la instrucción, el lote o la conexión; sin `XACT_ABORT ON`, una violación de restricción no detiene la transacción.
2. `SET XACT_ABORT ON` + `TRY...CATCH`: cualquier error deshace todo, también ante un *timeout* del cliente.
3. En el `CATCH`: `IF XACT_STATE() <> 0 ROLLBACK;`, después registrar, después `THROW;`.
4. `THROW` para código nuevo (respeta `XACT_ABORT` y relanza el original); `RAISERROR ... WITH NOWAIT` para mensajes de progreso.
5. Reintenta solo los transitorios (1205, 1222, 3960...) con espera creciente; los permanentes se registran y se informan.

-----

## Para profundizar

<details>
<summary>Mensajes de error propios con sys.messages</summary>

`sp_addmessage` permite registrar mensajes propios con número, severidad e idiomas, para lanzarlos con `RAISERROR(50201, 16, 1)` o formatearlos con `FORMATMESSAGE(50201, @valor)` y lanzarlos con `THROW`. Centraliza los textos, pero los mensajes quedan a nivel de **servidor** (no de base), lo que complica los despliegues y las restauraciones en otro servidor. Muchos equipos prefieren mantener los mensajes en el código o en una tabla de la base.

</details>

<details>
<summary>Cómo llega el error a la aplicación</summary>

En .NET, un error de SQL Server llega como `SqlException`, con la propiedad `Number` (el `ERROR_NUMBER()`) y, si hubo varios, la colección `Errors`. Una aplicación bien diseñada traduce los números conocidos: 2627 o 2601 → "ya existe un registro con esos datos"; 547 → "no se puede borrar porque tiene registros asociados"; 50102 → el mensaje de negocio tal cual; 1205 → reintentar sin mostrar nada al usuario. EF Core ofrece `EnableRetryOnFailure()` para los transitorios, y Polly permite políticas de reintento más finas (ver el wiki de .NET).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso `TRY...CATCH`: si hay un error dentro del `TRY`, el control pasa al `CATCH`, donde hago `ROLLBACK` de la transacción y relanzo el error con `THROW`. Con `ERROR_NUMBER()` y `ERROR_MESSAGE()` obtengo los datos del error para registrarlo.

### Respuesta ampliada (semi-senior)

Sé que en SQL Server un error puede terminar solo la instrucción, el lote o la conexión, y que por defecto una violación de restricción deja la transacción abierta y la ejecución sigue; por eso todo procedimiento que abre transacciones empieza con `SET XACT_ABORT ON`, que además deshace la transacción cuando la aplicación cancela por *timeout*, algo que el `CATCH` no captura. En el `CATCH`, uso `XACT_STATE()` y no solo `@@TRANCOUNT`, porque la transacción puede estar condenada; hago el `ROLLBACK` primero, registro el error después (si no, el `ROLLBACK` deshace también el log) y relanzo con `THROW;`, que conserva el número y la línea originales y respeta `XACT_ABORT`. Hago las validaciones de negocio antes de la transacción, con errores propios documentados desde el 50000. Clasifico los errores: reintento solo los transitorios, como el 1205, el 1222 o el 3960, con espera creciente y la transacción completa, y nunca los permanentes, como el 547 o el 2627. Y sé lo que `TRY...CATCH` no captura: errores de compilación del mismo nivel, severidades fatales y las atenciones del cliente.

### Preguntas frecuentes de seguimiento

**1. ¿Qué hace `SET XACT_ABORT ON`?**
Hace que cualquier error en tiempo de ejecución deshaga la transacción entera; con `TRY...CATCH`, el control pasa al `CATCH` con la transacción condenada.

**2. ¿Qué diferencia hay entre `@@TRANCOUNT` y `XACT_STATE()`?**
`@@TRANCOUNT` cuenta las transacciones abiertas; `XACT_STATE()` dice además si la transacción puede confirmarse (1) o solo deshacerse (-1).

**3. ¿`THROW` o `RAISERROR`?**
`THROW`: relanza el error original, respeta `XACT_ABORT` y es la opción moderna. `RAISERROR` para mensajes informativos de progreso con `WITH NOWAIT`.

**4. ¿Qué errores reintentarías?**
Los transitorios: *deadlock* (1205), *timeout* de bloqueo (1222), conflicto de `SNAPSHOT` (3960) y errores de conexión. Nunca los de restricciones o datos inválidos.

**5. ¿Cómo te aseguras de que el error quede registrado aunque se deshaga la transacción?**
Registrándolo después del `ROLLBACK`, todavía dentro del `CATCH`.

-----

## Práctica

**Ejercicio 1.** ¿Qué imprime este script, y qué queda en la tabla?

```sql
CREATE TABLE dbo.Prueba (Id INT PRIMARY KEY);

SET XACT_ABORT OFF;
BEGIN TRY
    BEGIN TRANSACTION;
        INSERT INTO dbo.Prueba VALUES (1);
        INSERT INTO dbo.Prueba VALUES (1);
        INSERT INTO dbo.Prueba VALUES (2);
    COMMIT;
END TRY
BEGIN CATCH
    PRINT CONCAT('Error ', ERROR_NUMBER(), ' - XACT_STATE: ', XACT_STATE());
    IF XACT_STATE() <> 0 ROLLBACK;
END CATCH;

SELECT COUNT(*) AS Filas FROM dbo.Prueba;
```

<details>
<summary>Solución</summary>

Imprime `Error 2627 - XACT_STATE: 1`, y la tabla queda **vacía** (`Filas = 0`).

A diferencia del ejercicio sin `TRY...CATCH` de [Transacciones y ACID](01-Transacciones%20y%20ACID.md), aquí el error de la segunda instrucción **sí** pasa el control al `CATCH`: el tercer `INSERT` y el `COMMIT` no se ejecutan. Con `XACT_ABORT OFF`, la transacción queda activa (1) en el `CATCH` (se podría confirmar el primer `INSERT`), pero el `ROLLBACK` la deshace. Con `XACT_ABORT ON`, el resultado sería el mismo, salvo que `XACT_STATE()` valdría -1.

</details>

**Ejercicio 2.** Corrige este `CATCH` para que el error quede registrado, la transacción se deshaga y la aplicación se entere:

```sql
BEGIN CATCH
    EXEC dbo.RegistrarError;
    ROLLBACK;
    SELECT 'ERROR' AS Resultado;
END CATCH;
```

<details>
<summary>Solución</summary>

```sql
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;   -- 1. deshacer primero
    EXEC dbo.RegistrarError;                      -- 2. registrar fuera de la transacción
    THROW;                                        -- 3. relanzar el error original
END CATCH;
```

Los tres problemas del original:

* **El registro estaba antes del `ROLLBACK`:** quedaba dentro de la transacción y se deshacía con ella.
* **`ROLLBACK` sin verificar:** si no hay transacción (`XACT_STATE() = 0`), el `ROLLBACK` da otro error dentro del `CATCH`, que oculta el original.
* **`SELECT 'ERROR'` en lugar de `THROW`:** para la aplicación, el procedimiento terminó bien y devolvió un resultado; tiene que leer una columna de texto para saber que falló. Con `THROW`, recibe una excepción con el número y el mensaje.

</details>

-----

## Siguiente lección

[Operaciones masivas en lotes](07-Operaciones%20masivas%20en%20lotes.md)
