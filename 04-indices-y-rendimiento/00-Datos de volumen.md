# Datos de volumen

## En una frase

Los índices y los planes de ejecución solo se entienden con **volumen**: con 5 filas cualquier consulta es instantánea, así que este módulo usa `TiendaGrande`, una copia del esquema de la tienda con 100.000 clientes, 1 millón de pedidos y unos 2,5 millones de líneas, generada con un script reproducible.

-----

## Antes de empezar

Conviene que ya sepas:

* El esquema de la tienda, del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md).
* `INSERT ... SELECT`, `ROW_NUMBER` y `CROSS JOIN`, del módulo de [Consultas](../03-consultas/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Tabla de números (tally table):** tabla con los enteros 1, 2, 3... que sirve para generar filas sin bucles.
* **Datos sintéticos:** datos generados por un script, con distribuciones controladas.
* **Distribución sesgada (skew):** cuando unos valores aparecen muchísimo más que otros (un cliente con 50.000 pedidos y otros con 9).
* **Modelo de recuperación SIMPLE:** configuración de la base que no conserva el log después de cada punto de control; útil para entornos de prueba.

-----

## El problema

En la base `Tienda`, esta consulta tarda 0 ms con o sin índice:

```sql
SELECT * FROM dbo.Pedido WHERE ClienteId = 1;
```

El plan de ejecución muestra un *scan* de 5 filas. Lo mismo da crear un índice que no crearlo: el motor lee una página y termina. Todo lo que este módulo quiere mostrar (la diferencia entre leer 3 páginas o 5.000, un *Key Lookup* que se repite 50.000 veces, un plan que es bueno para un cliente y malo para otro) **no existe** con 5 filas.

Hacen falta datos de volumen, con una forma realista:

* Suficientes filas como para que un *scan* cueste miles de páginas.
* Distribuciones **desiguales**, como en la vida real: la mayoría de los pedidos están entregados, pocos están pendientes, un cliente mayorista concentra miles de pedidos.
* **Reproducibles**: el mismo script genera los mismos datos, para poder comparar antes y después de cada cambio.

-----

## Cómo funciona

### Una base aparte

El script crea `TiendaGrande`, una base distinta de `Tienda`. Así los datos de referencia de los módulos anteriores (Ana, Sofía, el pedido cancelado de María) siguen intactos.

`TiendaGrande` tiene las mismas tablas y restricciones que `Tienda`, con una diferencia **a propósito**: **no** tiene los índices sobre las claves foráneas (`IX_Pedido_ClienteId`, `IX_DetallePedido_ProductoId`). Las lecciones los crean, y ver la diferencia es parte del aprendizaje.

### La tabla de números

Para generar un millón de filas sin un bucle `WHILE` (que tardaría muchísimo, como se ve en [Cursores](../06-programacion-t-sql/06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md)), se usa una tabla con los números del 1 al N. Se genera con un `CROSS JOIN` de una lista de 10 valores consigo misma: 10 × 10 × 10 × 10 × 10 × 10 × 10 = 10 millones de combinaciones posibles, de las que se toman las necesarias con `TOP`.

### Valores pseudoaleatorios pero reproducibles

En lugar de `NEWID()` o `RAND()` (que dan valores distintos en cada ejecución), el script usa `CHECKSUM` del número de fila. `CHECKSUM(n * 7919)` siempre devuelve lo mismo para el mismo `n`, pero los valores parecen aleatorios entre una fila y la siguiente. `& 2147483647` quita el signo sin el riesgo de desbordamiento que tiene `ABS` con el entero más negativo.

### Las distribuciones

| Dato | Distribución | Para qué sirve |
| --- | --- | --- |
| Ciudad del cliente | 60 % Lima, 10 % Arequipa, 10 % Cusco, 10 % Trujillo, 10 % `NULL` | Selectividad baja (Lima) frente a alta |
| Pedidos por cliente | El cliente 1 tiene ~50.000 pedidos; los demás, ~9 de media | *Parameter sniffing* y planes que cambian según el valor |
| Estado del pedido | 85 % Entregado, 5 % Enviado, 4 % Pagado, 1 % Pendiente, 5 % Cancelado | Índices filtrados y estadísticas |
| Fecha del pedido | Uniforme entre 2023-01-01 y 2026-06-30 | Rangos de fechas y consultas SARGables |
| Líneas por pedido | Entre 1 y 4 | Joins y multiplicación de filas |
| Productos | 5.000, en 10 categorías, el 5 % inactivos | Joins y agregaciones |

-----

## Ejemplo completo

El script completo. En una máquina de desarrollo típica tarda entre uno y cinco minutos, y ocupa alrededor de 300 MB.

```sql
CREATE DATABASE TiendaGrande;
GO
ALTER DATABASE TiendaGrande SET RECOVERY SIMPLE;   -- sin crecimiento del log en pruebas
GO
USE TiendaGrande;
GO

------------------------------------------------------------------------
-- 1. Tablas: mismo esquema que Tienda, SIN los índices de las FK
------------------------------------------------------------------------
CREATE TABLE dbo.Cliente (
    ClienteId     INT IDENTITY(1,1) NOT NULL,
    Nombre        NVARCHAR(100)     NOT NULL,
    Email         NVARCHAR(254)     NOT NULL,
    Ciudad        NVARCHAR(80)      NULL,
    FechaRegistro DATETIME2(0)      NOT NULL CONSTRAINT DF_Cliente_FechaRegistro DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Cliente PRIMARY KEY (ClienteId),
    CONSTRAINT UQ_Cliente_Email UNIQUE (Email)
);

CREATE TABLE dbo.Categoria (
    CategoriaId INT IDENTITY(1,1) NOT NULL,
    Nombre      NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_Categoria PRIMARY KEY (CategoriaId),
    CONSTRAINT UQ_Categoria_Nombre UNIQUE (Nombre)
);

CREATE TABLE dbo.Producto (
    ProductoId  INT IDENTITY(1,1) NOT NULL,
    CategoriaId INT               NOT NULL,
    Nombre      NVARCHAR(100)     NOT NULL,
    Precio      DECIMAL(10,2)     NOT NULL,
    Stock       INT               NOT NULL CONSTRAINT DF_Producto_Stock DEFAULT 0,
    Activo      BIT               NOT NULL CONSTRAINT DF_Producto_Activo DEFAULT 1,
    CONSTRAINT PK_Producto PRIMARY KEY (ProductoId),
    CONSTRAINT FK_Producto_Categoria FOREIGN KEY (CategoriaId) REFERENCES dbo.Categoria (CategoriaId),
    CONSTRAINT UQ_Producto_Categoria_Nombre UNIQUE (CategoriaId, Nombre),
    CONSTRAINT CK_Producto_Precio CHECK (Precio >= 0),
    CONSTRAINT CK_Producto_Stock CHECK (Stock >= 0)
);

CREATE TABLE dbo.Pedido (
    PedidoId    INT IDENTITY(1,1) NOT NULL,
    ClienteId   INT               NOT NULL,
    FechaPedido DATETIME2(0)      NOT NULL CONSTRAINT DF_Pedido_FechaPedido DEFAULT SYSUTCDATETIME(),
    Estado      VARCHAR(20)       NOT NULL CONSTRAINT DF_Pedido_Estado DEFAULT 'Pendiente',
    CONSTRAINT PK_Pedido PRIMARY KEY (PedidoId),
    CONSTRAINT FK_Pedido_Cliente FOREIGN KEY (ClienteId) REFERENCES dbo.Cliente (ClienteId),
    CONSTRAINT CK_Pedido_Estado CHECK (Estado IN ('Pendiente', 'Pagado', 'Enviado', 'Entregado', 'Cancelado'))
);

CREATE TABLE dbo.DetallePedido (
    PedidoId       INT           NOT NULL,
    ProductoId     INT           NOT NULL,
    Cantidad       INT           NOT NULL,
    PrecioUnitario DECIMAL(10,2) NOT NULL,
    CONSTRAINT PK_DetallePedido PRIMARY KEY (PedidoId, ProductoId),
    CONSTRAINT FK_DetallePedido_Pedido FOREIGN KEY (PedidoId) REFERENCES dbo.Pedido (PedidoId) ON DELETE CASCADE,
    CONSTRAINT FK_DetallePedido_Producto FOREIGN KEY (ProductoId) REFERENCES dbo.Producto (ProductoId),
    CONSTRAINT CK_DetallePedido_Cantidad CHECK (Cantidad > 0),
    CONSTRAINT CK_DetallePedido_PrecioUnitario CHECK (PrecioUnitario >= 0)
);
GO

------------------------------------------------------------------------
-- 2. Tabla de números del 1 al 3.000.000
------------------------------------------------------------------------
CREATE TABLE dbo.Numero (n INT NOT NULL PRIMARY KEY);

WITH Diez AS (
    SELECT x FROM (VALUES (1), (1), (1), (1), (1), (1), (1), (1), (1), (1)) AS t (x)
)
INSERT INTO dbo.Numero (n)
SELECT TOP (3000000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL))
FROM Diez AS a CROSS JOIN Diez AS b CROSS JOIN Diez AS c CROSS JOIN Diez AS d
     CROSS JOIN Diez AS e CROSS JOIN Diez AS f CROSS JOIN Diez AS g;
GO

------------------------------------------------------------------------
-- 3. Categorías (10)
------------------------------------------------------------------------
INSERT INTO dbo.Categoria (Nombre) VALUES
    (N'Electrónica'), (N'Hogar'), (N'Libros'), (N'Deportes'), (N'Juguetes'),
    (N'Jardín'), (N'Oficina'), (N'Mascotas'), (N'Belleza'), (N'Alimentos');

------------------------------------------------------------------------
-- 4. Productos (5.000)
------------------------------------------------------------------------
INSERT INTO dbo.Producto (CategoriaId, Nombre, Precio, Stock, Activo)
SELECT 1 + n % 10,
       CONCAT(N'Producto ', n),
       CAST(5 + (CHECKSUM(n * 7919) & 2147483647) % 2000
              + ((CHECKSUM(n * 104729) & 2147483647) % 100) / 100.0 AS DECIMAL(10,2)),
       (CHECKSUM(n * 31) & 2147483647) % 500,
       CASE WHEN n % 20 = 0 THEN 0 ELSE 1 END
FROM dbo.Numero
WHERE n <= 5000;

------------------------------------------------------------------------
-- 5. Clientes (100.000)
------------------------------------------------------------------------
INSERT INTO dbo.Cliente (Nombre, Email, Ciudad, FechaRegistro)
SELECT CONCAT(N'Cliente ', n),
       CONCAT(N'cliente', n, N'@correo.com'),
       CASE n % 10
           WHEN 0 THEN NULL
           WHEN 7 THEN N'Arequipa'
           WHEN 8 THEN N'Cusco'
           WHEN 9 THEN N'Trujillo'
           ELSE N'Lima'                       -- 1 a 6: el 60 %
       END,
       DATEADD(day, n % 1460, CAST('2022-01-01' AS DATETIME2(0)))
FROM dbo.Numero
WHERE n <= 100000;

------------------------------------------------------------------------
-- 6. Pedidos (1.000.000), con el cliente 1 como "mayorista"
------------------------------------------------------------------------
-- CAST a BIGINT: n * 7907 no cabe en un INT cuando n supera ~271.000
INSERT INTO dbo.Pedido (ClienteId, FechaPedido, Estado)
SELECT CASE WHEN n % 20 = 0 THEN 1                                                -- 5 %: cliente 1
            ELSE 2 + (CHECKSUM(CAST(n AS BIGINT) * 7907) & 2147483647) % 99999 END, -- resto: 2 a 100.000
       DATEADD(minute,
               (CHECKSUM(CAST(n AS BIGINT) * 6151) & 2147483647) % (1277 * 1440), -- ~3,5 años en minutos
               CAST('2023-01-01' AS DATETIME2(0))),
       CASE
           WHEN (CHECKSUM(CAST(n AS BIGINT) * 3571) & 2147483647) % 100 < 85 THEN 'Entregado'
           WHEN (CHECKSUM(CAST(n AS BIGINT) * 3571) & 2147483647) % 100 < 90 THEN 'Enviado'
           WHEN (CHECKSUM(CAST(n AS BIGINT) * 3571) & 2147483647) % 100 < 94 THEN 'Pagado'
           WHEN (CHECKSUM(CAST(n AS BIGINT) * 3571) & 2147483647) % 100 < 95 THEN 'Pendiente'
           ELSE 'Cancelado'
       END
FROM dbo.Numero
WHERE n <= 1000000;

------------------------------------------------------------------------
-- 7. Detalles (entre 1 y 4 líneas por pedido, ~2.500.000)
------------------------------------------------------------------------
INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
SELECT p.PedidoId,
       pr.ProductoId,
       1 + (p.PedidoId + k.n) % 3,
       pr.Precio
FROM dbo.Pedido AS p
JOIN dbo.Numero AS k ON k.n <= 1 + p.PedidoId % 4
JOIN dbo.Producto AS pr ON pr.ProductoId = 1 + (p.PedidoId * 7 + k.n * 997) % 5000;
GO

------------------------------------------------------------------------
-- 8. Verificación
------------------------------------------------------------------------
SELECT 'Cliente' AS Tabla, COUNT(*) AS Filas FROM dbo.Cliente
UNION ALL SELECT 'Producto', COUNT(*) FROM dbo.Producto
UNION ALL SELECT 'Pedido', COUNT(*) FROM dbo.Pedido
UNION ALL SELECT 'DetallePedido', COUNT(*) FROM dbo.DetallePedido;

SELECT Estado, COUNT(*) AS Pedidos
FROM dbo.Pedido
GROUP BY Estado
ORDER BY Pedidos DESC;

SELECT TOP (3) ClienteId, COUNT(*) AS Pedidos
FROM dbo.Pedido
GROUP BY ClienteId
ORDER BY Pedidos DESC;
```

Resultado esperado de la verificación (las cantidades por estado y por cliente son aproximadas, porque dependen de los valores pseudoaleatorios):

| Tabla | Filas |
| --- | --- |
| Cliente | 100000 |
| Producto | 5000 |
| Pedido | 1000000 |
| DetallePedido | 2500000 |

| Estado | Pedidos |
| --- | --- |
| Entregado | ~850.000 |
| Enviado | ~50.000 |
| Cancelado | ~50.000 |
| Pagado | ~40.000 |
| Pendiente | ~10.000 |

| ClienteId | Pedidos |
| --- | --- |
| 1 | 50000 |
| (otro cliente) | ~20 a 25 |
| (otro cliente) | ~20 a 25 |

El resto de los clientes tiene unos 9,5 pedidos de media; los que siguen al cliente 1 en el ranking son simplemente los que, por azar, recibieron más.

Qué decisión hay detrás de cada parte:

* **`RECOVERY SIMPLE`:** con el modelo por defecto (`FULL`), cargar millones de filas haría crecer el log hasta que se haga una copia de seguridad del log. En una base de prueba no hace falta.
* **La línea de detalle con `1 + p.PedidoId % 4`:** cada pedido tiene entre 1 y 4 líneas, en total 2,5 millones. Los productos de un mismo pedido nunca se repiten (los desplazamientos `k.n * 997` son distintos para k = 1 a 4), así que la clave primaria `(PedidoId, ProductoId)` se respeta.
* **El cliente 1 con exactamente 50.000 pedidos** (todos los `n` múltiplos de 20): es el caso extremo que se usa para mostrar que un mismo plan no sirve para todos los valores.
* **Las fechas de pedido no respetan la fecha de registro del cliente:** es una simplificación aceptable para estudiar rendimiento, no para estudiar reglas de negocio.

-----

## Errores comunes

**1. Generar los datos con un bucle `WHILE`.**
Qué pasa: un millón de `INSERT` de una fila tardan decenas de minutos.
Por qué: cada instrucción tiene su propio costo de compilación, bloqueo y escritura en el log.
Arreglo: una sola instrucción `INSERT ... SELECT` sobre una tabla de números.

**2. Cargar en la base `Tienda`.**
Qué pasa: los ejemplos de los módulos anteriores dejan de dar los resultados documentados.
Por qué: Sofía deja de ser "la única clienta sin pedidos".
Arreglo: una base aparte para el volumen.

**3. Multiplicar enteros grandes sin convertir.**
Qué pasa: `Arithmetic overflow error converting expression to data type int` a mitad de la carga.
Por qué: `n * 7907` es una multiplicación de dos `INT`, y el resultado también es `INT`: con n = 300.000 ya no cabe.
Arreglo: `CAST(n AS BIGINT) * 7907`.

**4. Usar `ABS(CHECKSUM(...))`.**
Qué pasa: de vez en cuando, `Arithmetic overflow error converting expression to data type int`.
Por qué: `CHECKSUM` puede devolver -2.147.483.648, y su valor absoluto no cabe en un `INT`.
Arreglo: `CHECKSUM(...) & 2147483647`.

**5. Comparar tiempos con la caché fría y la caché caliente mezcladas.**
Qué pasa: la primera ejecución tarda 3 segundos y la segunda 200 ms, sin cambiar nada.
Por qué: la primera lee del disco (*physical reads*); la segunda, de la memoria (*logical reads*).
Arreglo: compara **lecturas lógicas** (`SET STATISTICS IO ON`), que no dependen de la caché, más que tiempos. Se explica en [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Serie de números | Tabla de números o `GENERATE_SERIES` (desde 2022) | `generate_series(1, 1000000)` | CTE recursiva (con `cte_max_recursion_depth` alto) o tabla de números |
| Valores pseudoaleatorios reproducibles | `CHECKSUM(n * k)` | `hashtext(n::text)` o `setseed()` + `random()` | `CRC32(n)` o `RAND(semilla)` |
| Minimizar el log en cargas | `RECOVERY SIMPLE` | `UNLOGGED TABLE` (sin log, se pierde ante un fallo) | `innodb_flush_log_at_trx_commit = 2` (solo pruebas) |

### En MongoDB

Para generar volumen se usa `insertMany` con lotes desde `mongosh` (un bucle de JavaScript que arma arreglos de miles de documentos y los inserta juntos), o herramientas como `mongoimport` con un archivo generado. La misma regla vale: lotes grandes, no un documento por llamada.

-----

## Cuándo sí y cuándo no

**Usa datos de volumen cuando:**

* Pruebas índices, planes de ejecución o el rendimiento de una consulta: los resultados con pocas filas no dicen nada.

**Ten en cuenta:**

* Los datos sintéticos no reemplazan a los reales: las distribuciones de producción tienen sorpresas que ningún script imita. Para pruebas serias, una copia anonimizada de producción es mejor.
* Las lecturas lógicas que veas van a diferir un poco de las de este wiki, según la versión de SQL Server y la configuración. Lo que importa es el **orden de magnitud** y la diferencia entre antes y después.

-----

## Resumen en 5 líneas

1. Con pocas filas, cualquier consulta es instantánea: los índices solo se estudian con volumen.
2. `TiendaGrande` tiene el mismo esquema que `Tienda`, sin los índices de las claves foráneas, y 1 millón de pedidos.
3. Se genera con una tabla de números y `INSERT ... SELECT`, nunca con bucles.
4. `CHECKSUM(n * k) & 2147483647` da valores pseudoaleatorios reproducibles y sin desbordamiento.
5. Las distribuciones son desiguales a propósito: un cliente con 50.000 pedidos, un 1 % de pedidos pendientes.

-----

## Para profundizar

<details>
<summary>Por qué el CROSS JOIN de una lista de 10 genera millones</summary>

Un `CROSS JOIN` multiplica: 10 filas por 10 filas dan 100; por otras 10, mil. Siete listas de 10 dan 10⁷ = 10.000.000 de combinaciones. `ROW_NUMBER() OVER (ORDER BY (SELECT NULL))` las numera sin ordenarlas (el `SELECT NULL` le dice al motor que el orden no importa) y `TOP (3000000)` corta en la cantidad necesaria, sin generar las demás: el motor deja de producir filas cuando alcanza el `TOP`. La técnica es de Itzik Ben-Gan, y es la base de muchos trucos de T-SQL basados en conjuntos.

</details>

<details>
<summary>Limpiar la caché para medir en frío</summary>

En un entorno de **pruebas** (nunca en producción), se puede vaciar la caché de datos para medir como si fuera la primera ejecución:

```sql
CHECKPOINT;                 -- escribe a disco las páginas modificadas
DBCC DROPCLEANBUFFERS;      -- vacía la caché de datos (buffer pool)
```

En producción esto haría que todas las consultas de todas las bases tuvieran que volver a leer del disco. Para comparar dos versiones de una consulta, las lecturas lógicas suelen ser suficientes y no requieren tocar la caché.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para probar el rendimiento hace falta una base con muchos datos, porque con pocas filas todas las consultas son rápidas. Genero los datos con un script usando una tabla de números e `INSERT ... SELECT`, en lugar de insertar fila por fila.

### Respuesta ampliada (semi-senior)

Pruebo rendimiento sobre volúmenes realistas y con distribuciones parecidas a las de producción, incluidos los sesgos (un cliente que concentra pedidos, estados muy desiguales), porque son los que provocan problemas como el *parameter sniffing* o las malas estimaciones. Genero los datos con técnicas basadas en conjuntos (tabla de números y `INSERT ... SELECT`), con valores pseudoaleatorios reproducibles para poder comparar antes y después de cada cambio, en una base separada y con el modelo de recuperación `SIMPLE`. Comparo lecturas lógicas más que tiempos, porque no dependen del estado de la caché. Y cuando es posible, uso una copia anonimizada de producción, que tiene distribuciones que ningún script reproduce.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no un bucle `WHILE` para insertar un millón de filas?**
Porque cada `INSERT` de una fila tiene su propio costo fijo; una sola instrucción basada en conjuntos lo paga una vez.

**2. ¿Por qué importan las distribuciones desiguales?**
Porque el optimizador elige planes según las estadísticas; con datos uniformes no aparecen los problemas que causan los valores atípicos.

**3. ¿Qué es una tabla de números?**
Una tabla con los enteros consecutivos, que permite generar filas, series y combinaciones con consultas basadas en conjuntos.

-----

## Práctica

**Ejercicio 1.** Después de ejecutar el script, escribe una consulta que muestre cuántos clientes hay por ciudad (incluidos los `NULL`) y el porcentaje de cada una sobre el total.

<details>
<summary>Solución</summary>

```sql
USE TiendaGrande;

SELECT COALESCE(Ciudad, N'(Sin ciudad)')                          AS Ciudad,
       COUNT(*)                                                   AS Clientes,
       CAST(100.0 * COUNT(*) / SUM(COUNT(*)) OVER () AS DECIMAL(5,1)) AS Porcentaje
FROM dbo.Cliente
GROUP BY Ciudad
ORDER BY Clientes DESC;
```

| Ciudad | Clientes | Porcentaje |
| --- | --- | --- |
| Lima | 60000 | 60.0 |
| Arequipa | 10000 | 10.0 |
| Cusco | 10000 | 10.0 |
| Trujillo | 10000 | 10.0 |
| (Sin ciudad) | 10000 | 10.0 |

`SUM(COUNT(*)) OVER ()` es una función de ventana sobre el resultado agrupado: suma los conteos de todos los grupos. Las cifras son exactas porque la ciudad depende de `n % 10`, no de un valor pseudoaleatorio.

</details>

**Ejercicio 2.** Escribe una consulta que verifique que ningún pedido tiene más de 4 líneas ni menos de 1, y que muestre cuántos pedidos hay con cada cantidad de líneas.

<details>
<summary>Solución</summary>

```sql
SELECT Lineas, COUNT(*) AS Pedidos
FROM (SELECT PedidoId, COUNT(*) AS Lineas
      FROM dbo.DetallePedido
      GROUP BY PedidoId) AS t
GROUP BY Lineas
ORDER BY Lineas;
```

| Lineas | Pedidos |
| --- | --- |
| 1 | 250000 |
| 2 | 250000 |
| 3 | 250000 |
| 4 | 250000 |

La cantidad de líneas depende de `PedidoId % 4`, así que cada valor aparece exactamente en un cuarto de los pedidos. Total de líneas: 250.000 × (1 + 2 + 3 + 4) = 2.500.000.

</details>

-----

## Siguiente lección

[Cómo guarda los datos SQL Server](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md)
