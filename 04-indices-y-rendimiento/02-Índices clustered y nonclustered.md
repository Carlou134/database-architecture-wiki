# Índices clustered y nonclustered

## En una frase

El índice **agrupado** (*clustered*) es la tabla misma ordenada por su clave, y solo hay uno; un índice **no agrupado** (*nonclustered*) es una estructura aparte, ordenada por otras columnas, que guarda en cada entrada un "puntero" a la fila completa, y si la consulta necesita columnas que el índice no tiene, cada fila encontrada cuesta un viaje extra a la tabla: el *Key Lookup*.

-----

## Antes de empezar

Conviene que ya sepas:

* Páginas, heap, árbol B y lecturas lógicas, de [Cómo guarda los datos SQL Server](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md).
* Que SQL Server no indexa las claves foráneas, de [Claves foráneas](../01-fundamentos/05-Claves%20foráneas%20e%20integridad%20referencial.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Índice agrupado (clustered):** el índice cuyas hojas son las filas de la tabla, ordenadas por la clave.
* **Índice no agrupado (nonclustered):** índice aparte, con sus propias columnas clave y un localizador de la fila.
* **Localizador de fila (row locator):** lo que guarda un índice no agrupado para llegar a la fila: la clave del agrupado o, en un heap, el RID.
* **Index Seek:** navegar el árbol del índice hasta las filas buscadas.
* **Index Scan:** recorrer el índice completo (o una gran parte).
* **Key Lookup:** ir al índice agrupado a buscar las columnas que faltan en el no agrupado, una vez por fila.
* **Punto de inflexión (tipping point):** la cantidad de filas a partir de la cual al optimizador le conviene más un *scan* que un *seek* con *lookups*.

-----

## El problema

En `TiendaGrande`, la pantalla "mis pedidos" de la web ejecuta, para cada cliente que entra:

```sql
SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE ClienteId = 12345;
```

Devuelve unas 10 filas, pero lee **todas** las páginas de la tabla:

```text
Table 'Pedido'. Scan count 1, logical reads 4500, ...
```

La tabla está ordenada por `PedidoId` (su índice agrupado), no por `ClienteId`. Para encontrar los pedidos de un cliente, el motor no tiene más remedio que revisarlos todos. Con mil usuarios simultáneos, son 4,5 millones de lecturas de página para mostrar 10.000 filas.

Hace falta otro "orden" de la tabla, por `ClienteId`. Pero la tabla solo se puede ordenar físicamente de una forma.

-----

## Cómo funciona

### El índice agrupado: la tabla ordenada

Como se vio en la lección anterior, el índice agrupado organiza las filas en las hojas de un árbol B, ordenadas por la clave:

```text
Índice agrupado PK_Pedido (PedidoId)
Raíz → Intermedio → Hojas = LAS FILAS COMPLETAS
                    [1: Cliente 4521, 2023-05-02, Entregado]
                    [2: Cliente 81, 2024-11-30, Entregado]
                    ...
```

* Solo hay **uno** por tabla.
* En SQL Server se crea por defecto con la `PRIMARY KEY`, pero son conceptos distintos: la clave primaria es una regla de unicidad; el agrupado, una decisión de almacenamiento.
* Buscar por la clave del agrupado (`WHERE PedidoId = 500000`) es un *Clustered Index Seek*: 3 lecturas.
* Buscar por otra columna (`WHERE ClienteId = 12345`) es un *Clustered Index Scan*: todas las páginas.

### El índice no agrupado: un orden adicional

Un índice no agrupado es una **estructura aparte**: otro árbol B, ordenado por las columnas que elijas, cuyas hojas no tienen la fila completa sino la clave del índice **más un localizador** de la fila.

```sql
CREATE INDEX IX_Pedido_ClienteId ON dbo.Pedido (ClienteId);
```

```text
Índice no agrupado IX_Pedido_ClienteId (ClienteId)
Raíz → Intermedio → Hojas = (ClienteId, PedidoId)
                    [12344, 30871]
                    [12345, 9902]     ← los pedidos del cliente 12345,
                    [12345, 15077]       uno al lado del otro
                    [12345, 402116]
                    [12346, 777]
                    ...
```

El localizador es la **clave del índice agrupado** (aquí, `PedidoId`). Si la tabla fuera un heap, sería el RID. Como `PedidoId` es un `INT`, cada entrada ocupa unos pocos bytes: el índice entero es mucho más chico que la tabla, porque solo tiene dos columnas.

Con el índice, la consulta del problema:

1. **Index Seek** en `IX_Pedido_ClienteId` hasta `ClienteId = 12345`: 3 páginas, y ahí están los `PedidoId` de sus ~10 pedidos.
2. Pero la consulta pide `FechaPedido` y `Estado`, que **no están** en el índice. Para cada uno de los 10 `PedidoId`, el motor va al índice agrupado a buscar la fila: un **Key Lookup** (3 páginas cada uno).

```text
Table 'Pedido'. Scan count 1, logical reads ~33, ...
```

De 4.500 lecturas a ~33. El *Key Lookup* agrega trabajo (10 × 3 lecturas), pero con 10 filas es un precio pequeño.

### El costo de los Key Lookups y el punto de inflexión

Ahora el cliente 1, el mayorista con 50.000 pedidos:

```sql
SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE ClienteId = 1;
```

Con el mismo índice, la estrategia *seek* + *lookup* costaría 50.000 *lookups* × 3 lecturas = **150.000 lecturas**, frente a las 4.500 de recorrer la tabla entera. El optimizador lo calcula (con las estadísticas, ver [Planes de ejecución II](05-Planes%20de%20ejecución%20II.md)) y **elige el scan**, aunque el índice exista.

Ese cambio de estrategia tiene un nombre: el **punto de inflexión**. Cuando la cantidad de filas a buscar supera aproximadamente un cuarto o un tercio de la cantidad de **páginas** de la tabla, los *lookups* dejan de convenir. En `Pedido`, con ~4.500 páginas, eso es del orden de 1.000 a 1.500 filas: apenas el 0,1 % de la tabla.

```text
Filas que devuelve el filtro       Estrategia que elige el optimizador
──────────────────────────────     ─────────────────────────────────────
pocas (cliente 12345: ~10)     →   Index Seek + Key Lookup  (~33 lecturas)
muchas (cliente 1: 50.000)     →   Clustered Index Scan     (~4.500 lecturas)
```

La conclusión es importante: **un índice no agrupado que no cubre la consulta solo sirve para filtros muy selectivos**. Para que sirva también con muchas filas, el índice tiene que tener todas las columnas que la consulta necesita, y eso es un índice cubriente, en la [próxima lección](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md).

### Elegir la clave del índice agrupado

Como la clave del agrupado se copia en **cada entrada de cada índice no agrupado**, su elección afecta a toda la tabla. Una buena clave de agrupado es:

| Propiedad | Por qué | `PedidoId INT IDENTITY` |
| --- | --- | --- |
| **Angosta** | Se copia en todos los índices no agrupados | 4 bytes ✅ |
| **Única** | Si no lo es, SQL Server agrega 4 bytes ocultos (*uniqueifier*) a las filas repetidas | ✅ |
| **Estable** | Si cambia, hay que mover la fila y actualizar todos los índices | Nunca cambia ✅ |
| **Creciente** | Las filas nuevas van al final y no parten páginas | ✅ |

Por eso `INT` o `BIGINT IDENTITY` es la clave de agrupado por defecto, y un `UNIQUEIDENTIFIER` aleatorio, o un `NVARCHAR(254)` como el email, son malas elecciones (ver [Claves primarias](../01-fundamentos/04-Claves%20primarias.md)).

Hay casos en que conviene agrupar por otra columna: una tabla de eventos que **siempre** se consulta por rangos de fecha puede agruparse por `(Fecha, Id)`, para que cada rango esté en páginas contiguas. Se hace declarando la clave primaria como `NONCLUSTERED` y creando el agrupado aparte:

```sql
CREATE TABLE dbo.Evento (
    EventoId INT IDENTITY(1,1) NOT NULL,
    Fecha    DATETIME2(0)      NOT NULL,
    Detalle  NVARCHAR(400)     NOT NULL,
    CONSTRAINT PK_Evento PRIMARY KEY NONCLUSTERED (EventoId)
);
CREATE CLUSTERED INDEX CIX_Evento_Fecha ON dbo.Evento (Fecha, EventoId);
```

### Índices únicos

Un índice puede exigir que sus valores no se repitan:

```sql
CREATE UNIQUE INDEX UX_Cliente_Email ON dbo.Cliente (Email);
```

Una restricción `UNIQUE` (como `UQ_Cliente_Email`) crea por dentro exactamente este índice. La diferencia es de intención: la restricción expresa una regla de negocio; el índice único, además, permite filtros (`WHERE ... IS NOT NULL`, ver [Restricciones](../01-fundamentos/06-Restricciones.md)). Para el optimizador, saber que un índice es único es valioso: una búsqueda por igualdad devuelve como máximo una fila.

### El costo de cada índice

Cada índice acelera algunas lecturas y **encarece todas las escrituras**:

| Operación | Efecto en los índices |
| --- | --- |
| `INSERT` | Hay que insertar una entrada en **cada** índice de la tabla |
| `DELETE` | Hay que borrar la entrada de **cada** índice |
| `UPDATE` | Hay que actualizar los índices que contienen las columnas modificadas |

Una tabla con 12 índices hace 13 escrituras por cada fila insertada. Además, cada índice ocupa espacio en disco, en memoria y en las copias de seguridad. Por eso los índices se crean para **consultas concretas y frecuentes**, no para cada columna "por si acaso".

-----

## Ejemplo completo

El problema de la pantalla "mis pedidos", resuelto y medido en `TiendaGrande`:

```sql
USE TiendaGrande;
SET STATISTICS IO ON;

-- 1. Sin índice: scan de toda la tabla
SELECT PedidoId, FechaPedido, Estado FROM dbo.Pedido WHERE ClienteId = 12345;

-- 2. Crear el índice que faltaba sobre la clave foránea
CREATE INDEX IX_Pedido_ClienteId ON dbo.Pedido (ClienteId);

-- 3. Cliente común: seek + lookups
SELECT PedidoId, FechaPedido, Estado FROM dbo.Pedido WHERE ClienteId = 12345;

-- 4. Cliente mayorista: el optimizador vuelve al scan
SELECT PedidoId, FechaPedido, Estado FROM dbo.Pedido WHERE ClienteId = 1;

-- 5. Solo columnas que están en el índice: seek sin lookups, aunque sean 50.000 filas
SELECT PedidoId FROM dbo.Pedido WHERE ClienteId = 1;

SET STATISTICS IO OFF;
```

| Paso | Plan (operadores principales) | Lecturas lógicas aproximadas |
| --- | --- | --- |
| 1 | Clustered Index Scan | 4.500 |
| 3 | Index Seek (`IX_Pedido_ClienteId`) + Key Lookup | ~33 |
| 4 | Clustered Index Scan | 4.500 |
| 5 | Index Seek (`IX_Pedido_ClienteId`) | ~100 |

Qué muestra cada paso:

* **Paso 1 → 3:** el índice sobre la clave foránea reduce las lecturas más de cien veces para un cliente común. Es el índice que el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md) ya tiene, y que `TiendaGrande` omitió a propósito.
* **Paso 4:** el índice existe, pero el optimizador no lo usa: 50.000 *lookups* costarían más que el scan. No es un error del optimizador, es una buena decisión.
* **Paso 5:** si la consulta solo necesita columnas que el índice tiene (`ClienteId` y el localizador `PedidoId`), no hay *lookups*, y el *seek* recorre solo el tramo de hojas del cliente 1 (50.000 entradas angostas, del orden de cien páginas). El índice **cubre** la consulta.

La próxima lección parte de ahí: cómo hacer que el índice cubra también `FechaPedido` y `Estado`.

-----

## Errores comunes

**1. No indexar las claves foráneas.**
Qué pasa: los joins y las búsquedas por la FK recorren toda la tabla hija; los `DELETE` del padre también.
Por qué: SQL Server no crea esos índices automáticamente.
Arreglo: un índice no agrupado por cada FK que no sea ya la primera columna de otro índice.

**2. Esperar que el índice se use siempre.**
Qué pasa: "creé el índice y la consulta sigue haciendo scan".
Por qué: con muchas filas, los *Key Lookups* cuestan más que el scan, y el optimizador elige bien.
Arreglo: un índice que cubra la consulta, o aceptar el scan si la consulta devuelve una parte grande de la tabla.

**3. Una clave de agrupado ancha o aleatoria.**
Qué pasa: todos los índices no agrupados crecen, y las inserciones fragmentan la tabla.
Por qué: la clave del agrupado se copia en cada índice, y una clave aleatoria inserta en cualquier página.
Arreglo: clave angosta, única, estable y creciente (`INT`/`BIGINT IDENTITY`).

**4. Un índice por columna.**
Qué pasa: inserciones lentas, base enorme, y los índices de una sola columna rara vez cubren las consultas reales.
Por qué: se indexa "por si acaso", sin mirar las consultas.
Arreglo: índices diseñados para las consultas frecuentes (próxima lección).

**5. Confundir clave primaria con índice agrupado.**
Qué pasa: se cree que agregar un `UNIQUE` sobre `Email` "reordena" la tabla, o que no se puede agrupar por otra columna.
Por qué: en SQL Server suelen coincidir.
Arreglo: la PK es una regla lógica; el agrupado, una decisión física; se pueden separar.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Índice agrupado | Opcional, elegible (uno por tabla) | **No existe** (`CLUSTER` reordena una vez) | **Obligatorio**: la PK (o una clave oculta) |
| Localizador del índice secundario | Clave del agrupado (o RID en un heap) | Posición física de la fila (TID) | La clave primaria |
| Equivalente del Key Lookup | Key Lookup / RID Lookup | *Heap fetch* después de un *Index Scan* | Búsqueda por la PK después del índice secundario |
| Índice sobre las FK | Manual | Manual | Automático |
| Máximo de índices no agrupados | 999 por tabla | Sin límite práctico | 64 por tabla |

En MySQL, como el localizador es la clave primaria, una PK ancha (un `VARCHAR(255)`) agranda **todos** los índices secundarios: el mismo argumento que en SQL Server, pero sin la opción de elegir otro agrupado.

### En MongoDB

Toda colección tiene un índice obligatorio sobre `_id`, y los demás son índices secundarios (árboles B) que apuntan al documento. El equivalente del *Key Lookup* es la etapa `FETCH` que aparece en `explain()` después de un `IXSCAN`: el índice encuentra la entrada y después hay que traer el documento completo. Si todos los campos que pide la consulta están en el índice, el plan no tiene `FETCH`: es una consulta cubierta (*covered query*).

-----

## Cuándo sí y cuándo no

**Crea un índice no agrupado cuando:**

* Una consulta frecuente filtra, une u ordena por una columna que no es la clave del agrupado.
* La columna es una clave foránea.

**No lo crees cuando:**

* La tabla es chica (unas pocas páginas): el scan es igual de barato.
* La columna tiene muy pocos valores distintos y las consultas devuelven gran parte de la tabla (un `Activo BIT`), salvo como índice filtrado (próxima lección).
* La tabla recibe muchísimas escrituras y la consulta que se beneficiaría es rara.

-----

## Resumen en 5 líneas

1. El índice agrupado es la tabla ordenada por su clave; solo hay uno, y por defecto es la PK.
2. Un índice no agrupado es un árbol aparte con sus columnas clave más la clave del agrupado como localizador.
3. Si faltan columnas, cada fila encontrada cuesta un Key Lookup; con muchas filas, el optimizador prefiere el scan (punto de inflexión).
4. La clave del agrupado debe ser angosta, única, estable y creciente, porque se copia en todos los índices.
5. Cada índice encarece las escrituras: se crean para consultas concretas, empezando por las claves foráneas.

-----

## Para profundizar

<details>
<summary>Cómo calcular el punto de inflexión</summary>

El punto de inflexión no es un porcentaje fijo de filas: depende de cuántas **páginas** tiene la tabla. Una regla aproximada, popularizada por Kimberly Tripp, es que el optimizador deja de preferir los *lookups* cuando la cantidad de filas a buscar ronda entre un cuarto y un tercio de la cantidad de páginas de la tabla. Por eso una tabla con filas anchas (pocas filas por página, muchas páginas) tolera más *lookups* que una con filas angostas. Para la tabla `Pedido`, con ~4.500 páginas, el umbral está entre 1.100 y 1.500 filas; para `PedidoInflado`, con ~17.000 páginas, entre 4.300 y 5.700.

</details>

<details>
<summary>RID Lookup en un heap</summary>

Si la tabla es un heap, el localizador del índice no agrupado es el RID, y el plan muestra un **RID Lookup** en lugar de un Key Lookup. Es un poco más barato (va directo a la página, sin bajar por un árbol), pero si la fila se movió por una actualización, el RID apunta a un puntero de reenvío y hay que hacer una lectura más. Es una de las razones por las que los heaps con actualizaciones frecuentes degradan con el tiempo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El índice agrupado ordena físicamente la tabla por su clave, normalmente la clave primaria, y solo puede haber uno. Los índices no agrupados son estructuras aparte que guardan la columna indexada y un puntero a la fila, y puede haber muchos. Sirven para buscar rápido por columnas que no son la clave, pero cada índice hace más lentas las inserciones y actualizaciones.

### Respuesta ampliada (semi-senior)

El índice agrupado es la tabla misma organizada como un árbol B por su clave; un no agrupado es un árbol aparte con sus columnas clave y, como localizador, la clave del agrupado (o el RID en un heap). Si la consulta necesita columnas que el no agrupado no tiene, cada fila encontrada genera un Key Lookup al agrupado, y a partir del punto de inflexión (del orden de un cuarto de las páginas de la tabla) el optimizador prefiere un scan; por eso un índice no cubriente solo sirve para filtros selectivos. Elijo claves de agrupado angostas, únicas, estables y crecientes, porque se copian en todos los índices, y separo PK y agrupado cuando el patrón de acceso lo justifica, como agrupar eventos por fecha. Indexo las claves foráneas y diseño los demás índices para consultas frecuentes concretas, sopesando el costo en escrituras, espacio y memoria.

### Preguntas frecuentes de seguimiento

**1. ¿Cuántos índices agrupados puede tener una tabla?**
Uno, porque las filas solo se pueden ordenar físicamente de una forma.

**2. ¿Qué es un Key Lookup?**
Ir al índice agrupado a buscar las columnas que faltan en el índice no agrupado, una vez por cada fila encontrada.

**3. Creaste un índice y la consulta sigue haciendo scan. ¿Por qué puede ser?**
Porque devuelve tantas filas que los Key Lookups costarían más que el scan, porque el filtro no es SARGable, porque hay una conversión implícita o porque las estadísticas están desactualizadas.

**4. ¿Qué características debe tener la clave de un índice agrupado?**
Angosta, única, estable y creciente.

-----

## Práctica

**Ejercicio 1.** En `TiendaGrande`, la consulta "líneas de pedido de un producto" es lenta. Mide las lecturas, crea el índice adecuado y vuelve a medir.

```sql
SELECT PedidoId, Cantidad FROM dbo.DetallePedido WHERE ProductoId = 777;
```

<details>
<summary>Solución</summary>

```sql
SET STATISTICS IO ON;
SELECT PedidoId, Cantidad FROM dbo.DetallePedido WHERE ProductoId = 777;   -- antes

CREATE INDEX IX_DetallePedido_ProductoId ON dbo.DetallePedido (ProductoId);

SELECT PedidoId, Cantidad FROM dbo.DetallePedido WHERE ProductoId = 777;   -- después
SET STATISTICS IO OFF;
```

Antes: un *Clustered Index Scan* de las ~2,5 millones de líneas (unas 9.000 páginas: cada línea ocupa ~30 bytes). La clave primaria es `(PedidoId, ProductoId)`: empieza por `PedidoId`, así que no sirve para buscar por producto.

Después: *Index Seek* en `IX_DetallePedido_ProductoId` más un *Key Lookup* por cada línea del producto (unas 500, porque 2,5 millones / 5.000 productos), para traer `Cantidad`. `PedidoId` no necesita *lookup*: forma parte de la clave del agrupado, que todo índice no agrupado ya contiene. 500 *lookups* × 3 lecturas = ~1.500 lecturas, contra ~9.000 del scan: están por debajo del punto de inflexión (entre 2.000 y 3.000 filas para esta tabla), así que el optimizador elige el *seek*.

</details>

**Ejercicio 2.** Una tabla `Log(LogId UNIQUEIDENTIFIER DEFAULT NEWID() PRIMARY KEY, Fecha DATETIME2, Mensaje NVARCHAR(2000))` recibe 500 inserciones por segundo y se consulta siempre por rangos de fecha. ¿Qué problemas tiene su diseño de índices y cómo lo cambiarías?

<details>
<summary>Solución</summary>

Problemas:

* **Agrupado sobre un GUID aleatorio:** cada inserción cae en una página aleatoria y la parte (*page split*), fragmentando la tabla y multiplicando las escrituras.
* **Clave de agrupado de 16 bytes:** cualquier índice no agrupado que se agregue la copia.
* **Las consultas por fecha no tienen índice:** cada consulta recorre toda la tabla.

Rediseño:

```sql
CREATE TABLE dbo.Log (
    LogId   BIGINT IDENTITY(1,1) NOT NULL,
    Fecha   DATETIME2(3)         NOT NULL,
    Mensaje NVARCHAR(2000)       NOT NULL,
    CONSTRAINT PK_Log PRIMARY KEY NONCLUSTERED (LogId)
);
CREATE CLUSTERED INDEX CIX_Log_Fecha ON dbo.Log (Fecha, LogId);
```

El agrupado por `(Fecha, LogId)` es creciente (los logs llegan en orden de fecha), así que las inserciones van al final, y los rangos de fecha quedan en páginas contiguas. `LogId` en el agrupado lo hace único sin *uniqueifier*. Si hiciera falta un identificador global, un `UNIQUEIDENTIFIER` puede ir como columna aparte con `UNIQUE`, sin ser la clave del agrupado.

</details>

-----

## Siguiente lección

[Índices compuestos, cubrientes y filtrados](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md)
