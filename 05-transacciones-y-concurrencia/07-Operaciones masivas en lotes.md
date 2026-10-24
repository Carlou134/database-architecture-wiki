# Operaciones masivas en lotes

## En una frase

Borrar, actualizar o cargar millones de filas en **una sola instrucción** es una transacción gigante que llena el log, escala los bloqueos a toda la tabla y, si falla, tarda lo mismo en deshacerse; procesarlo en **lotes** pequeños (cada uno con su propia transacción corta, avanzando por la clave, con un registro de progreso y pausas) convierte una operación peligrosa en una rutina que se puede detener, reanudar y ejecutar con la base en uso.

-----

## Antes de empezar

Conviene que ya sepas:

* `DELETE TOP (n)`, `OUTPUT` y `MERGE`, de [Modificar datos](../03-consultas/11-Modificar%20datos.md).
* El escalamiento de bloqueos (~5.000 bloqueos), de [Bloqueos, hints y NOLOCK](03-Bloqueos%2C%20hints%20y%20NOLOCK.md).
* El log de transacciones, de [Transacciones y ACID](01-Transacciones%20y%20ACID.md).
* La base `TiendaGrande`, de [Datos de volumen](../04-indices-y-rendimiento/00-Datos%20de%20volumen.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Procesamiento por lotes (batching, chunking):** dividir una operación masiva en muchas operaciones pequeñas.
* **Tamaño de lote:** cuántas filas procesa cada iteración.
* **Recorrido por clave (keyset):** avanzar por rangos de la clave (`PedidoId` del 1 al 5.000, del 5.001 al 10.000...) en lugar de buscar "las siguientes n que cumplan la condición".
* **Punto de control del proceso (checkpoint):** registro del último lote procesado, para poder reanudar.
* **Regulación (throttling):** pausas y ventanas horarias para limitar el impacto en el resto del sistema.
* **Tabla de staging:** tabla intermedia donde se cargan los datos externos para validarlos antes de pasarlos a las tablas definitivas.
* **Registro mínimo (minimal logging):** modo de carga masiva que registra en el log solo la asignación de páginas, no cada fila.

-----

## El problema

Política de retención: borrar los pedidos cancelados anteriores a 2025. En `TiendaGrande` son decenas de miles de pedidos, con sus líneas (por la cascada de la clave foránea). Alguien lo ejecuta a las 10 de la mañana:

```sql
DELETE FROM dbo.Pedido
WHERE Estado = 'Cancelado' AND FechaPedido < '2025-01-01';
```

Durante los minutos que tarda:

* **El escalamiento** convierte los bloqueos de fila en un bloqueo exclusivo de **toda** la tabla `Pedido`: nadie puede crear un pedido ni consultar uno.
* **El log** crece con cada fila y cada línea borrada; si el disco del log se llena, la base deja de aceptar escrituras.
* **Si alguien lo cancela** a mitad de camino, el `ROLLBACK` tarda lo mismo que el trabajo hecho, con los mismos bloqueos.
* **En un servidor con réplicas,** todo ese log tiene que viajar a las réplicas, que quedan atrasadas.

Y en una base de producción real, con cientos de millones de filas, esto no son minutos sino horas.

-----

## Cómo funciona

### La idea: muchas transacciones cortas

```text
Una instrucción:   [=============== 80.000 filas, 1 transacción, bloqueo de tabla ===============]

En lotes:          [4.000][pausa][4.000][pausa][4.000][pausa] ... [4.000]
                   cada lote: su transacción, bloqueos de fila, log reutilizable
```

Cada lote:

* Toca **menos de ~5.000 filas**, así que no escala a un bloqueo de tabla.
* Dura poco: los bloqueos se liberan al terminar el lote, y los demás usuarios trabajan entre lote y lote.
* Se confirma por separado: si el proceso se detiene, lo ya procesado queda hecho, y se puede **reanudar** desde ahí.
* Permite que el log se reutilice entre lotes (en el modelo `SIMPLE`, automáticamente; en `FULL`, con las copias de seguridad del log que ya hace el servidor).

### El patrón básico: `DELETE TOP` en un bucle

Es el de tu cheat sheet de la sesión 10:

```sql
DECLARE @Lote INT = 4000, @Filas INT = 1;

WHILE @Filas > 0
BEGIN
    DELETE TOP (@Lote) FROM dbo.Pedido
    WHERE Estado = 'Cancelado' AND FechaPedido < '2025-01-01';

    SET @Filas = @@ROWCOUNT;
END;
```

Funciona, con una condición importante: **el filtro necesita un índice**. Sin un índice sobre `(Estado, FechaPedido)`, cada iteración recorre la tabla desde el principio para encontrar las siguientes 4.000 filas, y a medida que avanza tiene que saltarse más filas que ya no cumplen el filtro (o que nunca lo cumplieron). Con 20 iteraciones, son 20 *scans* de la tabla completa.

### El patrón por clave: avanzar por rangos

Una alternativa más predecible es recorrer la tabla **por su clave**, en rangos fijos, y aplicar el filtro dentro de cada rango:

```sql
DECLARE @Desde INT = 1, @Tamano INT = 5000, @Max INT;
SELECT @Max = MAX(PedidoId) FROM dbo.Pedido;

WHILE @Desde <= @Max
BEGIN
    DELETE FROM dbo.Pedido
    WHERE PedidoId >= @Desde AND PedidoId < @Desde + @Tamano     -- el rango, por la clave agrupada
      AND Estado = 'Cancelado' AND FechaPedido < '2025-01-01';  -- el filtro, dentro del rango

    SET @Desde += @Tamano;
END;
```

* Cada iteración lee **solo su rango** de la clave agrupada (un *seek* de rango), sin volver a recorrer lo ya procesado.
* El tamaño del rango limita las filas tocadas por lote (como mucho 5.000, y menos si el filtro es selectivo).
* El progreso es la clave: guardar `@Desde` alcanza para saber dónde reanudar.

| Patrón | Ventaja | Desventaja |
| --- | --- | --- |
| `DELETE TOP (n)` + filtro | Lotes siempre de n filas | Necesita un índice sobre el filtro; si no, cada lote recorre la tabla |
| Por rangos de clave | Cada lote lee solo su rango; fácil de reanudar | Lotes de tamaño variable (algunos rangos casi vacíos) |

### El tamaño del lote

No hay un número mágico, pero sí límites claros:

* **Menos de ~5.000 filas** por instrucción, para evitar el escalamiento de bloqueos (con las cascadas, cuenta también las filas hijas).
* **Lo suficientemente grande** para que la sobrecarga por iteración no domine: lotes de 10 filas convierten un millón de filas en 100.000 transacciones.
* **Ajustado por medición:** se prueba un lote, se mide su duración y se calibra. Un objetivo razonable es que cada lote tarde menos de uno o dos segundos.

La tabla de tu cheat sheet (5.000–10.000 para `DELETE`, 2.000–5.000 para `UPDATE` que agranda filas, 1.000–2.000 en producción crítica) es un buen punto de partida, con el límite del escalamiento en mente para los valores más altos.

### Regulación: pausas y ventanas horarias

Entre lotes, una pausa breve deja respirar al resto del sistema (y a las réplicas):

```sql
WAITFOR DELAY '00:00:00.500';
```

Y para que el proceso solo corra en horario de baja actividad, hay que tener cuidado con los rangos que cruzan la medianoche. La condición de tu cheat sheet, `CAST(GETDATE() AS TIME) NOT BETWEEN '22:00' AND '06:00'`, **no funciona**: `BETWEEN '22:00' AND '06:00'` exige una hora que sea a la vez mayor o igual a las 22:00 y menor o igual a las 06:00, lo que es imposible, así que el `NOT BETWEEN` es siempre verdadero y el proceso se detiene siempre. La ventana nocturna se escribe con dos condiciones:

```sql
DECLARE @Ahora TIME = CAST(SYSDATETIME() AS TIME);
IF NOT (@Ahora >= '22:00' OR @Ahora < '06:00')
    BREAK;   -- fuera de la ventana permitida
```

### Registrar el progreso para poder reanudar

Para procesos que pueden tardar horas, una tabla de control guarda hasta dónde se llegó:

```sql
CREATE TABLE dbo.ProcesoLote (
    Proceso        VARCHAR(50)  NOT NULL PRIMARY KEY,
    UltimaClave    INT          NOT NULL,
    FilasAfectadas BIGINT       NOT NULL DEFAULT 0,
    Estado         VARCHAR(20)  NOT NULL,          -- 'EnCurso', 'Terminado'
    ActualizadoEn  DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
);
```

En cada lote, en la **misma transacción** que la modificación, se actualiza la fila de control. Así, la modificación y el registro del progreso se confirman juntos: si el proceso se cae entre dos lotes, la tabla de control dice exactamente qué lote fue el último confirmado.

### Cargar datos externos: staging y validación

Las cargas desde archivos (CSV, Excel, otros sistemas) siguen otro patrón: primero a una **tabla de staging** con columnas de texto, después validación, y recién después a las tablas definitivas.

```sql
-- Staging: todo como texto, tal como viene
CREATE TABLE dbo.StagingProducto (
    Fila       INT IDENTITY(1,1) PRIMARY KEY,
    Nombre     NVARCHAR(200) NULL,
    PrecioTxt  NVARCHAR(50)  NULL,
    StockTxt   NVARCHAR(50)  NULL
);

-- Validar con conversiones que no fallan
SELECT Fila, Nombre,
       CASE
           WHEN NULLIF(TRIM(Nombre), N'') IS NULL                        THEN 'Nombre vacío'
           WHEN TRY_CONVERT(DECIMAL(10,2), PrecioTxt) IS NULL             THEN 'Precio inválido'
           WHEN TRY_CONVERT(DECIMAL(10,2), PrecioTxt) < 0                 THEN 'Precio negativo'
           WHEN TRY_CONVERT(INT, StockTxt) IS NULL                        THEN 'Stock inválido'
           ELSE 'OK'
       END AS Validacion
FROM dbo.StagingProducto;
```

Dos detalles sobre las conversiones, que tu cheat sheet de la sesión 10 deja en el aire:

* **`TRY_CONVERT(DECIMAL, ...)` sin precisión** convierte a `DECIMAL(18,0)`: `'12.50'` se convierte en **13**, sin error. Escribe siempre la precisión y la escala.
* **`TRY_PARSE`** (para formatos dependientes de la cultura, como `'1.234,56'` en `es-ES`) usa el motor de .NET y es mucho más lento que `TRY_CONVERT`. Para cargas grandes, conviene normalizar el formato (reemplazar separadores) y usar `TRY_CONVERT`.

Las filas válidas pasan a las tablas definitivas (en lotes, si son muchas); las inválidas se conservan en staging, o en una tabla de rechazos, con el motivo, para corregirlas.

### Inserciones masivas con registro mínimo

Para cargar millones de filas en una tabla vacía o en un heap, SQL Server puede registrar en el log solo la asignación de páginas, no cada fila. Requiere el modelo de recuperación `SIMPLE` o `BULK_LOGGED`, y el hint `TABLOCK` en la tabla de destino:

```sql
INSERT INTO dbo.PedidoHistorico WITH (TABLOCK) (PedidoId, ClienteId, FechaPedido, Estado)
SELECT PedidoId, ClienteId, FechaPedido, Estado FROM dbo.Pedido WHERE FechaPedido < '2024-01-01';
```

`TABLOCK` bloquea toda la tabla de destino durante la carga, así que es para tablas que nadie más está usando (históricos, staging). Para cargar archivos, `BULK INSERT` y la herramienta `bcp` aplican la misma técnica.

### `MERGE` en procesos de sincronización

`MERGE` es la instrucción natural para sincronizar una tabla con un origen (insertar lo nuevo, actualizar lo que cambió y, opcionalmente, borrar lo que ya no está). Además de las advertencias de [Modificar datos](../03-consultas/11-Modificar%20datos.md), en procesos masivos hay una más: **`WHEN NOT MATCHED BY SOURCE THEN DELETE` borra todo lo que no está en el origen**. Si el origen es parcial (el archivo de hoy trae solo los productos que cambiaron, o la carga falló a la mitad), `MERGE` borra el resto de la tabla. Esa cláusula solo se usa cuando el origen es la foto **completa** de lo que debe existir, y conviene limitarla con una condición (`WHEN NOT MATCHED BY SOURCE AND destino.CategoriaId = @Categoria THEN DELETE`).

-----

## Ejemplo completo

Archivar y borrar los pedidos cancelados anteriores a 2025 en `TiendaGrande`, por lotes, con progreso reanudable, pausas y mensajes de avance.

```sql
USE TiendaGrande;

-- Tabla de archivo (una vez)
CREATE TABLE dbo.PedidoArchivado (
    PedidoId    INT          NOT NULL PRIMARY KEY,
    ClienteId   INT          NOT NULL,
    FechaPedido DATETIME2(0) NOT NULL,
    Estado      VARCHAR(20)  NOT NULL,
    ArchivadoEn DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
);

-- Registro del proceso (una vez), con la tabla dbo.ProcesoLote de arriba
IF NOT EXISTS (SELECT 1 FROM dbo.ProcesoLote WHERE Proceso = 'ArchivarCancelados2024')
    INSERT INTO dbo.ProcesoLote (Proceso, UltimaClave, Estado) VALUES ('ArchivarCancelados2024', 0, 'EnCurso');
GO

SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @Tamano INT = 5000,
        @Desde  INT = (SELECT UltimaClave + 1 FROM dbo.ProcesoLote WHERE Proceso = 'ArchivarCancelados2024'),
        @Max    INT = (SELECT MAX(PedidoId) FROM dbo.Pedido),
        @Filas  INT,
        @Total  BIGINT = 0;

WHILE @Desde <= @Max
BEGIN
    BEGIN TRANSACTION;

        -- Borrar el rango y archivarlo en la misma instrucción
        DELETE FROM dbo.Pedido
        OUTPUT deleted.PedidoId, deleted.ClienteId, deleted.FechaPedido, deleted.Estado
        INTO dbo.PedidoArchivado (PedidoId, ClienteId, FechaPedido, Estado)
        WHERE PedidoId >= @Desde AND PedidoId < @Desde + @Tamano
          AND Estado = 'Cancelado'
          AND FechaPedido < '2025-01-01';        -- las líneas se borran por ON DELETE CASCADE

        SET @Filas = @@ROWCOUNT;
        SET @Total += @Filas;

        -- Registrar el progreso en la misma transacción
        UPDATE dbo.ProcesoLote
        SET UltimaClave = @Desde + @Tamano - 1,
            FilasAfectadas += @Filas,
            ActualizadoEn = SYSUTCDATETIME()
        WHERE Proceso = 'ArchivarCancelados2024';

    COMMIT TRANSACTION;

    RAISERROR(N'Hasta PedidoId %d: %d filas en este lote, %I64d en total', 0, 1,
              @Desde + @Tamano - 1, @Filas, @Total) WITH NOWAIT;

    SET @Desde += @Tamano;
    WAITFOR DELAY '00:00:00.200';                -- regulación
END;

UPDATE dbo.ProcesoLote SET Estado = 'Terminado' WHERE Proceso = 'ArchivarCancelados2024';
```

En la pestaña **Mensajes**, el avance aparece mientras se ejecuta:

```text
Hasta PedidoId 5000: 140 filas en este lote, 140 en total
Hasta PedidoId 10000: 137 filas en este lote, 277 en total
...
```

(Las cantidades son aproximadas: alrededor del 5 % de los pedidos están cancelados, y algo más de la mitad son anteriores a 2025.)

Qué decisión hay detrás:

* **Rangos de 5.000 por la clave agrupada:** cada lote es un *seek* de rango, sin depender de un índice sobre el estado, y nunca toca más de 5.000 pedidos (más sus líneas: hasta 4 por pedido, aunque solo los ~3 % que cumplen el filtro tienen líneas para borrar).
* **`DELETE ... OUTPUT ... INTO`:** el archivo y el borrado son la misma instrucción: no hay forma de borrar sin archivar.
* **El progreso en la misma transacción:** si el proceso se detiene (a mano o por un error), al volver a ejecutarlo lee `UltimaClave` y sigue desde el siguiente rango, sin repetir ni saltear.
* **`XACT_ABORT ON`:** un error en un lote deshace solo ese lote, y detiene el proceso; los lotes anteriores ya están confirmados.
* **`RAISERROR ... WITH NOWAIT`:** muestra el avance en tiempo real (`PRINT` esperaría a que terminara el lote de mensajes). `%I64d` es el formato para un `BIGINT`.
* **`WAITFOR` de 200 ms:** entre lotes, otras sesiones obtienen los bloqueos que necesitan.

-----

## Errores comunes

**1. Una sola instrucción para millones de filas.**
Qué pasa: bloqueo de tabla, log lleno, réplicas atrasadas, `ROLLBACK` eterno si se cancela.
Por qué: se prueba con pocas filas y se ejecuta en producción con muchas.
Arreglo: lotes de menos de ~5.000 filas, cada uno con su transacción.

**2. `DELETE TOP` sin índice sobre el filtro.**
Qué pasa: cada lote es más lento que el anterior.
Por qué: cada iteración recorre la tabla desde el principio buscando filas que cumplan el filtro.
Arreglo: un índice que soporte el filtro, o el recorrido por rangos de clave.

**3. Toda la carga en una transacción "para poder deshacerla".**
Qué pasa: se vuelve a tener una transacción gigante, con todos sus problemas.
Por qué: se confunde "poder deshacer" con "deshacer en un solo paso".
Arreglo: lotes confirmados por separado, con un registro de progreso; para deshacer, un proceso inverso (por ejemplo, restaurar desde la tabla de archivo).

**4. Ventanas horarias que cruzan la medianoche con `BETWEEN`.**
Qué pasa: el proceso nunca corre, o corre siempre.
Por qué: `BETWEEN '22:00' AND '06:00'` nunca es verdadero.
Arreglo: `@Ahora >= '22:00' OR @Ahora < '06:00'`.

**5. `TRY_CONVERT(DECIMAL, ...)` sin precisión.**
Qué pasa: precios redondeados a enteros, sin ningún error.
Por qué: `DECIMAL` sin parámetros es `DECIMAL(18,0)`.
Arreglo: `TRY_CONVERT(DECIMAL(10,2), ...)`.

**6. `MERGE` con `WHEN NOT MATCHED BY SOURCE THEN DELETE` sobre un origen parcial.**
Qué pasa: se borra todo lo que no venía en el archivo de hoy.
Por qué: la cláusula interpreta "no está en el origen" como "debe borrarse".
Arreglo: usarla solo con orígenes completos, y limitarla con una condición.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Borrar un lote | `DELETE TOP (n) ...` | `DELETE ... WHERE id IN (SELECT id ... LIMIT n)` | `DELETE ... LIMIT n` |
| Escalamiento de bloqueos | Sí (~5.000) | No | No |
| Lo que crece con transacciones grandes | El log de transacciones | El WAL, y las versiones muertas (que `VACUUM` limpia) | El *undo log* y el *binlog* |
| Carga masiva de archivos | `BULK INSERT`, `bcp` | `COPY` | `LOAD DATA INFILE` |
| Registro mínimo | `SIMPLE`/`BULK_LOGGED` + `TABLOCK` | Tablas `UNLOGGED` | No equivalente directo |
| Mensajes de progreso | `RAISERROR ... WITH NOWAIT` | `RAISE NOTICE` | `SELECT` intermedios |
| Pausa | `WAITFOR DELAY` | `pg_sleep()` | `DO SLEEP()` |

En PostgreSQL, además, un `DELETE` masivo deja las filas borradas como versiones muertas que `VACUUM` tiene que limpiar después; borrar en lotes también le da a *autovacuum* la oportunidad de trabajar entre lote y lote.

### En MongoDB

`deleteMany` y `updateMany` sobre millones de documentos generan mucha carga en el *oplog* (el log de replicación) y pueden atrasar a las réplicas, igual que en SQL Server. El patrón es el mismo: lotes por rangos de `_id` (o por un campo indexado), con pausas. Para cargas, `insertMany` con lotes de unos miles de documentos, o `mongoimport`. Para el borrado por antigüedad, MongoDB ofrece además los índices **TTL**, que borran automáticamente los documentos cuya fecha supera un tiempo de vida.

-----

## Cuándo sí y cuándo no

**Procesa en lotes cuando:**

* La operación toca más de unos miles de filas en una tabla que se usa mientras tanto.
* El proceso puede tardar lo suficiente como para necesitar detenerlo y reanudarlo.

**Puedes hacerlo en una instrucción cuando:**

* La tabla es chica, o se usa solo en una ventana de mantenimiento sin usuarios, y el log tiene espacio de sobra.

**Ten en cuenta:**

* Para borrar datos por antigüedad de forma recurrente, el particionado de tablas permite eliminar meses enteros instantáneamente (`TRUNCATE TABLE ... WITH (PARTITIONS (...))`), sin borrar fila por fila.

-----

## Resumen en 5 líneas

1. Una operación masiva en una instrucción llena el log, escala a un bloqueo de tabla y tarda lo mismo en deshacerse.
2. En lotes de menos de ~5.000 filas, cada uno con su transacción corta, la base sigue en uso y el proceso se puede detener.
3. Recorre por rangos de la clave (o usa `TOP` con un índice sobre el filtro), y guarda el progreso en la misma transacción que la modificación.
4. Regula con pausas y ventanas horarias bien escritas (sin `BETWEEN` a través de la medianoche).
5. Carga datos externos a staging, valida con `TRY_CONVERT` y precisión explícita, y cuida el `DELETE` de `MERGE`.

-----

## Para profundizar

<details>
<summary>Eliminar particiones completas</summary>

Una tabla particionada por mes (con una función y un esquema de partición sobre la fecha) guarda cada mes en una partición separada. Borrar un mes entero no requiere borrar sus filas: `TRUNCATE TABLE dbo.Log WITH (PARTITIONS (3));` (desde SQL Server 2016) lo vacía instantáneamente, y `ALTER TABLE ... SWITCH PARTITION` mueve la partición completa a otra tabla (para archivarla) como una operación de metadatos. Es la técnica estándar para tablas de eventos con retención por fecha, y la alternativa definitiva al borrado por lotes en esos casos.

</details>

<details>
<summary>El log en el modelo de recuperación FULL</summary>

En el modelo `FULL` (el habitual en producción), el log solo se libera para reutilizarse después de una **copia de seguridad del log**. Procesar en lotes no reduce el total de log generado, pero sí evita que una sola transacción lo retenga: entre lotes, las copias de seguridad del log programadas pueden liberar el espacio. Si el proceso genera log más rápido de lo que se respalda, el log crece igual; en procesos muy grandes, se coordina con el responsable de las copias de seguridad.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para borrar o actualizar millones de filas, lo hago en lotes, por ejemplo con `DELETE TOP (5000)` en un bucle, para que cada transacción sea corta, no se bloquee toda la tabla y el log no crezca demasiado. Agrego pausas entre lotes para no afectar a los usuarios.

### Respuesta ampliada (semi-senior)

Una operación masiva en una sola instrucción es una transacción gigante: escala a bloqueo de tabla a partir de unos 5.000 bloqueos, retiene el log, atrasa a las réplicas y, si se cancela, el `ROLLBACK` tarda lo mismo. La proceso en lotes de menos de 5.000 filas (contando las cascadas), cada uno con su transacción y `XACT_ABORT ON`. Prefiero recorrer por rangos de la clave agrupada, que lee solo cada rango y se reanuda fácilmente, o uso `DELETE TOP` solo si hay un índice sobre el filtro, porque si no cada lote recorre la tabla. Guardo el progreso en una tabla de control en la misma transacción que la modificación, para reanudar sin repetir ni saltear, archivo con `OUTPUT ... INTO` en la misma instrucción, regulo con pausas y ventanas horarias, y muestro el avance con `RAISERROR WITH NOWAIT`. Para cargas externas uso staging con validación por `TRY_CONVERT` con precisión explícita, y registro mínimo con `TABLOCK` cuando el destino lo permite. Y con `MERGE`, solo uso `WHEN NOT MATCHED BY SOURCE THEN DELETE` con orígenes completos.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué lotes de menos de 5.000 filas?**
Porque a partir de unos 5.000 bloqueos en una instrucción, SQL Server intenta escalarlos a un bloqueo de tabla.

**2. ¿Qué problema tiene `DELETE TOP (n)` en un bucle sin índice?**
Que cada iteración recorre la tabla desde el principio para encontrar las filas que cumplen el filtro.

**3. ¿Cómo haces que un proceso por lotes se pueda reanudar?**
Guardando la última clave procesada en una tabla de control, en la misma transacción que cada lote.

**4. ¿Qué es una tabla de staging?**
Una tabla intermedia, normalmente con columnas de texto, donde se cargan los datos externos para validarlos antes de pasarlos a las tablas definitivas.

-----

## Práctica

**Ejercicio 1.** En `TiendaGrande`, escribe un proceso por lotes que marque como `'Entregado'` todos los pedidos `'Enviado'` anteriores a 2026, en lotes de 4.000, mostrando el avance.

<details>
<summary>Solución</summary>

```sql
SET NOCOUNT ON; SET XACT_ABORT ON;

DECLARE @Lote INT = 4000, @Filas INT = 1, @Total INT = 0;

-- Índice que soporta el filtro (si no existe), para que cada lote no recorra la tabla
-- CREATE INDEX IX_Pedido_Estado_Fecha ON dbo.Pedido (Estado, FechaPedido);

WHILE @Filas > 0
BEGIN
    UPDATE TOP (@Lote) dbo.Pedido
    SET Estado = 'Entregado'
    WHERE Estado = 'Enviado' AND FechaPedido < '2026-01-01';

    SET @Filas = @@ROWCOUNT;
    SET @Total += @Filas;

    RAISERROR(N'%d filas en este lote, %d en total', 0, 1, @Filas, @Total) WITH NOWAIT;
    WAITFOR DELAY '00:00:00.200';
END;
```

Aquí `UPDATE TOP` funciona bien porque las filas actualizadas **dejan de cumplir el filtro** (ya no son `'Enviado'`), así que cada iteración encuentra filas nuevas, y con el índice sobre `(Estado, FechaPedido)` las encuentra sin recorrer la tabla. Cada `UPDATE` sin `BEGIN TRANSACTION` explícito es su propia transacción (confirmación automática), lo que para este caso alcanza. Son unos 40.000 pedidos (5 % enviados × ~80 % anteriores a 2026), unos 10 lotes.

</details>

**Ejercicio 2.** Un proceso nocturno debe correr solo entre las 23:00 y las 05:30. Escribe la condición que lo detiene fuera de ese horario, y explica por qué esta no sirve: `IF CAST(GETDATE() AS TIME) NOT BETWEEN '23:00' AND '05:30' BREAK;`.

<details>
<summary>Solución</summary>

```sql
DECLARE @Ahora TIME = CAST(SYSDATETIME() AS TIME);
IF NOT (@Ahora >= '23:00' OR @Ahora < '05:30')
    BREAK;
```

La versión con `BETWEEN` no sirve porque `BETWEEN '23:00' AND '05:30'` significa `>= '23:00' AND <= '05:30'`, y ninguna hora es a la vez posterior a las 23:00 y anterior a las 05:30 del **mismo** día. El `BETWEEN` es siempre falso, el `NOT BETWEEN` siempre verdadero, y el proceso se detiene en la primera iteración, a cualquier hora. Una ventana que cruza la medianoche es la **unión** de dos tramos (de 23:00 a 24:00 y de 00:00 a 05:30), por eso se escribe con `OR`.

</details>

-----

## Siguiente lección

Con esto termina el módulo de transacciones y concurrencia. El siguiente módulo trata cómo se programa dentro de SQL Server:

[Vistas](../06-programacion-t-sql/01-Vistas.md)
