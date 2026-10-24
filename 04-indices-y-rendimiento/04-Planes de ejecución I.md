# Planes de ejecución I

## En una frase

El plan de ejecución es la "receta" que el optimizador eligió para resolver una consulta (qué índices usa, en qué orden une las tablas, dónde ordena o agrega), y leerlo, junto con `SET STATISTICS IO` y `TIME`, es la forma de saber **por qué** una consulta es lenta en lugar de adivinarlo.

-----

## Antes de empezar

Conviene que ya sepas:

* Seek, scan, Key Lookup e índices cubrientes, de [Índices clustered y nonclustered](02-Índices%20clustered%20y%20nonclustered.md) y [Índices compuestos, cubrientes y filtrados](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md).
* Que el orden lógico de una consulta no es su orden físico, de [SELECT y el orden lógico](../03-consultas/01-SELECT%20y%20el%20orden%20lógico.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Optimizador de consultas:** el componente que, para cada consulta, evalúa planes posibles y elige el de menor costo estimado.
* **Plan estimado:** el plan que el optimizador **piensa** usar, sin ejecutar la consulta.
* **Plan real (actual):** el plan usado en una ejecución, con las filas y tiempos reales.
* **Operador:** cada paso del plan (Index Seek, Hash Match, Sort...).
* **Costo estimado:** una medida abstracta (sin unidades) que el optimizador usa para comparar planes.
* **Predicado de búsqueda (Seek Predicate) / predicado residual (Predicate):** la condición que el índice resuelve directamente / la que se evalúa fila por fila después.
* **Memory grant:** la memoria que se reserva para una consulta antes de ejecutarla (para ordenar o armar tablas hash).

-----

## El problema

Una consulta del reporte comercial tarda 4 segundos. Tres personas del equipo opinan:

* "Falta un índice en `Pedido`."
* "Es el `GROUP BY`, los agregados son lentos."
* "Hay que pasarla a un procedimiento almacenado."

Las tres pueden tener razón o estar completamente equivocadas, y no hay forma de saberlo **opinando**. Crear un índice al azar puede no cambiar nada (y encarecer las escrituras); reescribir la consulta puede dejarla igual. El plan de ejecución responde la pregunta de verdad: **¿en qué parte de la consulta se va el tiempo?**

-----

## Cómo funciona

### Plan estimado y plan real

| | Plan estimado (`Ctrl` + `L`) | Plan real (`Ctrl` + `M`, después `F5`) |
| --- | --- | --- |
| ¿Ejecuta la consulta? | **No** | **Sí** |
| Filas | Estimadas (según las estadísticas) | Estimadas **y** reales |
| Útil para | Ver el plan de una consulta peligrosa o muy lenta sin ejecutarla | Diagnosticar: comparar lo estimado con lo real |
| Incluye | Costos estimados | Además: filas reales, ejecuciones, tiempos, memoria usada, advertencias de ejecución |

El plan real es el que se usa para diagnosticar. Un `UPDATE` sobre producción, en cambio, se revisa con el plan **estimado**, que no lo ejecuta.

`Ctrl` + `M` es un interruptor: activa "incluir el plan real" para las siguientes ejecuciones, que aparecen en una pestaña *Execution plan* junto a los resultados.

### Cómo se lee un plan

Un plan gráfico es un árbol de operadores. Se lee **de derecha a izquierda y de arriba hacia abajo**: los datos nacen en los operadores de la derecha (los que leen tablas e índices) y fluyen hacia la izquierda, hasta el `SELECT` final.

```text
SELECT ← Hash Match (Aggregate) ← Hash Match (Inner Join) ← Clustered Index Scan (Pedido)
                                                          ↖
                                                            Clustered Index Scan (DetallePedido)
         └──── los datos fluyen de derecha a izquierda ────┘
```

Tres cosas se ven de un vistazo:

* **El costo relativo** de cada operador (un porcentaje del total), debajo de su ícono.
* **El grosor de las flechas**: es proporcional a la cantidad de filas que pasan. Una flecha muy gruesa que termina en un operador que la reduce a pocas filas es una señal de trabajo desperdiciado.
* **Los triángulos amarillos de advertencia** sobre un operador.

Y al pasar el mouse (o con `F4`, el panel de propiedades), los detalles de cada operador: filas estimadas y reales, cantidad de ejecuciones, predicados, columnas de salida.

### Operadores de acceso a datos

| Operador | Qué hace | Cuándo es un problema |
| --- | --- | --- |
| **Table Scan** | Recorre un heap completo | Casi siempre, en tablas de negocio |
| **Clustered Index Scan** | Recorre toda la tabla (el agrupado) | Cuando devuelve pocas filas: falta un índice o el filtro no es SARGable |
| **Index Scan** | Recorre un índice no agrupado completo | Igual; aunque si el índice es angosto o filtrado, puede ser lo correcto |
| **Clustered Index Seek / Index Seek** | Navega el árbol hasta las filas | Rara vez; salvo que se ejecute millones de veces |
| **Key Lookup / RID Lookup** | Busca en la tabla las columnas que faltan | Cuando se ejecuta miles de veces |

La regla del cheat sheet de tus apuntes es correcta: **no todo *scan* es malo ni todo *seek* es bueno**. Un *scan* de un índice filtrado de 40 páginas es excelente; un *seek* que se ejecuta 50.000 veces dentro de un *Nested Loops*, con un *Key Lookup* en cada una, puede ser lo más caro del plan.

### Operadores de join

| Operador | Cómo funciona | Bueno cuando | Señal de alerta |
| --- | --- | --- | --- |
| **Nested Loops** | Para cada fila de la entrada de arriba, busca sus parejas en la de abajo | La entrada de arriba tiene pocas filas y la de abajo tiene un índice | Muchas filas arriba: la búsqueda de abajo se repite miles de veces |
| **Merge Join** | Recorre las dos entradas ordenadas por la clave, como un cierre | Las dos llegan ordenadas (por un índice) | Necesita un *Sort* previo costoso |
| **Hash Match** | Arma una tabla hash con una entrada y la recorre con la otra | Entradas grandes sin índices útiles | Mucha memoria; si no alcanza, *spill* a `tempdb` |

Ningún algoritmo es "el bueno": el optimizador elige según la cantidad estimada de filas. Por eso un *Hash Match* donde esperabas un *Nested Loops* suele significar que el optimizador **estimó** muchas filas, y la pregunta pasa a ser si la estimación es correcta.

### Otros operadores frecuentes

| Operador | Qué hace |
| --- | --- |
| **Sort** | Ordena (para un `ORDER BY`, un *Merge Join* o un agregado). Costoso con muchas filas; un índice en el orden correcto lo evita |
| **Stream Aggregate / Hash Match (Aggregate)** | Calcula un `GROUP BY`. El primero necesita la entrada ordenada; el segundo arma una tabla hash |
| **Compute Scalar** | Calcula una expresión. Casi siempre barato |
| **Filter** | Descarta filas después de leerlas. Si filtra muchas, el filtro no se pudo aplicar antes |
| **Top** | Corta en las primeras n filas (`TOP`, `EXISTS`) |
| **Parallelism** | Reparte o reúne el trabajo entre varios hilos |

### Predicado de búsqueda frente a predicado residual

En las propiedades de un *Index Seek* aparecen dos cosas distintas:

* **Seek Predicate:** la parte del filtro que el índice resuelve navegando el árbol.
* **Predicate:** la parte que se comprueba **después**, fila por fila, sobre lo que el *seek* encontró.

Con el índice `(Estado, ClienteId, FechaPedido)` y `WHERE Estado = 'Enviado' AND FechaPedido >= '2026-01-01'`, el *Seek Predicate* es `Estado = 'Enviado'` y el *Predicate* es la fecha: el motor lee **todos** los pedidos enviados y descarta los anteriores a 2026. Si el *seek* devuelve 50.000 filas y el predicado residual deja 500, el índice no está bien ordenado para esa consulta.

### Estimado frente a real

En el plan real, cada operador tiene *Estimated Number of Rows* y *Actual Number of Rows*. Cuando difieren mucho (10 veces o más), el optimizador eligió el plan con información equivocada:

* Estimó 10 filas y fueron 50.000: eligió *Nested Loops* y *lookups* para lo que debió ser un *scan* o un *Hash Match*.
* Estimó 50.000 y fueron 10: reservó memoria de más y eligió un *Hash Match* innecesario.

Las causas habituales son estadísticas desactualizadas, *parameter sniffing*, variables de tabla, funciones escalares y predicados complejos. Se estudian en [Planes de ejecución II](05-Planes%20de%20ejecución%20II.md).

### El costo es una estimación, no una medición

El costo que muestra el plan (en el operador y en el `SELECT` raíz, *Estimated Subtree Cost*) es una **unidad abstracta**: no son segundos ni lecturas. Sirve para comparar operadores del mismo plan, y aun así está calculado con las filas **estimadas**. Si la estimación es mala, el operador con "2 %" puede ser el que consume el 90 % del tiempo real. Por eso el costo se contrasta siempre con mediciones.

### `SET STATISTICS IO` y `SET STATISTICS TIME`

```sql
SET STATISTICS IO, TIME ON;
-- consulta
SET STATISTICS IO, TIME OFF;
```

En la pestaña **Mensajes**:

```text
Table 'DetallePedido'. Scan count 1, logical reads 9312, physical reads 0, read-ahead reads 0, ...
Table 'Pedido'. Scan count 1, logical reads 4512, physical reads 0, ...

 SQL Server Execution Times:
   CPU time = 812 ms,  elapsed time = 1204 ms.
```

| Métrica | Qué dice |
| --- | --- |
| **Logical reads** | Páginas leídas (desde la memoria). **La métrica principal**: el trabajo de la consulta |
| Physical reads / read-ahead reads | Páginas traídas del disco. Dependen de la caché, no de la consulta |
| Scan count | Cuántas veces se recorrió o buscó en el objeto |
| CPU time | Tiempo de procesador consumido (sumando todos los hilos) |
| Elapsed time | Tiempo total. Si es mucho mayor que el CPU, la consulta **esperó** (disco, bloqueos, red) |

Si con paralelismo el *CPU time* supera al *elapsed time*, varios hilos trabajaron a la vez; es normal, pero si es por mucho en una consulta chica, el paralelismo puede estar costando más de lo que aporta.

El sitio statisticsparser.com convierte esta salida en una tabla legible, útil con consultas que tocan muchas tablas.

### Advertencias en el plan

| Advertencia | Qué significa | Qué hacer |
| --- | --- | --- |
| *Missing Index* (texto verde arriba del plan) | El optimizador cree que un índice ayudaría | Evaluarlo: es una sugerencia, no una orden (ver [Mantenimiento de índices](07-Mantenimiento%20de%20índices.md)) |
| *Type conversion ... may affect "SeekPlan"* | Una conversión implícita impide un *seek* | Igualar el tipo del parámetro y la columna |
| *No Join Predicate* | Un join sin condición: producto cartesiano | Revisar el `ON` |
| *Operator used tempdb to spill data* | Un *Sort* o *Hash* no tuvo memoria suficiente y usó el disco | Mejorar la estimación (estadísticas) o evitar el operador (índice) |
| *Excessive Grant* | Se reservó mucha más memoria de la usada | Estimación inflada |
| *Residual I/O* (en el plan real) | Se leyeron muchas más filas de las que pasaron el predicado | Mejorar el índice |

-----

## Ejemplo completo

El reporte comercial: facturación de 2026 por categoría, sin pedidos cancelados, en `TiendaGrande`.

```sql
USE TiendaGrande;
SET STATISTICS IO, TIME ON;
-- Ctrl + M para incluir el plan real, después F5

SELECT c.Nombre AS Categoria,
       SUM(d.Cantidad * d.PrecioUnitario) AS Facturado
FROM dbo.Pedido AS p
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
JOIN dbo.Categoria AS c ON c.CategoriaId = pr.CategoriaId
WHERE p.FechaPedido >= '2026-01-01' AND p.FechaPedido < '2027-01-01'
  AND p.Estado <> 'Cancelado'
GROUP BY c.Nombre
ORDER BY Facturado DESC;
```

**Lectura del plan** (de derecha a izquierda), en forma resumida:

```text
SELECT
 └─ Sort (ORDER BY Facturado)
     └─ Hash Match (Aggregate) — GROUP BY c.Nombre
         └─ Hash Match (Inner Join) — con Producto y Categoria
             └─ Hash Match (Inner Join) — Pedido ⋈ DetallePedido
                 ├─ Clustered Index Scan (Pedido)          ← ~4.500 lecturas, filtra fecha y estado
                 │     Predicate: FechaPedido >= ... AND Estado <> 'Cancelado'
                 └─ Clustered Index Scan (DetallePedido)   ← ~9.000 lecturas, lee las 2,5 M líneas
```

**Diagnóstico:**

1. **Los dos operadores de la derecha concentran casi todo el costo.** `Pedido` se recorre entero para quedarse con ~14 % de las filas (medio año de 3,5 años): el filtro de fecha aparece como *Predicate* de un *scan*, no como *Seek Predicate*. Falta un índice por fecha.
2. **`DetallePedido` se recorre entero** aunque solo interesan las líneas de los pedidos de 2026: como la entrada de `Pedido` es grande, el optimizador eligió *Hash Match* en lugar de buscar las líneas pedido por pedido.

**Cambio:**

```sql
CREATE INDEX IX_Pedido_Fecha ON dbo.Pedido (FechaPedido) INCLUDE (Estado);
```

**Después:** `Pedido` pasa a un *Index Seek* sobre `IX_Pedido_Fecha` (el rango de fechas como *Seek Predicate*, el estado como *Predicate* residual), con unas pocas centenas de lecturas en lugar de 4.500. El join con `DetallePedido` puede seguir siendo un *Hash Match* con *scan* (son ~350.000 líneas, una parte considerable), pero el total de lecturas y de CPU baja.

Qué muestra el ejemplo:

* **El plan dice dónde está el trabajo:** antes de crear el índice, se sabía qué tabla y qué filtro lo justificaban.
* **`INCLUDE (Estado)`** permite aplicar el filtro de estado sin *Key Lookups*.
* **No todo se resuelve con un índice:** si el reporte suma una fracción grande de los detalles, leerlos tiene un costo inevitable. Para reportes recurrentes sobre volúmenes grandes, las opciones siguientes son un índice columnstore o una tabla de resumen (ver [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md)).

-----

## Errores comunes

**1. Optimizar sin mirar el plan.**
Qué pasa: índices creados al azar, reescrituras que no cambian nada.
Por qué: se adivina la causa.
Arreglo: plan real + `STATISTICS IO` antes de cambiar nada, y otra vez después.

**2. Creer en el porcentaje de costo como si fuera tiempo.**
Qué pasa: se optimiza el operador "del 60 %" y la consulta sigue igual de lenta.
Por qué: el costo es una estimación basada en filas estimadas.
Arreglo: comparar filas estimadas y reales, y medir con lecturas y tiempos.

**3. Leer el plan de izquierda a derecha.**
Qué pasa: se interpreta el `SELECT` como el primer paso.
Por qué: es el orden de lectura natural en español.
Arreglo: los datos nacen a la derecha y fluyen hacia la izquierda.

**4. Comparar tiempos con distinta caché.**
Qué pasa: "con el índice tarda 200 ms y sin él 3 segundos", pero la primera ejecución leyó del disco.
Por qué: las lecturas físicas solo ocurren la primera vez.
Arreglo: comparar lecturas lógicas.

**5. Ignorar las advertencias.**
Qué pasa: una conversión implícita o un *spill* que explican toda la lentitud quedan sin ver.
Por qué: el triángulo amarillo es pequeño.
Arreglo: revisa siempre el operador `SELECT` raíz (que resume las advertencias) y los triángulos.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Plan estimado | `Ctrl` + `L`, `SET SHOWPLAN_XML ON` | `EXPLAIN` | `EXPLAIN` |
| Plan real | `Ctrl` + `M`, `SET STATISTICS XML ON` | `EXPLAIN (ANALYZE, BUFFERS)` | `EXPLAIN ANALYZE` (desde 8.0.18) |
| Lecturas de páginas | `SET STATISTICS IO ON` | `BUFFERS` en el `EXPLAIN` (*shared hit/read*) | No directamente (contadores de `SHOW STATUS`) |
| Formato del plan | Gráfico y XML | Texto, JSON, YAML | Árbol en texto (`FORMAT=TREE`), JSON |
| Visualizadores | SSMS, Plan Explorer | explain.dalibo.com, explain.depesz.com | MySQL Workbench (*Visual Explain*) |
| Nombres de operadores | Index Seek, Key Lookup, Hash Match | Index Scan, Bitmap Heap Scan, Hash Join | `ref`, `range`, `ALL` (en la columna `type`) |

En PostgreSQL, un `Index Scan` es lo que en SQL Server sería un *Index Seek* + *lookup* (navega el índice y visita la tabla); un `Seq Scan` es el *Table Scan*.

### En MongoDB

`explain("executionStats")` muestra el plan elegido por etapas: `COLLSCAN` (recorre toda la colección, como un *scan*), `IXSCAN` (usa un índice), `FETCH` (trae el documento, como un *Key Lookup*) y `SORT` (ordena en memoria). Las métricas clave son `totalKeysExamined`, `totalDocsExamined` y `nReturned`: si se examinan 50.000 documentos para devolver 10, falta un índice o está mal ordenado.

```javascript
db.pedidos.find({ clienteId: 12345 }).sort({ fechaPedido: -1 }).explain("executionStats");
```

-----

## Cuándo sí y cuándo no

**Mira el plan real cuando:**

* Una consulta es lenta o consume muchos recursos: siempre, antes de cambiar nada.
* Creas un índice: para confirmar que se usa y cómo.

**Usa el plan estimado cuando:**

* La consulta modifica datos o tarda demasiado como para ejecutarla solo para ver el plan.

**Ten en cuenta:**

* Incluir el plan real agrega un pequeño costo a la ejecución. En producción, los planes se obtienen de la caché o de Query Store (módulo de monitoreo), no ejecutando consultas a ciegas.

**Herramientas que ayudan:**

* **SolarWinds Plan Explorer** (gratuito): muestra el plan con las filas reales en cada flecha y ordena los operadores por costo real.
* **Live Query Statistics** en SSMS: muestra el plan mientras la consulta se ejecuta, con el avance de cada operador.

-----

## Resumen en 5 líneas

1. El plan muestra cómo se resuelve la consulta; se lee de derecha a izquierda, siguiendo el grosor de las flechas.
2. Plan estimado (`Ctrl` + `L`) sin ejecutar; plan real (`Ctrl` + `M`) con filas reales para diagnosticar.
3. No todo *scan* es malo ni todo *seek* es bueno: importa cuántas páginas se leen y cuántas veces se ejecuta cada operador.
4. Filas estimadas muy distintas de las reales significan que el optimizador eligió con información equivocada.
5. El costo es una estimación abstracta: mide siempre con `SET STATISTICS IO, TIME ON` y compara lecturas lógicas.

-----

## Para profundizar

<details>
<summary>Cómo elige el optimizador</summary>

El optimizador de SQL Server es *cost-based*: genera planes alternativos (distintos órdenes de join, algoritmos, índices), estima el costo de cada uno con las estadísticas y elige el más barato que encuentra. No los evalúa todos: con 10 tablas hay millones de órdenes de join posibles, así que trabaja por fases y se detiene cuando encuentra un plan "suficientemente bueno" (en las propiedades del `SELECT`, *Reason for Early Termination: Good Enough Plan Found* o *Time Out*). Un *Time Out* en consultas muy complejas significa que pudo haber un plan mejor que no llegó a evaluar; partir la consulta en pasos (con tablas temporales) a veces ayuda.

</details>

<details>
<summary>El plan en XML</summary>

Todo plan gráfico es un XML por debajo (clic derecho → *Show Execution Plan XML*). Se puede guardar como `.sqlplan` y abrir en otra máquina, lo que es útil para compartir un plan con alguien sin darle acceso a la base. También se puede obtener desde la caché con `sys.dm_exec_query_plan`, como en las consultas de tus apuntes de la sesión 7, y desde Query Store.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El plan de ejecución muestra cómo SQL Server resuelve una consulta: qué índices usa, si hace un seek o un scan, cómo une las tablas. Lo veo en SSMS con `Ctrl` + `M` y lo combino con `SET STATISTICS IO ON` para ver cuántas páginas lee. Un scan sobre una tabla grande cuando la consulta devuelve pocas filas suele indicar que falta un índice.

### Respuesta ampliada (semi-senior)

Antes de optimizar, miro el plan real y `STATISTICS IO, TIME`. Leo el plan de derecha a izquierda, siguiendo las flechas más gruesas y los operadores con más costo, pero contrasto el costo, que es una estimación abstracta, con las filas estimadas y reales de cada operador: una diferencia grande indica estadísticas desactualizadas, *parameter sniffing* o predicados que el optimizador no puede estimar, y explica elecciones como un *Nested Loops* con miles de *lookups*. Distingo *Seek Predicate* de predicado residual para saber si el índice está bien ordenado, reviso las advertencias (conversiones implícitas, *spills*, joins sin predicado) y evalúo los *missing index* como sugerencias, no como órdenes. Comparo lecturas lógicas antes y después de cada cambio, porque los tiempos dependen de la caché, y en producción obtengo los planes de la caché o de Query Store en lugar de ejecutar consultas para verlos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre el plan estimado y el real?**
El estimado no ejecuta la consulta y muestra filas estimadas; el real se obtiene al ejecutarla e incluye filas reales, ejecuciones, memoria y advertencias de ejecución.

**2. ¿Un Index Scan siempre es malo?**
No. Si la consulta devuelve gran parte de la tabla, o si el índice es angosto o filtrado, un scan puede ser lo más eficiente.

**3. ¿Qué indica que las filas estimadas sean muy distintas de las reales?**
Que el optimizador eligió el plan con información equivocada; las causas típicas son estadísticas desactualizadas y *parameter sniffing*.

**4. ¿Qué métrica usas para comparar dos versiones de una consulta?**
Las lecturas lógicas de `SET STATISTICS IO`, porque miden el trabajo independientemente de la caché.

-----

## Práctica

**Ejercicio 1.** En `TiendaGrande`, obtén el plan real de esta consulta e identifica: el operador de acceso a `Cliente`, el operador de join y si hay advertencias.

```sql
SELECT c.Nombre, COUNT(*) AS Pedidos
FROM dbo.Cliente AS c
JOIN dbo.Pedido AS p ON p.ClienteId = c.ClienteId
WHERE c.Email = N'cliente500@correo.com'
GROUP BY c.Nombre;
```

<details>
<summary>Solución</summary>

* **Acceso a `Cliente`:** *Index Seek* sobre el índice de `UQ_Cliente_Email` (la restricción única creó un índice), seguido de un *Key Lookup* al agrupado para traer `Nombre`. Es una sola fila, así que el *lookup* cuesta 3 lecturas.
* **Join:** *Nested Loops*: la entrada de arriba tiene una fila (un cliente), y por cada una se buscan sus pedidos. Si existe un índice que empiece por `Pedido.ClienteId` (de la lección anterior), es un *Index Seek*; si no, un *Clustered Index Scan* de `Pedido` dentro del join.
* **Advertencias:** si no hay índice sobre `Pedido.ClienteId`, probablemente aparezca la sugerencia *Missing Index* sobre `(ClienteId)`.

Prueba también con `WHERE c.Email = 'cliente500@correo.com'` (sin la `N`): como el literal `VARCHAR` se convierte a `NVARCHAR` (el tipo de la columna), **no** hay advertencia de conversión. La advertencia aparece en el caso inverso: columna `VARCHAR` comparada con un valor `NVARCHAR`.

</details>

**Ejercicio 2.** Una consulta muestra este resumen de `STATISTICS IO, TIME`. ¿Qué te dice, y qué mirarías después?

```text
Table 'Pedido'. Scan count 50000, logical reads 160000, physical reads 0, ...
 SQL Server Execution Times:
   CPU time = 310 ms,  elapsed time = 6200 ms.
```

<details>
<summary>Solución</summary>

* **`Scan count 50000` con 160.000 lecturas:** se buscó en `Pedido` 50.000 veces, unas 3 lecturas cada una. Es el patrón de un *Nested Loops* (o de *Key Lookups*) ejecutado una vez por cada fila de otra entrada. Con 4.500 páginas en la tabla, recorrerla una vez habría sido 35 veces más barato.
* **`elapsed` (6,2 s) mucho mayor que `CPU` (0,3 s):** la consulta pasó casi todo el tiempo **esperando**, no trabajando. Con `physical reads 0`, no fue el disco: lo más probable es un bloqueo (otra transacción tenía filas bloqueadas) o el envío de resultados al cliente.

Siguiente paso: el plan real, para ver qué operador se ejecuta 50.000 veces y comparar sus filas estimadas con las reales (probablemente el optimizador estimó muy pocas filas en la entrada externa); y, para la espera, revisar bloqueos con las DMV de [Bloqueos](../05-transacciones-y-concurrencia/03-Bloqueos%2C%20hints%20y%20NOLOCK.md).

</details>

-----

## Siguiente lección

[Planes de ejecución II](05-Planes%20de%20ejecución%20II.md)
