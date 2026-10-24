# Glosario: Consultas

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Agregación condicional:** un agregado sobre un `CASE` (`SUM(CASE WHEN ... THEN 1 ELSE 0 END)`) para contar o sumar solo las filas que cumplen una condición. ([05](05-Agregados%20y%20GROUP%20BY.md))

**Alias:** nombre alternativo para una columna o tabla, con `AS`. ([01](01-SELECT%20y%20el%20orden%20lógico.md))

**Alucinación:** respuesta falsa de una IA dicha con seguridad (una función o columna que no existe). ([12](12-SQL%20con%20IA.md))

**Anti join:** quedarse con las filas de una tabla que no tienen pareja en otra (`LEFT JOIN ... IS NULL` o `NOT EXISTS`). ([06](06-JOIN.md))

**`CASE`:** expresión condicional que devuelve un valor distinto según la primera condición verdadera. ([04](04-Expresiones%2C%20CASE%20y%20funciones.md))

**`CAST` / `CONVERT` / `TRY_CAST`:** funciones de conversión de tipo; `CONVERT` acepta estilos de formato y las `TRY_` devuelven `NULL` en lugar de fallar. ([04](04-Expresiones%2C%20CASE%20y%20funciones.md))

**Cláusula:** cada parte de una consulta (`SELECT`, `FROM`, `WHERE`...). ([01](01-SELECT%20y%20el%20orden%20lógico.md))

**Comodín (wildcard):** carácter especial de `LIKE`: `%` (cualquier cantidad de caracteres) y `_` (uno). ([02](02-Filtrar%20con%20WHERE.md))

**Compatibilidad de unión:** requisito de los operadores de conjuntos: misma cantidad de columnas, tipos compatibles y mismo orden. ([07](07-Operaciones%20de%20conjuntos.md))

**Consulta (query):** instrucción que pide datos a la base. ([01](01-SELECT%20y%20el%20orden%20lógico.md))

**`CROSS JOIN`:** producto cartesiano: cada fila de una tabla con cada fila de la otra. ([06](06-JOIN.md))

**CTE (Common Table Expression):** consulta con nombre, definida con `WITH`, que existe solo durante la instrucción siguiente. ([09](09-CTE.md))

**CTE recursiva:** CTE que se referencia a sí misma, con un miembro ancla y un miembro recursivo unidos por `UNION ALL`. ([09](09-CTE.md))

**Desempate (tie-breaker):** columna única al final de un `ORDER BY` que hace el orden determinista. ([03](03-Ordenar%20y%20paginar.md))

**`DISTINCT`:** elimina las filas repetidas del resultado. ([01](01-SELECT%20y%20el%20orden%20lógico.md))

**DML:** instrucciones que modifican datos: `INSERT`, `UPDATE`, `DELETE`, `MERGE`. ([11](11-Modificar%20datos.md))

**Embudo (funnel):** proceso en etapas donde en cada una quedan menos casos; se mide con agregación condicional. ([05](05-Agregados%20y%20GROUP%20BY.md))

**`EXCEPT`:** filas de la primera consulta que no están en la segunda. ([07](07-Operaciones%20de%20conjuntos.md))

**`EXISTS` / `NOT EXISTS`:** verdadero si la subconsulta devuelve al menos una fila / ninguna; nunca da `UNKNOWN`. ([08](08-Subconsultas.md))

**Función de agregado:** función que resume un conjunto de filas en un valor (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`). ([05](05-Agregados%20y%20GROUP%20BY.md))

**Función de ventana:** función calculada sobre un conjunto de filas definido con `OVER`, sin agruparlas. ([10](10-Funciones%20de%20ventana.md))

**Función escalar:** función que recibe valores de una fila y devuelve un valor (`UPPER`, `YEAR`). ([04](04-Expresiones%2C%20CASE%20y%20funciones.md))

**`FULL JOIN`:** conserva las filas sin pareja de las dos tablas. ([06](06-JOIN.md))

**`GO`:** separador de lotes de SSMS y `sqlcmd`; no es una instrucción de SQL. ([00](00-Entorno%20de%20trabajo.md))

**`GROUP BY`:** agrupa las filas que comparten valores en las columnas indicadas. ([05](05-Agregados%20y%20GROUP%20BY.md))

**`HAVING`:** filtra grupos según sus agregados, después del `GROUP BY`. ([05](05-Agregados%20y%20GROUP%20BY.md))

**`INNER JOIN`:** devuelve solo las filas con pareja en las dos tablas. ([06](06-JOIN.md))

**`INTERSECT`:** filas que están en las dos consultas. ([07](07-Operaciones%20de%20conjuntos.md))

**`LAG` / `LEAD`:** funciones de ventana que devuelven el valor de la fila anterior / siguiente. ([10](10-Funciones%20de%20ventana.md))

**`LEFT JOIN`:** devuelve todas las filas de la tabla izquierda, con `NULL` donde no hay pareja en la derecha. ([06](06-JOIN.md))

**LLM (modelo de lenguaje grande):** el tipo de modelo detrás de los asistentes de IA. ([12](12-SQL%20con%20IA.md))

**Lote (batch):** grupo de instrucciones que se envía junto al servidor. ([00](00-Entorno%20de%20trabajo.md))

**Marco (frame):** subconjunto de filas de la partición que entra en el cálculo de una función de ventana (`ROWS BETWEEN ...`). ([10](10-Funciones%20de%20ventana.md))

**`MAXRECURSION`:** límite de niveles de una CTE recursiva (100 por defecto en SQL Server). ([09](09-CTE.md))

**`MERGE`:** instrucción que inserta, actualiza o borra comparando un destino con un origen; tiene limitaciones conocidas en concurrencia. ([11](11-Modificar%20datos.md))

**Multiplicación de filas (fan-out):** el aumento de filas que produce un join con una tabla del lado "muchos". ([06](06-JOIN.md))

**`OFFSET ... FETCH`:** salta filas y devuelve las siguientes; la paginación estándar de SQL Server. ([03](03-Ordenar%20y%20paginar.md))

**Orden determinista:** orden sin empates, que da siempre el mismo resultado. ([03](03-Ordenar%20y%20paginar.md))

**Orden lógico de procesamiento:** `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `TOP`. ([01](01-SELECT%20y%20el%20orden%20lógico.md))

**`OUTPUT`:** cláusula de SQL Server que devuelve las filas afectadas por una modificación, con sus valores antes (`deleted`) y después (`inserted`). ([11](11-Modificar%20datos.md))

**`OVER`:** cláusula que define la ventana de una función de ventana. ([10](10-Funciones%20de%20ventana.md))

**Paginación por clave (keyset):** paginar pidiendo "las siguientes después de la última vista", en lugar de saltar filas con `OFFSET`. ([03](03-Ordenar%20y%20paginar.md))

**`PARTITION BY`:** divide las filas en grupos independientes para una función de ventana. ([10](10-Funciones%20de%20ventana.md))

**Precedencia:** orden de evaluación de los operadores sin paréntesis: `NOT`, `AND`, `OR`. ([02](02-Filtrar%20con%20WHERE.md))

**Predicado:** condición que se evalúa como verdadera, falsa o desconocida. ([02](02-Filtrar%20con%20WHERE.md))

**Prompt:** instrucción o pregunta que se le da a un modelo de IA. ([12](12-SQL%20con%20IA.md))

**Rango semiabierto:** rango que incluye el inicio y excluye el final (`>= inicio AND < fin`); la forma correcta de filtrar fechas con hora. ([02](02-Filtrar%20con%20WHERE.md))

**`ROW_NUMBER` / `RANK` / `DENSE_RANK`:** funciones de ranking: numeran sin repetir / repiten y saltan / repiten sin saltar. ([10](10-Funciones%20de%20ventana.md))

**`@@ROWCOUNT`:** cantidad de filas afectadas por la última instrucción. ([11](11-Modificar%20datos.md))

**SARGable:** condición que el motor puede resolver con un índice, porque deja la columna sin transformar. ([02](02-Filtrar%20con%20WHERE.md))

**Self join:** unir una tabla consigo misma, con dos alias. ([06](06-JOIN.md))

**Semi join:** quedarse con las filas que tienen pareja, sin duplicarlas (lo que hace `EXISTS`). ([08](08-Subconsultas.md))

**Sesgo de confirmación:** tendencia a aceptar un resultado porque "se ve bien", sin verificarlo. ([12](12-SQL%20con%20IA.md))

**`sqlcmd`:** cliente de línea de comandos de SQL Server. ([00](00-Entorno%20de%20trabajo.md))

**SSMS:** SQL Server Management Studio, el cliente gráfico oficial. ([00](00-Entorno%20de%20trabajo.md))

**`STRING_AGG`:** concatena los textos de un grupo; el equivalente de `GROUP_CONCAT`. ([05](05-Agregados%20y%20GROUP%20BY.md))

**Subconsulta:** un `SELECT` dentro de otra instrucción, entre paréntesis. ([08](08-Subconsultas.md))

**Subconsulta correlacionada:** subconsulta que usa columnas de la consulta exterior. ([08](08-Subconsultas.md))

**Subconsulta escalar:** subconsulta que devuelve una fila y una columna, y se usa como un valor. ([08](08-Subconsultas.md))

**Tabla derivada:** subconsulta en el `FROM`, con alias obligatorio. ([08](08-Subconsultas.md))

**`TOP (n)`:** devuelve las primeras `n` filas según el `ORDER BY`. ([03](03-Ordenar%20y%20paginar.md))

**`TRUNCATE`:** vacía una tabla completa, rápido; reinicia el `IDENTITY` y no funciona si otra tabla la referencia. ([11](11-Modificar%20datos.md))

**`UNION` / `UNION ALL`:** apilan los resultados de dos consultas, sin repetidos / con repetidos. ([07](07-Operaciones%20de%20conjuntos.md))

**Upsert:** "actualizar si existe, insertar si no". ([11](11-Modificar%20datos.md))

**`WITH TIES`:** con `TOP`, incluye las filas que empatan con la última. ([03](03-Ordenar%20y%20paginar.md))
