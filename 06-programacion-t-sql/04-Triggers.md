# Triggers

## En una frase

Un *trigger* (disparador) es código que el motor ejecuta **automáticamente** cuando ocurre una modificación (`INSERT`, `UPDATE`, `DELETE`) sobre una tabla, dentro de la **misma transacción**, con acceso a las filas afectadas a través de las tablas virtuales `inserted` y `deleted`; es la herramienta para auditar o hacer cumplir reglas que ninguna restricción puede expresar, y también una de las fuentes más comunes de lógica escondida y lentitud.

-----

## Antes de empezar

Conviene que ya sepas:

* Restricciones y sus límites (un `CHECK` no puede mirar otras tablas), de [Restricciones](../01-fundamentos/06-Restricciones.md).
* `OUTPUT inserted.*, deleted.*`, de [Modificar datos](../03-consultas/11-Modificar%20datos.md).
* Transacciones y manejo de errores, del módulo de [Transacciones y concurrencia](../05-transacciones-y-concurrencia/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Trigger DML:** se ejecuta ante un `INSERT`, `UPDATE` o `DELETE` sobre una tabla o vista.
* **`AFTER` (o `FOR`):** el *trigger* se ejecuta después de la modificación, cuando ya se aplicaron las restricciones.
* **`INSTEAD OF`:** el *trigger* se ejecuta **en lugar** de la modificación; él decide qué hacer.
* **`inserted` / `deleted`:** tablas virtuales con las filas nuevas y las filas anteriores de la modificación.
* **Trigger DDL:** se ejecuta ante cambios de estructura (`CREATE`, `ALTER`, `DROP`).
* **Trigger anidado / recursivo:** un *trigger* cuya modificación dispara otro *trigger* / a sí mismo.

-----

## El problema

El área de precios quiere saber, para cada cambio de precio: qué producto, el precio anterior, el nuevo, cuándo y quién. Los precios se cambian desde:

* La aplicación de administración.
* El procedimiento `dbo.CambiarPrecio` de la lección anterior.
* Un script de actualización masiva que corre cada lunes.
* Un analista que, de vez en cuando, ejecuta un `UPDATE` en SSMS.

Agregar el registro en cada uno de esos lugares es frágil: el próximo proceso que alguien escriba no lo va a tener. Lo que hace falta es que el registro ocurra **siempre** que cambie un precio, sin importar quién lo cambie ni cómo. Eso es exactamente lo que hace un *trigger*, y también es la razón por la que hay que usarlos con cuidado: se ejecutan siempre, aunque nadie se acuerde de que existen.

-----

## Cómo funciona

### `inserted` y `deleted`

Dentro de un *trigger*, dos tablas virtuales contienen las filas de la modificación:

| Operación | `inserted` | `deleted` |
| --- | --- | --- |
| `INSERT` | Las filas nuevas | Vacía |
| `DELETE` | Vacía | Las filas borradas |
| `UPDATE` | Las filas **después** del cambio | Las filas **antes** del cambio |

Tienen las mismas columnas que la tabla, y se usan como cualquier tabla: con `SELECT`, `JOIN`, `EXISTS`.

### Un trigger `AFTER` de auditoría

```sql
CREATE TABLE dbo.HistorialPrecio (
    HistorialPrecioId INT IDENTITY(1,1) PRIMARY KEY,
    ProductoId        INT           NOT NULL,
    PrecioAnterior    DECIMAL(10,2) NOT NULL,
    PrecioNuevo       DECIMAL(10,2) NOT NULL,
    CambiadoEn        DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    CambiadoPor       NVARCHAR(128) NOT NULL DEFAULT SUSER_SNAME()
);
GO

CREATE OR ALTER TRIGGER dbo.trProducto_HistorialPrecio
ON dbo.Producto
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT UPDATE(Precio) RETURN;   -- la instrucción no tocó la columna Precio

    INSERT INTO dbo.HistorialPrecio (ProductoId, PrecioAnterior, PrecioNuevo)
    SELECT d.ProductoId, d.Precio, i.Precio
    FROM inserted AS i
    JOIN deleted  AS d ON d.ProductoId = i.ProductoId
    WHERE i.Precio <> d.Precio;      -- solo las filas en las que el precio cambió de verdad
END;
GO
```

* **`AFTER UPDATE`:** se ejecuta después de que el `UPDATE` se aplicó y las restricciones se verificaron.
* **`UPDATE(Precio)`** es verdadero si la instrucción **mencionó** la columna en el `SET`, aunque el valor no haya cambiado; el `WHERE i.Precio <> d.Precio` filtra las filas que de verdad cambiaron.
* **`SET NOCOUNT ON`:** sin él, el `INSERT` del *trigger* agrega un "(n rows affected)" extra que confunde a algunas aplicaciones.

### La regla más importante: un trigger por instrucción, no por fila

En SQL Server, un *trigger* se ejecuta **una vez por instrucción**, no una vez por fila. Si un `UPDATE` modifica 500 productos, el *trigger* se ejecuta **una vez**, con 500 filas en `inserted` y `deleted`.

Este es el error más frecuente con los *triggers*:

```sql
-- ❌ Asume una sola fila
DECLARE @ProductoId INT, @Anterior DECIMAL(10,2), @Nuevo DECIMAL(10,2);
SELECT @ProductoId = i.ProductoId, @Nuevo = i.Precio, @Anterior = d.Precio
FROM inserted AS i JOIN deleted AS d ON d.ProductoId = i.ProductoId;

INSERT INTO dbo.HistorialPrecio (ProductoId, PrecioAnterior, PrecioNuevo)
VALUES (@ProductoId, @Anterior, @Nuevo);
```

Con un `UPDATE` de una fila, funciona. Con el script masivo del lunes (500 filas), las variables se quedan con los valores de **una** fila cualquiera, y se registra un solo cambio de los 500. Sin ningún error. La regla: **todo *trigger* se escribe basado en conjuntos**, con `INSERT ... SELECT` desde `inserted` y `deleted`, como en el ejemplo correcto de arriba.

### Hacer cumplir reglas entre tablas

Un `CHECK` no puede mirar otras tablas. Un *trigger* sí:

```sql
-- No se pueden agregar líneas a un pedido que ya se envió o entregó
CREATE OR ALTER TRIGGER dbo.trDetallePedido_PedidoAbierto
ON dbo.DetallePedido
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted AS i
        JOIN dbo.Pedido AS p ON p.PedidoId = i.PedidoId
        WHERE p.Estado IN ('Enviado', 'Entregado', 'Cancelado')
    )
        THROW 50301, N'No se pueden modificar las líneas de un pedido enviado, entregado o cancelado.', 1;
END;
```

El `THROW` dentro del *trigger* deshace la instrucción que lo disparó (y la transacción entera, porque los *triggers* se ejecutan con `XACT_ABORT ON` implícito). Antes de usar un *trigger* para una regla, conviene preguntarse si no la resuelve mejor una restricción, el diseño o el procedimiento que hace la operación: un *trigger* es la última línea de defensa, no la primera.

### `INSTEAD OF`: reemplazar la operación

Un *trigger* `INSTEAD OF` se ejecuta **en lugar** de la modificación. Sus dos usos principales:

* **Hacer actualizable una vista** que el motor no sabría cómo modificar (una vista de varias tablas).
* **Convertir un `DELETE` en un borrado lógico:**

```sql
CREATE OR ALTER TRIGGER dbo.trCliente_BorradoLogico
ON dbo.ClienteV2
INSTEAD OF DELETE
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE c
    SET c.EliminadoEn = SYSUTCDATETIME()
    FROM dbo.ClienteV2 AS c
    JOIN deleted AS d ON d.ClienteId = c.ClienteId;
END;
```

Con este *trigger*, un `DELETE FROM dbo.ClienteV2 WHERE ClienteId = 5` no borra la fila: la marca como eliminada. Es cómodo, y también es exactamente el tipo de comportamiento "mágico" que sorprende a quien no sabe que el *trigger* existe.

### Triggers anidados y recursivos

* **Anidados:** si el *trigger* de `Producto` inserta en `HistorialPrecio`, y `HistorialPrecio` tiene su propio *trigger*, este también se ejecuta. Está activado por defecto, con un límite de 32 niveles.
* **Recursivos:** un *trigger* que modifica su propia tabla vuelve a dispararse **solo** si la opción `RECURSIVE_TRIGGERS` de la base está activada (por defecto no lo está).

Las cadenas de *triggers* son la forma más rápida de convertir una base en un sistema imposible de entender: un `UPDATE` sencillo dispara cuatro *triggers* en tres tablas, y nadie sabe por qué una inserción tarda dos segundos.

### Lo que no dispara un trigger

* **`TRUNCATE TABLE`:** no dispara los *triggers* de `DELETE`.
* **`BULK INSERT` y `bcp`:** no los disparan, salvo con la opción `FIRE_TRIGGERS`.
* **Las cascadas de claves foráneas** sí disparan los *triggers* de la tabla hija en SQL Server.

### Su costo

Un *trigger* se ejecuta dentro de la transacción de la instrucción que lo disparó:

* **Alarga la transacción:** sus lecturas y escrituras suman tiempo y bloqueos.
* **Su error deshace la instrucción original:** si el `INSERT` del historial falla, el `UPDATE` del precio también se deshace.
* **Es invisible:** no aparece en el código que hace el `UPDATE`; solo se ve en el plan de ejecución real o mirando la tabla.

Antes de crear un *trigger* de auditoría, conviene conocer las alternativas que no tienen estos costos: las **tablas temporales con versionado del sistema** y **Change Data Capture** (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md) y [Auditoría](../07-monitoreo-y-administracion/06-Auditoría.md)).

-----

## Ejemplo completo

La auditoría de precios del problema, probada con una actualización de varias filas en `Tienda`:

```sql
-- (con la tabla dbo.HistorialPrecio y el trigger dbo.trProducto_HistorialPrecio de arriba)

BEGIN TRANSACTION;   -- envoltorio de prueba

-- El "script masivo del lunes": sube un 10 % la electrónica
UPDATE dbo.Producto
SET Precio = Precio * 1.10
WHERE CategoriaId = 1;

-- Un cambio que no cambia nada (mismo precio)
UPDATE dbo.Producto SET Precio = Precio WHERE ProductoId = 6;

SELECT ProductoId, PrecioAnterior, PrecioNuevo FROM dbo.HistorialPrecio ORDER BY ProductoId;

ROLLBACK;
```

| ProductoId | PrecioAnterior | PrecioNuevo |
| --- | --- | --- |
| 1 | 120.00 | 132.00 |
| 2 | 45.50 | 50.05 |
| 3 | 899.90 | 989.89 |

Qué muestra el resultado:

* **Las tres filas del `UPDATE` masivo quedaron registradas:** el *trigger* procesa `inserted` y `deleted` como conjuntos.
* **El cambio de Clean Code no aparece:** `UPDATE(Precio)` era verdadero (la columna se mencionó), pero el `WHERE i.Precio <> d.Precio` descartó la fila porque el precio no cambió.
* **El `ROLLBACK` deshace también el historial:** el *trigger* se ejecutó dentro de la misma transacción, así que el registro de auditoría y el cambio de precio se confirman o se deshacen juntos.
* **`CambiadoPor`** guarda el usuario de la conexión (`SUSER_SNAME()`). Si la aplicación se conecta siempre con el mismo usuario, eso no dice quién fue la persona; para eso, la aplicación tiene que informar el usuario, por ejemplo con `SESSION_CONTEXT` (ver [Auditoría](../07-monitoreo-y-administracion/06-Auditoría.md)).

-----

## Errores comunes

**1. Triggers que asumen una sola fila.**
Qué pasa: en las modificaciones de varias filas, solo se procesa una, sin error.
Por qué: el *trigger* se ejecuta una vez por instrucción, y las variables toman el valor de una fila cualquiera.
Arreglo: siempre `INSERT ... SELECT` o `UPDATE ... FROM` sobre `inserted` y `deleted`.

**2. Lógica de negocio escondida en triggers.**
Qué pasa: comportamientos inesperados (un `DELETE` que no borra, un `INSERT` que modifica otra tabla) que nadie entiende.
Por qué: el *trigger* no aparece en el código que hace la operación.
Arreglo: reservarlos para auditoría y reglas que no pueden ir en otro lugar, y documentarlos.

**3. Llamadas externas o trabajo pesado dentro de un trigger.**
Qué pasa: cada modificación de la tabla se vuelve lenta, y los bloqueos duran más.
Por qué: el *trigger* se ejecuta dentro de la transacción, de forma sincrónica.
Arreglo: el *trigger* solo registra (en una tabla de cola, por ejemplo); otro proceso hace el trabajo pesado después.

**4. Confiar en `UPDATE(columna)` para saber si el valor cambió.**
Qué pasa: registros de auditoría de "cambios" que no cambiaron nada.
Por qué: `UPDATE(columna)` solo indica que la columna se mencionó en el `SET`.
Arreglo: comparar `inserted` con `deleted` (cuidando los `NULL`: `i.Col <> d.Col OR (i.Col IS NULL) <> (d.Col IS NULL)`, o `IS DISTINCT FROM` en 2022+).

**5. Olvidar que `TRUNCATE` y las cargas masivas no los disparan.**
Qué pasa: una auditoría con huecos.
Por qué: esas operaciones no pasan por los *triggers* de `DELETE`/`INSERT`.
Arreglo: restringir esos permisos, o usar `FIRE_TRIGGERS` en las cargas.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Frecuencia | Una vez por **instrucción** | Por fila (`FOR EACH ROW`) o por instrucción (`FOR EACH STATEMENT`) | Solo por **fila** |
| Filas afectadas | `inserted` / `deleted` | `NEW` / `OLD` (por fila); tablas de transición `NEW TABLE` / `OLD TABLE` (por instrucción) | `NEW` / `OLD` |
| Antes de la modificación | No (`INSTEAD OF` en su lugar) | `BEFORE` (puede cambiar `NEW`) | `BEFORE` (puede cambiar `NEW`) |
| `INSTEAD OF` | Tablas y vistas | Solo vistas | No |
| El código | Bloque T-SQL en el *trigger* | Una función aparte (`EXECUTE FUNCTION`) | Bloque en el *trigger* |
| Disparan en cascadas de FK | Sí | Sí | **No** |
| Triggers DDL | Sí (de base y de servidor) | *Event triggers* | No |

La diferencia por fila frente a por instrucción es la trampa clásica al migrar: un *trigger* de MySQL o de PostgreSQL con `FOR EACH ROW`, que usa `NEW.Precio`, no se traduce a SQL Server con variables, sino con un `INSERT ... SELECT` desde `inserted`.

### En MongoDB

MongoDB no tiene *triggers* en el servidor de base de datos. El equivalente son los **Change Streams**: la aplicación se suscribe a los cambios de una colección y reacciona a cada uno, de forma asíncrona y fuera de la transacción. En MongoDB Atlas existen además los *Atlas Triggers*, que ejecutan funciones ante cambios. En los dos casos, a diferencia de SQL Server, la reacción no puede deshacer el cambio que la originó.

-----

## Cuándo sí y cuándo no

**Usa un trigger cuando:**

* Una auditoría debe registrar los cambios sin excepción, venga de donde venga la modificación, y no te sirven las tablas temporales ni CDC.
* Una regla entre tablas debe cumplirse siempre, y no se puede expresar con restricciones ni garantizar en un único procedimiento.

**No lo uses cuando:**

* La regla cabe en una restricción (`CHECK`, `UNIQUE`, FK).
* La lógica pertenece al flujo de negocio de la aplicación (enviar un correo, llamar a una API).
* Hay muchas escrituras por segundo y el *trigger* agregaría trabajo pesado a cada una.

-----

## Resumen en 5 líneas

1. Un *trigger* se ejecuta automáticamente ante una modificación, dentro de la misma transacción.
2. `inserted` y `deleted` tienen las filas nuevas y anteriores; en SQL Server, el *trigger* se ejecuta una vez por **instrucción**, con todas las filas.
3. Escríbelos siempre basados en conjuntos; nunca asumas una sola fila.
4. `AFTER` para auditar y validar; `INSTEAD OF` para reemplazar la operación (vistas, borrado lógico).
5. Son lógica invisible que alarga cada transacción: resérvalos para lo que no puede ir en otro lugar, y considera tablas temporales o CDC para auditar.

-----

## Para profundizar

<details>
<summary>Triggers DDL: auditar cambios de estructura</summary>

Un *trigger* DDL se dispara ante cambios de estructura, a nivel de base o de servidor, y la función `EVENTDATA()` devuelve un XML con el detalle (quién, qué instrucción, sobre qué objeto):

```sql
CREATE TRIGGER trAuditarDDL ON DATABASE
FOR CREATE_TABLE, ALTER_TABLE, DROP_TABLE
AS
BEGIN
    INSERT INTO dbo.AuditoriaDDL (Evento, Usuario, Instruccion, Fecha)
    SELECT EVENTDATA().value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(100)'),
           EVENTDATA().value('(/EVENT_INSTANCE/LoginName)[1]', 'NVARCHAR(128)'),
           EVENTDATA().value('(/EVENT_INSTANCE/TSQLCommand/CommandText)[1]', 'NVARCHAR(MAX)'),
           SYSUTCDATETIME();
END;
```

También pueden impedir cambios (con `ROLLBACK`), por ejemplo, prohibir `DROP TABLE` en producción fuera de una ventana de despliegue.

</details>

<details>
<summary>El orden de ejecución de varios triggers</summary>

Si una tabla tiene varios *triggers* `AFTER` para la misma operación, el orden de ejecución no está definido, salvo el primero y el último, que se fijan con `sp_settriggerorder`. Si el orden importa, suele ser mejor un solo *trigger* que haga las dos cosas en el orden correcto. Para ver todos los *triggers* de la base: `SELECT name, OBJECT_NAME(parent_id) AS Tabla, is_disabled FROM sys.triggers;`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un *trigger* es código que se ejecuta automáticamente cuando se inserta, actualiza o borra en una tabla. Dentro del *trigger*, las tablas `inserted` y `deleted` tienen las filas nuevas y las anteriores. Se usan, por ejemplo, para guardar un historial de cambios.

### Respuesta ampliada (semi-senior)

Un *trigger* DML se ejecuta dentro de la transacción de la instrucción que lo dispara, y en SQL Server se dispara una vez por **instrucción**, no por fila, así que lo escribo siempre basado en conjuntos sobre `inserted` y `deleted`; el error más frecuente es usar variables y procesar solo una fila de un `UPDATE` masivo. Uso `AFTER` para auditar y para reglas entre tablas que una restricción no puede expresar, e `INSTEAD OF` para hacer actualizable una vista o convertir un borrado en lógico. Distingo `UPDATE(columna)` (la columna se mencionó) de que el valor haya cambiado, y para eso comparo `inserted` con `deleted` cuidando los `NULL`. Sé lo que no los dispara (`TRUNCATE`, cargas masivas sin `FIRE_TRIGGERS`) y su costo: alargan la transacción, su error deshace la operación original y son lógica invisible. Por eso los reservo para lo que no puede ir en otro lugar, no hago trabajo pesado ni llamadas externas dentro, y para auditar considero primero las tablas temporales o Change Data Capture.

### Preguntas frecuentes de seguimiento

**1. ¿Cuántas veces se ejecuta un trigger ante un `UPDATE` de 1.000 filas?**
Una vez, con las 1.000 filas en `inserted` y `deleted`.

**2. ¿Qué diferencia hay entre `AFTER` e `INSTEAD OF`?**
`AFTER` se ejecuta después de la modificación; `INSTEAD OF` en su lugar, y decide qué hacer.

**3. ¿Qué contienen `inserted` y `deleted` en un `UPDATE`?**
`inserted`, las filas después del cambio; `deleted`, las filas antes del cambio.

**4. ¿Qué riesgos tienen los triggers?**
Lógica escondida, transacciones más largas, errores que deshacen la operación original y, si se escriben por fila, modificaciones masivas procesadas a medias.

-----

## Práctica

**Ejercicio 1.** Escribe un *trigger* que registre en `dbo.HistorialEstadoPedido(PedidoId, EstadoAnterior, EstadoNuevo, CambiadoEn)` cada cambio de estado de los pedidos, y que funcione con modificaciones de varias filas.

<details>
<summary>Solución</summary>

```sql
CREATE TABLE dbo.HistorialEstadoPedido (
    HistorialEstadoPedidoId INT IDENTITY(1,1) PRIMARY KEY,
    PedidoId       INT          NOT NULL,
    EstadoAnterior VARCHAR(20)  NOT NULL,
    EstadoNuevo    VARCHAR(20)  NOT NULL,
    CambiadoEn     DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME()
);
GO

CREATE OR ALTER TRIGGER dbo.trPedido_HistorialEstado
ON dbo.Pedido
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT UPDATE(Estado) RETURN;

    INSERT INTO dbo.HistorialEstadoPedido (PedidoId, EstadoAnterior, EstadoNuevo)
    SELECT d.PedidoId, d.Estado, i.Estado
    FROM inserted AS i
    JOIN deleted  AS d ON d.PedidoId = i.PedidoId
    WHERE i.Estado <> d.Estado;
END;
```

`Estado` es `NOT NULL`, así que la comparación `<>` no necesita contemplar nulos. Con este historial, el embudo de [Agregados y GROUP BY](../03-consultas/05-Agregados%20y%20GROUP%20BY.md) podría medir también cuánto tarda cada etapa.

</details>

**Ejercicio 2.** ¿Qué está mal en este *trigger*, y qué pasa con `UPDATE dbo.Producto SET Stock = 0 WHERE CategoriaId = 2;`?

```sql
CREATE TRIGGER dbo.trProducto_SinStock ON dbo.Producto AFTER UPDATE
AS
BEGIN
    DECLARE @Id INT, @Stock INT;
    SELECT @Id = ProductoId, @Stock = Stock FROM inserted;
    IF @Stock = 0
        UPDATE dbo.Producto SET Activo = 0 WHERE ProductoId = @Id;
END;
```

<details>
<summary>Solución</summary>

Asume una sola fila. El `UPDATE` afecta a dos productos (la lámpara y la cafetera); `@Id` y `@Stock` toman los valores de **una** de las dos filas, cualquiera, así que se desactiva solo uno de los dos productos. Además, le falta `SET NOCOUNT ON`, y el `UPDATE` del *trigger* sobre su propia tabla lo dispararía otra vez si `RECURSIVE_TRIGGERS` estuviera activado.

Versión basada en conjuntos:

```sql
CREATE OR ALTER TRIGGER dbo.trProducto_SinStock ON dbo.Producto AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT UPDATE(Stock) RETURN;

    UPDATE p
    SET p.Activo = 0
    FROM dbo.Producto AS p
    JOIN inserted AS i ON i.ProductoId = p.ProductoId
    WHERE i.Stock = 0 AND p.Activo = 1;
END;
```

Y una pregunta de diseño: ¿desactivar un producto sin stock es una regla de la base, o una decisión del negocio que debería tomar la aplicación (que quizá quiere mostrarlo como "agotado")? Muchas veces, la respuesta correcta es no tener este *trigger*.

</details>

-----

## Siguiente lección

[Tablas temporales y variables de tabla](05-Tablas%20temporales%20y%20variables%20de%20tabla.md)
