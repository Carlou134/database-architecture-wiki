# Índices compuestos, cubrientes y filtrados

## En una frase

Un índice útil se diseña **para una consulta**: las columnas de la clave, en el orden correcto (primero las de igualdad, después las de orden, después las de rango), permiten buscar y ordenar; las columnas de `INCLUDE` permiten devolver los datos sin volver a la tabla; y un filtro `WHERE` permite indexar solo las filas que de verdad se consultan.

-----

## Antes de empezar

Conviene que ya sepas:

* Índices agrupados y no agrupados, Key Lookup y punto de inflexión, de [Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md).
* Índices únicos filtrados, de [Restricciones](../01-fundamentos/06-Restricciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Índice compuesto:** índice con varias columnas en la clave.
* **Prefijo izquierdo:** las primeras columnas de un índice compuesto; solo un filtro sobre el prefijo permite un *seek*.
* **Índice cubriente (covering):** índice que contiene todas las columnas que necesita una consulta, así que no hace falta ir a la tabla.
* **`INCLUDE`:** columnas que se guardan solo en las hojas del índice, para cubrir consultas, sin formar parte de la clave.
* **Índice filtrado:** índice con un `WHERE`, que solo contiene las filas que cumplen la condición.
* **Selectividad:** qué proporción de filas descarta un filtro; un filtro muy selectivo devuelve pocas filas.
* **Predicado residual:** condición que el motor evalúa fila por fila después del *seek*, porque el índice no permite buscarla directamente.

-----

## El problema

El índice `IX_Pedido_ClienteId` de la lección anterior resolvió la pantalla "mis pedidos" para los clientes comunes, pero no para el mayorista: con 50.000 filas, los *Key Lookups* eran tan caros que el optimizador volvía al scan de 4.500 páginas. Y la pantalla real es un poco más exigente:

```sql
SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE ClienteId = @ClienteId
ORDER BY FechaPedido DESC;
```

Hay que buscar por cliente, **ordenar** por fecha y **devolver** el estado. Un índice sobre `ClienteId` solo resuelve lo primero. Y el área de logística tiene otra consulta, que se ejecuta cada minuto:

```sql
SELECT PedidoId, ClienteId, FechaPedido
FROM dbo.Pedido
WHERE Estado = 'Pendiente';
```

Son 10.000 pedidos pendientes entre un millón. Un índice sobre `Estado` tendría un millón de entradas, el 85 % con el valor "Entregado", que nadie consulta.

-----

## Cómo funciona

### Índices compuestos y el prefijo izquierdo

Un índice compuesto ordena por la primera columna, después por la segunda dentro de cada valor de la primera, y así:

```sql
CREATE INDEX IX_Pedido_Cliente_Fecha ON dbo.Pedido (ClienteId, FechaPedido);
```

```text
(ClienteId, FechaPedido, → PedidoId)
(12345, 2023-04-02 ...)
(12345, 2024-01-15 ...)    ← los pedidos de un cliente, ya ordenados por fecha
(12345, 2025-09-30 ...)
(12346, 2023-02-11 ...)
```

Es como una guía telefónica ordenada por apellido y nombre: sirve para buscar por apellido, o por apellido y nombre, pero **no** para buscar solo por nombre, porque los "Juan" están repartidos por toda la guía.

| Filtro | ¿Puede hacer *seek*? |
| --- | --- |
| `WHERE ClienteId = 12345` | Sí (prefijo) |
| `WHERE ClienteId = 12345 AND FechaPedido >= '2025-01-01'` | Sí (las dos columnas) |
| `WHERE FechaPedido >= '2025-01-01'` | **No**: no puede saltarse `ClienteId`; recorre el índice entero |
| `WHERE ClienteId = 12345 ORDER BY FechaPedido` | Sí, y **sin ordenar**: las filas ya salen en orden de fecha |

Una consecuencia práctica: un índice sobre `(ClienteId, FechaPedido)` **hace innecesario** uno sobre `(ClienteId)` solo, porque cualquier búsqueda por `ClienteId` usa el prefijo. Tener los dos es duplicar el costo de escritura.

### El orden de las columnas: igualdad, orden, rango

La regla más útil para ordenar las columnas de la clave se conoce como **ESR** (*Equality, Sort, Range*):

1. **Igualdad (`=`)** primero.
2. **Orden** (las columnas del `ORDER BY`) después.
3. **Rango** (`>`, `<`, `BETWEEN`, `LIKE 'x%'`) al final.

¿Por qué el orden antes que el rango? Porque después de una columna de **rango**, las siguientes ya no están ordenadas. Con `WHERE Estado = @E AND FechaPedido >= @F ORDER BY ClienteId`:

| Índice | Qué hace el motor |
| --- | --- |
| `(Estado, FechaPedido, ClienteId)` (igualdad, rango, orden) | *Seek* exacto al rango, pero dentro del rango los `ClienteId` vienen ordenados **por fecha**, no por cliente: necesita un **Sort** |
| `(Estado, ClienteId, FechaPedido)` (igualdad, orden, rango) | *Seek* al estado, lee en orden de cliente (sin *Sort*) y descarta las fechas fuera de rango como **predicado residual** |

Ninguna de las dos es siempre mejor. ESR evita el ordenamiento a cambio de leer más filas; si el rango es muy selectivo (pocas filas) y ordenarlas es barato, `(igualdad, rango)` + *Sort* puede ganar. ESR es el punto de partida; el plan de ejecución decide.

Un mito frecuente es "la columna más selectiva primero". Entre columnas de **igualdad**, el orden casi no cambia el costo del *seek*: el motor llega directamente a la combinación de valores. Lo que sí cambia es qué **otras** consultas pueden usar el índice por su prefijo. Ordena las columnas de igualdad pensando en reutilizar el índice, no en la selectividad.

### `INCLUDE`: cubrir la consulta sin agrandar la clave

Las columnas que la consulta solo **devuelve** (no filtra ni ordena) no necesitan estar en la clave. Con `INCLUDE`, se guardan solo en las hojas:

```sql
CREATE INDEX IX_Pedido_Cliente_Fecha
    ON dbo.Pedido (ClienteId, FechaPedido DESC)
    INCLUDE (Estado);
```

| | Columnas clave | Columnas `INCLUDE` |
| --- | --- | --- |
| Dónde se guardan | En todos los niveles del árbol | Solo en las hojas |
| Ordenadas | Sí | No |
| Sirven para | `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY` | Solo para devolverlas en el `SELECT` |
| Límite | 16 columnas y 1.700 bytes (900 en un agrupado) | Hasta 1.023 columnas, sin límite de bytes de la clave |
| Tipos `MAX` | **No** pueden ser clave | **Sí** pueden incluirse |

Con este índice, la consulta de "mis pedidos" es **cubierta**: el índice tiene `ClienteId` (para buscar), `FechaPedido` en orden descendente (para el `ORDER BY` sin *Sort*), `Estado` (incluida) y `PedidoId` (el localizador, que todo índice no agrupado ya tiene). No hay *Key Lookup*, ni siquiera para el cliente mayorista.

No conviene incluir columnas "por si acaso": cada columna incluida se copia en el índice y lo agranda, y un índice con 15 columnas incluidas es casi una segunda copia de la tabla, con su costo en escrituras.

### Índices filtrados

Un índice filtrado contiene solo las filas que cumplen una condición:

```sql
CREATE INDEX IX_Pedido_Pendiente
    ON dbo.Pedido (FechaPedido)
    INCLUDE (ClienteId)
    WHERE Estado = 'Pendiente';
```

Para la consulta de logística, este índice tiene 10.000 entradas en lugar de un millón: es cien veces más chico, se lee más rápido y solo se actualiza cuando cambia un pedido pendiente.

Es la opción natural cuando:

* Las consultas buscan un subconjunto pequeño y estable: pendientes, activos, no eliminados (`WHERE EliminadoEn IS NULL`, del [borrado lógico](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)).
* La columna tiene muchos `NULL` y solo interesan los no nulos.
* Se quiere unicidad solo entre algunas filas (índice único filtrado).

Sus trampas:

* **La consulta tiene que implicar el filtro.** `WHERE Estado = 'Pendiente'` lo usa; `WHERE Estado IN ('Pendiente', 'Pagado')`, no.
* **Las consultas parametrizadas no lo usan:** con `WHERE Estado = @Estado`, el plan se compila para cualquier valor de `@Estado`, así que no puede depender de un índice que solo sirve para `'Pendiente'`. Hace falta el literal, o `OPTION (RECOMPILE)`.
* La condición del filtro solo admite comparaciones simples: no puede usar funciones ni referirse a columnas calculadas.

### Cuántos índices y cuáles

Antes de crear un índice, revisa los que ya existen:

| Si existe | Y quieres crear | Entonces |
| --- | --- | --- |
| `(A)` | `(A, B)` | Crea `(A, B)` y **elimina** `(A)`: el nuevo cubre sus búsquedas |
| `(A, B)` | `(A)` | No lo crees: `(A, B)` ya sirve |
| `(A) INCLUDE (X)` | `(A) INCLUDE (Y)` | Uno solo: `(A) INCLUDE (X, Y)` |
| `(A, B)` | `(B, A)` | Son distintos: sirven a consultas distintas |

Una tabla OLTP con 5 o 6 índices bien diseñados suele rendir mejor que una con 20 índices de una columna.

-----

## Ejemplo completo

Las dos consultas del problema, en `TiendaGrande`, partiendo del índice `IX_Pedido_ClienteId` de la lección anterior:

```sql
USE TiendaGrande;
SET STATISTICS IO ON;

-- "Mis pedidos" del mayorista, con el índice de una columna
SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE ClienteId = 1
ORDER BY FechaPedido DESC;
-- Clustered Index Scan + Sort: ~4.500 lecturas

-- Reemplazar el índice por uno compuesto y cubriente
DROP INDEX IX_Pedido_ClienteId ON dbo.Pedido;
CREATE INDEX IX_Pedido_Cliente_Fecha
    ON dbo.Pedido (ClienteId, FechaPedido DESC)
    INCLUDE (Estado);

SELECT PedidoId, FechaPedido, Estado
FROM dbo.Pedido
WHERE ClienteId = 1
ORDER BY FechaPedido DESC;
-- Index Seek, sin Sort ni Key Lookup: unas pocas centenas de lecturas

-- Logística: pedidos pendientes
SELECT PedidoId, ClienteId, FechaPedido FROM dbo.Pedido WHERE Estado = 'Pendiente';
-- Clustered Index Scan: ~4.500 lecturas

CREATE INDEX IX_Pedido_Pendiente
    ON dbo.Pedido (FechaPedido)
    INCLUDE (ClienteId)
    WHERE Estado = 'Pendiente';

SELECT PedidoId, ClienteId, FechaPedido FROM dbo.Pedido WHERE Estado = 'Pendiente';
-- Index Scan sobre IX_Pedido_Pendiente: unas decenas de lecturas

SET STATISTICS IO OFF;
```

| Consulta | Antes | Después |
| --- | --- | --- |
| "Mis pedidos", cliente 1 (50.000 filas) | Scan + Sort, ~4.500 lecturas | Seek ordenado, unos cientos de lecturas |
| "Mis pedidos", cliente común (~10 filas) | Seek + Lookups, ~33 lecturas | Seek, 3 lecturas |
| Pedidos pendientes (10.000 filas) | Scan, ~4.500 lecturas | Scan del índice filtrado, ~40 lecturas |

Qué decisión hay detrás:

* **`(ClienteId, FechaPedido DESC)`:** igualdad primero, orden después. El `DESC` en el índice no es imprescindible (SQL Server puede recorrer un índice hacia atrás), pero deja claro el uso previsto.
* **`INCLUDE (Estado)`:** `Estado` solo se devuelve, así que va en las hojas y no agranda la clave.
* **Se eliminó `IX_Pedido_ClienteId`:** el nuevo índice empieza por `ClienteId`, así que también sirve para el join con `Cliente` y para las verificaciones de la clave foránea. Mantener los dos sería pagar doble en cada escritura.
* **El índice filtrado es un "scan" y es una buena noticia:** recorre el índice entero, pero el índice entero son los 10.000 pendientes. No todo *scan* es malo: lo que importa es cuántas páginas lee.

-----

## Errores comunes

**1. Columnas en el orden equivocado.**
Qué pasa: un índice `(FechaPedido, ClienteId)` que no sirve para "los pedidos del cliente X".
Por qué: el *seek* solo funciona sobre el prefijo izquierdo.
Arreglo: igualdad, orden, rango, y verifica en el plan.

**2. Índices de una columna para consultas de varias.**
Qué pasa: varios índices de una columna, ninguno cubre la consulta, y el optimizador hace *lookups* o *scans*.
Por qué: se indexan columnas sueltas en lugar de consultas.
Arreglo: un índice compuesto, con `INCLUDE`, diseñado para la consulta frecuente.

**3. Índices redundantes.**
Qué pasa: `(A)` y `(A, B)` en la misma tabla, el doble de costo de escritura sin beneficio.
Por qué: se agregan índices sin revisar los existentes.
Arreglo: consolida; `(A, B)` cubre las búsquedas por `A`.

**4. Todo en `INCLUDE`.**
Qué pasa: un índice casi tan grande como la tabla, que encarece cada escritura.
Por qué: se busca cubrir todas las consultas con un solo índice.
Arreglo: cubre las consultas críticas; para las demás, un *lookup* ocasional es aceptable.

**5. Índice filtrado con consultas parametrizadas.**
Qué pasa: el índice existe y nunca se usa.
Por qué: el plan de `WHERE Estado = @Estado` tiene que servir para cualquier valor.
Arreglo: el literal en la consulta, `OPTION (RECOMPILE)`, o un índice sin filtro si el parámetro varía.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Índice compuesto y prefijo izquierdo | Sí | Sí | Sí |
| Saltar la primera columna | No (*scan*) | No (salvo *skip scan* en la 18) | *Skip scan* desde 8.0.13, en casos limitados |
| `INCLUDE` | Sí | Sí, desde la 11 | **No**: para cubrir, todas las columnas van en la clave |
| Índice filtrado / parcial | `WHERE` (filtrado) | `WHERE` (parcial), muy flexible, admite expresiones | **No** |
| Índice sobre una expresión | Columna calculada + índice | Índice de expresión: `CREATE INDEX ON t (lower(email))` | Índice funcional (desde 8.0.13) |
| Consulta cubierta en el plan | Sin Key Lookup | *Index Only Scan* | `Using index` en `EXPLAIN` |

En PostgreSQL, un *Index Only Scan* además depende del mapa de visibilidad (que `VACUUM` mantiene): si la tabla tiene muchas filas modificadas recientemente, el motor igual tiene que visitar la tabla.

### En MongoDB

Los índices compuestos de MongoDB siguen las mismas reglas, y de hecho la regla **ESR** (*Equality, Sort, Range*) es la recomendación oficial de MongoDB para ordenar sus campos. Una consulta es cubierta (*covered query*) si todos los campos del filtro y de la proyección están en el índice y se excluye `_id` (salvo que esté en el índice). Los **índices parciales** (`partialFilterExpression`) son el equivalente de los filtrados:

```javascript
db.pedidos.createIndex(
  { fechaPedido: 1 },
  { partialFilterExpression: { estado: "Pendiente" } }
);
```

-----

## Cuándo sí y cuándo no

**Usa un índice compuesto cuando:**

* Una consulta frecuente filtra por varias columnas, o filtra por una y ordena por otra.

**Usa `INCLUDE` cuando:**

* Una consulta frecuente devuelve pocas columnas extra y sufre *Key Lookups* costosos.

**Usa un índice filtrado cuando:**

* Las consultas siempre buscan el mismo subconjunto pequeño (pendientes, activos, no nulos) con un valor literal.

**Ten en cuenta:**

* Diseña índices para las consultas **más frecuentes y más costosas** (Query Store y las DMV del módulo de monitoreo te dicen cuáles son), no para todas.

-----

## Resumen en 5 líneas

1. Un índice compuesto solo permite *seek* sobre su prefijo izquierdo: `(A, B)` sirve para `A` y para `A + B`, no para `B` sola.
2. Orden de la clave: igualdad, orden, rango (ESR); después de una columna de rango, las siguientes ya no están ordenadas.
3. `INCLUDE` agrega columnas solo en las hojas para cubrir la consulta y evitar los Key Lookups, sin agrandar la clave.
4. Un índice filtrado indexa solo un subconjunto; la consulta tiene que implicar el filtro, y las parametrizadas no lo usan.
5. Consolida índices redundantes: `(A, B)` reemplaza a `(A)`.

-----

## Para profundizar

<details>
<summary>Índices columnstore</summary>

Todos los índices de este módulo son de **filas** (*rowstore*): cada página guarda filas completas o entradas de índice. Un índice **columnstore** guarda cada **columna** por separado, comprimida, en segmentos de hasta ~1 millón de filas. Para consultas analíticas que leen pocas columnas de muchísimas filas (`SUM(Total)` agrupado por mes sobre 500 millones de filas), es varias veces más rápido y ocupa una fracción del espacio, porque lee solo las columnas necesarias y procesa las filas en lotes (*batch mode*).

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_DetallePedido
    ON dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario);
```

Es malo para buscar una fila concreta o para tablas con muchas actualizaciones fila por fila. Es la herramienta para reportes y almacenes de datos, y se puede combinar con índices de filas en la misma tabla.

</details>

<details>
<summary>Selectividad, densidad y por qué un índice sobre un BIT rara vez sirve</summary>

La selectividad de un filtro es la fracción de filas que devuelve. `WHERE Activo = 1` sobre una tabla donde el 95 % está activo devuelve casi todo: un índice sobre `Activo` no sirve, porque recorrer la tabla es más barato que hacer millones de *lookups*. Pero `WHERE Activo = 0` (el 5 %) podría beneficiarse de un **índice filtrado** `WHERE Activo = 0`. La regla no es "nunca indexar columnas con pocos valores", sino "indexar el valor poco frecuente que se consulta".

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un índice compuesto tiene varias columnas, y solo sirve si la consulta filtra por las primeras. `INCLUDE` agrega columnas al índice para que la consulta no tenga que ir a la tabla, lo que se llama un índice cubriente. Un índice filtrado solo indexa las filas que cumplen una condición, como los pedidos pendientes.

### Respuesta ampliada (semi-senior)

Diseño los índices para consultas concretas. En un índice compuesto el *seek* solo funciona sobre el prefijo izquierdo, así que ordeno la clave con la regla ESR: igualdad, después las columnas del `ORDER BY` y por último los rangos, porque después de un rango las columnas siguientes ya no están ordenadas; es un punto de partida que verifico en el plan, porque si el rango es muy selectivo puede convenir más ordenar. Entre columnas de igualdad no ordeno por selectividad sino pensando en qué otras consultas pueden reutilizar el prefijo. Las columnas que solo se devuelven van en `INCLUDE`, que las guarda en las hojas, evita los Key Lookups y no agranda la clave, sin incluir de más porque cada columna encarece las escrituras. Uso índices filtrados para subconjuntos chicos y estables, sabiendo que la consulta tiene que implicar el filtro y que las consultas parametrizadas no los usan. Y consolido los redundantes: un `(A, B)` reemplaza a un `(A)`.

### Preguntas frecuentes de seguimiento

**1. ¿Un índice sobre `(A, B)` sirve para `WHERE B = 1`?**
No para un *seek*: el índice está ordenado primero por `A`. A lo sumo, el motor lo recorre entero.

**2. ¿Qué diferencia hay entre una columna clave y una de `INCLUDE`?**
La clave está en todos los niveles del árbol, ordenada, y sirve para buscar y ordenar; la de `INCLUDE` está solo en las hojas y solo sirve para devolverla.

**3. ¿Qué es un índice cubriente?**
Uno que contiene todas las columnas que la consulta necesita, de modo que no hace falta ir a la tabla.

**4. ¿Cuándo no se usa un índice filtrado?**
Cuando la consulta no implica el filtro, por ejemplo cuando el valor viene en un parámetro, o cuando usa un `IN` con otros valores.

-----

## Práctica

**Ejercicio 1.** Diseña el índice para esta consulta frecuente de `TiendaGrande`, y explica el orden de las columnas:

```sql
SELECT PedidoId, FechaPedido
FROM dbo.Pedido
WHERE Estado = 'Enviado'
  AND FechaPedido >= '2026-01-01'
ORDER BY FechaPedido;
```

<details>
<summary>Solución</summary>

```sql
CREATE INDEX IX_Pedido_Estado_Fecha ON dbo.Pedido (Estado, FechaPedido);
```

* **`Estado`** primero: es la igualdad.
* **`FechaPedido`** después: es a la vez el rango y el orden. Como es la misma columna, no hay conflicto entre "orden" y "rango": el *seek* llega al primer pedido enviado de 2026 y lee en orden de fecha, sin *Sort*.
* **`PedidoId`** no hace falta: es el localizador del índice (la clave del agrupado).

Alternativa: si casi todas las consultas son sobre `'Enviado'` con un literal, un índice filtrado `ON dbo.Pedido (FechaPedido) WHERE Estado = 'Enviado'` sería más chico. Pero si `Estado` llega como parámetro, el filtrado no se usaría.

</details>

**Ejercicio 2.** Una tabla tiene estos índices. ¿Cuáles eliminarías o consolidarías, y por qué?

```text
IX_1 (ClienteId)
IX_2 (ClienteId, FechaPedido)
IX_3 (FechaPedido)
IX_4 (ClienteId) INCLUDE (Estado)
IX_5 (FechaPedido, ClienteId)
```

<details>
<summary>Solución</summary>

* **`IX_1`: eliminar.** `IX_2` empieza por `ClienteId` y cubre las mismas búsquedas.
* **`IX_4`: consolidar con `IX_2`** como `(ClienteId, FechaPedido) INCLUDE (Estado)`. Así cubre las consultas que necesitaban `Estado` y las búsquedas por cliente y fecha, con un solo índice.
* **`IX_3`: eliminar.** `IX_5` empieza por `FechaPedido` y cubre sus búsquedas.
* **`IX_5`: conservar** (como reemplazo de `IX_3`) solo si hay consultas que filtran por fecha sin cliente. Si no las hay, eliminarlo también.

Resultado: de 5 índices a 1 o 2. Antes de eliminar en producción, verifica el uso real con `sys.dm_db_index_usage_stats` (ver [Mantenimiento de índices](07-Mantenimiento%20de%20índices.md)): un índice "redundante" puede estar sosteniendo una restricción `UNIQUE` o una pista de índice (`WITH (INDEX(...))`) en alguna consulta.

</details>

-----

## Siguiente lección

[Planes de ejecución I](04-Planes%20de%20ejecución%20I.md)
