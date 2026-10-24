# Cómo guarda los datos SQL Server

## En una frase

SQL Server guarda las filas en **páginas de 8 KB**, y toda lectura, escritura e índice se mide en páginas: una consulta rápida es una consulta que lee **pocas páginas**, y por eso el tamaño de las filas, la organización de la tabla (heap o índice agrupado) y la memoria donde se guardan las páginas leídas explican casi todo el rendimiento.

-----

## Antes de empezar

Conviene que ya sepas:

* Tipos de datos y su tamaño, de [Tipos de datos](../01-fundamentos/03-Tipos%20de%20datos.md).
* Que la clave primaria es por defecto el índice agrupado, de [Claves primarias](../01-fundamentos/04-Claves%20primarias.md).
* La base `TiendaGrande`, de [Datos de volumen](00-Datos%20de%20volumen.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Página:** la unidad mínima de lectura y escritura: 8 KB.
* **Extensión (extent):** 8 páginas contiguas (64 KB).
* **Heap (montón):** tabla sin índice agrupado; las filas no tienen orden.
* **B-tree (árbol B):** estructura de árbol balanceado con la que se organizan los índices.
* **Buffer pool:** la memoria donde SQL Server guarda las páginas leídas.
* **Lectura lógica / física:** leer una página desde la memoria / desde el disco.
* **Densidad de página:** qué porcentaje de cada página está ocupado con datos.

-----

## El problema

Dos tablas con los mismos datos de pedidos. En la segunda, alguien eligió los tipos "por las dudas":

```sql
-- Pedido:         PedidoId INT, ClienteId INT, FechaPedido DATETIME2(0), Estado VARCHAR(20)
-- PedidoInflado:  PedidoId BIGINT, ClienteId BIGINT, FechaPedido DATETIME, Estado NCHAR(50)
```

Con un millón de filas, contar los pedidos entregados en la primera lee unas 4.500 páginas; en la segunda, más de 17.000. La misma consulta, los mismos datos, casi **cuatro veces** más trabajo. Y no solo esa consulta: cada *scan*, cada copia de seguridad, cada índice y cada megabyte de memoria cuesta lo mismo de más.

Para entender por qué, y para entender todo lo que sigue sobre índices, hay que bajar un nivel: cómo están guardadas las filas en el disco.

-----

## Cómo funciona

### Archivos, páginas y extensiones

Una base de SQL Server tiene, como mínimo, dos archivos:

| Archivo | Extensión habitual | Contiene |
| --- | --- | --- |
| Datos | `.mdf` (y `.ndf` si hay más) | Las tablas y los índices, en páginas |
| Log de transacciones | `.ldf` | El registro de cada cambio, para recuperar y deshacer |

Los archivos de datos se dividen en **páginas de 8 KB** (8.192 bytes), que se agrupan en **extensiones** de 8 páginas (64 KB). La página es la unidad **mínima**: para leer una fila de 40 bytes, el motor lee la página entera de 8 KB que la contiene.

```text
Archivo de datos (.mdf)
├── Extensión 0  [pág 0][pág 1][pág 2]...[pág 7]   ← 64 KB
├── Extensión 1  [pág 8][pág 9]...[pág 15]
└── ...

Página (8.192 bytes)
┌──────────────────────────────────────┐
│ Encabezado (96 bytes)                │
├──────────────────────────────────────┤
│ Fila 1 │ Fila 2 │ Fila 3 │ ...       │  ← hasta 8.060 bytes de datos
│                                      │
│          espacio libre               │
├──────────────────────────────────────┤
│ Matriz de posiciones (2 bytes/fila)  │  ← dónde empieza cada fila
└──────────────────────────────────────┘
```

Una fila "normal" no puede ocupar más de 8.060 bytes. Las columnas de longitud variable que no caben (un `NVARCHAR(4000)` muy lleno) se mueven a páginas de desbordamiento, y los tipos `MAX` (`NVARCHAR(MAX)`, `VARBINARY(MAX)`) pueden guardarse en páginas aparte (*LOB*). Leer esos datos cuesta lecturas adicionales.

### Cuántas filas entran en una página

Cada fila ocupa sus columnas más unos bytes de control (encabezado, mapa de nulos, posiciones de las columnas variables). Para `Pedido`:

| Parte | Bytes |
| --- | --- |
| `PedidoId INT` + `ClienteId INT` | 8 |
| `FechaPedido DATETIME2(0)` | 6 |
| `Estado VARCHAR(20)` ('Entregado') | 9 + 2 de posición |
| Encabezado y mapa de nulos | ~9 |
| Entrada en la matriz de posiciones | 2 |
| **Total aproximado** | **~36** |

8.060 / 36 ≈ **220 filas por página**, y un millón de filas ocupan unas **4.500 páginas** (~35 MB). Con los tipos "inflados" del problema (`BIGINT`, `BIGINT`, `DATETIME`, `NCHAR(50)` = 8 + 8 + 8 + 100 bytes), cada fila supera los 135 bytes: unas 60 filas por página y más de 17.000 páginas.

Esta es la razón por la que elegir bien los tipos (lección [Tipos de datos](../01-fundamentos/03-Tipos%20de%20datos.md)) es una decisión de **rendimiento** y no solo de prolijidad:

* **Filas más angostas → más filas por página → menos páginas → menos lecturas.**
* Las claves (primaria, foráneas, columnas de índices) se copian en cada índice: un byte de más en la clave se multiplica por cada índice y cada fila.

### Heap: una tabla sin orden

Una tabla **sin índice agrupado** es un *heap*: las filas se guardan donde haya espacio, sin ningún orden. Para encontrar una fila concreta, el motor no tiene más opción que leer **todas** las páginas (un *Table Scan*). Cada fila se identifica por su **RID** (*row identifier*: archivo, página y posición).

```sql
-- Una copia sin índices: SELECT ... INTO crea un heap
SELECT * INTO dbo.PedidoHeap FROM dbo.Pedido;
```

Los heaps tienen usos (tablas de *staging* que se cargan y se leen enteras), pero para tablas de negocio son casi siempre una mala idea: además de los *scans*, las actualizaciones que agrandan una fila la mueven a otra página y dejan un "puntero de reenvío" (*forwarded record*), que agrega lecturas.

### Índice agrupado: la tabla ordenada como un árbol

Con un índice agrupado (en `Pedido`, la clave primaria `PedidoId`), las páginas de datos se organizan como las **hojas de un árbol B**, ordenadas por la clave, y por encima se construyen niveles de páginas que funcionan como un índice de un libro:

```text
                    ┌──────────────────────┐
Raíz                │ 1 | 450.001 | ...    │        1 página
                    └──┬────────┬──────────┘
            ┌──────────┘        └──────────┐
Intermedio  │ 1 | 221 | 441 ...│   │ 450.001 | ... │    ~20 páginas
            └──┬────┬──────────┘
       ┌───────┘    └────┐
Hoja   │ filas 1 a 220 │ │ filas 221 a 440 │ ...       ~4.500 páginas
       └───────────────┘ └─────────────────┘           (los datos)
```

Para encontrar `PedidoId = 500000`, el motor lee la raíz, baja a la página intermedia que corresponde y llega a la hoja: **3 páginas** en lugar de 4.500. Eso es un *Index Seek*. La cantidad de niveles crece muy despacio (cada nivel multiplica la capacidad por cientos), así que una tabla de mil millones de filas rara vez pasa de 4 o 5 niveles.

En un índice agrupado, **las hojas son la tabla**: no hay una copia aparte de los datos. Por eso solo puede haber un índice agrupado por tabla: las filas solo se pueden ordenar físicamente de una forma.

### Memoria: lecturas lógicas y físicas

SQL Server no lee del disco cada vez. Las páginas leídas se guardan en el **buffer pool**, en memoria, y las siguientes lecturas de esa página salen de ahí:

| Lectura | De dónde | Costo |
| --- | --- | --- |
| **Física** | Del disco, la primera vez | Alto (milisegundos por lectura en disco mecánico; mucho menos en SSD) |
| **Lógica** | De la memoria | Bajo, pero no gratis: cada página hay que recorrerla |

Toda lectura física es también una lectura lógica. Por eso, para comparar dos versiones de una consulta, se miran las **lecturas lógicas**: son el trabajo de la consulta, independiente de si la caché estaba caliente o fría. Se miden con `SET STATISTICS IO ON` (ver [Planes de ejecución I](04-Planes%20de%20ejecución%20I.md)).

Y hay una consecuencia de diseño: si las tablas que se usan a diario caben en la memoria del servidor, la mayoría de las lecturas son lógicas. Filas más angostas significan que **más datos caben en la misma memoria**.

### Medir lo que ocupa una tabla

```sql
EXEC sp_spaceused 'dbo.Pedido';
```

| name | rows | reserved | data | index_size | unused |
| --- | --- | --- | --- | --- | --- |
| Pedido | 1000000 | ~36 MB | ~35 MB | ~0,2 MB | ~0,5 MB |

Y con más detalle (cantidad de páginas por nivel, tamaño medio de fila, densidad):

```sql
SELECT index_level, page_count, avg_record_size_in_bytes, avg_page_space_used_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Pedido'), 1, NULL, 'DETAILED');
```

| index_level | page_count | avg_record_size_in_bytes | avg_page_space_used_in_percent |
| --- | --- | --- | --- |
| 0 (hoja) | ~4.500 | ~34 | ~99 |
| 1 | ~20 | ~11 | ~95 |
| 2 (raíz) | 1 | ~11 | ~25 |

(Los valores son aproximados: dependen de la versión y del estado de la tabla.) El nivel 0 son los datos; los niveles superiores, la navegación. El modo `DETAILED` lee todas las páginas, así que en tablas grandes de producción conviene `SAMPLED`.

-----

## Ejemplo completo

El problema del principio, medido en `TiendaGrande`:

```sql
USE TiendaGrande;

-- La tabla con tipos "por las dudas"
CREATE TABLE dbo.PedidoInflado (
    PedidoId    BIGINT    NOT NULL PRIMARY KEY,
    ClienteId   BIGINT    NOT NULL,
    FechaPedido DATETIME  NOT NULL,
    Estado      NCHAR(50) NOT NULL
);

INSERT INTO dbo.PedidoInflado (PedidoId, ClienteId, FechaPedido, Estado)
SELECT PedidoId, ClienteId, FechaPedido, Estado FROM dbo.Pedido;

-- Comparar
SET STATISTICS IO ON;

SELECT COUNT(*) FROM dbo.Pedido        WHERE Estado = 'Entregado';
SELECT COUNT(*) FROM dbo.PedidoInflado WHERE Estado = N'Entregado';

SET STATISTICS IO OFF;
```

En la pestaña **Mensajes** (los números exactos varían):

```text
Table 'Pedido'.        Scan count 1, logical reads 4500, physical reads 0, ...
Table 'PedidoInflado'. Scan count 1, logical reads 17200, physical reads 0, ...
```

Y el espacio:

```sql
EXEC sp_spaceused 'dbo.Pedido';         -- ~36 MB
EXEC sp_spaceused 'dbo.PedidoInflado';  -- ~135 MB
```

Qué explica la diferencia:

* **`NCHAR(50)` en lugar de `VARCHAR(20)`:** 100 bytes fijos por fila, aunque el estado tenga 9 letras. Es la mayor parte del aumento.
* **`BIGINT` en lugar de `INT`:** 4 bytes más por cada clave. Poco por fila, pero se copia en cada índice que incluya esas columnas.
* **`DATETIME` en lugar de `DATETIME2(0)`:** 2 bytes más, con menos precisión útil.
* **La consulta no cambió:** las dos hacen un *scan* completo (ninguna tiene un índice sobre `Estado`). La diferencia es solo cuántas páginas hay que recorrer. Los índices, desde la próxima lección, reducen **cuántas** de esas páginas hay que leer.

```sql
DROP TABLE dbo.PedidoInflado;   -- limpieza
```

-----

## Errores comunes

**1. Elegir tipos más grandes "por las dudas".**
Qué pasa: tablas varias veces más grandes, más lecturas, más memoria, copias de seguridad más lentas.
Por qué: se piensa en un byte por fila, no en millones de filas por cada índice.
Arreglo: el tipo más chico que cubra el dominio con margen razonable.

**2. Tablas de negocio sin índice agrupado.**
Qué pasa: cada búsqueda es un *Table Scan*, y las actualizaciones generan punteros de reenvío.
Por qué: se creó la tabla sin clave primaria, o con `SELECT ... INTO`.
Arreglo: toda tabla de negocio con clave primaria (que crea el índice agrupado).

**3. Medir en milisegundos en lugar de en lecturas.**
Qué pasa: "la consulta mejoró" porque la segunda ejecución tardó menos, cuando en realidad la primera leyó del disco.
Por qué: el tiempo depende del estado de la caché, del servidor y de la concurrencia.
Arreglo: comparar lecturas lógicas con `SET STATISTICS IO ON`.

**4. `CHAR`/`NCHAR` para texto de longitud variable.**
Qué pasa: cada fila ocupa la longitud máxima, aunque el valor sea corto.
Por qué: `CHAR(n)` rellena con espacios.
Arreglo: `VARCHAR`/`NVARCHAR` salvo que todos los valores tengan la misma longitud.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Tamaño de página | 8 KB (fijo) | 8 KB (por defecto, fijo al compilar) | 16 KB (por defecto, configurable) |
| Organización de la tabla | Heap o índice agrupado (a elección) | **Siempre heap**; los índices apuntan a la posición de la fila | **Siempre agrupada** por la clave primaria |
| Datos grandes fuera de la fila | Desbordamiento y LOB | TOAST (comprime y saca valores grandes) | Páginas de desbordamiento |
| Memoria caché | Buffer pool | `shared_buffers` + caché del sistema operativo | Buffer pool de InnoDB |
| Ver el tamaño | `sp_spaceused` | `pg_relation_size`, `pg_total_relation_size` | `information_schema.TABLES` (`DATA_LENGTH`) |

La diferencia de organización tiene consecuencias. En PostgreSQL, como la tabla es siempre un heap, `CLUSTER` reordena las filas por un índice **una vez**, pero el orden no se mantiene con las inserciones siguientes. En MySQL, como la tabla es siempre el índice de la clave primaria, una clave aleatoria (UUID) fragmenta la tabla completa.

### En MongoDB

El motor WiredTiger guarda cada colección como un árbol B de documentos, comprimidos por defecto (con *snappy*), con su propia caché en memoria. Los documentos tienen un tamaño máximo de 16 MB. El mismo principio aplica: documentos más chicos (sin campos innecesarios ni nombres de campo larguísimos, que se repiten en cada documento) significan más documentos por página y por megabyte de caché.

-----

## Cuándo sí y cuándo no

**Piensa en páginas cuando:**

* Diseñas una tabla que va a crecer: el tamaño de la fila multiplica todo.
* Analizas una consulta lenta: la pregunta siempre es "¿cuántas páginas lee y por qué?".

**Ten en cuenta:**

* Para tablas chicas (cientos o pocos miles de filas), nada de esto se nota: caben en pocas páginas que viven siempre en memoria.
* La compresión de datos (`DATA_COMPRESSION = ROW` o `PAGE`) reduce las páginas a costa de CPU; es una opción para tablas grandes de lectura frecuente.

**Herramientas que ayudan:**

* `sp_spaceused` y `sys.dm_db_index_physical_stats` para medir.
* El informe *Disk Usage by Top Tables* de SSMS (clic derecho en la base → *Reports* → *Standard Reports*).

-----

## Resumen en 5 líneas

1. SQL Server lee y escribe en páginas de 8 KB; una consulta rápida es una que lee pocas páginas.
2. Filas más angostas → más filas por página → menos lecturas: los tipos de datos son una decisión de rendimiento.
3. Un heap no tiene orden y obliga a recorrer toda la tabla; un índice agrupado ordena las filas como las hojas de un árbol B.
4. Un *seek* en un árbol B lee unas pocas páginas (una por nivel) en lugar de todas.
5. Las lecturas lógicas (desde la memoria) miden el trabajo de una consulta; compáralas en lugar de los tiempos.

-----

## Para profundizar

<details>
<summary>Ver las páginas por dentro</summary>

Existen comandos no documentados para ver el contenido de una página, muy usados para aprender. `DBCC PAGE` muestra el encabezado y las filas de una página concreta, y la función `sys.dm_db_page_info` (desde SQL Server 2019, documentada) muestra el encabezado. Para encontrar en qué página está una fila, `sys.fn_PhysLocFormatter(%%physloc%%)` devuelve su ubicación `(archivo:página:posición)`:

```sql
SELECT TOP (3) PedidoId, sys.fn_PhysLocFormatter(%%physloc%%) AS Ubicacion
FROM dbo.Pedido;
```

Úsalos solo en bases de prueba.

</details>

<details>
<summary>Extensiones uniformes y mixtas</summary>

Las primeras páginas de un objeto chico pueden asignarse en extensiones **mixtas** (compartidas entre varios objetos), para no reservar 64 KB a una tabla de 3 filas. Desde SQL Server 2016, por defecto las bases de usuario asignan siempre extensiones **uniformes** (de un solo objeto), lo que reduce la contención en la asignación. Es un detalle interno, pero explica por qué una tabla vacía con un índice ya ocupa 64 KB o más en `sp_spaceused`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

SQL Server guarda los datos en páginas de 8 KB. Si las filas son más chicas, entran más en cada página y las consultas leen menos. Una tabla sin índice agrupado es un heap, y para buscar una fila hay que recorrerla toda; con un índice agrupado, las filas están ordenadas en un árbol B y se encuentran leyendo pocas páginas.

### Respuesta ampliada (semi-senior)

La unidad de trabajo es la página de 8 KB, agrupada en extensiones de 64 KB, y una fila no puede superar los 8.060 bytes en la fila; lo que no cabe va a páginas de desbordamiento o LOB, con lecturas extra. Por eso el tamaño de la fila es una decisión de rendimiento: determina cuántas filas entran por página, cuántas páginas lee un *scan*, cuánto de la tabla cabe en el buffer pool y cuánto pesa cada índice, porque las claves se copian en todos. Un heap obliga a recorrer la tabla y sufre punteros de reenvío; un índice agrupado organiza los datos como las hojas de un árbol B ordenado por la clave, con una profundidad que crece logarítmicamente, así que un *seek* lee unas pocas páginas. Para evaluar consultas comparo lecturas lógicas, que miden el trabajo independientemente de la caché, y para medir tablas uso `sp_spaceused` y `sys.dm_db_index_physical_stats`.

### Preguntas frecuentes de seguimiento

**1. ¿De qué tamaño es una página en SQL Server?**
8 KB, de los cuales unos 8.060 bytes están disponibles para los datos de las filas.

**2. ¿Qué es un heap?**
Una tabla sin índice agrupado: las filas no tienen orden y se identifican por su RID.

**3. ¿Qué diferencia hay entre una lectura lógica y una física?**
La lógica lee una página desde la memoria; la física la trae primero desde el disco. Toda física es también lógica.

**4. ¿Por qué solo puede haber un índice agrupado por tabla?**
Porque el índice agrupado es la tabla misma ordenada por su clave, y las filas solo se pueden ordenar físicamente de una forma.

-----

## Práctica

**Ejercicio 1.** Estima cuántas filas entran por página y cuántas páginas ocupa una tabla `Visita(VisitaId INT, ClienteId INT, Fecha DATETIME2(0), Url NVARCHAR(200))` con 10 millones de filas, si las URL tienen 60 caracteres de media.

<details>
<summary>Solución</summary>

| Parte | Bytes |
| --- | --- |
| `VisitaId` + `ClienteId` | 8 |
| `Fecha DATETIME2(0)` | 6 |
| `Url` (60 caracteres × 2 bytes en `NVARCHAR`) + 2 de posición | 122 |
| Encabezado, mapa de nulos, matriz de posiciones | ~11 |
| **Total** | **~147** |

8.060 / 147 ≈ **54 filas por página**, y 10.000.000 / 54 ≈ **185.000 páginas** (~1,4 GB). Si las URL son solo ASCII, guardarlas en `VARCHAR(200)` (60 bytes en lugar de 120) daría ~87 bytes por fila, unas 92 filas por página y ~109.000 páginas: un 40 % menos.

</details>

**Ejercicio 2.** En `TiendaGrande`, compara las lecturas lógicas de buscar el pedido 500.000 en `Pedido` (con índice agrupado) y en una copia sin índices.

<details>
<summary>Solución</summary>

```sql
SELECT * INTO dbo.PedidoHeap FROM dbo.Pedido;

SET STATISTICS IO ON;
SELECT * FROM dbo.Pedido     WHERE PedidoId = 500000;
SELECT * FROM dbo.PedidoHeap WHERE PedidoId = 500000;
SET STATISTICS IO OFF;

DROP TABLE dbo.PedidoHeap;
```

```text
Table 'Pedido'.     Scan count 0, logical reads 3, ...
Table 'PedidoHeap'. Scan count 1, logical reads ~4500, ...
```

En la tabla con índice agrupado, el motor baja por el árbol (raíz, intermedio, hoja): 3 páginas. En el heap no hay orden, así que lee todas las páginas para encontrar una sola fila. Mil quinientas veces más trabajo para el mismo resultado.

</details>

-----

## Siguiente lección

[Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md)
