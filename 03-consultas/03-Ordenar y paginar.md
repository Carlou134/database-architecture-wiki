# Ordenar y paginar

## En una frase

`ORDER BY` es la **única** forma de garantizar el orden de un resultado, y en SQL Server se combina con `TOP` para quedarse con las primeras filas y con `OFFSET ... FETCH` para paginar; el equivalente de `LIMIT` de SQLite, PostgreSQL y MySQL.

-----

## Antes de empezar

Conviene que ya sepas:

* Que una relación no tiene orden, de [El modelo relacional](../01-fundamentos/02-El%20modelo%20relacional.md).
* El orden lógico de procesamiento, de [SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`ORDER BY`:** ordena el resultado por una o varias columnas o expresiones.
* **`ASC` / `DESC`:** orden ascendente (por defecto) o descendente.
* **`TOP (n)`:** devuelve solo las primeras `n` filas.
* **`OFFSET ... FETCH`:** salta una cantidad de filas y devuelve las siguientes; sirve para paginar.
* **Orden determinista:** un orden en el que no hay empates, así que el resultado es siempre el mismo.
* **Desempate (tie-breaker):** columna única que se agrega al final del `ORDER BY` para que el orden sea determinista.

-----

## El problema

En tus apuntes, la consulta para "los 5 productos más caros" es:

```sql
SELECT * FROM producto ORDER BY precio DESC LIMIT 5;
```

En SQL Server:

```text
Msg 102, Level 15, State 1
Incorrect syntax near 'LIMIT'.
```

`LIMIT` es de SQLite, PostgreSQL y MySQL; T-SQL usa `TOP` y `OFFSET ... FETCH`. Ese es el problema fácil. El difícil aparece al paginar una lista ordenada por precio con empates: la página 1 y la página 2 muestran el mismo producto, y otro no aparece en ninguna. Ninguna consulta da error. Simplemente, el orden no era determinista.

-----

## Cómo funciona

### `ORDER BY`: ascendente, descendente y varias columnas

```sql
SELECT Nombre, CategoriaId, Precio
FROM dbo.Producto
ORDER BY CategoriaId ASC, Precio DESC;
```

| Nombre | CategoriaId | Precio |
| --- | --- | --- |
| Monitor 27 pulgadas | 1 | 899.90 |
| Teclado mecánico | 1 | 120.00 |
| Mouse inalámbrico | 1 | 45.50 |
| Cafetera | 2 | 210.00 |
| Lámpara de escritorio | 2 | 75.00 |
| Clean Code | 3 | 95.00 |

* Se ordena por la primera columna; la segunda solo decide **entre empates** de la primera.
* `ASC` es el valor por defecto y se puede omitir. Cada columna lleva su propia dirección: `DESC` se aplica solo a la columna que lo tiene al lado.
* Se puede ordenar por columnas que no están en el `SELECT`, por alias y por expresiones:

```sql
SELECT Nombre, Precio * Stock AS ValorInventario
FROM dbo.Producto
ORDER BY ValorInventario DESC, Nombre;
```

También se puede ordenar por posición (`ORDER BY 2 DESC`), pero conviene evitarlo: si alguien agrega una columna al `SELECT`, el orden cambia sin que nadie lo note.

### Sin `ORDER BY` no hay orden

Una consulta sin `ORDER BY` devuelve las filas en el orden que le resulte más barato al motor. Hoy puede coincidir con el orden de la clave primaria; mañana, con un índice nuevo, un plan paralelo o más datos, puede ser otro. Si el orden importa, `ORDER BY` siempre.

Y el orden se aplica al resultado **final**: un `ORDER BY` dentro de una vista o de una subconsulta no garantiza el orden de la consulta exterior. SQL Server ni siquiera lo permite, salvo que vaya acompañado de `TOP` u `OFFSET`.

### `NULL` al ordenar

En SQL Server, los `NULL` van **primero** en orden ascendente y **últimos** en descendente. Para mandarlos al final en orden ascendente:

```sql
SELECT Nombre, Ciudad
FROM dbo.Cliente
ORDER BY CASE WHEN Ciudad IS NULL THEN 1 ELSE 0 END, Ciudad;
```

| Nombre | Ciudad |
| --- | --- |
| Luis Ramos | Arequipa |
| Sofía Mendoza | Cusco |
| Ana Torres | Lima |
| Jorge Díaz | Lima |
| María Quispe | `NULL` |

Ana y Jorge empatan en Lima, así que su orden entre ellos no está garantizado. Para fijarlo, se agrega un desempate: `..., Ciudad, Nombre`.

### `TOP`: las primeras filas

```sql
SELECT TOP (3) Nombre, Precio
FROM dbo.Producto
ORDER BY Precio DESC;
```

| Nombre | Precio |
| --- | --- |
| Monitor 27 pulgadas | 899.90 |
| Cafetera | 210.00 |
| Teclado mecánico | 120.00 |

* `TOP` sin `ORDER BY` devuelve **cualquier** grupo de 3 filas, no "las primeras". Siempre van juntos.
* Los paréntesis son opcionales con un número fijo (`TOP 3`), pero obligatorios con una variable (`TOP (@n)`); escríbelos siempre.
* `TOP (10) PERCENT` devuelve el 10 % de las filas, redondeado hacia arriba.
* Si `n` es mayor que la cantidad de filas, devuelve todas, sin error.

### `WITH TIES`: no cortar empates

Si el corte cae en medio de un empate, `TOP` elige arbitrariamente cuáles entran. `WITH TIES` incluye todos los que empatan con la última fila:

```sql
-- Los 2 productos con más unidades vendidas
SELECT TOP (2) WITH TIES ProductoId, SUM(Cantidad) AS Unidades
FROM dbo.DetallePedido
GROUP BY ProductoId
ORDER BY Unidades DESC;
```

| ProductoId | Unidades |
| --- | --- |
| 2 | 3 |
| 6 | 3 |

Aquí el corte cae justo después de un empate, así que no se agregan filas. Con `TOP (1) WITH TIES`, en cambio, se devolverían igual las dos (el mouse y Clean Code empatan con 3), mientras que `TOP (1)` sin `WITH TIES` devolvería solo uno, cualquiera de los dos. `WITH TIES` exige `ORDER BY`.

### `OFFSET ... FETCH`: paginar

Desde SQL Server 2012, la paginación estándar es:

```sql
DECLARE @Pagina INT = 2, @TamanoPagina INT = 2;

SELECT ProductoId, Nombre, Precio
FROM dbo.Producto
ORDER BY Precio DESC, ProductoId          -- desempate: ProductoId es único
OFFSET (@Pagina - 1) * @TamanoPagina ROWS
FETCH NEXT @TamanoPagina ROWS ONLY;
```

| ProductoId | Nombre | Precio |
| --- | --- | --- |
| 1 | Teclado mecánico | 120.00 |
| 6 | Clean Code | 95.00 |

* `OFFSET n ROWS` salta `n` filas; `FETCH NEXT m ROWS ONLY` devuelve las `m` siguientes.
* `ORDER BY` es **obligatorio**.
* `FETCH` es opcional: `OFFSET 10 ROWS` solo, devuelve desde la fila 11 hasta el final.

### Paginar con orden determinista

Paginar exige que el orden sea **siempre el mismo** entre una página y la siguiente. Si dos productos tienen el mismo precio y el `ORDER BY` es solo `Precio DESC`, el motor puede ponerlos en un orden distinto en cada consulta, y uno de los dos puede aparecer en las dos páginas mientras el otro no aparece en ninguna.

La regla: **el `ORDER BY` de una paginación termina con una columna única** (normalmente la clave primaria), que actúa de desempate.

### El costo de `OFFSET` en páginas lejanas

Para devolver la página 5.000, `OFFSET` tiene que leer y descartar las 99.980 filas anteriores. En tablas grandes, las páginas lejanas son cada vez más lentas. La alternativa es la **paginación por clave** (*keyset* o *seek*): en lugar de "saltar n filas", se pide "las siguientes después de la última que vi":

```sql
-- Página siguiente a la que terminó en (Precio = 120.00, ProductoId = 1)
SELECT TOP (2) ProductoId, Nombre, Precio
FROM dbo.Producto
WHERE Precio < 120.00 OR (Precio = 120.00 AND ProductoId > 1)
ORDER BY Precio DESC, ProductoId;
```

Con un índice adecuado, cada página cuesta lo mismo, sea la segunda o la milésima. El costo: no se puede saltar directamente a la página 37. Se profundiza en el módulo de índices.

-----

## Ejemplo completo

Un listado paginado de pedidos, del más reciente al más antiguo, de a 2 por página, con el total de páginas.

```sql
DECLARE @Pagina INT = 1, @TamanoPagina INT = 2;

SELECT p.PedidoId, p.FechaPedido, p.Estado
FROM dbo.Pedido AS p
ORDER BY p.FechaPedido DESC, p.PedidoId DESC
OFFSET (@Pagina - 1) * @TamanoPagina ROWS
FETCH NEXT @TamanoPagina ROWS ONLY;

SELECT CEILING(COUNT(*) * 1.0 / @TamanoPagina) AS TotalPaginas
FROM dbo.Pedido;
```

Página 1:

| PedidoId | FechaPedido | Estado |
| --- | --- | --- |
| 5 | 2026-05-02 11:00:00 | Pendiente |
| 4 | 2026-04-20 18:30:00 | Cancelado |

| TotalPaginas |
| --- |
| 3 |

Con `@Pagina = 3`, se obtiene solo el pedido 1: la última página puede tener menos filas que las demás.

Qué decisión hay detrás:

* **`FechaPedido DESC, PedidoId DESC`:** dos pedidos pueden tener la misma fecha y hora; `PedidoId` desempata y hace el orden determinista.
* **`* 1.0`:** `COUNT(*) / @TamanoPagina` es una división entera (5 / 2 = 2); multiplicar por `1.0` la convierte en decimal (2.5), y `CEILING` redondea hacia arriba (3).
* **Dos consultas:** una para la página y otra para el total. Calcular el total en la misma consulta es posible con funciones de ventana (`COUNT(*) OVER ()`), que se ven en [Funciones de ventana](10-Funciones%20de%20ventana.md).

-----

## Errores comunes

**1. Usar `LIMIT` en SQL Server.**
Qué pasa: `Msg 102: Incorrect syntax near 'LIMIT'`.
Por qué: `LIMIT` no existe en T-SQL.
Arreglo: `TOP (n)` o `OFFSET ... FETCH`.

**2. `TOP` sin `ORDER BY`.**
Qué pasa: "los 10 últimos pedidos" devuelve 10 pedidos cualesquiera.
Por qué: sin orden, no hay "primeros".
Arreglo: `TOP` siempre con `ORDER BY`.

**3. Paginar sin desempate.**
Qué pasa: filas repetidas en dos páginas y filas que no aparecen en ninguna.
Por qué: con empates, el orden entre las filas empatadas puede cambiar en cada ejecución.
Arreglo: termina el `ORDER BY` con una columna única.

**4. Confiar en el orden de una vista o subconsulta.**
Qué pasa: una vista con `ORDER BY` (y un `TOP 100 PERCENT` para que compile) devuelve las filas desordenadas.
Por qué: el orden solo se garantiza en el `ORDER BY` de la consulta exterior; el optimizador ignora `TOP 100 PERCENT ... ORDER BY`.
Arreglo: ordena en la consulta que usa la vista.

**5. `OFFSET` para páginas muy lejanas en tablas grandes.**
Qué pasa: cada página es más lenta que la anterior.
Por qué: `OFFSET` lee y descarta todas las filas anteriores.
Arreglo: paginación por clave para listados grandes o desplazamiento infinito.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL | SQLite |
| --- | --- | --- | --- | --- |
| Primeras n filas | `TOP (n)` | `LIMIT n` o `FETCH FIRST n ROWS ONLY` | `LIMIT n` | `LIMIT n` |
| Paginar | `OFFSET x ROWS FETCH NEXT n ROWS ONLY` | `LIMIT n OFFSET x` (o la forma estándar) | `LIMIT x, n` o `LIMIT n OFFSET x` | `LIMIT n OFFSET x` |
| Incluir empates | `TOP (n) WITH TIES` | `FETCH FIRST n ROWS WITH TIES` (desde la 13) | No | No |
| `NULL` en `ASC` | Primero | **Último** | Primero | Primero |
| `NULLS FIRST` / `NULLS LAST` | No (usar `CASE`) | Sí | No (usar `ORDER BY col IS NULL, col`) | Sí (desde 3.30) |
| ¿Exige `ORDER BY` para paginar? | Sí | No (pero sin él el resultado es arbitrario) | No | No |

La forma estándar SQL (`OFFSET ... FETCH`) es la única que funciona en SQL Server, PostgreSQL y Oracle sin cambios; MySQL y SQLite solo entienden `LIMIT`.

### En MongoDB

```javascript
db.productos.find().sort({ precio: -1, _id: 1 }).skip(2).limit(2);
```

`sort` (1 ascendente, -1 descendente), `skip` (como `OFFSET`) y `limit` (como `FETCH`). Los mismos principios valen: sin `sort`, el orden no está garantizado; `skip` en páginas lejanas es lento; y la paginación por clave (`find({ _id: { $gt: ultimoId } })`) es la alternativa escalable.

-----

## Cuándo sí y cuándo no

**Usa `TOP` cuando:**

* Necesitas "los n primeros" según un criterio: los 5 más vendidos, el último pedido.

**Usa `OFFSET ... FETCH` cuando:**

* Paginas listados con números de página y volúmenes moderados.

**Usa paginación por clave cuando:**

* La tabla es grande o la interfaz es de desplazamiento infinito ("cargar más").

**Ten en cuenta:**

* Ordenar tiene un costo: el motor ordena el resultado completo, salvo que un índice ya lo entregue ordenado.

-----

## Resumen en 5 líneas

1. Sin `ORDER BY` no hay orden garantizado, ni siquiera dentro de una vista o subconsulta.
2. Varias columnas: la segunda desempata la primera; `DESC` se aplica a cada columna por separado.
3. SQL Server no tiene `LIMIT`: usa `TOP (n)` (siempre con `ORDER BY`) o `OFFSET ... FETCH`.
4. `WITH TIES` incluye los empates del corte; en SQL Server los `NULL` van primero en `ASC`.
5. Para paginar, termina el `ORDER BY` con una columna única; en tablas grandes, paginación por clave.

-----

## Para profundizar

<details>
<summary>TOP en INSERT, UPDATE y DELETE</summary>

`TOP` también funciona en las instrucciones que modifican datos (`DELETE TOP (1000) FROM ...`), pero **no** admite `ORDER BY` ahí: borra 1000 filas cualesquiera. Se usa para procesar tablas grandes en lotes, repitiendo hasta que no queden filas, sin bloquear la tabla entera en una sola transacción. Si el orden importa, se elige primero qué filas con un `SELECT TOP ... ORDER BY` dentro de una CTE y después se borra a través de ella. Se ve en [Modificar datos](11-Modificar%20datos.md).

</details>

<details>
<summary>Por qué ordenar es caro</summary>

Ordenar un conjunto de n filas cuesta del orden de n·log(n) comparaciones, y si el resultado no entra en la memoria asignada a la consulta, el motor lo escribe temporalmente en `tempdb` (un *spill*). Un índice cuyas columnas coinciden con el `ORDER BY` evita el ordenamiento: las filas ya se leen en orden. Por eso `TOP (10) ... ORDER BY FechaPedido DESC` sobre una tabla con un índice en `FechaPedido` lee solo 10 filas, y sin índice lee y ordena la tabla entera.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`ORDER BY` ordena el resultado de forma ascendente o descendente, por una o varias columnas. En SQL Server se usa `TOP (n)` para quedarse con las primeras filas y `OFFSET ... FETCH` para paginar, porque `LIMIT` no existe. Sin `ORDER BY`, el orden no está garantizado.

### Respuesta ampliada (semi-senior)

El orden solo está garantizado en el `ORDER BY` de la consulta exterior: ni las vistas ni las subconsultas lo preservan. Uso `TOP` siempre con `ORDER BY`, y `WITH TIES` cuando el corte no debe partir un empate. Para paginar uso `OFFSET ... FETCH` con un orden determinista, terminando el `ORDER BY` con una columna única; sin eso, las filas empatadas pueden repetirse o perderse entre páginas. En tablas grandes, `OFFSET` degrada con las páginas lejanas porque lee y descarta las filas previas, así que prefiero la paginación por clave, que con un índice sobre las columnas del orden tiene un costo constante por página. También tengo en cuenta que en SQL Server los `NULL` van primero en orden ascendente, a diferencia de PostgreSQL.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es el equivalente de `LIMIT` en SQL Server?**
`TOP (n)` para las primeras filas, y `OFFSET x ROWS FETCH NEXT n ROWS ONLY` para paginar.

**2. ¿Qué hace `WITH TIES`?**
Agrega al resultado de `TOP` todas las filas que empatan con la última según el `ORDER BY`.

**3. ¿Por qué hace falta un desempate al paginar?**
Porque sin un orden determinista, las filas empatadas pueden aparecer en un orden distinto en cada consulta, y repetirse o perderse entre páginas.

**4. ¿Qué problema tiene `OFFSET` con páginas lejanas?**
Que tiene que leer y descartar todas las filas anteriores. La paginación por clave lo evita.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta para cada caso:

1. Los 2 clientes registrados más recientemente.
2. Los productos ordenados por categoría y, dentro de cada categoría, alfabéticamente.
3. La segunda página de clientes, de a 2, ordenados por nombre.

<details>
<summary>Solución</summary>

```sql
-- 1
SELECT TOP (2) Nombre, FechaRegistro
FROM dbo.Cliente
ORDER BY FechaRegistro DESC;

-- 2
SELECT CategoriaId, Nombre
FROM dbo.Producto
ORDER BY CategoriaId, Nombre;

-- 3
SELECT ClienteId, Nombre
FROM dbo.Cliente
ORDER BY Nombre, ClienteId
OFFSET 2 ROWS FETCH NEXT 2 ROWS ONLY;
```

Resultados:

1. Sofía Mendoza (2026-04-01) y Jorge Díaz (2026-03-15).
2. Categoría 1: Monitor 27 pulgadas, Mouse inalámbrico, Teclado mecánico; categoría 2: Cafetera, Lámpara de escritorio; categoría 3: Clean Code.
3. Orden por nombre: Ana, Jorge, Luis, María, Sofía. La página 2 es **Luis Ramos** y **María Quispe**.

</details>

**Ejercicio 2.** Escribe la consulta de "el producto más vendido en unidades", de forma que si hay un empate en el primer puesto aparezcan todos los empatados.

<details>
<summary>Solución</summary>

```sql
SELECT TOP (1) WITH TIES p.Nombre, SUM(d.Cantidad) AS Unidades
FROM dbo.DetallePedido AS d
JOIN dbo.Producto AS p ON p.ProductoId = d.ProductoId
GROUP BY p.Nombre
ORDER BY Unidades DESC;
```

| Nombre | Unidades |
| --- | --- |
| Mouse inalámbrico | 3 |
| Clean Code | 3 |

Sin `WITH TIES`, devolvería uno solo de los dos, y cuál sería no está garantizado. El orden entre las dos filas tampoco: el `ORDER BY` solo dice que van primero las de más unidades.

</details>

-----

## Siguiente lección

[Expresiones, CASE y funciones](04-Expresiones%2C%20CASE%20y%20funciones.md)
