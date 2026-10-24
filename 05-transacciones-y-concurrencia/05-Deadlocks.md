# Deadlocks

## En una frase

Un *deadlock* (interbloqueo) ocurre cuando dos transacciones se esperan **mutuamente**, cada una con un recurso que la otra necesita, y ninguna puede avanzar; SQL Server lo detecta, elige una **víctima**, deshace su transacción con el error 1205 y deja seguir a la otra, así que los *deadlocks* no se "arreglan" esperando: se previenen accediendo a los datos en un orden consistente, con transacciones cortas e índices adecuados, y se manejan reintentando.

-----

## Antes de empezar

Conviene que ya sepas:

* Modos de bloqueo, compatibilidad y el bloqueo U, de [Bloqueos, hints y NOLOCK](03-Bloqueos%2C%20hints%20y%20NOLOCK.md).
* Versionado y RCSI, de [Versionado con RCSI y SNAPSHOT](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Deadlock (interbloqueo):** ciclo de esperas en el que cada transacción espera un recurso que tiene otra del ciclo.
* **Víctima:** la transacción que SQL Server elige deshacer para romper el ciclo.
* **Monitor de bloqueos (lock monitor):** el proceso interno que busca ciclos de espera.
* **Gráfico de deadlock (deadlock graph):** el XML que describe el ciclo: qué sesiones, qué recursos y qué consultas participaron.
* **`DEADLOCK_PRIORITY`:** configuración de la sesión que influye en cuál se elige como víctima.
* **Orden de acceso consistente:** acceder a los recursos compartidos siempre en el mismo orden.

-----

## El problema

Dos transferencias de stock entre productos, al mismo tiempo:

| Momento | Sesión A: pasar 5 unidades del producto 1 al 2 | Sesión B: pasar 3 unidades del producto 2 al 1 |
| --- | --- | --- |
| t1 | `UPDATE ... WHERE ProductoId = 1;` → bloqueo X en el producto 1 | |
| t2 | | `UPDATE ... WHERE ProductoId = 2;` → bloqueo X en el producto 2 |
| t3 | `UPDATE ... WHERE ProductoId = 2;` → **espera a B** | |
| t4 | | `UPDATE ... WHERE ProductoId = 1;` → **espera a A** |

A espera a B y B espera a A. Si nadie interviniera, esperarían para siempre. Unos segundos después, una de las dos recibe:

```text
Msg 1205, Level 13, State 51
Transaction (Process ID 61) was deadlocked on lock resources with another process and has been
chosen as the deadlock victim. Rerun the transaction.
```

Su transacción se deshizo entera, la otra terminó bien, y la aplicación muestra un error que el usuario no entiende. Sucede "de vez en cuando", nunca en las pruebas, y casi siempre en los momentos de más tráfico.

-----

## Cómo funciona

### Bloqueo frente a deadlock

| | Bloqueo (*blocking*) | Deadlock |
| --- | --- | --- |
| Qué es | Una sesión espera a otra | Dos (o más) sesiones se esperan mutuamente, en ciclo |
| ¿Se resuelve solo? | Sí, cuando la otra termina | **No**: sin intervención, esperarían para siempre |
| Qué hace SQL Server | Nada: la sesión espera | Elige una víctima y la deshace |
| Síntoma | Lentitud, *timeouts* | Error 1205 |

Un bloqueo largo es lento pero termina; un *deadlock* nunca termina por sí mismo.

### Cómo los detecta SQL Server

El **monitor de bloqueos** revisa periódicamente (cada 5 segundos por defecto, y con más frecuencia cuando encuentra *deadlocks*) el grafo de quién espera a quién. Cuando encuentra un ciclo, elige la víctima:

1. La sesión con menor **`DEADLOCK_PRIORITY`** (`LOW`, `NORMAL`, `HIGH` o un número de -10 a 10; por defecto `NORMAL`).
2. A igual prioridad, la que sea **más barata de deshacer** (la que menos escribió en el log).

```sql
SET DEADLOCK_PRIORITY LOW;   -- en un proceso de fondo que prefiere ceder ante los usuarios
```

### Los patrones más frecuentes

| Patrón | Qué pasa | Prevención |
| --- | --- | --- |
| **Orden inverso** | A bloquea 1 y quiere 2; B bloquea 2 y quiere 1 (el problema de arriba) | Acceder siempre en el mismo orden |
| **Conversión** | A y B leen la misma fila con S (`REPEATABLE READ` o una lectura dentro de la transacción) y después las dos quieren X | Leer con `UPDLOCK`, o hacer la operación con un `UPDATE` atómico |
| **Key Lookup** | Una consulta busca por un índice no agrupado y hace *lookup* al agrupado; un `UPDATE` bloquea el agrupado y quiere actualizar el no agrupado | Un índice cubriente, para que la consulta no necesite el agrupado |
| **Clave foránea y cascadas** | El `DELETE` de un padre revisa o borra filas hijas mientras otra transacción inserta hijas | Índices sobre las FK, y orden consistente padre → hija |
| **Paralelismo** (*intra-query*) | Los hilos de una misma consulta paralela se esperan entre sí | Normalmente un índice que evite el plan paralelo, o `MAXDOP` |

El de la conversión es especialmente traicionero, porque aparece en el código que parece más "seguro":

```sql
BEGIN TRANSACTION;
    SELECT @Stock = Stock FROM dbo.Producto WITH (HOLDLOCK) WHERE ProductoId = 3;   -- S, mantenido
    UPDATE dbo.Producto SET Stock = @Stock - 1 WHERE ProductoId = 3;               -- quiere X
COMMIT;
```

Dos sesiones que ejecutan esto a la vez obtienen las dos el bloqueo S (son compatibles), y después las dos esperan convertirlo a X, que es incompatible con el S de la otra: *deadlock* garantizado. Con `UPDLOCK` en lugar de `HOLDLOCK`, la segunda sesión espera desde el `SELECT` y no hay ciclo.

### Ver el gráfico de un deadlock

SQL Server registra automáticamente los *deadlocks* en la sesión de Extended Events `system_health`, que está siempre activa:

```sql
SELECT xed.value('@timestamp', 'datetime2') AS Momento,
       xed.query('.')                       AS GraficoDeadlock
FROM (
    SELECT CAST(st.target_data AS XML) AS TargetData
    FROM sys.dm_xe_session_targets AS st
    JOIN sys.dm_xe_sessions AS s ON s.address = st.event_session_address
    WHERE s.name = 'system_health' AND st.target_name = 'ring_buffer'
) AS d
CROSS APPLY d.TargetData.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS x(xed)
ORDER BY Momento DESC;
```

Al hacer clic en el XML, SSMS lo muestra; guardado con extensión `.xdl`, SSMS lo dibuja como gráfico. Se lee así:

```text
 ┌───────────────┐   tiene X sobre   ┌──────────────────────┐
 │ Proceso 58 ✕  │ ────────────────► │ KEY: PK_Producto (1) │
 │ (víctima)     │                   └──────────────────────┘
 └───────────────┘                              ▲
        │ espera U/X sobre                      │ espera U/X sobre
        ▼                                       │
 ┌──────────────────────┐   tiene X sobre ┌───────────────┐
 │ KEY: PK_Producto (2) │ ◄───────────────│ Proceso 61    │
 └──────────────────────┘                 └───────────────┘
```

* Los **óvalos** (aquí, cajas) son los procesos; el tachado es la víctima. Al pasar el mouse se ve la consulta que estaba ejecutando cada uno.
* Los **rectángulos** son los recursos (una clave de un índice, una página, un objeto).
* Las **flechas** dicen quién tiene qué y quién espera qué.

Con el gráfico, la pregunta "¿por qué hubo un *deadlock*?" deja de ser una especulación: muestra las dos consultas exactas y el orden en que tomaron los recursos. El gráfico es solo el XML del ring buffer en memoria; para conservarlos más tiempo se crea una sesión de Extended Events propia con destino a un archivo (ver [Extended Events](../07-monitoreo-y-administracion/03-Extended%20Events.md)).

### Prevenir

1. **Orden de acceso consistente.** Si todas las transacciones que tocan productos los toman en orden de `ProductoId`, el ciclo es imposible: la segunda espera a la primera desde el primer recurso. Es la técnica del patrón bancario de tu cheat sheet (bloquear primero la cuenta con el ID menor).
2. **Transacciones cortas.** Menos tiempo con bloqueos = menos oportunidades de cruzarse. Calcular y validar antes del `BEGIN TRANSACTION`.
3. **Índices adecuados.** Una consulta que va directo a sus filas bloquea menos y menos tiempo; un índice cubriente elimina el *deadlock* de *Key Lookup*.
4. **`UPDLOCK` al leer para modificar,** en lugar de `HOLDLOCK` o `REPEATABLE READ`, para eliminar los de conversión.
5. **RCSI.** Las lecturas dejan de pedir bloqueos compartidos, así que los *deadlocks* entre lectores y escritores desaparecen. Los de escritor contra escritor siguen siendo posibles.

Nada de esto los elimina al 100 % en un sistema con mucha concurrencia. Por eso, además de prevenir, hay que **manejarlos**.

### Manejar: reintentar

El error 1205 es **transitorio**: la misma transacción, ejecutada otra vez un instante después, normalmente funciona. La forma correcta de manejarlo es reintentar la transacción completa, con una espera breve:

```sql
DECLARE @Intento INT = 1, @MaxIntentos INT = 3;

WHILE @Intento <= @MaxIntentos
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
            -- ... la transacción completa ...
        COMMIT TRANSACTION;
        BREAK;                                       -- éxito: salir del bucle
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        IF ERROR_NUMBER() = 1205 AND @Intento < @MaxIntentos
        BEGIN
            SET @Intento += 1;
            WAITFOR DELAY '00:00:00.200';            -- breve espera antes de reintentar
        END
        ELSE
            THROW;                                   -- otro error, o se agotaron los intentos
    END CATCH;
END;
```

En la práctica, el reintento suele estar en la **aplicación** (por ejemplo, con Polly en .NET, o con las estrategias de reintento de EF Core), que puede reintentar toda la operación de negocio, incluidas las partes que no están en la base. El patrón completo, con espera exponencial y clasificación de errores, se ve en [Manejo de errores](06-Manejo%20de%20errores.md).

-----

## Ejemplo completo

La transferencia de stock del problema, corregida en `Tienda`:

**Antes** (cada sesión bloquea en el orden en que recibe los parámetros):

```sql
CREATE OR ALTER PROCEDURE dbo.TransferirStock @Desde INT, @Hacia INT, @Cantidad INT
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        UPDATE dbo.Producto SET Stock = Stock - @Cantidad WHERE ProductoId = @Desde;
        WAITFOR DELAY '00:00:05';   -- solo para poder cruzar dos ventanas en la prueba
        UPDATE dbo.Producto SET Stock = Stock + @Cantidad WHERE ProductoId = @Hacia;
    COMMIT;
END;
```

Ventana A: `EXEC dbo.TransferirStock 1, 2, 5;` — Ventana B, un segundo después: `EXEC dbo.TransferirStock 2, 1, 3;`. Resultado: una de las dos recibe el error 1205.

**Después** (orden consistente, validación y reintento):

```sql
CREATE OR ALTER PROCEDURE dbo.TransferirStock @Desde INT, @Hacia INT, @Cantidad INT
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;

    DECLARE @Primero INT = IIF(@Desde < @Hacia, @Desde, @Hacia);
    DECLARE @Segundo INT = IIF(@Desde < @Hacia, @Hacia, @Desde);
    DECLARE @Intento INT = 1;

    WHILE 1 = 1
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;

            -- 1. Bloquear las dos filas SIEMPRE en el mismo orden (el ID menor primero)
            SELECT ProductoId FROM dbo.Producto WITH (UPDLOCK) WHERE ProductoId = @Primero;
            SELECT ProductoId FROM dbo.Producto WITH (UPDLOCK) WHERE ProductoId = @Segundo;

            -- 2. Modificar, con la regla de negocio en el WHERE
            UPDATE dbo.Producto SET Stock = Stock - @Cantidad
            WHERE ProductoId = @Desde AND Stock >= @Cantidad;
            IF @@ROWCOUNT = 0 THROW 50002, N'Stock insuficiente en el producto de origen.', 1;

            UPDATE dbo.Producto SET Stock = Stock + @Cantidad WHERE ProductoId = @Hacia;

            COMMIT TRANSACTION;
            RETURN;
        END TRY
        BEGIN CATCH
            IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
            IF ERROR_NUMBER() = 1205 AND @Intento < 3
            BEGIN
                SET @Intento += 1;
                WAITFOR DELAY '00:00:00.100';
            END
            ELSE
                THROW;
        END CATCH;
    END;
END;
```

Con las mismas dos ventanas:

| Momento | Ventana A (1 → 2) | Ventana B (2 → 1) |
| --- | --- | --- |
| t1 | Bloquea el producto **1** (U) | |
| t2 | | Quiere el producto **1** primero (su `@Primero` también es 1): **espera a A** |
| t3 | Bloquea el 2, transfiere, confirma | |
| t4 | | Obtiene el 1, después el 2, transfiere, confirma |

Qué cambió y por qué:

* **Orden consistente:** las dos sesiones toman primero el producto de ID menor, sin importar la dirección de la transferencia. La segunda espera desde el primer recurso, así que no hay ciclo: hay un bloqueo breve, que se resuelve solo.
* **`UPDLOCK` en los `SELECT`:** reserva las filas para modificar, sin el *deadlock* de conversión que tendría `HOLDLOCK`.
* **La regla de stock en el `WHERE`,** con `@@ROWCOUNT`, como en [Niveles de aislamiento](02-Niveles%20de%20aislamiento%20y%20anomalías.md).
* **El reintento** cubre los *deadlocks* que todavía puedan ocurrir con **otros** procesos que no respeten el orden. Se limita a 3 intentos y solo para el error 1205: cualquier otro error se relanza.

-----

## Errores comunes

**1. Tratar el 1205 como un error definitivo.**
Qué pasa: el usuario ve un error y pierde su operación, que habría funcionado un instante después.
Por qué: no se distingue entre errores transitorios y permanentes.
Arreglo: reintentar la transacción completa ante un 1205, con un límite de intentos.

**2. Reintentar solo la instrucción que falló.**
Qué pasa: datos inconsistentes, porque la víctima ya se deshizo entera.
Por qué: se asume que solo se perdió la última instrucción.
Arreglo: reintentar **toda** la transacción, desde el `BEGIN`.

**3. Leer con `HOLDLOCK` o `REPEATABLE READ` para "proteger" una modificación.**
Qué pasa: *deadlocks* de conversión cuando dos sesiones hacen lo mismo.
Por qué: dos bloqueos S compatibles, y después las dos quieren X.
Arreglo: `UPDLOCK`.

**4. Acceder a las tablas en distinto orden según el procedimiento.**
Qué pasa: un procedimiento actualiza `Pedido` y después `Producto`, y otro al revés; se cruzan en los momentos de más tráfico.
Por qué: cada procedimiento se escribió por separado.
Arreglo: una convención de orden (por ejemplo, siempre padre antes que hija, y por clave dentro de cada tabla).

**5. "Resolverlos" con `NOLOCK`.**
Qué pasa: desaparecen los *deadlocks* de lectura contra escritura, y aparecen lecturas sucias.
Por qué: se cambia un problema por otro.
Arreglo: RCSI, que elimina esos *deadlocks* sin lecturas sucias.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Detección | Monitor de bloqueos (cada 5 s por defecto) | Después de esperar `deadlock_timeout` (1 s por defecto) | Inmediata, en cada espera (`innodb_deadlock_detect`) |
| Error | 1205 | `40P01` (`deadlock detected`) | 1213 (`Deadlock found when trying to get lock`) |
| Elegir la víctima | `DEADLOCK_PRIORITY`, después la más barata de deshacer | La que detecta el ciclo | La que modificó menos filas |
| Ver el último | `system_health` (Extended Events) | El log del servidor (`log_lock_waits`) | `SHOW ENGINE INNODB STATUS` (sección *LATEST DETECTED DEADLOCK*) |
| Lectores contra escritores | Posible sin RCSI | No (MVCC) | No en lecturas normales (MVCC) |

Los tres motores tienen los mismos patrones de escritor contra escritor y las mismas soluciones: orden consistente, transacciones cortas, índices y reintento.

### En MongoDB

Dentro de una transacción multidocumento, dos transacciones que modifican los mismos documentos en distinto orden producen un `WriteConflict` en lugar de un *deadlock* clásico: una de las dos aborta y la etiqueta de error `TransientTransactionError` indica que se puede reintentar entera. Los drivers oficiales ofrecen una API (`session.withTransaction(...)`) que reintenta automáticamente ante esos errores.

-----

## Cuándo sí y cuándo no

**Investiga un deadlock cuando:**

* Se repite: el gráfico de `system_health` muestra las consultas y el orden de los recursos.

**Agrega reintentos cuando:**

* La operación es una transacción de negocio que puede ejecutarse dos veces sin efectos duplicados (idempotente dentro de la transacción).

**Ten en cuenta:**

* Un *deadlock* esporádico en un sistema con mucha concurrencia es normal y se maneja con reintentos; muchos *deadlocks* por minuto son un problema de diseño que hay que corregir.

-----

## Resumen en 5 líneas

1. Un *deadlock* es un ciclo de esperas; SQL Server elige una víctima, la deshace y devuelve el error 1205.
2. Patrones típicos: orden inverso, conversión de S a X, *Key Lookup* contra `UPDATE`, claves foráneas y paralelismo.
3. El gráfico de `system_health` muestra las consultas y los recursos de cada *deadlock*: no hay que adivinar.
4. Prevención: orden consistente, transacciones cortas, índices adecuados, `UPDLOCK` en lugar de `HOLDLOCK`, y RCSI.
5. El 1205 es transitorio: reintenta la transacción completa, con un límite y una espera breve.

-----

## Para profundizar

<details>
<summary>El deadlock de Key Lookup, paso a paso</summary>

Una consulta `SELECT Nombre, Precio FROM dbo.Producto WHERE CategoriaId = 1` usa un índice no agrupado sobre `CategoriaId` y hace *Key Lookup* al agrupado para traer `Precio`: bloquea primero la entrada del no agrupado (S) y después la del agrupado (S). Al mismo tiempo, un `UPDATE dbo.Producto SET Precio = ... WHERE ProductoId = 1` bloquea primero el agrupado (X) y, si `CategoriaId` también cambiara o el índice incluyera `Precio`, después necesitaría el no agrupado (X). Cada una toma los índices en orden inverso a la otra. La solución es un índice cubriente (`(CategoriaId) INCLUDE (Nombre, Precio)`), con el que la consulta no necesita el agrupado; o RCSI, con el que la consulta no pide bloqueos S.

</details>

<details>
<summary>Habilitar más detalle de los deadlocks en el log de errores</summary>

Los *trace flags* 1222 y 1204 escriben el detalle de cada *deadlock* en el log de errores de SQL Server (`DBCC TRACEON (1222, -1)`). Eran la técnica clásica antes de Extended Events. Hoy, `system_health` (siempre activo) o una sesión de Extended Events propia con el evento `xml_deadlock_report` dan la misma información en un formato más útil.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un *deadlock* pasa cuando dos transacciones se bloquean mutuamente: cada una tiene un recurso que la otra necesita. SQL Server lo detecta, cancela una (la víctima) con el error 1205 y deja seguir a la otra. Se previene accediendo a las tablas siempre en el mismo orden y con transacciones cortas, y se maneja reintentando.

### Respuesta ampliada (semi-senior)

Distingo un bloqueo, que se resuelve cuando la otra sesión termina, de un *deadlock*, que es un ciclo que el monitor de bloqueos detecta y rompe eligiendo como víctima la sesión de menor `DEADLOCK_PRIORITY` o la más barata de deshacer. Para diagnosticar uso el gráfico de `system_health`, que muestra las consultas y el orden en que tomaron los recursos. Los patrones que más encuentro son el orden inverso, la conversión de S a X (por leer con `HOLDLOCK` o `REPEATABLE READ` antes de modificar), el de *Key Lookup* contra un `UPDATE` y los de claves foráneas sin índice. Los prevengo con orden de acceso consistente (por ejemplo, el ID menor primero), transacciones cortas, índices cubrientes, `UPDLOCK` para leer antes de modificar y RCSI para eliminar los de lector contra escritor. Y como no se eliminan al 100 %, trato el 1205 como transitorio: reintento la transacción completa, con un límite y una espera, normalmente desde la aplicación.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre un bloqueo y un deadlock?**
Un bloqueo es una espera que termina cuando la otra sesión libera el recurso; un *deadlock* es un ciclo de esperas que nunca terminaría solo.

**2. ¿Cómo elige SQL Server la víctima?**
Por `DEADLOCK_PRIORITY` y, a igual prioridad, la transacción más barata de deshacer.

**3. ¿Cómo encuentras la causa de un deadlock que ya ocurrió?**
Con el gráfico de *deadlock* de la sesión de Extended Events `system_health`.

**4. ¿Qué haces cuando la aplicación recibe un error 1205?**
Reintentar la transacción completa, con un límite de intentos y una espera breve.

-----

## Práctica

**Ejercicio 1.** ¿Puede producir un *deadlock* este procedimiento si dos sesiones lo ejecutan a la vez con el mismo `@PedidoId`? ¿Por qué? ¿Cómo lo corregirías?

```sql
CREATE PROCEDURE dbo.AgregarLinea @PedidoId INT, @ProductoId INT, @Cantidad INT
AS
BEGIN
    SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    BEGIN TRANSACTION;
        DECLARE @Lineas INT = (SELECT COUNT(*) FROM dbo.DetallePedido WHERE PedidoId = @PedidoId);
        IF @Lineas >= 10 THROW 50020, N'Máximo 10 líneas por pedido.', 1;
        INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
        SELECT @PedidoId, @ProductoId, @Cantidad, Precio FROM dbo.Producto WHERE ProductoId = @ProductoId;
    COMMIT;
END;
```

<details>
<summary>Solución</summary>

**Sí.** Con `REPEATABLE READ`, las dos sesiones leen las líneas del pedido con bloqueos S que se mantienen hasta el final; después las dos intentan insertar, lo que necesita bloqueos incompatibles en las mismas páginas o claves del pedido. Es un *deadlock* de conversión. Y además, `REPEATABLE READ` no evita los fantasmas: las dos podrían contar 9 e insertar, dejando 11 líneas.

Corrección: serializar las operaciones sobre **el pedido**, bloqueando su fila con `UPDLOCK` antes de contar:

```sql
CREATE OR ALTER PROCEDURE dbo.AgregarLinea @PedidoId INT, @ProductoId INT, @Cantidad INT
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        -- Serializa a quienes agregan líneas al MISMO pedido
        SELECT PedidoId FROM dbo.Pedido WITH (UPDLOCK) WHERE PedidoId = @PedidoId;

        IF (SELECT COUNT(*) FROM dbo.DetallePedido WHERE PedidoId = @PedidoId) >= 10
            THROW 50020, N'Máximo 10 líneas por pedido.', 1;

        INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
        SELECT @PedidoId, @ProductoId, @Cantidad, Precio FROM dbo.Producto WHERE ProductoId = @ProductoId;
    COMMIT;
END;
```

La segunda sesión espera en el `SELECT ... WITH (UPDLOCK)` sobre el pedido hasta que la primera confirme, y recién entonces cuenta: ya ve la línea nueva. Es la técnica de bloquear el "padre" para serializar las operaciones sobre sus hijas, y resuelve tanto el *deadlock* como la regla de máximo 10 líneas.

</details>

**Ejercicio 2.** Escribe la consulta que lista los *deadlocks* registrados por `system_health` y explica qué buscarías en cada gráfico.

<details>
<summary>Solución</summary>

La consulta es la de la sección "Ver el gráfico de un deadlock". En cada gráfico buscaría:

1. **Las consultas de cada proceso** (al pasar el mouse sobre el proceso, o en el nodo `inputbuf` del XML): qué procedimientos o instrucciones se cruzaron.
2. **Los recursos** (`keylock`, `pagelock`, `objectlock`) y el **índice** de cada uno: si un recurso es de un índice no agrupado y el otro del agrupado de la misma tabla, sospecharía del patrón de *Key Lookup*.
3. **Los modos**: si los dos procesos tienen S y esperan X, es un *deadlock* de conversión.
4. **El orden**: qué recurso tomó primero cada proceso; si es inverso, el arreglo es un orden consistente.
5. **El nivel de aislamiento** de cada proceso (atributo `isolationlevel`): un `serializable` o `repeatable read` inesperado suele venir de un `TransactionScope` de .NET, que por defecto usa `SERIALIZABLE`.

</details>

-----

## Siguiente lección

[Manejo de errores](06-Manejo%20de%20errores.md)
