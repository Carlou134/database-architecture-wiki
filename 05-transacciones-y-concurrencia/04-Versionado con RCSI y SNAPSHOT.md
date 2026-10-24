# Versionado con RCSI y SNAPSHOT

## En una frase

Con el **versionado de filas**, cada modificación guarda la versión anterior de la fila, y las lecturas leen la última versión **confirmada** en lugar de esperar los bloqueos de las escrituras: RCSI lo hace para cada instrucción (sin cambiar el código), `SNAPSHOT` para toda una transacción (una foto consistente), y los dos eliminan la necesidad de `NOLOCK` sin sus lecturas sucias.

-----

## Antes de empezar

Conviene que ya sepas:

* Niveles de aislamiento y anomalías, de [Niveles de aislamiento y anomalías](02-Niveles%20de%20aislamiento%20y%20anomalías.md).
* Bloqueos y por qué `NOLOCK` es peligroso, de [Bloqueos, hints y NOLOCK](03-Bloqueos%2C%20hints%20y%20NOLOCK.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Versionado de filas (row versioning):** guardar la versión anterior de cada fila modificada, para que las lecturas puedan usarla.
* **Almacén de versiones (version store):** donde se guardan esas versiones; en `tempdb` (o en la propia base con ADR).
* **MVCC (control de concurrencia multiversión):** el modelo general en el que lectores y escritores trabajan sobre versiones distintas sin bloquearse.
* **RCSI (READ COMMITTED SNAPSHOT ISOLATION):** opción de la base que hace que `READ COMMITTED` lea versiones en lugar de pedir bloqueos.
* **Aislamiento `SNAPSHOT`:** nivel en el que toda la transacción lee la foto de los datos confirmados al momento de su primera lectura.
* **Conflicto de actualización:** error de `SNAPSHOT` cuando una transacción intenta modificar una fila que otra modificó después de que empezó la foto.

-----

## El problema

En la lección anterior, una transacción de escritura olvidada dejó a todos los vendedores esperando, y la "solución" de `NOLOCK` hizo que las pantallas mostraran datos que nunca se confirmaron. Hay un tercer camino, que combina lo bueno de los dos:

| | Espera las escrituras | Lee datos sin confirmar |
| --- | --- | --- |
| `READ COMMITTED` (por defecto) | ❌ Sí, espera | ✅ No |
| `NOLOCK` | ✅ No espera | ❌ Sí, lee sucio |
| **Versionado (RCSI / `SNAPSHOT`)** | ✅ **No espera** | ✅ **No lee sucio** |

La idea: si una fila está siendo modificada, la lectura no espera ni lee el valor a medio cambiar: lee **la última versión confirmada**, que el motor guardó en el momento de la modificación.

-----

## Cómo funciona

### Cómo funciona el versionado

Con el versionado activo, cada `UPDATE` o `DELETE` copia la versión anterior de la fila al **almacén de versiones** antes de modificarla, y la fila nueva apunta a la anterior:

```text
Fila actual (sin confirmar, de la transacción T2):  Pedido 3, Estado = 'Enviado'
      │ apunta a
      ▼
Versión anterior (confirmada, en el almacén):       Pedido 3, Estado = 'Pagado'  (confirmada por T1)
```

Una lectura que llega mientras T2 está abierta no necesita esperar: recorre la cadena hasta la versión confirmada que le corresponde y la devuelve. Las escrituras siguen bloqueándose entre sí (dos `UPDATE` sobre la misma fila se esperan, como siempre); lo que cambia es que **las lecturas y las escrituras dejan de bloquearse mutuamente**.

Cada fila de una base con versionado activo lleva 14 bytes extra (un puntero y el número de transacción), que se agregan la primera vez que la fila se modifica.

### RCSI: `READ COMMITTED` con versiones

RCSI es una opción **de la base**:

```sql
ALTER DATABASE Tienda SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;

SELECT name, is_read_committed_snapshot_on FROM sys.databases WHERE name = 'Tienda';
```

`WITH ROLLBACK IMMEDIATE` desconecta y deshace las transacciones abiertas, porque el cambio necesita que nadie más esté usando la base en ese momento. Se hace en una ventana de mantenimiento.

A partir de ahí, toda sesión que use el nivel por defecto (`READ COMMITTED`) lee versiones:

* Cada **instrucción** ve los datos confirmados **al momento de empezar esa instrucción**.
* No hace falta cambiar ni una línea de código: las consultas, los procedimientos y los ORM siguen usando `READ COMMITTED`, pero ya no esperan a las escrituras.
* Las anomalías son las mismas que con `READ COMMITTED` por bloqueos: no hay lecturas sucias, pero sí puede haber lecturas no repetibles y fantasmas **entre dos instrucciones** de la misma transacción.

Es la configuración por defecto de **Azure SQL Database**, y la recomendación habitual para aplicaciones OLTP en SQL Server.

### `SNAPSHOT`: una foto para toda la transacción

`SNAPSHOT` es un **nivel de aislamiento** que hay que habilitar en la base y después pedir explícitamente:

```sql
ALTER DATABASE Tienda SET ALLOW_SNAPSHOT_ISOLATION ON;   -- no requiere desconectar a nadie

-- En la sesión que lo necesita
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    SELECT SUM(Stock * Precio) FROM dbo.Producto;     -- ve la foto de este momento
    -- ... otras consultas: todas ven la misma foto ...
COMMIT;
```

* **Toda la transacción** ve los datos confirmados al momento de su **primera lectura**, aunque otras transacciones confirmen cambios mientras tanto.
* Evita las lecturas sucias, las no repetibles y las fantasma: es la herramienta para reportes que tienen que cuadrar entre sí.

| | RCSI | `SNAPSHOT` |
| --- | --- | --- |
| Cómo se activa | Opción de la base; afecta a todas las sesiones en `READ COMMITTED` | Se habilita en la base y cada sesión lo pide |
| Cambia el código | No | Sí (`SET TRANSACTION ISOLATION LEVEL SNAPSHOT`) |
| Foto de los datos | Por instrucción | Por transacción |
| Lecturas no repetibles y fantasma | Posibles entre instrucciones | Evitadas |
| Conflictos al escribir | No (se comporta como `READ COMMITTED` al escribir) | **Sí**: error 3960 |

### Conflictos de actualización en `SNAPSHOT`

Si una transacción `SNAPSHOT` intenta modificar una fila que **otra transacción modificó y confirmó después** de que empezó la foto, SQL Server no puede aplicar el cambio sin perder el de la otra, y aborta:

```text
Msg 3960, Level 16, State 2
Snapshot isolation transaction aborted due to update conflict. You cannot use snapshot isolation
to access table 'dbo.Producto' directly or indirectly in database 'Tienda' to update, delete,
or insert the row that has been modified or deleted by another transaction. Retry the transaction
or change the isolation level for the update/delete statement.
```

Es una forma de **concurrencia optimista automática**: la actualización perdida no ocurre, pero la transacción tiene que reintentarse. Por eso `SNAPSHOT` encaja mejor en transacciones de **lectura** (reportes) que en transacciones que leen y escriben con mucha concurrencia.

### El costo del versionado

| Costo | Por qué | Cómo se controla |
| --- | --- | --- |
| **Espacio y actividad en `tempdb`** | Las versiones anteriores viven ahí | Dimensionar `tempdb`; desde 2019, ADR guarda las versiones en la propia base |
| **14 bytes por fila modificada** | El puntero a la versión anterior | Puede causar divisiones de página la primera vez que se modifican muchas filas |
| **Lecturas algo más caras** | Recorrer la cadena de versiones | Normalmente despreciable; crece con transacciones muy largas |
| **Transacciones largas retienen versiones** | Una transacción `SNAPSHOT` abierta durante horas obliga a guardar todas las versiones desde que empezó | Mantener las transacciones cortas; monitorear `sys.dm_tran_version_store_space_usage` |

En la gran mayoría de los sistemas OLTP, el beneficio (lecturas que no esperan) es mucho mayor que el costo.

### El riesgo de activar RCSI en una aplicación existente

RCSI no cambia el código, pero **sí** cambia su comportamiento en un caso: el código que dependía, sin saberlo, de que las lecturas **esperaran** a las escrituras.

```sql
-- "Si no existe, lo creo", con READ COMMITTED por bloqueos:
-- la lectura espera a una inserción en curso de la misma clave, y la ve al terminar.
IF NOT EXISTS (SELECT 1 FROM dbo.Cliente WHERE Email = @Email)
    INSERT INTO dbo.Cliente (...) VALUES (...);
```

Con RCSI, la lectura no espera: no ve la inserción en curso de otra sesión, y las dos intentan insertar. (Aquí la restricción `UNIQUE` sobre `Email` salva la situación con un error; sin ella, habría un duplicado.) Este patrón ya tenía una carrera con el nivel por defecto; RCSI la hace más probable. La forma correcta, con cualquier nivel, es la del *upsert* con `UPDLOCK, HOLDLOCK` (ver [Modificar datos](../03-consultas/11-Modificar%20datos.md)), o simplemente insertar y manejar el error de la restricción única.

Antes de activar RCSI en una aplicación existente, conviene buscar estos patrones de "leer y después escribir según lo leído" y verificar que usen `UPDLOCK` o una instrucción atómica.

-----

## Ejemplo completo

El problema de la lección anterior, con RCSI activado en `Tienda`:

```sql
ALTER DATABASE Tienda SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

| Momento | Ventana A | Ventana B (pantalla de vendedores) |
| --- | --- | --- |
| t1 | `BEGIN TRAN; UPDATE dbo.Pedido SET Estado = 'Enviado' WHERE PedidoId = 3;` | |
| t2 | | `SELECT PedidoId, Estado FROM dbo.Pedido WHERE ClienteId = 1;` |
| t2 | | **Responde enseguida:** pedido 1 `Entregado`, pedido 3 **`Pagado`** (la versión confirmada) |
| t3 | `ROLLBACK;` | |
| t4 | | Lo que se mostró en t2 sigue siendo cierto: el pedido 3 está `Pagado` |

Y un reporte que necesita consistencia entre dos consultas, con `SNAPSHOT`:

```sql
ALTER DATABASE Tienda SET ALLOW_SNAPSHOT_ISOLATION ON;
```

| Momento | Ventana A (reporte) | Ventana B (ventas) |
| --- | --- | --- |
| t1 | `SET TRANSACTION ISOLATION LEVEL SNAPSHOT; BEGIN TRAN;` | |
| t2 | `SELECT SUM(Stock) FROM dbo.Producto;` → **95** | |
| t3 | | `UPDATE dbo.Producto SET Stock = Stock - 5 WHERE ProductoId = 2; COMMIT;` |
| t4 | `SELECT Stock FROM dbo.Producto WHERE ProductoId = 2;` → **40** (la misma foto de t2) | |
| t5 | `COMMIT;` | |

El total (95) y el detalle (40 del mouse) del reporte cuadran entre sí, aunque en medio se confirmó una venta. Con RCSI, la consulta de t4 habría visto 35 (la foto de **esa** instrucción), y el reporte no cuadraría con el total de t2.

(El stock total de los datos de ejemplo es 15 + 40 + 8 + 20 + 0 + 12 = 95.)

Qué decisión hay detrás:

* **RCSI para la aplicación en general:** las pantallas dejan de esperar a las escrituras, sin lecturas sucias y sin cambiar el código.
* **`SNAPSHOT` solo donde hace falta una foto:** el reporte lo pide explícitamente; el resto de la aplicación sigue con RCSI.
* **Las escrituras siguen protegidas:** si la ventana B hubiera intentado un `UPDATE` sobre el pedido 3 en el primer ejemplo, habría esperado a la ventana A, como siempre.

-----

## Errores comunes

**1. Seguir usando `NOLOCK` con RCSI activado.**
Qué pasa: lecturas sucias innecesarias, porque `NOLOCK` sigue leyendo sin confirmar aunque la base tenga RCSI.
Por qué: el hint tiene prioridad sobre el nivel de la sesión.
Arreglo: con RCSI activado, quitar los `NOLOCK`: ya no aportan nada.

**2. Activar RCSI sin revisar los patrones de "leer y después escribir".**
Qué pasa: duplicados o decisiones tomadas con un dato que otra transacción estaba cambiando.
Por qué: el código dependía de que la lectura esperara.
Arreglo: `UPDLOCK` en esas lecturas, o instrucciones atómicas.

**3. `SNAPSHOT` para transacciones con muchas escrituras concurrentes.**
Qué pasa: errores 3960 frecuentes y reintentos constantes.
Por qué: cada conflicto aborta la transacción entera.
Arreglo: `SNAPSHOT` para lecturas consistentes; para escrituras concurrentes, `READ COMMITTED` (con o sin RCSI) y `UPDLOCK`.

**4. Olvidar `tempdb`.**
Qué pasa: `tempdb` crece sin control por una transacción larga que retiene versiones.
Por qué: mientras la transacción más vieja siga abierta, el motor no puede descartar las versiones posteriores.
Arreglo: transacciones cortas, monitoreo del almacén de versiones, y ADR en 2019+.

**5. Activar RCSI en horario productivo.**
Qué pasa: el `ALTER DATABASE` espera indefinidamente, o desconecta a todos con `ROLLBACK IMMEDIATE`.
Por qué: el cambio requiere que no haya otras conexiones activas en la base.
Arreglo: una ventana de mantenimiento.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Lecturas con versiones por defecto | No (hay que activar RCSI); sí en Azure SQL | **Sí, siempre** (MVCC nativo) | **Sí, siempre** (MVCC nativo) |
| Dónde se guardan las versiones | `tempdb` (o la propia base con ADR) | En la propia tabla (filas viejas, que limpia `VACUUM`) | En el *undo log* |
| Foto por instrucción | RCSI | `READ COMMITTED` | `READ COMMITTED` |
| Foto por transacción | `SNAPSHOT` | `REPEATABLE READ` | `REPEATABLE READ` (el nivel por defecto) |
| Conflicto al escribir en la foto | Error 3960 | Error de serialización (`could not serialize access due to concurrent update`) | Se aplica sobre la última versión (sin error, con *gap locks*) |
| Costo de mantenimiento | `tempdb` | `VACUUM` y el *bloat* de las tablas | Purga del *undo log* |

Lo que en SQL Server es una opción, en PostgreSQL y MySQL es el comportamiento estándar. Es una de las diferencias de arquitectura más importantes entre los motores, y explica por qué la cultura de `NOLOCK` es propia de SQL Server.

### En MongoDB

WiredTiger es multiversión de forma nativa: las lecturas ven una versión consistente de cada documento sin bloquear a las escrituras. Las transacciones multidocumento leen una foto (*snapshot*) de los datos al momento de empezar, y un conflicto de escritura con otra transacción produce un error `WriteConflict` que se resuelve reintentando, igual que el error 3960 de SQL Server.

-----

## Cuándo sí y cuándo no

**Activa RCSI cuando:**

* La aplicación sufre esperas de lectores detrás de escritores, o está llena de `NOLOCK`: es la recomendación por defecto para OLTP.

**Usa `SNAPSHOT` cuando:**

* Un reporte o un proceso necesita que varias consultas vean exactamente los mismos datos.

**Ten en cuenta:**

* Ninguno de los dos evita que dos escrituras sobre la misma fila se esperen: el versionado resuelve el conflicto entre lectores y escritores, no entre escritores.

-----

## Resumen en 5 líneas

1. El versionado guarda la versión anterior de cada fila modificada; las lecturas usan la última confirmada en lugar de esperar.
2. RCSI es una opción de la base: `READ COMMITTED` lee versiones, foto por instrucción, sin cambiar el código.
3. `SNAPSHOT` es un nivel que se pide: foto por transacción, ideal para reportes, con error 3960 si se escribe sobre una fila que cambió.
4. El costo es `tempdb`, 14 bytes por fila y versiones retenidas por transacciones largas.
5. Al activar RCSI, revisa el código que lee y después escribe según lo leído: debe usar `UPDLOCK` o instrucciones atómicas.

-----

## Para profundizar

<details>
<summary>Por qué RCSI es la recomendación por defecto</summary>

Con el `READ COMMITTED` por bloqueos, cada lectura compite con las escrituras, y la reacción habitual de los equipos es llenar el código de `NOLOCK`, cambiando esperas por datos incorrectos. RCSI elimina la competencia entre lectores y escritores con un costo predecible, sin cambiar el código y sin lecturas sucias. Por eso es la configuración de Azure SQL Database, por eso muchos expertos (Brent Ozar, Kendra Little, entre otros) lo recomiendan como punto de partida, y por eso los demás motores relacionales importantes funcionan así por defecto.

</details>

<details>
<summary>Ver el almacén de versiones</summary>

```sql
-- Espacio usado por las versiones, por base
SELECT DB_NAME(database_id) AS Base, reserved_space_kb / 1024 AS ReservadoMB
FROM sys.dm_tran_version_store_space_usage;

-- Transacciones que usan versiones, de la más vieja a la más nueva
SELECT transaction_id, transaction_sequence_num, elapsed_time_seconds
FROM sys.dm_tran_active_snapshot_database_transactions
ORDER BY elapsed_time_seconds DESC;
```

Una transacción con `elapsed_time_seconds` muy alto es la que está reteniendo versiones y haciendo crecer `tempdb`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Con el versionado de filas, las lecturas no esperan a las escrituras: leen la última versión confirmada de la fila. RCSI lo activa para toda la base sin cambiar el código, y es una mejor alternativa que `NOLOCK` porque no lee datos sin confirmar. `SNAPSHOT` hace que toda una transacción vea los mismos datos.

### Respuesta ampliada (semi-senior)

El versionado copia la versión anterior de cada fila modificada al almacén de versiones, en `tempdb` (o en la propia base con ADR), para que las lecturas usen la última versión confirmada sin pedir bloqueos compartidos: lectores y escritores dejan de bloquearse, aunque los escritores siguen esperándose entre sí. RCSI es una opción de la base que cambia el comportamiento de `READ COMMITTED` con una foto por instrucción, sin tocar el código, y es lo que recomiendo por defecto en OLTP en lugar de `NOLOCK`; antes de activarlo reviso los patrones de "leer y después escribir" que dependían de la espera, y les pongo `UPDLOCK` o los vuelvo atómicos. `SNAPSHOT` es un nivel que se pide por sesión, con una foto por transacción, ideal para reportes consistentes, pero con el error 3960 si la transacción intenta modificar una fila que cambió, así que no lo uso para escrituras muy concurrentes. Vigilo el costo: `tempdb`, los 14 bytes por fila y las transacciones largas que retienen versiones.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre RCSI y `SNAPSHOT`?**
RCSI es una opción de la base que da una foto por instrucción sin cambiar el código; `SNAPSHOT` es un nivel que se pide explícitamente y da una foto para toda la transacción.

**2. ¿Por qué RCSI es mejor que `NOLOCK`?**
Porque las lecturas tampoco esperan, pero leen la última versión confirmada, sin lecturas sucias ni filas duplicadas o perdidas.

**3. ¿Qué costo tiene el versionado?**
Espacio y actividad en `tempdb`, 14 bytes por fila modificada y versiones retenidas por las transacciones largas.

**4. ¿Qué es el error 3960?**
Un conflicto de actualización en `SNAPSHOT`: la transacción intentó modificar una fila que otra modificó después de que empezó su foto; hay que reintentarla.

-----

## Práctica

**Ejercicio 1.** Con RCSI activado en `Tienda`, ¿qué devuelve la ventana B en t3 y en t5?

| Momento | Ventana A | Ventana B |
| --- | --- | --- |
| t1 | `BEGIN TRAN;` | `BEGIN TRAN;` |
| t2 | `UPDATE dbo.Producto SET Precio = 130 WHERE ProductoId = 1;` | |
| t3 | | `SELECT Precio FROM dbo.Producto WHERE ProductoId = 1;` |
| t4 | `COMMIT;` | |
| t5 | | `SELECT Precio FROM dbo.Producto WHERE ProductoId = 1;` |

<details>
<summary>Solución</summary>

* **t3: 120.00.** La ventana A no confirmó; RCSI devuelve la última versión confirmada, sin esperar.
* **t5: 130.00.** Con RCSI, cada instrucción ve la foto del momento en que empieza; en t5, el cambio ya está confirmado. Es una lectura no repetible dentro de la transacción de B.

Con `SNAPSHOT` en la ventana B (en lugar de RCSI), t5 también devolvería **120.00**: la foto es la de la primera lectura de la transacción (t3).

</details>

**Ejercicio 2.** Con `SNAPSHOT` en las dos ventanas, ¿qué pasa en t4?

| Momento | Ventana A | Ventana B |
| --- | --- | --- |
| t1 | `SET TRANSACTION ISOLATION LEVEL SNAPSHOT; BEGIN TRAN;` | `SET TRANSACTION ISOLATION LEVEL SNAPSHOT; BEGIN TRAN;` |
| t2 | `SELECT Stock FROM dbo.Producto WHERE ProductoId = 4;` | `SELECT Stock FROM dbo.Producto WHERE ProductoId = 4;` |
| t3 | `UPDATE dbo.Producto SET Stock = Stock - 1 WHERE ProductoId = 4; COMMIT;` | |
| t4 | | `UPDATE dbo.Producto SET Stock = Stock - 1 WHERE ProductoId = 4;` |

<details>
<summary>Solución</summary>

La ventana B recibe el **error 3960** (conflicto de actualización): intenta modificar una fila que la ventana A modificó y confirmó después de que empezó la foto de B. La transacción de B queda abortada y hay que reintentarla desde el principio.

Es la protección contra la actualización perdida de `SNAPSHOT`: en lugar de que B pise el cambio de A (dejando el stock en 19 en lugar de 18), la operación falla. Si en lugar de `SNAPSHOT` las dos usaran `READ COMMITTED` (con o sin RCSI), el `UPDATE` de B aplicaría `Stock - 1` sobre el valor ya confirmado por A (19 → 18), sin error: la instrucción atómica también evita la pérdida. (Si A todavía no hubiera confirmado, el `UPDATE` de B esperaría a A y después haría lo mismo.)

</details>

-----

## Siguiente lección

[Deadlocks](05-Deadlocks.md)
