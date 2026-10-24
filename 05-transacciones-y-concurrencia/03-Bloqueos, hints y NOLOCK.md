# Bloqueos, hints y NOLOCK

## En una frase

SQL Server protege los datos que una transacción está usando con **bloqueos** de distintos modos (compartido para leer, exclusivo para escribir, de actualización para "leer con intención de escribir") y granularidades (fila, página, tabla); cuando dos bloqueos no son compatibles, una sesión **espera** a la otra, y entender esas esperas (y no esconderlas con `NOLOCK`) es la clave de la concurrencia en SQL Server.

-----

## Antes de empezar

Conviene que ya sepas:

* Niveles de aislamiento y la actualización perdida, de [Niveles de aislamiento y anomalías](02-Niveles%20de%20aislamiento%20y%20anomalías.md).
* Que los índices determinan cuántas filas lee una consulta, del módulo de [Índices y rendimiento](../04-indices-y-rendimiento/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Bloqueo (lock):** marca que una transacción pone sobre un recurso para controlar el acceso de las demás.
* **Modo de bloqueo:** el tipo de acceso que protege: compartido (S), exclusivo (X), de actualización (U), de intención (IS, IX).
* **Granularidad:** el tamaño del recurso bloqueado: fila (clave o RID), página, tabla.
* **Bloqueo (blocking):** una sesión esperando a que otra libere un recurso incompatible.
* **Escalamiento de bloqueos (lock escalation):** convertir muchos bloqueos de fila o página en uno de tabla, para ahorrar memoria.
* **Hint de bloqueo:** instrucción en la consulta que cambia cómo se bloquea una tabla (`UPDLOCK`, `NOLOCK`...).

-----

## El problema

Un lunes a las 9:00, la pantalla de pedidos "se cuelga" para todos los vendedores. El servidor no está al 100 % de CPU, no hay falta de memoria, no hay errores. Las consultas simplemente **no terminan**.

Lo que pasó: a las 8:55, alguien ejecutó en SSMS un `UPDATE dbo.Pedido SET Estado = 'Enviado' WHERE ...` dentro de un `BEGIN TRANSACTION` para revisar el resultado antes de confirmar, y se fue a buscar un café. Esa transacción tiene bloqueos exclusivos sobre miles de filas de pedidos, y cada consulta que necesita leerlas **espera**. En cola, detrás de ellas, esperan otras.

La "solución" que alguien propone en el chat del equipo: agregar `WITH (NOLOCK)` a todas las consultas. Las pantallas vuelven a funcionar... y empiezan a mostrar pedidos en estados que nunca se confirmaron.

-----

## Cómo funciona

### Modos de bloqueo y compatibilidad

| Modo | Lo pide | Significa |
| --- | --- | --- |
| **S** (compartido) | Una lectura | "Estoy leyendo esto; nadie lo modifique" |
| **X** (exclusivo) | Una escritura (`INSERT`, `UPDATE`, `DELETE`) | "Estoy modificando esto; nadie lo lea ni lo modifique" |
| **U** (actualización) | Un `UPDATE` mientras busca las filas, o `WITH (UPDLOCK)` | "Voy a modificar esto; los demás pueden leer, pero nadie más puede tener esta intención" |
| **IS, IX** (de intención) | Automáticamente, en la página y la tabla, antes de bloquear una fila | "Hay bloqueos S o X más abajo en esta tabla" |
| **Sch-M** (modificación de esquema) | `ALTER TABLE`, `CREATE INDEX` sin `ONLINE` | "Estoy cambiando la estructura; nadie toque la tabla" |

Dos bloqueos sobre el mismo recurso conviven o no según esta matriz:

| Pedido \ Existente | S | U | X |
| --- | --- | --- | --- |
| **S** | ✅ | ✅ | ❌ espera |
| **U** | ✅ | ❌ espera | ❌ espera |
| **X** | ❌ espera | ❌ espera | ❌ espera |

* Muchas lecturas conviven entre sí.
* Una escritura excluye a todos, y todos la excluyen a ella.
* El bloqueo **U** existe para evitar un *deadlock* clásico: si dos sesiones leyeran con S y después ambas quisieran convertir a X, cada una esperaría a la otra para siempre. Con U, la segunda espera desde el principio.

Los bloqueos de intención son los que permiten que, para bloquear una tabla entera, el motor no tenga que revisar millones de filas: le basta mirar si hay un IX en la tabla.

### Cuánto duran

| Bloqueo | En `READ COMMITTED` (por defecto) | En `REPEATABLE READ` / `SERIALIZABLE` |
| --- | --- | --- |
| S (lectura) | Mientras se lee la fila: se libera enseguida | Hasta el final de la transacción |
| X (escritura) | **Hasta el final de la transacción** | Hasta el final de la transacción |

Los bloqueos exclusivos duran siempre hasta el `COMMIT` o el `ROLLBACK`, en cualquier nivel de aislamiento. Por eso una transacción abierta con escrituras (el `UPDATE` sin confirmar del problema) bloquea a todo el que necesite esas filas, durante todo el tiempo que siga abierta.

### Granularidad y escalamiento

El motor bloquea lo más chico que puede: normalmente **filas** (en un índice, la clave; en un heap, el RID). Cada bloqueo ocupa memoria, así que cuando **una instrucción** acumula unos 5.000 bloqueos sobre la misma tabla, SQL Server intenta **escalarlos** a un bloqueo de tabla:

```text
5.000 bloqueos X de fila en Pedido  →  1 bloqueo X de tabla en Pedido
```

El escalamiento ahorra memoria, pero bloquea la tabla entera para todos los demás. Es una de las razones principales para procesar los cambios masivos en lotes de menos de 5.000 filas (ver [Operaciones masivas en lotes](07-Operaciones%20masivas%20en%20lotes.md)).

### Por qué los índices reducen los bloqueos

Para encontrar las filas que va a modificar, un `UPDATE ... WHERE ClienteId = 700` **lee** con bloqueos U. Si no hay un índice sobre `ClienteId`, recorre toda la tabla, pidiendo un bloqueo U fila por fila, y **espera** en cualquier fila que otra transacción tenga bloqueada, aunque no sea de ese cliente. Con un índice, va directo a las ~10 filas del cliente y no se cruza con nadie más. Muchos problemas de bloqueos son, en el fondo, problemas de índices.

### Ver los bloqueos y las esperas

```sql
-- ¿Quién está esperando a quién?
SELECT r.session_id            AS SesionQueEspera,
       r.blocking_session_id   AS SesionQueBloquea,
       r.wait_type,
       r.wait_time / 1000      AS SegundosEsperando,
       t.text                  AS Consulta
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0;

-- ¿Qué bloqueos tiene una sesión?
SELECT resource_type, request_mode, request_status, COUNT(*) AS Cantidad
FROM sys.dm_tran_locks
WHERE request_session_id = 57      -- la sesión que bloquea
GROUP BY resource_type, request_mode, request_status;

-- ¿Qué transacciones llevan más tiempo abiertas?
SELECT s.session_id, s.login_name, s.host_name, t.transaction_begin_time
FROM sys.dm_tran_active_transactions AS t
JOIN sys.dm_tran_session_transactions AS st ON st.transaction_id = t.transaction_id
JOIN sys.dm_exec_sessions AS s ON s.session_id = st.session_id
ORDER BY t.transaction_begin_time;
```

La sesión que encabeza una cadena de bloqueos (la que bloquea y no espera a nadie) se llama **cabeza de la cadena** (*head blocker*). Es la que hay que encontrar: las demás solo esperan. Si es una transacción olvidada, la opción de último recurso es terminarla con `KILL 57;`, sabiendo que eso **deshace** todo lo que hizo.

### Limitar la espera

Por defecto, una sesión espera un bloqueo indefinidamente. Se puede poner un límite:

```sql
SET LOCK_TIMEOUT 5000;   -- 5 segundos; -1 = esperar siempre (por defecto), 0 = no esperar
```

Pasado el límite, la instrucción falla con `Msg 1222: Lock request time out period exceeded`, que es uno de los errores transitorios que vale la pena reintentar (ver [Manejo de errores](06-Manejo%20de%20errores.md)). En las aplicaciones, lo más habitual es el *timeout* del lado del cliente (por defecto, 30 segundos en ADO.NET).

### Hints de bloqueo

Los hints cambian cómo se bloquea una tabla en una consulta concreta:

| Hint | Efecto | Uso razonable |
| --- | --- | --- |
| `UPDLOCK` | Lee con bloqueo U y lo mantiene hasta el final de la transacción | Leer para después modificar, evitando la actualización perdida |
| `HOLDLOCK` (= `SERIALIZABLE`) | Mantiene los bloqueos de lectura hasta el final, con bloqueo de rango | Junto con `UPDLOCK` en un *upsert*: impedir que otro inserte la misma clave |
| `READPAST` | Se salta las filas bloqueadas, en lugar de esperarlas | Colas de trabajo: cada proceso toma el siguiente elemento libre |
| `ROWLOCK` / `PAGLOCK` / `TABLOCK` | Pide una granularidad concreta | `TABLOCK` en cargas masivas (permite el registro mínimo); los demás, rara vez |
| `XLOCK` | Bloqueo exclusivo al leer | Muy raro |
| `NOLOCK` (= `READUNCOMMITTED`) | Lee sin pedir bloqueos S, ignorando los X de los demás | Ver abajo |

Sobre `ROWLOCK`: tu cheat sheet lo recomienda para evitar *deadlocks*, pero SQL Server **ya** bloquea por fila por defecto. `ROWLOCK` impide que el motor elija bloqueos de página, lo que puede aumentar la cantidad de bloqueos y acelerar el escalamiento a tabla. No es una herramienta general de prevención.

El patrón de cola con `READPAST` es uno de los usos más útiles de los hints:

```sql
-- Cada proceso trabajador toma el siguiente pedido pendiente que nadie más está procesando
WITH Siguiente AS (
    SELECT TOP (1) PedidoId, Estado
    FROM dbo.Pedido WITH (UPDLOCK, READPAST, ROWLOCK)
    WHERE Estado = 'Pagado'
    ORDER BY FechaPedido
)
UPDATE Siguiente
SET Estado = 'Enviado'
OUTPUT inserted.PedidoId;
```

Dos procesos que ejecutan esto a la vez toman pedidos **distintos**: cada uno se salta la fila que el otro tiene bloqueada.

### `NOLOCK`: lo que realmente hace

`WITH (NOLOCK)` (o el nivel `READ UNCOMMITTED`) lee sin respetar los bloqueos exclusivos de los demás. La consulta no espera a nadie, y por eso parece "solucionar" los bloqueos. El costo:

| Riesgo | Qué pasa |
| --- | --- |
| **Lecturas sucias** | Se leen cambios que todavía no se confirmaron y que pueden deshacerse: un pedido "Enviado" que nunca se envió |
| **Filas duplicadas** | Si una división de página mueve filas mientras la consulta recorre el índice, algunas se leen dos veces |
| **Filas faltantes** | Por el mismo motivo, otras no se leen nunca |
| **Datos inconsistentes** | Una fila a medio actualizar, o una consulta que falla con el error 601 (*Could not continue scan with NOLOCK due to data movement*) |

`NOLOCK` **no** hace que la consulta no bloquee nada: sigue pidiendo un bloqueo de esquema (Sch-S), así que un `ALTER TABLE` la hace esperar. Y no tiene efecto sobre las escrituras: un `UPDATE` sigue bloqueando.

Tus apuntes lo resumen bien: aceptable para tableros de monitoreo, conteos aproximados y estadísticas no críticas; **nunca** para saldos, inventario, facturación, auditoría o cualquier dato a partir del cual se tome una decisión. Y una advertencia de tus notas anteriores que vale repetir: tampoco se usa sobre las tablas que se modifican, porque en un `UPDATE` o `DELETE` el hint se ignora o da error.

Lo que normalmente se busca con `NOLOCK` (que las lecturas no esperen a las escrituras) tiene una solución **correcta**: el versionado de filas con RCSI, en la [próxima lección](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md).

-----

## Ejemplo completo

El problema del lunes, reproducido con dos ventanas de SSMS sobre la base `Tienda`:

**Ventana A** (la transacción olvidada):

```sql
BEGIN TRANSACTION;
UPDATE dbo.Pedido SET Estado = 'Enviado' WHERE PedidoId = 3;
-- (sin COMMIT: la ventana queda abierta)
```

**Ventana B** (la pantalla de los vendedores):

```sql
SELECT PedidoId, Estado FROM dbo.Pedido WHERE ClienteId = 1;
-- No termina: espera el bloqueo X de la fila del pedido 3
```

**Ventana C** (diagnóstico):

```sql
SELECT r.session_id AS Espera, r.blocking_session_id AS Bloquea,
       r.wait_type, r.wait_time / 1000 AS Segundos, t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0;
```

| Espera | Bloquea | wait_type | Segundos | text |
| --- | --- | --- | --- | --- |
| 61 | 58 | LCK_M_S | 42 | SELECT PedidoId, Estado FROM dbo.Pedido WHERE ClienteId = 1 |

* **`LCK_M_S`:** la sesión 61 espera un bloqueo compartido (S): quiere leer.
* **La sesión 58** es la ventana A: la cabeza de la cadena.

Con `NOLOCK` en la ventana B, la consulta termina enseguida y muestra el pedido 3 como **"Enviado"**, un cambio que la ventana A todavía puede deshacer. Si después la ventana A ejecuta `ROLLBACK`, la pantalla mostró un dato que nunca existió.

La solución correcta tiene tres partes:

* **Inmediata:** encontrar la sesión 58 y que su dueño confirme o deshaga (o, como último recurso, `KILL 58`).
* **De proceso:** no dejar transacciones abiertas en SSMS; revisar con `SELECT` antes, y ejecutar la modificación con `COMMIT` o `ROLLBACK` en el mismo lote.
* **De configuración:** activar RCSI en la base, para que las lecturas usen la última versión confirmada en lugar de esperar (próxima lección). Con RCSI, la ventana B termina enseguida y muestra el estado **confirmado** ("Pagado"), sin lecturas sucias.

-----

## Errores comunes

**1. `NOLOCK` en todas las consultas.**
Qué pasa: lecturas sucias, filas duplicadas o faltantes en reportes y pantallas.
Por qué: se usa para esconder los bloqueos en lugar de resolver su causa.
Arreglo: transacciones cortas, índices adecuados y RCSI.

**2. Transacciones abiertas en una herramienta.**
Qué pasa: bloqueos de minutos u horas sobre tablas muy usadas.
Por qué: `BEGIN TRANSACTION` para revisar y nunca un `COMMIT` o `ROLLBACK`.
Arreglo: revisar con `SELECT` antes; ejecutar la modificación y su cierre juntos.

**3. Modificaciones sin índice en el filtro.**
Qué pasa: un `UPDATE` de diez filas que recorre la tabla y espera en filas ajenas.
Por qué: para encontrar las filas, el motor pide bloqueos U sobre todas las que revisa.
Arreglo: un índice sobre las columnas del `WHERE` de las modificaciones frecuentes.

**4. Modificaciones masivas en una sola instrucción.**
Qué pasa: escalamiento a bloqueo de tabla, y toda la tabla inaccesible durante el proceso.
Por qué: más de ~5.000 bloqueos en una instrucción.
Arreglo: lotes de menos de 5.000 filas.

**5. `ROWLOCK` como remedio general.**
Qué pasa: más bloqueos que gestionar, escalamiento más rápido, y los *deadlocks* siguen igual.
Por qué: las filas ya son la granularidad por defecto.
Arreglo: atacar la causa del conflicto (orden de acceso, índices, duración de la transacción).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| ¿Las lecturas normales bloquean las escrituras? | Sí (sin RCSI) | **No** (MVCC) | **No** (MVCC) |
| Bloquear al leer para modificar | `WITH (UPDLOCK)` | `SELECT ... FOR UPDATE` | `SELECT ... FOR UPDATE` |
| Saltarse filas bloqueadas | `WITH (READPAST)` | `FOR UPDATE SKIP LOCKED` | `FOR UPDATE SKIP LOCKED` (desde 8.0) |
| No esperar | `SET LOCK_TIMEOUT 0` | `FOR UPDATE NOWAIT` | `FOR UPDATE NOWAIT` |
| Escalamiento a tabla | Sí (~5.000 bloqueos) | **No** (los bloqueos de fila se guardan en la propia fila) | No |
| Lecturas sucias | `NOLOCK` / `READ UNCOMMITTED` | Imposibles | `READ UNCOMMITTED` |
| Ver bloqueos | `sys.dm_tran_locks`, `sys.dm_exec_requests` | `pg_locks`, `pg_stat_activity` | `performance_schema.data_locks`, `sys.innodb_lock_waits` |

En PostgreSQL, el problema del lunes no existiría tal cual: la pantalla habría leído la última versión confirmada sin esperar. Es la razón por la que `NOLOCK` es un hábito tan propio de SQL Server: es un parche para un comportamiento que otros motores no tienen por defecto, y que SQL Server también puede evitar con RCSI.

### En MongoDB

WiredTiger usa control de concurrencia a nivel de **documento**: las lecturas no bloquean a las escrituras ni al revés, y dos escrituras sobre el mismo documento se serializan (una espera o se reintenta automáticamente). Dentro de una transacción multidocumento, si dos transacciones modifican el mismo documento, una recibe un *write conflict* y debe reintentarse. No hay hints de bloqueo como en SQL Server; el patrón de cola equivalente se hace con `findOneAndUpdate` sobre un campo de estado.

-----

## Cuándo sí y cuándo no

**Usa `UPDLOCK` cuando:**

* Lees una fila para decidir cómo modificarla dentro de la misma transacción.

**Usa `READPAST` cuando:**

* Implementas una cola de trabajo con varios procesos que toman tareas.

**Usa `NOLOCK` solo cuando:**

* El dato es aproximado por naturaleza (un tablero de monitoreo, un conteo estimado) y nadie toma decisiones con él.

**Ten en cuenta:**

* Antes de cualquier hint de bloqueo, pregúntate si la causa no es una transacción larga, un índice que falta o una modificación masiva sin lotes.

-----

## Resumen en 5 líneas

1. Leer pide bloqueos S, escribir pide X, "leer para modificar" pide U; los X duran hasta el final de la transacción en cualquier nivel.
2. Cuando dos bloqueos son incompatibles, una sesión espera: encuentra la cabeza de la cadena con `sys.dm_exec_requests`.
3. Con ~5.000 bloqueos en una instrucción, SQL Server escala a un bloqueo de tabla: procesa en lotes más chicos.
4. Muchos problemas de bloqueos son de índices (se revisan filas de más) o de transacciones largas.
5. `NOLOCK` no resuelve los bloqueos: lee datos sin confirmar, duplica y pierde filas; la solución correcta es RCSI.

-----

## Para profundizar

<details>
<summary>Bloqueos de rango en SERIALIZABLE</summary>

Con `SERIALIZABLE` (o `HOLDLOCK`), además de las filas leídas, el motor bloquea los **rangos de clave** del índice entre ellas (`RangeS-S`, `RangeS-U`, `RangeI-N` en `sys.dm_tran_locks`). Así, una transacción que leyó "los pedidos del cliente 700" impide que otra inserte un pedido nuevo de ese cliente. Si no hay un índice que permita bloquear solo ese rango, el motor bloquea un rango mucho mayor, o la tabla entera: otra vez, el índice decide el impacto de la concurrencia.

</details>

<details>
<summary>sp_getapplock: bloquear un concepto, no una fila</summary>

A veces lo que hay que serializar no es una fila sino un proceso: "solo una ejecución del cierre diario a la vez". `sp_getapplock` permite bloquear un nombre arbitrario con los mismos modos y la misma compatibilidad que los bloqueos de datos:

```sql
BEGIN TRANSACTION;
EXEC sp_getapplock @Resource = N'CierreDiario', @LockMode = 'Exclusive', @LockTimeout = 0;
-- ... el proceso ...
COMMIT;   -- libera el bloqueo
```

Si otra sesión ya lo tiene, `@LockTimeout = 0` hace que falle enseguida en lugar de esperar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

SQL Server bloquea los datos que se están usando: las lecturas ponen bloqueos compartidos y las escrituras bloqueos exclusivos, que duran hasta el `COMMIT`. Si dos sesiones necesitan bloqueos incompatibles, una espera. `NOLOCK` lee sin esperar, pero puede leer datos que todavía no se confirmaron, así que no debe usarse para datos importantes.

### Respuesta ampliada (semi-senior)

Distingo los modos S, U y X y su compatibilidad: muchas lecturas conviven, una escritura excluye a todos, y U existe para evitar el *deadlock* de conversión de S a X cuando dos sesiones leen para modificar. Los bloqueos X duran hasta el final de la transacción en cualquier nivel, así que las transacciones largas bloquean; los S, en `READ COMMITTED`, se liberan al leer cada fila. Diagnostico las esperas con `sys.dm_exec_requests` buscando la cabeza de la cadena, y con `sys.dm_tran_locks` veo qué tiene bloqueado. Sé que muchas esperas vienen de modificaciones sin índice en el filtro, que revisan y bloquean filas ajenas, o de instrucciones masivas que escalan a tabla a partir de unos 5.000 bloqueos, y por eso proceso en lotes. Uso `UPDLOCK` para leer antes de modificar y `READPAST` para colas. `NOLOCK` lo evito salvo en datos aproximados: además de lecturas sucias, puede duplicar o perder filas por movimientos de página. Para que las lecturas no esperen a las escrituras, la solución correcta es RCSI.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `NOLOCK` y `READ UNCOMMITTED`?**
El efecto es el mismo; `NOLOCK` se aplica a una tabla en una consulta y `READ UNCOMMITTED` a toda la sesión.

**2. ¿Qué es el escalamiento de bloqueos?**
Convertir muchos bloqueos de fila o página (unos 5.000 en una instrucción) en un bloqueo de tabla, para ahorrar memoria.

**3. ¿Para qué sirve el bloqueo U?**
Para leer con la intención de modificar: es compatible con S pero no con otro U, lo que evita el *deadlock* de dos sesiones que leen y después quieren escribir.

**4. ¿Cómo encuentras la sesión que está bloqueando a las demás?**
Con `sys.dm_exec_requests`, siguiendo `blocking_session_id` hasta la sesión que bloquea y no espera a nadie.

-----

## Práctica

**Ejercicio 1.** Con dos ventanas de SSMS sobre `Tienda`, reproduce un bloqueo y responde: ¿qué tipo de espera aparece en cada caso?

1. Ventana A: `BEGIN TRAN; UPDATE dbo.Producto SET Stock = Stock - 1 WHERE ProductoId = 2;` (sin confirmar). Ventana B: `SELECT Stock FROM dbo.Producto WHERE ProductoId = 2;`.
2. Igual, pero la ventana B ejecuta `UPDATE dbo.Producto SET Stock = Stock + 1 WHERE ProductoId = 2;`.
3. Igual que el 1, pero la ventana B lee el producto 4 (`WHERE ProductoId = 4`).

<details>
<summary>Solución</summary>

1. **`LCK_M_S`:** B quiere leer (S) una fila con bloqueo X.
2. **`LCK_M_U`:** el `UPDATE` de B busca la fila con un bloqueo U, incompatible con el X de A.
3. **No espera:** A solo bloqueó la fila del producto 2, y B va directo a la fila del producto 4 por la clave primaria. Si la consulta de B hubiera tenido que recorrer la tabla (por ejemplo, `WHERE Nombre = N'Lámpara de escritorio'`, sin índice sobre `Nombre`), habría esperado al pasar por la fila del producto 2. Con 6 filas el motor podría además elegir otra estrategia; con volumen, la diferencia entre tener o no índice se nota claramente.

En todos los casos, `ROLLBACK` en la ventana A libera los bloqueos.

</details>

**Ejercicio 2.** Escribe la consulta de cola para que varios procesos tomen, cada uno, **un** pedido `'Pagado'` distinto y lo marquen como `'Enviado'`, devolviendo cuál tomaron. ¿Qué hint es imprescindible y por qué?

<details>
<summary>Solución</summary>

```sql
WITH Siguiente AS (
    SELECT TOP (1) PedidoId, Estado
    FROM dbo.Pedido WITH (UPDLOCK, READPAST, ROWLOCK)
    WHERE Estado = 'Pagado'
    ORDER BY FechaPedido, PedidoId
)
UPDATE Siguiente
SET Estado = 'Enviado'
OUTPUT inserted.PedidoId;
```

* **`READPAST`** es el imprescindible: sin él, el segundo proceso esperaría en la fila que tomó el primero en lugar de pasar a la siguiente.
* **`UPDLOCK`** asegura que la fila elegida quede bloqueada para modificar desde el momento en que se lee.
* **`ROWLOCK`** aquí sí tiene sentido: evita que el motor elija un bloqueo de página, que impediría a los demás procesos tomar otros pedidos de la misma página.

Para que escale con volumen, hace falta un índice que empiece por `Estado` (o filtrado `WHERE Estado = 'Pagado'`) y ordenado por `FechaPedido`.

</details>

-----

## Siguiente lección

[Versionado con RCSI y SNAPSHOT](04-Versionado%20con%20RCSI%20y%20SNAPSHOT.md)
