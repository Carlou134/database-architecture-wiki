# Procedimientos almacenados

## En una frase

Un procedimiento almacenado es un **programa con nombre** guardado en la base, que recibe parámetros, ejecuta varias instrucciones (con transacciones, validaciones y manejo de errores) y devuelve resultados; encapsula una operación de negocio completa detrás de una interfaz estable, controla los permisos y reutiliza su plan de ejecución.

-----

## Antes de empezar

Conviene que ya sepas:

* Transacciones y manejo de errores, de [Transacciones y ACID](../05-transacciones-y-concurrencia/01-Transacciones%20y%20ACID.md) y [Manejo de errores](../05-transacciones-y-concurrencia/06-Manejo%20de%20errores.md).
* La caché de planes y el *parameter sniffing*, de [Planes de ejecución II](../04-indices-y-rendimiento/05-Planes%20de%20ejecución%20II.md).
* El encadenamiento de propiedad, de [Vistas](01-Vistas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Procedimiento almacenado (stored procedure):** conjunto de instrucciones T-SQL guardado con nombre, que se ejecuta con `EXEC`.
* **Parámetro de entrada / de salida (`OUTPUT`):** valor que recibe el procedimiento / que el procedimiento devuelve en una variable del llamador.
* **Código de retorno:** número entero que el procedimiento devuelve con `RETURN`, por convención 0 si terminó bien.
* **Conjunto de resultados (result set):** las filas que devuelve un `SELECT` dentro del procedimiento.
* **`SET NOCOUNT ON`:** desactiva los mensajes "(n rows affected)" de cada instrucción.
* **`EXECUTE AS`:** cláusula que hace que el procedimiento se ejecute con los permisos de otro usuario.

-----

## El problema

La aplicación registra un pedido enviando, una por una, siete instrucciones SQL: buscar el precio, validar el stock, insertar el pedido, leer el ID generado, insertar cada línea, descontar el stock y confirmar. Eso significa:

* **Siete viajes de red** por pedido, cada uno con su latencia.
* **La transacción abierta** mientras viajan los mensajes entre la aplicación y la base, con los bloqueos que eso implica.
* **La lógica repetida** en la aplicación web, en la app móvil y en el proceso de importación, cada una con sus propias variantes (y sus propios errores).
* **Permisos amplios:** el usuario de la aplicación necesita `INSERT`, `UPDATE` y `SELECT` sobre cuatro tablas, aunque solo debería poder "registrar pedidos".

Un procedimiento almacenado convierte esas siete instrucciones en una sola llamada, con la lógica en un solo lugar y un permiso concreto: `EXECUTE` sobre `dbo.RegistrarPedido`.

-----

## Cómo funciona

### Crear y ejecutar

```sql
CREATE OR ALTER PROCEDURE dbo.ObtenerPedidosCliente
    @ClienteId INT,
    @Estado    VARCHAR(20) = NULL          -- parámetro opcional, con valor por defecto
AS
BEGIN
    SET NOCOUNT ON;

    SELECT PedidoId, FechaPedido, Estado
    FROM dbo.Pedido
    WHERE ClienteId = @ClienteId
      AND (@Estado IS NULL OR Estado = @Estado)
    ORDER BY FechaPedido DESC
    OPTION (RECOMPILE);   -- filtro opcional: ver "consultas para todo"
END;
GO

EXEC dbo.ObtenerPedidosCliente @ClienteId = 1;
EXEC dbo.ObtenerPedidosCliente @ClienteId = 1, @Estado = 'Pagado';
```

| PedidoId | FechaPedido | Estado |
| --- | --- | --- |
| 3 | 2026-04-12 09:05 | Pagado |
| 1 | 2026-03-01 10:15 | Entregado |

* **Parámetros con nombre** (`@ClienteId = 1`) al llamar: no dependen del orden y siguen funcionando si se agrega un parámetro nuevo.
* **Valores por defecto** (`= NULL`): hacen opcionales los parámetros.
* **`SET NOCOUNT ON`** al principio: evita que cada instrucción envíe un mensaje "(n rows affected)", que algunas aplicaciones interpretan como un conjunto de resultados y que agrega tráfico.
* **No uses el prefijo `sp_`:** SQL Server busca primero los `sp_` en la base `master` (ver [Convenciones de nombres](../02-diseno-y-modelado/10-Convenciones%20de%20nombres.md)).

### Tres formas de devolver información

| Mecanismo | Para qué | Ejemplo |
| --- | --- | --- |
| **Conjuntos de resultados** (`SELECT`) | Filas: listados, detalles | Los pedidos del cliente |
| **Parámetros `OUTPUT`** | Valores escalares: un ID generado, un total, un estado | El `PedidoId` recién creado |
| **Código de retorno** (`RETURN n`) | Un entero: éxito (0) o un código de resultado | Poco usado hoy; los errores se comunican con `THROW` |

```sql
CREATE OR ALTER PROCEDURE dbo.ContarPedidos
    @ClienteId INT,
    @Cantidad  INT OUTPUT,
    @Total     DECIMAL(12,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT @Cantidad = COUNT(DISTINCT p.PedidoId),
           @Total    = COALESCE(SUM(d.Cantidad * d.PrecioUnitario), 0)
    FROM dbo.Pedido AS p
    LEFT JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.ClienteId = @ClienteId;
END;
GO

DECLARE @n INT, @t DECIMAL(12,2);
EXEC dbo.ContarPedidos @ClienteId = 1, @Cantidad = @n OUTPUT, @Total = @t OUTPUT;
SELECT @n AS Pedidos, @t AS Total;
```

| Pedidos | Total |
| --- | --- |
| 2 | 476.00 |

`OUTPUT` tiene que escribirse **en los dos lados**: en la declaración del parámetro y en la llamada. Si se olvida en la llamada, el procedimiento funciona pero la variable del llamador no recibe nada.

### Qué ganas con un procedimiento

| Beneficio | Por qué |
| --- | --- |
| **Menos viajes de red** | Una llamada en lugar de muchas instrucciones |
| **Transacciones más cortas** | La transacción empieza y termina dentro del servidor, sin esperar a la red |
| **Lógica en un solo lugar** | Todas las aplicaciones llaman al mismo procedimiento |
| **Seguridad** | Permiso `EXECUTE` sobre el procedimiento, sin permisos sobre las tablas (encadenamiento de propiedad) |
| **Protección contra la inyección de SQL** | Los parámetros son valores, nunca código (salvo que el procedimiento arme SQL dinámico concatenando: ver [SQL dinámico seguro](07-SQL%20dinámico%20seguro.md)) |
| **Plan reutilizado** | El plan se compila una vez y queda en caché (con el riesgo del *parameter sniffing*) |

Sobre el último punto: las consultas **parametrizadas** que envía una aplicación (por ejemplo, desde EF Core o Dapper) también reutilizan su plan. El procedimiento no tiene ninguna ventaja mágica de rendimiento frente a una consulta parametrizada bien escrita; sus ventajas reales son las de la tabla: encapsulamiento, transacciones cortas, seguridad y menos viajes.

### Seguridad y `EXECUTE AS`

```sql
GRANT EXECUTE ON dbo.RegistrarPedido TO RolAplicacion;
-- RolAplicacion no tiene INSERT ni UPDATE sobre las tablas
```

El encadenamiento de propiedad funciona igual que con las vistas: si el procedimiento y las tablas tienen el mismo dueño, basta el permiso de ejecución. Se rompe con el SQL dinámico (que se verifica con los permisos del llamador) y con objetos de otro dueño o de otra base; para esos casos, `WITH EXECUTE AS OWNER` (o un usuario concreto) hace que el procedimiento se ejecute con los permisos de ese usuario. Es poderoso y hay que usarlo con cuidado: dentro del procedimiento, cualquier cosa se ejecuta con esos permisos.

### Lógica de negocio: ¿en la base o en la aplicación?

Es uno de los debates clásicos de la arquitectura de software, y no tiene una respuesta única:

| En procedimientos | En la aplicación |
| --- | --- |
| Operaciones con muchos datos (cálculos sobre millones de filas, procesos por lotes) | Reglas de negocio complejas que cambian a menudo |
| Operaciones que tienen que ser atómicas y cortas (registrar un pedido) | Lógica que se prueba mejor con pruebas unitarias |
| Sistemas con muchas aplicaciones distintas sobre la misma base | Lógica que combina la base con servicios externos (pagos, correos) |
| Seguridad estricta: las aplicaciones solo ejecutan procedimientos | Equipos que trabajan principalmente en C#, Kotlin o Java |

Una posición equilibrada, frecuente en equipos maduros: la **integridad** (restricciones, claves) siempre en la base; las **operaciones atómicas sobre datos** en procedimientos o en consultas bien escritas; y la **orquestación del negocio** (qué hacer cuando un pedido se paga: facturar, notificar, reservar el envío) en la aplicación, donde se prueba y se versiona junto con el resto del código.

### Buenas prácticas

* `SET NOCOUNT ON` y `SET XACT_ABORT ON` al principio.
* Validaciones antes de la transacción; transacción corta; `TRY...CATCH` con `THROW` (ver [Manejo de errores](../05-transacciones-y-concurrencia/06-Manejo%20de%20errores.md)).
* Parámetros con el **mismo tipo** que las columnas que filtran (para evitar conversiones implícitas).
* Columnas explícitas en los `SELECT` que devuelve: los resultados son parte del contrato con la aplicación.
* Los procedimientos versionados en el repositorio, como el resto del código, y desplegados con migraciones.

-----

## Ejemplo completo

Una página de pedidos para la API, con paginación, total de filas y validación:

```sql
CREATE OR ALTER PROCEDURE dbo.ListarPedidos
    @ClienteId    INT         = NULL,
    @Estado       VARCHAR(20) = NULL,
    @Pagina       INT         = 1,
    @TamanoPagina INT         = 20,
    @TotalFilas   INT         OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    -- Validaciones: errores de negocio claros, antes de consultar
    IF @Pagina < 1 OR @TamanoPagina NOT BETWEEN 1 AND 100
        THROW 50201, N'Página o tamaño de página inválidos (tamaño entre 1 y 100).', 1;

    -- Total de filas para la paginación
    SELECT @TotalFilas = COUNT(*)
    FROM dbo.Pedido
    WHERE (@ClienteId IS NULL OR ClienteId = @ClienteId)
      AND (@Estado    IS NULL OR Estado    = @Estado)
    OPTION (RECOMPILE);

    -- La página
    SELECT p.PedidoId, p.FechaPedido, p.Estado, c.Nombre AS Cliente
    FROM dbo.Pedido AS p
    JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
    WHERE (@ClienteId IS NULL OR p.ClienteId = @ClienteId)
      AND (@Estado    IS NULL OR p.Estado    = @Estado)
    ORDER BY p.FechaPedido DESC, p.PedidoId DESC
    OFFSET (@Pagina - 1) * @TamanoPagina ROWS
    FETCH NEXT @TamanoPagina ROWS ONLY
    OPTION (RECOMPILE);
END;
GO

DECLARE @Total INT;
EXEC dbo.ListarPedidos @Pagina = 1, @TamanoPagina = 2, @TotalFilas = @Total OUTPUT;
SELECT @Total AS TotalFilas;
```

| PedidoId | FechaPedido | Estado | Cliente |
| --- | --- | --- | --- |
| 5 | 2026-05-02 11:00 | Pendiente | Jorge Díaz |
| 4 | 2026-04-20 18:30 | Cancelado | María Quispe |

| TotalFilas |
| --- |
| 5 |

Y el permiso para la aplicación:

```sql
GRANT EXECUTE ON dbo.ListarPedidos TO RolAplicacion;
```

Qué decisión hay detrás:

* **Validaciones con `THROW` y número propio (50201):** la aplicación puede traducir el número a un mensaje para el usuario.
* **Total y página en dos consultas,** con el total como `OUTPUT`: la aplicación recibe un único conjunto de resultados con las filas, y el total aparte. (Con `COUNT(*) OVER ()` iría en la misma consulta, como columna, a costa de calcularlo en cada fila de la página.)
* **`OPTION (RECOMPILE)`:** los filtros son opcionales, y sin recompilar, el plan se compilaría para la primera combinación de parámetros (ver [Consultas SARGables y antipatrones](../04-indices-y-rendimiento/06-Consultas%20SARGables%20y%20antipatrones.md)).
* **Orden determinista** con `PedidoId` como desempate, para que la paginación no repita ni pierda filas.
* **Un permiso `EXECUTE`:** la aplicación no necesita `SELECT` sobre `Pedido` ni `Cliente`.

-----

## Errores comunes

**1. Prefijo `sp_`.**
Qué pasa: una búsqueda extra en `master` en cada llamada, y el riesgo de que un procedimiento del sistema con el mismo nombre tenga prioridad.
Por qué: `sp_` está reservado para los procedimientos del sistema.
Arreglo: sin prefijo, o con un verbo (`dbo.RegistrarPedido`).

**2. Olvidar `OUTPUT` en la llamada.**
Qué pasa: la variable del llamador queda en `NULL`, sin ningún error.
Por qué: `OUTPUT` hay que escribirlo en la declaración y en la llamada.
Arreglo: `EXEC ... @Total = @t OUTPUT`.

**3. Procedimientos "para todo" sin `RECOMPILE`.**
Qué pasa: rendimiento errático según qué parámetros se usaron primero.
Por qué: un solo plan para todas las combinaciones de filtros opcionales.
Arreglo: `OPTION (RECOMPILE)` o SQL dinámico parametrizado.

**4. Parámetros de un tipo distinto al de la columna.**
Qué pasa: conversiones implícitas y *scans* (por ejemplo, `@Estado NVARCHAR(20)` contra `Estado VARCHAR(20)`).
Por qué: se declaran los parámetros "genéricos".
Arreglo: el mismo tipo y longitud que la columna.

**5. Lógica de presentación en el procedimiento.**
Qué pasa: procedimientos que devuelven textos formateados, HTML o fechas en el formato de un país.
Por qué: se mezcla la capa de datos con la de presentación.
Arreglo: devolver datos tipados; formatear en la aplicación.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Crear | `CREATE OR ALTER PROCEDURE` | `CREATE OR REPLACE PROCEDURE` (desde la 11) o `FUNCTION` | `CREATE PROCEDURE` (sin `OR REPLACE`) |
| Lenguaje | T-SQL | PL/pgSQL (y otros: Python, JavaScript...) | SQL/PSM de MySQL |
| Devolver filas | `SELECT` dentro del procedimiento | Lo habitual es una **función** que devuelve una tabla (`RETURNS TABLE`) | `SELECT` dentro del procedimiento |
| Parámetros de salida | `OUTPUT` | `INOUT` / `OUT` | `OUT` / `INOUT` |
| Transacciones dentro | Sí | Sí en procedimientos (desde la 11); no en funciones | Sí |
| Ejecutar | `EXEC nombre @p = 1` | `CALL nombre(1)` | `CALL nombre(1)` |

En PostgreSQL, la cultura es distinta: durante años solo existieron funciones, y todavía hoy la forma habitual de encapsular una consulta que devuelve filas es una función, no un procedimiento.

### En MongoDB

MongoDB no tiene procedimientos almacenados: la lógica vive en la aplicación (o en funciones de servidor como las de Atlas). Lo más parecido para encapsular consultas complejas son las vistas definidas con un *aggregation pipeline* y las validaciones con `$jsonSchema`.

-----

## Cuándo sí y cuándo no

**Usa un procedimiento cuando:**

* Una operación de negocio necesita varias instrucciones atómicas (registrar un pedido, cerrar una caja).
* Quieres que las aplicaciones solo puedan ejecutar operaciones concretas, sin permisos sobre las tablas.
* El trabajo es sobre muchos datos y conviene hacerlo donde están.

**Considera la aplicación cuando:**

* La lógica cambia a menudo, necesita pruebas unitarias o combina la base con servicios externos.

-----

## Resumen en 5 líneas

1. Un procedimiento encapsula una operación con parámetros, transacciones y manejo de errores, detrás de una sola llamada.
2. Devuelve filas con `SELECT`, valores con parámetros `OUTPUT` (escritos en los dos lados) y errores con `THROW`.
3. `SET NOCOUNT ON` y `XACT_ABORT ON`, parámetros del mismo tipo que las columnas, y nunca el prefijo `sp_`.
4. Con el mismo dueño, basta el permiso `EXECUTE`: las aplicaciones no necesitan permisos sobre las tablas.
5. Su ventaja no es un rendimiento mágico sino encapsulamiento, transacciones cortas, seguridad y menos viajes de red.

-----

## Para profundizar

<details>
<summary>Procedimientos compilados de forma nativa</summary>

Con las tablas optimizadas para memoria (*In-Memory OLTP*, desde SQL Server 2014), un procedimiento puede compilarse a código máquina (`WITH NATIVE_COMPILATION, SCHEMABINDING`) y ejecutarse muchas veces más rápido. Tiene restricciones importantes (solo puede acceder a tablas en memoria y admite un subconjunto de T-SQL) y se usa en casos muy específicos de altísima concurrencia, como sesiones o colas con decenas de miles de operaciones por segundo.

</details>

<details>
<summary>Llamar a un procedimiento desde .NET</summary>

Con ADO.NET, se usa `CommandType.StoredProcedure` y un `SqlParameter` por parámetro, con `Direction = ParameterDirection.Output` para los de salida. Con Dapper, `connection.Query<Pedido>("dbo.ListarPedidos", parametros, commandType: CommandType.StoredProcedure)` y `DynamicParameters` para los `OUTPUT`. Con EF Core, `FromSqlInterpolated($"EXEC dbo.ListarPedidos @Pagina = {pagina} ...")` para los que devuelven filas de una entidad. En todos los casos, los parámetros viajan como valores tipados: nunca concatenes valores en el texto de la llamada.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un procedimiento almacenado es un conjunto de instrucciones SQL guardado en la base que se ejecuta con `EXEC` y puede recibir parámetros. Sirve para no repetir lógica, para reducir las llamadas entre la aplicación y la base, y para dar permisos de ejecución sin dar acceso directo a las tablas.

### Respuesta ampliada (semi-senior)

Uso procedimientos para encapsular operaciones atómicas sobre datos: una llamada en lugar de varias instrucciones, con la transacción entera dentro del servidor (más corta, sin esperar a la red), validaciones antes, `XACT_ABORT ON` y `TRY...CATCH` con `THROW`, y errores de negocio con números propios que la aplicación traduce. Devuelvo filas con `SELECT`, valores con `OUTPUT` y errores con `THROW`, no con códigos de retorno. Declaro los parámetros con el mismo tipo que las columnas y, si tienen filtros opcionales, uso `OPTION (RECOMPILE)` o SQL dinámico parametrizado por el *parameter sniffing*. Para seguridad, doy `EXECUTE` sin permisos sobre las tablas y uso `EXECUTE AS` con cuidado cuando se rompe el encadenamiento. No les atribuyo ventajas mágicas de rendimiento frente a consultas parametrizadas; y en el debate de dónde va la lógica, dejo la integridad en la base, las operaciones atómicas sobre datos en procedimientos o consultas, y la orquestación de negocio en la aplicación, donde se prueba mejor.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar el prefijo `sp_`?**
Porque SQL Server busca esos procedimientos primero en `master`, y puede ejecutar uno del sistema con el mismo nombre.

**2. ¿Un procedimiento es más rápido que una consulta enviada desde la aplicación?**
No necesariamente: una consulta parametrizada también reutiliza su plan. Lo que ahorra el procedimiento son viajes de red y tiempo de transacción.

**3. ¿Cómo devuelves un valor calculado desde un procedimiento?**
Con un parámetro `OUTPUT`, indicado en la declaración y en la llamada.

**4. ¿Cómo das acceso a un procedimiento sin dar acceso a las tablas?**
Con `GRANT EXECUTE`; si el procedimiento y las tablas tienen el mismo dueño, el encadenamiento de propiedad hace el resto.

-----

## Práctica

**Ejercicio 1.** Escribe un procedimiento `dbo.CambiarPrecio` que reciba un producto y un precio nuevo, rechace precios negativos o cambios mayores al 50 % (con errores propios), actualice el precio y devuelva el precio anterior en un parámetro `OUTPUT`.

<details>
<summary>Solución</summary>

```sql
CREATE OR ALTER PROCEDURE dbo.CambiarPrecio
    @ProductoId     INT,
    @PrecioNuevo    DECIMAL(10,2),
    @PrecioAnterior DECIMAL(10,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    IF @PrecioNuevo < 0
        THROW 50211, N'El precio no puede ser negativo.', 1;

    BEGIN TRANSACTION;

        SELECT @PrecioAnterior = Precio
        FROM dbo.Producto WITH (UPDLOCK)
        WHERE ProductoId = @ProductoId;

        IF @PrecioAnterior IS NULL
            THROW 50212, N'El producto no existe.', 1;

        IF ABS(@PrecioNuevo - @PrecioAnterior) > @PrecioAnterior * 0.5
            THROW 50213, N'El cambio de precio supera el 50 %.', 1;

        UPDATE dbo.Producto SET Precio = @PrecioNuevo WHERE ProductoId = @ProductoId;

    COMMIT;
END;
GO

DECLARE @Anterior DECIMAL(10,2);
EXEC dbo.CambiarPrecio @ProductoId = 1, @PrecioNuevo = 130.00, @PrecioAnterior = @Anterior OUTPUT;
SELECT @Anterior AS PrecioAnterior;   -- 120.00
```

`UPDLOCK` en la lectura evita que otra sesión cambie el precio entre la validación y el `UPDATE`. Los `THROW` dentro de la transacción, con `XACT_ABORT ON`, la deshacen automáticamente (en un procedimiento real, iría además el `TRY...CATCH` de [Manejo de errores](../05-transacciones-y-concurrencia/06-Manejo%20de%20errores.md)). Un cambio de 120 a 200 (66 %) devolvería el error 50213.

</details>

**Ejercicio 2.** ¿Qué devuelve esta llamada, y por qué?

```sql
DECLARE @n INT = -1, @t DECIMAL(12,2) = -1;
EXEC dbo.ContarPedidos @ClienteId = 1, @Cantidad = @n, @Total = @t OUTPUT;
SELECT @n AS Pedidos, @t AS Total;
```

<details>
<summary>Solución</summary>

| Pedidos | Total |
| --- | --- |
| -1 | 476.00 |

`@Cantidad` se pasó **sin** `OUTPUT` en la llamada: el procedimiento recibe el valor -1 y calcula 2 en su copia local, pero la variable `@n` del llamador no se actualiza. `@Total` sí lleva `OUTPUT` y recibe 476.00. No hay ningún error: es el tipo de detalle que pasa inadvertido.

</details>

-----

## Siguiente lección

[Funciones](03-Funciones.md)
