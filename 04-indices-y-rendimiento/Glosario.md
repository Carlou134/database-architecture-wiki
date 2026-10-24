# Glosario: Índices y rendimiento

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Árbol B (B-tree):** estructura de árbol balanceado con la que se organizan los índices: raíz, niveles intermedios y hojas. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Buffer pool:** la memoria donde SQL Server guarda las páginas leídas. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Caché de planes:** memoria donde se guardan los planes compilados para reutilizarlos. ([05](05-Planes%20de%20ejecución%20II.md))

**Consulta "para todo" (catch-all):** consulta con filtros opcionales del tipo `@p IS NULL OR Col = @p`. ([06](06-Consultas%20SARGables%20y%20antipatrones.md))

**Conversión implícita:** conversión de tipo que agrega el motor; si convierte la columna, impide el *seek*. ([06](06-Consultas%20SARGables%20y%20antipatrones.md))

**Costo estimado:** medida abstracta que el optimizador usa para comparar planes; no son segundos ni lecturas. ([04](04-Planes%20de%20ejecución%20I.md))

**Densidad de página:** qué tan llenas están las páginas; su baja es la "fragmentación interna". ([07](07-Mantenimiento%20de%20índices.md))

**Distribución sesgada (skew):** cuando unos valores aparecen muchísimo más que otros. ([00](00-Datos%20de%20volumen.md))

**División de página (page split):** partir una página llena en dos para insertar una fila en el medio. ([07](07-Mantenimiento%20de%20índices.md))

**ESR (Equality, Sort, Range):** regla para ordenar las columnas de un índice compuesto: igualdad, orden y rango. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Estadísticas:** resumen de la distribución de los valores de una o varias columnas, usado para estimar filas. ([05](05-Planes%20de%20ejecución%20II.md))

**Estimación de cardinalidad:** el cálculo de cuántas filas devuelve cada operador. ([05](05-Planes%20de%20ejecución%20II.md))

**Extensión (extent):** 8 páginas contiguas, 64 KB. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Fill factor:** porcentaje de cada página hoja que se llena al crear o reconstruir un índice. ([07](07-Mantenimiento%20de%20índices.md))

**Fragmentación lógica:** desorden entre el orden lógico de las páginas y su orden físico. ([07](07-Mantenimiento%20de%20índices.md))

**Hash Match:** operador que arma una tabla hash para unir o agregar; bueno para entradas grandes sin índices. ([04](04-Planes%20de%20ejecución%20I.md))

**Heap:** tabla sin índice agrupado; las filas no tienen orden y se identifican por su RID. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Hint:** instrucción que fuerza una decisión del optimizador. ([05](05-Planes%20de%20ejecución%20II.md))

**Histograma:** parte de las estadísticas con hasta 200 tramos de la primera columna y la cantidad de filas de cada uno. ([05](05-Planes%20de%20ejecución%20II.md))

**`INCLUDE`:** columnas que se guardan solo en las hojas de un índice, para cubrir consultas. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Index Scan / Index Seek:** recorrer un índice completo / navegar el árbol hasta las filas buscadas. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**Índice agrupado (clustered):** índice cuyas hojas son las filas de la tabla, ordenadas por su clave. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**Índice columnstore:** índice que guarda cada columna por separado y comprimida, para consultas analíticas. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Índice compuesto:** índice con varias columnas en la clave. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Índice cubriente (covering):** índice que contiene todas las columnas que necesita una consulta. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Índice filtrado:** índice con un `WHERE`, que solo contiene algunas filas. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Índice no agrupado (nonclustered):** índice aparte, con sus columnas clave y un localizador de la fila. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**Key Lookup:** ir al índice agrupado a buscar las columnas que faltan, una vez por fila. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**Lectura lógica / física:** leer una página desde la memoria / desde el disco. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Localizador de fila (row locator):** lo que guarda un índice no agrupado para llegar a la fila: la clave del agrupado o el RID. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**Memory grant:** memoria reservada para una consulta antes de ejecutarla. ([04](04-Planes%20de%20ejecución%20I.md))

**Merge Join:** operador que une dos entradas ordenadas recorriéndolas en paralelo. ([04](04-Planes%20de%20ejecución%20I.md))

**Nested Loops:** operador que, por cada fila de una entrada, busca sus parejas en la otra. ([04](04-Planes%20de%20ejecución%20I.md))

**Optimizador de consultas:** componente que evalúa planes posibles y elige el de menor costo estimado. ([04](04-Planes%20de%20ejecución%20I.md))

**Página:** unidad mínima de lectura y escritura de SQL Server: 8 KB. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**Parameter sniffing:** compilar el plan con el valor del primer parámetro y reutilizarlo para valores que necesitarían otro plan. ([05](05-Planes%20de%20ejecución%20II.md))

**Plan estimado / plan real:** el plan que se piensa usar, sin ejecutar / el plan usado en una ejecución, con filas reales. ([04](04-Planes%20de%20ejecución%20I.md))

**Predicado de búsqueda / residual:** la condición que el índice resuelve navegando / la que se evalúa fila por fila después. ([04](04-Planes%20de%20ejecución%20I.md))

**Prefijo izquierdo:** las primeras columnas de un índice compuesto; solo un filtro sobre ellas permite un *seek*. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Problema N+1:** ejecutar una consulta por cada fila de un resultado anterior. ([06](06-Consultas%20SARGables%20y%20antipatrones.md))

**Punto de inflexión (tipping point):** cantidad de filas a partir de la cual conviene más un *scan* que *seeks* con *lookups*. ([02](02-Índices%20clustered%20y%20nonclustered.md))

**`REBUILD` / `REORGANIZE`:** reconstruir un índice completo / reordenar y compactar sus hojas en pasos pequeños. ([07](07-Mantenimiento%20de%20índices.md))

**RID (row identifier):** identificador físico de una fila en un heap: archivo, página y posición. ([01](01-Cómo%20guarda%20los%20datos%20SQL%20Server.md))

**SARGable:** condición que el motor puede resolver con un *seek*, porque deja la columna sin transformar. ([06](06-Consultas%20SARGables%20y%20antipatrones.md))

**Selectividad:** proporción de filas que descarta un filtro; muy selectivo = pocas filas. ([03](03-Índices%20compuestos%2C%20cubrientes%20y%20filtrados.md))

**Tabla de números (tally table):** tabla con enteros consecutivos para generar filas sin bucles. ([00](00-Datos%20de%20volumen.md))
