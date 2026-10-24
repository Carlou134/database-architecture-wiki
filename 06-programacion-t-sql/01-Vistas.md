# Vistas

## En una frase

Una vista es una **consulta guardada con nombre**, que se usa como si fuera una tabla; no guarda datos (salvo que sea indexada), sino que el motor la expande dentro de cada consulta que la usa, y sirve para simplificar consultas repetidas, ocultar columnas o filas sensibles y mantener estable lo que ven las aplicaciones aunque cambien las tablas de abajo.

-----

## Antes de empezar

Conviene que ya sepas:

* Joins, agregados y CTE, del módulo de [Consultas](../03-consultas/README.md).
* Vistas indexadas, de [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md).
* El borrado lógico y su filtro centralizado, de [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Vista:** consulta con nombre que se usa como una tabla.
* **Expansión de la vista:** el reemplazo de la vista por su definición dentro de la consulta que la usa, antes de optimizar.
* **`WITH SCHEMABINDING`:** opción que impide modificar las tablas de la vista de forma que la rompan.
* **`WITH CHECK OPTION`:** opción que impide insertar o actualizar a través de la vista filas que la vista no mostraría.
* **Vista actualizable:** vista a través de la cual se pueden hacer `INSERT`, `UPDATE` y `DELETE`.
* **Encadenamiento de propiedad (ownership chaining):** regla por la cual, si la vista y sus tablas tienen el mismo dueño, basta tener permiso sobre la vista.

-----

## El problema

La consulta "pedidos con su cliente y su total" aparece en el panel de administración, en tres reportes, en la exportación contable y en la API móvil:

```sql
SELECT p.PedidoId, p.FechaPedido, p.Estado, c.Nombre, c.Email,
       SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY p.PedidoId, p.FechaPedido, p.Estado, c.Nombre, c.Email;
```

Seis copias de la misma lógica. El día que el negocio decide que el total incluye un costo de envío, hay que encontrar y cambiar las seis, y alguna se olvida. Además, el equipo de reportes tiene acceso a la tabla `Cliente` completa solo para leer el nombre, cuando en esa tabla también están el documento y el teléfono.

-----

## Cómo funciona

### Crear y usar una vista

```sql
CREATE OR ALTER VIEW dbo.vPedidoResumen
AS
SELECT p.PedidoId, p.FechaPedido, p.Estado,
       c.ClienteId, c.Nombre AS Cliente, c.Email,
       SUM(d.Cantidad * d.PrecioUnitario) AS Total,
       COUNT(*) AS Lineas
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY p.PedidoId, p.FechaPedido, p.Estado, c.ClienteId, c.Nombre, c.Email;
GO

SELECT PedidoId, Cliente, Total
FROM dbo.vPedidoResumen
WHERE Estado <> 'Cancelado'
ORDER BY Total DESC;
```

| PedidoId | Cliente | Total |
| --- | --- | --- |
| 2 | Luis Ramos | 849.90 |
| 3 | Ana Torres | 265.00 |
| 1 | Ana Torres | 211.00 |
| 5 | Jorge Díaz | 140.50 |

* `CREATE VIEW` debe ser la **primera** instrucción del lote (por eso el `GO` antes y después).
* `CREATE OR ALTER` (desde SQL Server 2016 SP1) crea la vista o la reemplaza, sin perder los permisos asignados.
* Todas las columnas tienen que tener nombre: las expresiones (`SUM(...)`) necesitan un alias.

### Una vista no guarda datos

Cuando una consulta usa `dbo.vPedidoResumen`, el motor **reemplaza** la vista por su definición y optimiza la consulta resultante completa. Es como si se hubiera escrito la consulta larga a mano: el filtro `WHERE Estado <> 'Cancelado'` se puede aplicar antes del `GROUP BY`, y si la consulta no usa una columna de la vista, el motor puede incluso evitar un join innecesario.

Dos consecuencias:

* **La vista no es más rápida ni más lenta** que la consulta equivalente: es la misma consulta con nombre. (La excepción son las vistas **indexadas**, que sí guardan el resultado: ver [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md).)
* **Los datos siempre están actualizados:** cada consulta lee las tablas en ese momento.

### Para qué sirven

| Uso | Ejemplo |
| --- | --- |
| **Reutilizar lógica** | El cálculo del total del pedido, en un solo lugar |
| **Seguridad por columnas** | Una vista de clientes sin documento ni teléfono, con permiso solo sobre la vista |
| **Seguridad por filas** | Una vista que solo muestra los pedidos de la sucursal del usuario |
| **Filtros que no se pueden olvidar** | Una vista `vClienteActivo` con `WHERE EliminadoEn IS NULL`, para el borrado lógico |
| **Estabilidad ante cambios** | Si una tabla se divide en dos, una vista con el nombre y las columnas viejas mantiene funcionando a las aplicaciones mientras se migran |

### `ORDER BY` en una vista

Una vista no puede tener `ORDER BY` (`Msg 1033`), salvo con `TOP` u `OFFSET`. El truco `SELECT TOP 100 PERCENT ... ORDER BY` compila, pero **el orden no se garantiza**: el optimizador lo ignora. Como se vio en [Ordenar y paginar](../03-consultas/03-Ordenar%20y%20paginar.md), el orden se pide en la consulta que usa la vista.

### `SCHEMABINDING`

Sin opciones, una vista depende de sus tablas sin protegerlas: alguien puede borrar una columna que la vista usa, y la vista falla la próxima vez que se consulta. Con `WITH SCHEMABINDING`:

```sql
CREATE OR ALTER VIEW dbo.vClientePublico
WITH SCHEMABINDING
AS
SELECT ClienteId, Nombre, Ciudad
FROM dbo.Cliente;                -- nombres de dos partes obligatorios
```

* No se pueden modificar las columnas que la vista usa, ni borrar la tabla, sin borrar o cambiar antes la vista.
* Exige nombres de dos partes (`dbo.Cliente`) y prohíbe `SELECT *`.
* Es un requisito para las vistas indexadas, y para que una función o vista que la usa pueda considerarse determinista.

### El problema de `SELECT *` en las vistas

```sql
CREATE VIEW dbo.vCliente AS SELECT * FROM dbo.Cliente;
```

La lista de columnas de la vista se fija **al crearla**. Si después se agrega una columna a `Cliente`, la vista **no** la muestra; y si se agregan y quitan columnas, las columnas de la vista pueden quedar desfasadas respecto de sus nombres, devolviendo datos de una columna con el nombre de otra. `sp_refreshview 'dbo.vCliente'` actualiza los metadatos, pero la solución es no usar `SELECT *` en las vistas (y `SCHEMABINDING` directamente lo impide).

### Modificar datos a través de una vista

Una vista que lee de **una sola** tabla, sin agregados, `DISTINCT` ni `TOP`, es **actualizable**: se puede hacer `INSERT`, `UPDATE` o `DELETE` sobre ella, y el cambio va a la tabla. Con `WITH CHECK OPTION`, el motor rechaza los cambios que dejarían la fila fuera de la vista:

```sql
CREATE OR ALTER VIEW dbo.vProductoActivo
AS
SELECT ProductoId, CategoriaId, Nombre, Precio, Stock, Activo
FROM dbo.Producto
WHERE Activo = 1
WITH CHECK OPTION;
GO

UPDATE dbo.vProductoActivo SET Activo = 0 WHERE ProductoId = 1;
```

```text
Msg 550, Level 16, State 1
The attempted insert or update failed because the target view either specifies WITH CHECK OPTION
or spans a view that specifies WITH CHECK OPTION and one or more rows resulting from the operation
did not qualify under the CHECK OPTION constraint.
```

Para vistas de varias tablas, la modificación tiene que afectar a una sola tabla base; para casos más complejos existen los *triggers* `INSTEAD OF` (ver [Triggers](04-Triggers.md)).

### Seguridad: el encadenamiento de propiedad

```sql
GRANT SELECT ON dbo.vClientePublico TO RolReportes;
-- RolReportes NO tiene permisos sobre dbo.Cliente
```

`RolReportes` puede consultar la vista aunque no tenga permiso sobre la tabla, porque la vista y la tabla tienen el mismo dueño (`dbo`): el motor verifica el permiso sobre la vista y no vuelve a verificarlo sobre la tabla. Es el **encadenamiento de propiedad**, la base de la seguridad con vistas y procedimientos. Se rompe si la vista y la tabla tienen dueños distintos.

### El antipatrón: vistas sobre vistas

Una vista puede usar otras vistas. Con dos niveles, es razonable; con cinco, se vuelve un problema:

* Cada nivel agrega joins que la consulta final puede no necesitar, y el optimizador no siempre logra eliminarlos.
* Leer una consulta exige abrir cinco definiciones para saber qué tablas toca.
* Un cambio en la vista de abajo afecta a todo lo de arriba sin que se note.

Si una consulta sobre una vista es lenta y la vista se apoya en otras, el primer paso es mirar el plan: muchas veces aparecen tablas que la consulta no necesitaba.

-----

## Ejemplo completo

Una capa de vistas para el equipo de reportes, con seguridad por columnas y la lógica del total en un solo lugar:

```sql
USE Tienda;
GO

-- 1. Clientes sin datos sensibles
CREATE OR ALTER VIEW reportes.vCliente
WITH SCHEMABINDING
AS
SELECT ClienteId, Nombre, Ciudad, FechaRegistro
FROM dbo.Cliente;
GO

-- 2. El total del pedido, definido una sola vez
CREATE OR ALTER VIEW reportes.vVentas
WITH SCHEMABINDING
AS
SELECT p.PedidoId, p.ClienteId, p.FechaPedido, p.Estado,
       SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.Pedido AS p
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
WHERE p.Estado <> 'Cancelado'          -- regla de negocio: lo cancelado no es venta
GROUP BY p.PedidoId, p.ClienteId, p.FechaPedido, p.Estado;
GO

-- 3. Permisos: solo sobre las vistas
GRANT SELECT ON SCHEMA::reportes TO RolReportes;
```

(El esquema `reportes` se crea antes con `CREATE SCHEMA reportes;`, y el rol con `CREATE ROLE RolReportes;`.)

El equipo de reportes consulta así:

```sql
SELECT c.Nombre, c.Ciudad, SUM(v.Total) AS Comprado
FROM reportes.vVentas AS v
JOIN reportes.vCliente AS c ON c.ClienteId = v.ClienteId
GROUP BY c.Nombre, c.Ciudad
ORDER BY Comprado DESC;
```

| Nombre | Ciudad | Comprado |
| --- | --- | --- |
| Luis Ramos | Arequipa | 849.90 |
| Ana Torres | Lima | 476.00 |
| Jorge Díaz | Lima | 140.50 |

Qué decisión hay detrás:

* **Un esquema `reportes`** agrupa las vistas y permite dar el permiso de una vez a todo el esquema (ver [Convenciones de nombres](../02-diseno-y-modelado/10-Convenciones%20de%20nombres.md)).
* **`reportes.vCliente` no tiene el email:** el equipo de reportes no lo necesita, y por el encadenamiento de propiedad no tiene forma de llegar a la tabla.
* **La regla "lo cancelado no es venta" vive en la vista:** si cambia, se cambia en un solo lugar.
* **`SCHEMABINDING`:** nadie puede borrar `Pedido.Estado` sin antes cambiar la vista que depende de ella.

-----

## Errores comunes

**1. `SELECT *` en una vista.**
Qué pasa: columnas nuevas que no aparecen, o columnas desfasadas después de cambios en la tabla.
Por qué: la lista de columnas se fija al crear la vista.
Arreglo: listar las columnas; `SCHEMABINDING` lo exige.

**2. Esperar que la vista sea más rápida.**
Qué pasa: "creé una vista y la consulta sigue igual".
Por qué: una vista común se expande en la consulta; no guarda resultados.
Arreglo: optimizar la consulta (índices); si el resultado es caro y se lee mucho, una vista indexada.

**3. `ORDER BY` en la vista.**
Qué pasa: `Msg 1033`, o con `TOP 100 PERCENT`, un orden que no se respeta.
Por qué: el orden solo se garantiza en la consulta final.
Arreglo: ordenar en la consulta que usa la vista.

**4. Vistas apiladas en muchos niveles.**
Qué pasa: consultas con joins que nadie necesitaba y planes difíciles de leer.
Por qué: cada vista agrega su propia lógica.
Arreglo: pocos niveles; vistas que lean de tablas, no de otras vistas.

**5. Vistas sin `SCHEMABINDING` sobre tablas que cambian.**
Qué pasa: un cambio en una tabla rompe una vista que nadie recordaba.
Por qué: las dependencias no están protegidas.
Arreglo: `SCHEMABINDING` en las vistas importantes, y `sys.sql_expression_dependencies` para ver qué depende de qué antes de un cambio.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Crear o reemplazar | `CREATE OR ALTER VIEW` | `CREATE OR REPLACE VIEW` (sin cambiar columnas existentes) | `CREATE OR REPLACE VIEW` |
| `ORDER BY` en la vista | No (salvo con `TOP`/`OFFSET`) | Sí (pero el orden final lo decide la consulta exterior) | Sí (con la misma advertencia) |
| Proteger dependencias | `WITH SCHEMABINDING` | Siempre: no se puede borrar una columna usada por una vista sin `CASCADE` | No |
| `WITH CHECK OPTION` | Sí | Sí (`LOCAL` / `CASCADED`) | Sí |
| Vista materializada | Vista indexada (siempre actualizada) | `MATERIALIZED VIEW` + `REFRESH` | No |
| Seguridad por filas | Vistas, o *Row-Level Security* (políticas) | Vistas, o *Row-Level Security* (políticas) | Vistas |

PostgreSQL protege las dependencias de las vistas por defecto, sin una opción: es más seguro, a costa de tener que borrar y recrear vistas para cambiar las tablas de abajo.

### En MongoDB

MongoDB tiene **vistas de solo lectura** definidas con un *aggregation pipeline* (`db.createView("vPedidoResumen", "pedidos", [ ...etapas... ])`), que, como en SQL Server, no guardan datos y se calculan al consultarlas. Para resultados guardados, se usa un pipeline con `$merge` o `$out` hacia otra colección (una "vista materializada" manual, que hay que refrescar).

-----

## Cuándo sí y cuándo no

**Usa una vista cuando:**

* La misma lógica de consulta se repite en varios lugares.
* Necesitas exponer solo algunas columnas o filas a un grupo de usuarios.
* Quieres mantener un "contrato" estable para las aplicaciones mientras cambias las tablas.

**No uses una vista cuando:**

* Necesitas parámetros: una función de tabla en línea es una "vista con parámetros" (ver [Funciones](03-Funciones.md)).
* La quieres para acelerar una consulta: una vista común no la acelera.

-----

## Resumen en 5 líneas

1. Una vista es una consulta con nombre; el motor la expande dentro de cada consulta, así que no guarda datos ni acelera por sí misma.
2. Sirve para reutilizar lógica, ocultar columnas o filas y mantener estable lo que ven las aplicaciones.
3. Sin `ORDER BY` y sin `SELECT *`; con `SCHEMABINDING` para proteger sus dependencias.
4. Las vistas de una tabla son actualizables; `WITH CHECK OPTION` rechaza cambios que la dejarían fuera de la vista.
5. Con el mismo dueño, basta el permiso sobre la vista (encadenamiento de propiedad); evita apilar vistas en muchos niveles.

-----

## Para profundizar

<details>
<summary>Row-Level Security</summary>

Desde SQL Server 2016, *Row-Level Security* permite filtrar filas por usuario sin vistas: se define una función de predicado (una función de tabla en línea que devuelve una fila si el usuario puede ver la fila) y una política de seguridad que la aplica a la tabla. A diferencia de una vista, el filtro se aplica a **toda** consulta sobre la tabla, sin que el código tenga que acordarse de usar la vista. Es útil para sistemas con varios clientes en la misma base (*multi-tenant*).

</details>

<details>
<summary>Ver las dependencias antes de cambiar una tabla</summary>

```sql
SELECT OBJECT_SCHEMA_NAME(referencing_id) AS Esquema,
       OBJECT_NAME(referencing_id)        AS Objeto,
       referenced_entity_name             AS Usa
FROM sys.sql_expression_dependencies
WHERE referenced_id = OBJECT_ID('dbo.Pedido');
```

Lista las vistas, procedimientos, funciones y *triggers* que mencionan `dbo.Pedido`. Es el primer paso antes de renombrar o borrar columnas. El SQL dinámico no aparece aquí: el motor no puede saber qué tablas usa un texto que se arma en tiempo de ejecución.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una vista es una consulta guardada con nombre que se usa como una tabla. Sirve para no repetir consultas complejas y para dar acceso solo a algunas columnas o filas. No guarda datos: cada vez que se consulta, se ejecuta la consulta que la define.

### Respuesta ampliada (semi-senior)

Uso las vistas para centralizar lógica repetida (como la definición de "venta"), para seguridad por columnas o filas aprovechando el encadenamiento de propiedad (permiso sobre la vista, no sobre las tablas) y como contrato estable mientras cambio las tablas. Sé que una vista común se expande dentro de la consulta antes de optimizar, así que no la acelera; para eso están las vistas indexadas. Las creo con columnas explícitas y `SCHEMABINDING`, que protege sus dependencias y es requisito de las indexadas, y nunca con `SELECT *`, porque sus columnas se fijan al crearla. No pongo `ORDER BY` en las vistas, porque el orden solo vale en la consulta final. Las vistas de una sola tabla son actualizables, y con `WITH CHECK OPTION` impido cambios que dejen la fila fuera de la vista. Evito apilar vistas, porque arrastran joins innecesarios y vuelven ilegibles los planes; si necesito parámetros, uso una función de tabla en línea.

### Preguntas frecuentes de seguimiento

**1. ¿Una vista mejora el rendimiento?**
Una vista común no: se expande dentro de la consulta. Una vista indexada sí, porque guarda el resultado.

**2. ¿Qué hace `SCHEMABINDING`?**
Impide modificar o borrar las columnas y tablas que la vista usa sin cambiar antes la vista.

**3. ¿Por qué no usar `SELECT *` en una vista?**
Porque la lista de columnas se fija al crearla: las columnas nuevas no aparecen, y los cambios pueden desfasar los nombres.

**4. ¿Cómo das acceso a una vista sin dar acceso a la tabla?**
Con `GRANT SELECT` sobre la vista; si la vista y la tabla tienen el mismo dueño, el encadenamiento de propiedad evita verificar el permiso sobre la tabla.

-----

## Práctica

**Ejercicio 1.** Crea una vista `dbo.vProductoCatalogo` con los productos activos, el nombre de su categoría y un indicador de disponibilidad ("Disponible" o "Sin stock"), protegida contra cambios en las tablas.

<details>
<summary>Solución</summary>

```sql
CREATE OR ALTER VIEW dbo.vProductoCatalogo
WITH SCHEMABINDING
AS
SELECT p.ProductoId,
       p.Nombre,
       c.Nombre AS Categoria,
       p.Precio,
       CASE WHEN p.Stock > 0 THEN N'Disponible' ELSE N'Sin stock' END AS Disponibilidad
FROM dbo.Producto AS p
JOIN dbo.Categoria AS c ON c.CategoriaId = p.CategoriaId
WHERE p.Activo = 1;
GO

SELECT Nombre, Categoria, Precio, Disponibilidad
FROM dbo.vProductoCatalogo
ORDER BY Categoria, Nombre;
```

| Nombre | Categoria | Precio | Disponibilidad |
| --- | --- | --- | --- |
| Monitor 27 pulgadas | Electrónica | 899.90 | Disponible |
| Mouse inalámbrico | Electrónica | 45.50 | Disponible |
| Teclado mecánico | Electrónica | 120.00 | Disponible |
| Lámpara de escritorio | Hogar | 75.00 | Disponible |
| Clean Code | Libros | 95.00 | Disponible |

La cafetera no aparece porque está inactiva (y por eso tampoco aparece como "Sin stock"). Con `SCHEMABINDING`, un `ALTER TABLE dbo.Producto DROP COLUMN Stock` falla mientras exista la vista.

</details>

**Ejercicio 2.** Sobre la vista `dbo.vProductoActivo` (con `WITH CHECK OPTION`), ¿cuáles de estas instrucciones funcionan?

```sql
-- a
UPDATE dbo.vProductoActivo SET Precio = 130.00 WHERE ProductoId = 1;
-- b
UPDATE dbo.vProductoActivo SET Activo = 0 WHERE ProductoId = 2;
-- c
INSERT INTO dbo.vProductoActivo (CategoriaId, Nombre, Precio, Stock, Activo) VALUES (1, N'Webcam', 150, 5, 1);
-- d
UPDATE dbo.vProductoActivo SET Precio = 200.00 WHERE ProductoId = 5;
```

<details>
<summary>Solución</summary>

* **a: funciona.** El teclado sigue activo después del cambio.
* **b: falla** (`Msg 550`): la fila dejaría de cumplir `Activo = 1`.
* **c: funciona.** La fila nueva cumple el filtro de la vista; el `IDENTITY` lo genera la tabla.
* **d: no falla, pero actualiza 0 filas.** La cafetera (5) está inactiva, así que no es visible a través de la vista: el `UPDATE` no la encuentra. La vista también sirve para que una aplicación no pueda tocar lo que no ve.

</details>

-----

## Siguiente lección

[Procedimientos almacenados](02-Procedimientos%20almacenados.md)
