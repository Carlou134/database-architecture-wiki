# CTE

## En una frase

Una CTE (*Common Table Expression*, expresión de tabla común) es una consulta con **nombre**, definida con `WITH` antes de la consulta principal, que permite escribir SQL complejo **por pasos legibles**; y en su versión **recursiva**, la única forma estándar de recorrer jerarquías como organigramas o categorías anidadas.

-----

## Antes de empezar

Conviene que ya sepas:

* Tablas derivadas y subconsultas, de [Subconsultas](08-Subconsultas.md).
* `UNION ALL`, de [Operaciones de conjuntos](07-Operaciones%20de%20conjuntos.md).
* La autorreferencia (`JefeId`, `CategoriaPadreId`), de [Claves foráneas](../01-fundamentos/05-Claves%20foráneas%20e%20integridad%20referencial.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **CTE:** consulta con nombre, definida con `WITH`, que existe solo durante la instrucción que la sigue.
* **CTE recursiva:** CTE que se referencia a sí misma, para recorrer estructuras jerárquicas.
* **Miembro ancla:** la parte de una CTE recursiva que define el punto de partida.
* **Miembro recursivo:** la parte que se une con el resultado anterior para avanzar un nivel.
* **`MAXRECURSION`:** límite de niveles de recursión (100 por defecto en SQL Server).

-----

## El problema

La consulta de "clientes que gastaron más que el promedio" de la lección anterior tenía el cálculo del gasto **escrito dos veces**, y se leía de adentro hacia afuera: para entenderla hay que encontrar la subconsulta más interna, entenderla, salir un nivel, y así.

Y hay preguntas que ni siquiera se pueden escribir con lo visto hasta ahora. Con una tabla de categorías con subcategorías:

```text
Electrónica
├── Computación
│   ├── Laptops
│   └── Periféricos
└── Audio
```

"¿Cuáles son todas las subcategorías de Electrónica, en cualquier nivel?" Con joins, cada nivel es un join más: con tres niveles hacen falta tres joins, y si mañana hay un cuarto nivel, la consulta deja de ser correcta. Hace falta una consulta que **se repita** hasta que no haya más niveles.

-----

## Cómo funciona

### La sintaxis

```sql
WITH GastoPorCliente AS (
    SELECT p.ClienteId, SUM(d.Cantidad * d.PrecioUnitario) AS Gastado
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.ClienteId
)
SELECT c.Nombre, g.Gastado
FROM GastoPorCliente AS g
JOIN dbo.Cliente AS c ON c.ClienteId = g.ClienteId
ORDER BY g.Gastado DESC;
```

| Nombre | Gastado |
| --- | --- |
| Luis Ramos | 849.90 |
| Ana Torres | 476.00 |
| Jorge Díaz | 140.50 |

* `WITH Nombre AS (consulta)` define la CTE, y la consulta que sigue la usa como si fuera una tabla.
* La CTE existe **solo** durante esa instrucción. En la siguiente, ya no está.
* Se pueden nombrar las columnas en la definición: `WITH GastoPorCliente (ClienteId, Gastado) AS (...)`.

### Varias CTE encadenadas

Se separan con comas, y cada una puede usar las anteriores. Así se escribe una consulta compleja **de arriba hacia abajo**, como una receta:

```sql
WITH GastoPorCliente AS (
    SELECT p.ClienteId, SUM(d.Cantidad * d.PrecioUnitario) AS Gastado
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.ClienteId
),
Promedio AS (
    SELECT SUM(Gastado) / (SELECT COUNT(*) FROM dbo.Cliente) AS GastoPromedio
    FROM GastoPorCliente
)
SELECT c.Nombre, g.Gastado, pr.GastoPromedio
FROM GastoPorCliente AS g
JOIN dbo.Cliente AS c ON c.ClienteId = g.ClienteId
CROSS JOIN Promedio AS pr
WHERE g.Gastado > pr.GastoPromedio
ORDER BY g.Gastado DESC;
```

| Nombre | Gastado | GastoPromedio |
| --- | --- | --- |
| Luis Ramos | 849.90 | 293.280000 |
| Ana Torres | 476.00 | 293.280000 |

Es la consulta del ejemplo completo de [Subconsultas](08-Subconsultas.md), reescrita: el gasto se calcula **una vez**, cada paso tiene un nombre que dice qué es, y el `CROSS JOIN` con una CTE de una sola fila pone el promedio al lado de cada cliente.

### El punto y coma antes de `WITH`

`WITH` también se usa en otras partes de T-SQL (por ejemplo, `WITH (NOLOCK)`), así que si la instrucción anterior no termina en `;`, SQL Server no sabe que empieza una CTE:

```text
Msg 319, Level 15, State 1
Incorrect syntax near the keyword 'with'. If this statement is a common table expression,
an xmlnamespaces clause or a change tracking context clause, the previous statement must be
terminated with a semicolon.
```

Por eso es común ver `;WITH` en código ajeno. La solución correcta es terminar **todas** las instrucciones con `;`, que es lo que pide el estándar.

### CTE frente a tabla derivada y tabla temporal

| | Tabla derivada | CTE | Tabla temporal (`#Tabla`) |
| --- | --- | --- | --- |
| Dónde se define | Dentro del `FROM` | Antes de la consulta, con `WITH` | Con `CREATE TABLE #...` o `SELECT ... INTO #...` |
| Se puede usar varias veces en la consulta | No (hay que repetirla) | Sí | Sí, y en varias instrucciones |
| Se guarda el resultado | No | **No**: se vuelve a evaluar cada vez que se referencia | Sí, en `tempdb` |
| Admite índices y estadísticas | No | No | Sí |
| Recursiva | No | Sí | No |

Un punto que sorprende: en SQL Server, una CTE **no** se materializa. Si la usas tres veces en la consulta, el motor puede calcularla tres veces. La CTE es una herramienta de **legibilidad**, no de rendimiento. Si un resultado intermedio es caro y se usa varias veces, o si conviene indexarlo, una tabla temporal suele ser mejor.

### CTE recursivas

Una CTE recursiva tiene dos partes unidas con `UNION ALL`:

1. **El miembro ancla:** la consulta de partida (por ejemplo, la categoría raíz).
2. **El miembro recursivo:** una consulta que se une con **la propia CTE**, para avanzar un nivel.

El motor ejecuta el ancla, después el miembro recursivo sobre lo que obtuvo, después otra vez sobre lo nuevo, y así hasta que el miembro recursivo no devuelve filas.

Con la tabla de categorías jerárquicas de la práctica de [Claves foráneas](../01-fundamentos/05-Claves%20foráneas%20e%20integridad%20referencial.md):

```sql
CREATE TABLE dbo.CategoriaJerarquica (
    CategoriaId      INT IDENTITY(1,1) PRIMARY KEY,
    Nombre           NVARCHAR(50) NOT NULL,
    CategoriaPadreId INT NULL REFERENCES dbo.CategoriaJerarquica (CategoriaId)
);

INSERT INTO dbo.CategoriaJerarquica (Nombre, CategoriaPadreId) VALUES
    (N'Electrónica', NULL),   -- 1
    (N'Computación', 1),      -- 2
    (N'Audio', 1),            -- 3
    (N'Laptops', 2),          -- 4
    (N'Periféricos', 2),      -- 5
    (N'Hogar', NULL);         -- 6
```

Todas las subcategorías de Electrónica, con su nivel y su ruta:

```sql
WITH Arbol AS (
    -- Ancla: el punto de partida
    SELECT CategoriaId, Nombre, CategoriaPadreId,
           0 AS Nivel,
           CAST(Nombre AS NVARCHAR(500)) AS Ruta
    FROM dbo.CategoriaJerarquica
    WHERE CategoriaId = 1

    UNION ALL

    -- Recursivo: los hijos de lo que ya está en el árbol
    SELECT h.CategoriaId, h.Nombre, h.CategoriaPadreId,
           a.Nivel + 1,
           CAST(a.Ruta + N' > ' + h.Nombre AS NVARCHAR(500))
    FROM dbo.CategoriaJerarquica AS h
    JOIN Arbol AS a ON h.CategoriaPadreId = a.CategoriaId
)
SELECT Nivel, Nombre, Ruta
FROM Arbol
ORDER BY Ruta;
```

| Nivel | Nombre | Ruta |
| --- | --- | --- |
| 0 | Electrónica | Electrónica |
| 1 | Audio | Electrónica > Audio |
| 1 | Computación | Electrónica > Computación |
| 2 | Laptops | Electrónica > Computación > Laptops |
| 2 | Periféricos | Electrónica > Computación > Periféricos |

Cómo se construyó, paso a paso:

```text
Ancla:        Electrónica (1)
Iteración 1:  hijos de {1}     → Computación (2), Audio (3)
Iteración 2:  hijos de {2, 3}  → Laptops (4), Periféricos (5)
Iteración 3:  hijos de {4, 5}  → ninguno → fin
```

Detalles importantes:

* **Los tipos del ancla y del recursivo deben coincidir exactamente.** Por eso `Ruta` se convierte con `CAST(... AS NVARCHAR(500))` en las dos partes: sin eso, la concatenación cambia la longitud y falla con `Msg 240: Types don't match between the anchor and the recursive part`.
* **El recorrido funciona con cualquier cantidad de niveles:** agregar un quinto nivel no cambia la consulta.
* Para recorrer **hacia arriba** (de una categoría a sus ancestros), se invierte el join: `JOIN Arbol AS a ON h.CategoriaId = a.CategoriaPadreId`.

### `MAXRECURSION`: el límite de seguridad

Si los datos tienen un ciclo (A es padre de B y B es padre de A), la recursión no terminaría nunca. SQL Server la corta a los **100 niveles** por defecto:

```text
Msg 530, Level 16, State 1
The statement terminated. The maximum recursion 100 has been exhausted before statement completion.
```

Si una jerarquía legítima es más profunda, se ajusta el límite al final de la consulta:

```sql
SELECT ... FROM Arbol
OPTION (MAXRECURSION 500);   -- 0 = sin límite (solo si estás seguro de que no hay ciclos)
```

### Generar series

Una CTE recursiva también genera filas que no existen en ninguna tabla, como una lista de fechas:

```sql
WITH Meses AS (
    SELECT CAST('2026-01-01' AS DATE) AS Mes
    UNION ALL
    SELECT DATEADD(month, 1, Mes) FROM Meses WHERE Mes < '2026-06-01'
)
SELECT Mes FROM Meses;
```

Devuelve los seis primeros días de mes de 2026. Unida con un `LEFT JOIN` a los pedidos, da un reporte mensual **que incluye los meses sin ventas**, algo que un `GROUP BY` solo no puede hacer (los meses sin filas no aparecen). En SQL Server 2022, `GENERATE_SERIES` hace lo mismo con números, sin recursión.

-----

## Ejemplo completo

Reporte de ventas mensuales de enero a junio de 2026, sin pedidos cancelados, incluidos los meses sin ventas y con la variación respecto del mes anterior.

```sql
WITH Meses AS (
    SELECT CAST('2026-01-01' AS DATE) AS Mes
    UNION ALL
    SELECT DATEADD(month, 1, Mes) FROM Meses WHERE Mes < '2026-06-01'
),
VentasPorMes AS (
    SELECT DATEFROMPARTS(YEAR(p.FechaPedido), MONTH(p.FechaPedido), 1) AS Mes,
           SUM(d.Cantidad * d.PrecioUnitario)                           AS Vendido
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY DATEFROMPARTS(YEAR(p.FechaPedido), MONTH(p.FechaPedido), 1)
),
Serie AS (
    SELECT m.Mes, COALESCE(v.Vendido, 0) AS Vendido
    FROM Meses AS m
    LEFT JOIN VentasPorMes AS v ON v.Mes = m.Mes
)
SELECT s.Mes,
       s.Vendido,
       s.Vendido - COALESCE((SELECT a.Vendido FROM Serie AS a
                             WHERE a.Mes = DATEADD(month, -1, s.Mes)), 0) AS Variacion
FROM Serie AS s
ORDER BY s.Mes;
```

| Mes | Vendido | Variacion |
| --- | --- | --- |
| 2026-01-01 | 0.00 | 0.00 |
| 2026-02-01 | 0.00 | 0.00 |
| 2026-03-01 | 1060.90 | 1060.90 |
| 2026-04-01 | 265.00 | -795.90 |
| 2026-05-01 | 140.50 | -124.50 |
| 2026-06-01 | 0.00 | -140.50 |

Qué decisión hay en cada CTE:

* **`Meses`:** la grilla de todos los meses, generada con recursión, para que enero, febrero y junio aparezcan aunque no haya ventas.
* **`VentasPorMes`:** agrupa por el primer día del mes (`DATEFROMPARTS`), un valor de tipo fecha que se puede unir con la grilla.
* **`Serie`:** el `LEFT JOIN` de la grilla con las ventas, con `COALESCE` para los meses vacíos.
* **La variación** usa una subconsulta correlacionada sobre `Serie` para leer el mes anterior. Funciona, pero hace que la CTE `Serie` se evalúe de nuevo por cada fila: en la [próxima lección](10-Funciones%20de%20ventana.md), `LAG` resuelve lo mismo de forma más simple y eficiente.

-----

## Errores comunes

**1. Olvidar el `;` antes de `WITH`.**
Qué pasa: `Msg 319: Incorrect syntax near the keyword 'with'`.
Por qué: `WITH` tiene otros usos en T-SQL, y sin el `;` el motor no sabe que empieza una CTE.
Arreglo: termina todas las instrucciones con `;`.

**2. Usar la CTE en una segunda instrucción.**
Qué pasa: `Invalid object name 'GastoPorCliente'`.
Por qué: la CTE solo existe durante la instrucción que la sigue inmediatamente.
Arreglo: repite la CTE o guarda el resultado en una tabla temporal.

**3. Esperar que la CTE se calcule una sola vez.**
Qué pasa: una consulta lenta porque una CTE cara se referencia varias veces.
Por qué: SQL Server no materializa las CTE.
Arreglo: si el resultado intermedio es caro y se reutiliza, usa una tabla temporal.

**4. Tipos distintos entre el ancla y el miembro recursivo.**
Qué pasa: `Msg 240: Types don't match between the anchor and the recursive part`.
Por qué: una concatenación o una suma cambió el tipo o la longitud en el miembro recursivo.
Arreglo: `CAST` explícito en las dos partes.

**5. Recursión sin condición de corte, o datos con ciclos.**
Qué pasa: `Msg 530: The maximum recursion 100 has been exhausted`.
Por qué: el miembro recursivo nunca deja de devolver filas.
Arreglo: verifica la condición de corte; para detectar ciclos, guarda la ruta y no vuelvas a visitar un nodo que ya está en ella.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| CTE | Sí | Sí | Sí, desde 8.0 |
| Palabra para la recursiva | `WITH` (la detecta sola) | `WITH RECURSIVE` (obligatoria) | `WITH RECURSIVE` (obligatoria) |
| ¿Se materializa? | No | Desde la 12, solo si se referencia más de una vez o se pide `MATERIALIZED` | Depende del optimizador |
| Forzar materialización | No (usar una tabla temporal) | `AS MATERIALIZED (...)` / `AS NOT MATERIALIZED (...)` | No |
| Límite de recursión | 100 por defecto (`OPTION (MAXRECURSION n)`) | Sin límite (cuidado con los ciclos) | `cte_max_recursion_depth` = 1000 por defecto |
| CTE con `INSERT`/`UPDATE`/`DELETE` dentro | No | Sí (*data-modifying CTE*) | No |
| Detección de ciclos | Manual | `CYCLE ... SET ...` (desde la 14) | Manual |
| Generar series sin recursión | `GENERATE_SERIES` (desde 2022) | `generate_series` | No |

La diferencia que más se nota al copiar consultas: PostgreSQL y MySQL exigen `WITH RECURSIVE`; SQL Server no lo acepta.

### En MongoDB

No hay CTE: el *aggregation pipeline* ya se escribe por pasos (cada etapa recibe la salida de la anterior), que es exactamente lo que una CTE aporta a SQL. Para jerarquías, existe la etapa `$graphLookup`, que recorre recursivamente una colección siguiendo un campo:

```javascript
db.categorias.aggregate([
  { $match: { _id: 1 } },
  { $graphLookup: {
      from: "categorias",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "categoriaPadreId",
      as: "subcategorias",
      depthField: "nivel"
  } }
]);
```

-----

## Cuándo sí y cuándo no

**Usa una CTE cuando:**

* La consulta tiene varios pasos y quieres leerla de arriba hacia abajo.
* El mismo resultado intermedio se usa más de una vez en la consulta (por legibilidad, sabiendo que no se materializa).
* Necesitas recursión: jerarquías o series.

**Usa una tabla temporal cuando:**

* El resultado intermedio es caro, se reutiliza varias veces o se usa en varias instrucciones.
* Conviene indexarlo o que tenga estadísticas propias.

**Ten en cuenta:**

* Para jerarquías muy grandes que se consultan muchísimo, existen modelos alternativos a la autorreferencia: `hierarchyid` en SQL Server, la tabla de clausura o los conjuntos anidados.

-----

## Resumen en 5 líneas

1. Una CTE es una consulta con nombre (`WITH Nombre AS (...)`) que existe solo durante la instrucción siguiente.
2. Varias CTE separadas por comas permiten escribir consultas complejas por pasos, de arriba hacia abajo.
3. En SQL Server no se materializan: son legibilidad, no rendimiento; para reutilizar resultados caros, tablas temporales.
4. Recursiva: ancla + `UNION ALL` + miembro recursivo que se une con la propia CTE, hasta que no haya filas nuevas.
5. Termina la instrucción anterior con `;`, iguala los tipos con `CAST` y respeta el límite de `MAXRECURSION`.

-----

## Para profundizar

<details>
<summary>Detectar ciclos en una CTE recursiva</summary>

Si los datos pueden tener ciclos por error, se lleva la ruta de identificadores y se corta cuando un nodo ya aparece en ella:

```sql
WITH Arbol AS (
    SELECT CategoriaId, CategoriaPadreId,
           CAST(CONCAT(N'/', CategoriaId, N'/') AS NVARCHAR(1000)) AS RutaIds
    FROM dbo.CategoriaJerarquica
    WHERE CategoriaPadreId IS NULL
    UNION ALL
    SELECT h.CategoriaId, h.CategoriaPadreId,
           CAST(CONCAT(a.RutaIds, h.CategoriaId, N'/') AS NVARCHAR(1000))
    FROM dbo.CategoriaJerarquica AS h
    JOIN Arbol AS a ON h.CategoriaPadreId = a.CategoriaId
    WHERE a.RutaIds NOT LIKE CONCAT(N'%/', h.CategoriaId, N'/%')   -- no volver a un nodo visitado
)
SELECT * FROM Arbol;
```

Los separadores `/` al principio y al final evitan falsos positivos (que el 1 se encuentre dentro del 12).

</details>

<details>
<summary>CTE y vistas</summary>

Una CTE es como una vista que solo vive durante una consulta. Si la misma CTE aparece en muchas consultas distintas, es una buena candidata a convertirse en una **vista**, que tiene nombre permanente y se puede reutilizar y asignar permisos. Al igual que las CTE, una vista común no guarda datos: se expande dentro de la consulta que la usa. Las vistas indexadas, que sí guardan datos, se vieron en [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una CTE es una consulta con nombre que se define con `WITH` antes de la consulta principal y se usa como una tabla. Hace las consultas complejas más fáciles de leer, porque se escriben por pasos. Las CTE recursivas sirven para recorrer jerarquías, como un organigrama.

### Respuesta ampliada (semi-senior)

Uso CTE para estructurar consultas complejas en pasos con nombre, de arriba hacia abajo, en lugar de anidar tablas derivadas, y para reutilizar un resultado intermedio dentro de la misma instrucción. Sé que en SQL Server no se materializan: cada referencia puede volver a evaluarse, así que si el resultado intermedio es caro o se usa en varias instrucciones, prefiero una tabla temporal, que además admite índices y estadísticas. Las CTE recursivas tienen un ancla y un miembro recursivo unidos con `UNION ALL`; las uso para jerarquías y para generar series (meses sin ventas en un reporte), cuidando que los tipos coincidan con `CAST`, que haya una condición de corte y el límite de `MAXRECURSION`, y detectando ciclos con una ruta si los datos lo requieren. También termino siempre la instrucción anterior con `;`, y al portar consultas recuerdo que PostgreSQL y MySQL exigen `WITH RECURSIVE`.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre una CTE y una tabla temporal?**
La CTE existe solo durante una instrucción y no guarda su resultado; la tabla temporal vive en `tempdb` durante la sesión, guarda los datos y admite índices.

**2. ¿Una CTE mejora el rendimiento?**
En SQL Server, no por sí misma: es una herramienta de legibilidad. El optimizador la expande dentro de la consulta.

**3. ¿Cómo funciona una CTE recursiva?**
El miembro ancla define el punto de partida; el miembro recursivo se une con la propia CTE para avanzar un nivel, y se repite hasta que no devuelve filas nuevas.

**4. ¿Qué es `MAXRECURSION`?**
El límite de niveles de recursión: 100 por defecto en SQL Server, ajustable con `OPTION (MAXRECURSION n)`.

-----

## Práctica

**Ejercicio 1.** Reescribe con CTE la consulta del ejercicio 1.1 de [Subconsultas](08-Subconsultas.md) ("pedidos cuyo total supera el del pedido 3"), de forma que el total por pedido se calcule una sola vez en la consulta.

<details>
<summary>Solución</summary>

```sql
WITH TotalPorPedido AS (
    SELECT PedidoId, SUM(Cantidad * PrecioUnitario) AS Total
    FROM dbo.DetallePedido
    GROUP BY PedidoId
)
SELECT t.PedidoId, t.Total
FROM TotalPorPedido AS t
WHERE t.Total > (SELECT Total FROM TotalPorPedido WHERE PedidoId = 3);
```

| PedidoId | Total |
| --- | --- |
| 2 | 849.90 |

La lógica del total está escrita una vez. (Recuerda: aunque se escriba una vez, SQL Server puede evaluar `TotalPorPedido` dos veces, una por cada referencia.)

</details>

**Ejercicio 2.** Con la tabla `CategoriaJerarquica` de esta lección, escribe una CTE recursiva que, partiendo de "Periféricos", devuelva todos sus ancestros hasta la raíz, con la distancia de cada uno.

<details>
<summary>Solución</summary>

```sql
WITH Ancestros AS (
    SELECT CategoriaId, Nombre, CategoriaPadreId, 0 AS Distancia
    FROM dbo.CategoriaJerarquica
    WHERE Nombre = N'Periféricos'

    UNION ALL

    SELECT padre.CategoriaId, padre.Nombre, padre.CategoriaPadreId, a.Distancia + 1
    FROM dbo.CategoriaJerarquica AS padre
    JOIN Ancestros AS a ON padre.CategoriaId = a.CategoriaPadreId
)
SELECT Distancia, Nombre
FROM Ancestros
ORDER BY Distancia;
```

| Distancia | Nombre |
| --- | --- |
| 0 | Periféricos |
| 1 | Computación |
| 2 | Electrónica |

El join está invertido respecto del recorrido hacia abajo: en cada paso se busca la fila cuyo `CategoriaId` es el **padre** de la última encontrada. La recursión termina en Electrónica, cuyo padre es `NULL` y no empareja con nada.

</details>

-----

## Siguiente lección

[Funciones de ventana](10-Funciones%20de%20ventana.md)
