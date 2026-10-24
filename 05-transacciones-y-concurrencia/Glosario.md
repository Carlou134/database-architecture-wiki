# Glosario: Transacciones y concurrencia

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**ACID:** atomicidad, consistencia, aislamiento y durabilidad: las garantías de una transacción. ([01](01-Transacciones%20y%20ACID.md))

**Actualización perdida (lost update):** dos transacciones leen el mismo valor, lo modifican y la segunda escritura pisa a la primera. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Almacén de versiones (version store):** donde se guardan las versiones anteriores de las filas; en `tempdb`, o en la propia base con ADR. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**Bloqueo (lock):** marca que una transacción pone sobre un recurso para controlar el acceso de las demás. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**Cabeza de la cadena (head blocker):** la sesión que bloquea a otras y no espera a nadie. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**`COMMIT` / `ROLLBACK`:** confirmar / deshacer una transacción. ([01](01-Transacciones%20y%20ACID.md))

**Concurrencia optimista:** no bloquear al leer y verificar al escribir, con una columna de versión, que nadie modificó la fila. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Conflicto de actualización:** error 3960 de `SNAPSHOT` al modificar una fila que otra transacción modificó después de empezar la foto. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**Deadlock (interbloqueo):** ciclo de esperas en el que cada transacción espera un recurso que tiene otra del ciclo. ([05](05-Deadlocks.md))

**`DEADLOCK_PRIORITY`:** configuración de la sesión que influye en cuál se elige como víctima de un *deadlock*. ([05](05-Deadlocks.md))

**Error transitorio:** error que probablemente no se repita al reintentar (1205, 1222, 3960). ([06](06-Manejo%20de%20errores.md))

**Escalamiento de bloqueos (lock escalation):** convertir muchos bloqueos de fila o página (~5.000 en una instrucción) en uno de tabla. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**Gráfico de deadlock:** XML que describe los procesos, recursos y consultas de un *deadlock*; lo registra `system_health`. ([05](05-Deadlocks.md))

**Lectura fantasma (phantom read):** repetir una consulta y obtener filas que antes no estaban. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Lectura no repetible (non-repeatable read):** leer dos veces la misma fila y obtener valores distintos. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Lectura sucia (dirty read):** leer un cambio que otra transacción todavía no confirmó. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Modos de bloqueo:** S (compartido, para leer), X (exclusivo, para escribir), U (actualización, leer para modificar), IS/IX (intención). ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**MVCC (control de concurrencia multiversión):** modelo en el que lectores y escritores trabajan sobre versiones distintas sin bloquearse. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**Nivel de aislamiento:** configuración de la sesión que decide qué anomalías de concurrencia se permiten. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**`NOLOCK`:** hint que lee sin respetar los bloqueos exclusivos; permite lecturas sucias, filas duplicadas y faltantes. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**Procesamiento por lotes (batching):** dividir una operación masiva en muchas operaciones pequeñas con transacciones cortas. ([07](07-Operaciones%20masivas%20en%20lotes.md))

**Punto de guardado (savepoint):** marca dentro de una transacción a la que se puede volver con `ROLLBACK TRANSACTION nombre`. ([01](01-Transacciones%20y%20ACID.md))

**RCSI (READ COMMITTED SNAPSHOT ISOLATION):** opción de la base que hace que `READ COMMITTED` lea versiones en lugar de pedir bloqueos. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**`READPAST`:** hint que se salta las filas bloqueadas; base del patrón de colas. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**Registro mínimo (minimal logging):** modo de carga masiva que registra solo la asignación de páginas. ([07](07-Operaciones%20masivas%20en%20lotes.md))

**`ROWVERSION`:** columna que cambia sola en cada modificación de la fila; base de la concurrencia optimista. ([02](02-Niveles%20de%20aislamiento%20y%20anomalías.md))

**Severidad:** número de 0 a 25 que indica la gravedad de un error; de 20 en adelante cierra la conexión. ([06](06-Manejo%20de%20errores.md))

**`SNAPSHOT`:** nivel de aislamiento en el que toda la transacción lee la foto de los datos al momento de su primera lectura. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**Tabla de staging:** tabla intermedia para cargar y validar datos externos antes de pasarlos a las definitivas. ([07](07-Operaciones%20masivas%20en%20lotes.md))

**`THROW`:** instrucción que lanza un error propio o relanza el capturado; respeta `XACT_ABORT`. ([06](06-Manejo%20de%20errores.md))

**Transacción:** conjunto de operaciones que se confirman o se deshacen juntas. ([01](01-Transacciones%20y%20ACID.md))

**Transacción condenada (doomed):** transacción abierta que solo admite `ROLLBACK` (`XACT_STATE() = -1`). ([06](06-Manejo%20de%20errores.md))

**`@@TRANCOUNT`:** cuántos `BEGIN TRANSACTION` hay abiertos en la sesión. ([01](01-Transacciones%20y%20ACID.md))

**`TRY...CATCH`:** bloque que captura los errores del `TRY` y pasa el control al `CATCH`. ([06](06-Manejo%20de%20errores.md))

**`UPDLOCK`:** hint que lee con bloqueo de actualización hasta el final de la transacción; evita la actualización perdida. ([03](03-Bloqueos%2C%20hints%20y%20NOLOCK.md))

**Versionado de filas (row versioning):** guardar la versión anterior de cada fila modificada para que las lecturas no esperen. ([04](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md))

**Víctima:** la transacción que SQL Server deshace para romper un *deadlock* (error 1205). ([05](05-Deadlocks.md))

**`XACT_ABORT`:** configuración que hace que cualquier error deshaga la transacción entera. ([06](06-Manejo%20de%20errores.md))

**`XACT_STATE()`:** indica si hay una transacción y si puede confirmarse: 1 (activa), 0 (ninguna), -1 (condenada). ([06](06-Manejo%20de%20errores.md))
