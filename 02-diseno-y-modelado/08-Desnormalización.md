# Desnormalización

## En una frase

Desnormalizar es introducir redundancia **a propósito** en un diseño normalizado, para que ciertas lecturas sean más rápidas o más simples, aceptando a cambio el costo de mantener las copias sincronizadas; se hace después de normalizar, con una necesidad medida y con un mecanismo que garantice la consistencia.

-----

## Antes de empezar

Conviene que ya sepas:

* 1FN a BCNF y las anomalías que evitan, de [Normalización 1FN 2FN 3FN](06-Normalización%201FN%202FN%203FN.md) y [BCNF y formas superiores](07-BCNF%20y%20formas%20superiores.md).
* Atributos derivados y columnas calculadas, de [Del modelo ER a tablas](04-Del%20modelo%20ER%20a%20tablas.md).
* OLTP frente a OLAP, de [Qué es una base de datos y un SGBD](../01-fundamentos/01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Desnormalización:** redundancia introducida a propósito para mejorar la lectura.
* **Fuente de verdad:** el lugar donde vive el valor original; las copias se derivan de él.
* **Vista indexada (SQL Server) / vista materializada:** una vista cuyo resultado se guarda físicamente y se mantiene actualizado.
* **Esquema estrella:** diseño analítico con una tabla de hechos central y tablas de dimensiones desnormalizadas alrededor.
* **Snapshot (instantánea) histórica:** copia de un dato tal como era en el momento de un hecho, que **no** debe cambiar después.

-----

## El problema

La tienda creció. La pantalla principal del panel de administración lista los últimos 50 pedidos con su total:

```sql
SELECT TOP (50) p.PedidoId, p.FechaPedido, c.Nombre,
       SUM(d.Cantidad * d.PrecioUnitario) AS Total
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY p.PedidoId, p.FechaPedido, c.Nombre
ORDER BY p.FechaPedido DESC;
```

Con 5 pedidos, es instantánea. Con 20 millones de pedidos y 80 millones de detalles, y con el panel refrescándose cada pocos segundos para cien usuarios, la consulta empieza a consumir CPU y a competir con las inserciones de pedidos nuevos.

La tentación es inmediata: "agreguemos una columna `Total` en `Pedido`". Puede ser la decisión correcta. Pero si se hace sin pensar, aparece la anomalía que la normalización evitaba: alguien modifica un detalle, nadie actualiza el total, y el panel muestra un importe que no coincide con la factura.

Desnormalizar no es un error; desnormalizar **sin control** sí.

-----

## Cómo funciona

### Primero normalizar, después medir, después desnormalizar

El orden importa:

1. **Normaliza** (3FN o BCNF). Es el diseño de referencia y la fuente de verdad.
2. **Mide** la consulta lenta: plan de ejecución, tiempos, frecuencia. Muchas veces el problema es un **índice que falta**, no el diseño.
3. **Prueba las alternativas sin redundancia:** índices, reescribir la consulta, caché en la aplicación.
4. **Desnormaliza** solo si lo anterior no alcanza, y con un mecanismo de sincronización.

Desnormalizar sin haber normalizado no es desnormalizar: es un diseño redundante sin fuente de verdad.

### No todo lo que parece redundante lo es

Antes de seguir, una distinción importante. `DetallePedido.PrecioUnitario` parece una copia de `Producto.Precio`, pero **no** es desnormalización:

| | Copia redundante | Snapshot histórica |
| --- | --- | --- |
| Ejemplo | `Pedido.NombreCliente` copiado de `Cliente.Nombre` | `DetallePedido.PrecioUnitario` (el precio pagado) |
| ¿Debe cambiar si cambia el original? | Sí: si no cambia, está mal | **No**: si cambia, está mal |
| ¿Es un hecho distinto? | No, es el mismo hecho repetido | Sí: "lo que se pagó" no es "lo que cuesta hoy" |
| ¿Viola la normalización? | Sí | No: tiene su propio determinante |

Otros ejemplos de snapshots legítimas: la dirección de envío de un pedido, la razón social en una factura emitida, el nombre del plan en una suscripción contratada. La pregunta que las distingue es **"¿si el original cambia, esta copia debería cambiar?"**.

### Técnicas de desnormalización

| Técnica | Ejemplo | Cómo se mantiene |
| --- | --- | --- |
| **Columna calculada persistida** | `DetallePedido.Subtotal AS (Cantidad * PrecioUnitario) PERSISTED` | El motor; no puede desincronizarse |
| **Vista indexada / materializada** | Totales por pedido precalculados | El motor (SQL Server) o un refresco periódico (PostgreSQL) |
| **Columna agregada** | `Pedido.Total` | *Trigger*, la misma transacción de la aplicación o un proceso periódico |
| **Columna copiada de otra tabla** | `Pedido.NombreCliente` | *Trigger* o la aplicación, al cambiar el original |
| **Tabla de resumen** | `VentaDiaria(Fecha, CategoriaId, Total)` | Proceso periódico (ETL nocturno) |
| **Esquema analítico separado** | Data warehouse en estrella | ETL desde el OLTP |

Están ordenadas de **más segura** a **más riesgosa** dentro de la misma base: las primeras las mantiene el motor; las del medio dependen de código que hay que escribir, probar y no olvidar.

### Columnas calculadas: desnormalización sin riesgo

Cuando el valor sale de columnas **de la misma fila**, una columna calculada persistida guarda el resultado y el motor lo recalcula en cada cambio:

```sql
ALTER TABLE dbo.DetallePedido
    ADD Subtotal AS (Cantidad * PrecioUnitario) PERSISTED;
```

No se puede escribir a mano, así que no puede quedar desactualizada. Y se puede indexar.

### Vistas indexadas: agregados mantenidos por el motor

Cuando el valor sale de **varias filas**, SQL Server permite materializar una vista con un índice agrupado único. El motor la actualiza en la misma transacción que modifica las tablas base:

```sql
CREATE VIEW dbo.vTotalPedido
WITH SCHEMABINDING
AS
SELECT d.PedidoId,
       SUM(d.Cantidad * d.PrecioUnitario) AS Total,
       COUNT_BIG(*) AS Lineas
FROM dbo.DetallePedido AS d
GROUP BY d.PedidoId;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vTotalPedido ON dbo.vTotalPedido (PedidoId);
```

Requisitos clave: `WITH SCHEMABINDING`, nombres de dos partes (`dbo.Tabla`), `COUNT_BIG(*)` cuando hay `GROUP BY` y columnas de la suma que no admitan `NULL` (si la expresión puede ser nula, SQL Server la rechaza). El costo: cada `INSERT`, `UPDATE` o `DELETE` en `DetallePedido` actualiza también la vista, así que las escrituras se vuelven un poco más lentas.

En la edición Standard, el optimizador solo usa la vista si la consulta la nombra con la sugerencia `WITH (NOEXPAND)`; en Enterprise, puede usarla automáticamente aunque la consulta lea las tablas base.

### Columnas agregadas mantenidas a mano

Si se decide guardar `Pedido.Total`, **todo** cambio en los detalles tiene que actualizarlo. Las opciones:

* **En la misma transacción de la aplicación:** simple, pero cualquier cliente que escriba en `DetallePedido` sin pasar por ese código rompe la regla.
* **Con un *trigger*:** lo hace cumplir la base, para todos los clientes; a cambio, es lógica escondida que hay que conocer y probar.
* **Con un proceso periódico:** aceptable si el valor puede estar desactualizado unos minutos (un contador de "vistas" de un producto).

Y siempre con una forma de **verificar**: una consulta que compare el valor guardado con el calculado sirve para detectar desincronizaciones.

### Desnormalización analítica: el esquema estrella

Para reportes sobre años de datos, la solución no es desnormalizar la base OLTP, sino copiar los datos a un **almacén analítico** con otro diseño:

```text
                 ┌───────────────┐
                 │  DimFecha     │
                 │  Fecha, Mes,  │
                 │  Trimestre    │
                 └──────┬────────┘
┌──────────────┐ ┌──────┴────────┐ ┌──────────────────┐
│ DimCliente   ├─┤  HechoVenta   ├─┤ DimProducto      │
│ Nombre,      │ │  Cantidad,    │ │ Nombre,          │
│ Ciudad       │ │  Importe      │ │ Categoria (texto)│
└──────────────┘ └───────────────┘ └──────────────────┘
```

* **Tabla de hechos:** cada venta, con las medidas (cantidad, importe) y claves hacia las dimensiones.
* **Dimensiones:** desnormalizadas a propósito. `DimProducto` guarda el nombre de la categoría como texto, sin tabla aparte, para que los reportes no necesiten joins encadenados.

La base OLTP sigue normalizada para escribir bien; el almacén está desnormalizado para leer bien. Cada uno con su propósito.

-----

## Ejemplo completo

El panel de pedidos del problema, resuelto en tres pasos:

**Paso 1 — medir y probar un índice.** La consulta ordena por `FechaPedido`; sin índice, recorre toda la tabla. Un índice sobre la fecha puede bastar:

```sql
CREATE INDEX IX_Pedido_FechaPedido ON dbo.Pedido (FechaPedido DESC) INCLUDE (ClienteId);
```

Con esto, los 50 pedidos más recientes se leen directamente. Pero el total todavía se suma desde los detalles.

**Paso 2 — materializar el total con una vista indexada** (la vista `dbo.vTotalPedido` de arriba). La consulta del panel queda:

```sql
SELECT TOP (50) p.PedidoId, p.FechaPedido, c.Nombre, t.Total
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
JOIN dbo.vTotalPedido AS t WITH (NOEXPAND) ON t.PedidoId = p.PedidoId
ORDER BY p.FechaPedido DESC;
```

Resultado con los datos de ejemplo:

| PedidoId | FechaPedido | Nombre | Total |
| --- | --- | --- | --- |
| 5 | 2026-05-02 11:00 | Jorge Díaz | 140.50 |
| 4 | 2026-04-20 18:30 | María Quispe | 210.00 |
| 3 | 2026-04-12 09:05 | Ana Torres | 265.00 |
| 2 | 2026-03-05 16:40 | Luis Ramos | 849.90 |
| 1 | 2026-03-01 10:15 | Ana Torres | 211.00 |

**Paso 3 — verificar la consistencia.** Con una vista indexada, el motor la garantiza. Si se hubiera elegido una columna `Pedido.Total` mantenida a mano, esta consulta detectaría las diferencias:

```sql
SELECT p.PedidoId, p.Total AS Guardado, SUM(d.Cantidad * d.PrecioUnitario) AS Calculado
FROM dbo.Pedido AS p
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
GROUP BY p.PedidoId, p.Total
HAVING p.Total <> SUM(d.Cantidad * d.PrecioUnitario);
```

Qué cambió y por qué:

* **Se midió y se indexó primero:** el índice resolvió el orden sin agregar redundancia.
* **Se eligió la técnica más segura disponible:** la vista indexada guarda el total y el motor la mantiene en cada transacción.
* **La fuente de verdad sigue siendo `DetallePedido`:** el total es una copia derivada, nunca un dato que se escriba por su cuenta.

-----

## Errores comunes

**1. Desnormalizar por anticipado.**
Qué pasa: columnas copiadas "por rendimiento" en un sistema que todavía no tiene usuarios, y bugs de sincronización desde el primer día.
Por qué: se asume que los joins son lentos.
Arreglo: normaliza, mide y desnormaliza solo lo que una medición justifique.

**2. Copias sin mecanismo de sincronización.**
Qué pasa: `Pedido.NombreCliente` dice "Ana Torres" y `Cliente.Nombre` dice "Ana Torres de Ruiz".
Por qué: se agregó la copia, pero nadie escribió el código que la actualiza.
Arreglo: cada copia redundante necesita un dueño: el motor (columna calculada, vista indexada), un *trigger* o una transacción, y una consulta que verifique.

**3. Confundir una snapshot con una copia redundante.**
Qué pasa: un "arreglo" de normalización hace que los pedidos viejos lean el precio actual del producto, y los importes históricos cambian.
Por qué: se ve redundancia donde hay un hecho distinto.
Arreglo: pregunta "¿si el original cambia, esta copia debe cambiar?". Si la respuesta es no, no es redundancia.

**4. Desnormalizar la base OLTP para los reportes.**
Qué pasa: tablas transaccionales llenas de columnas agregadas y *triggers*, con escrituras cada vez más lentas.
Por qué: se usa la misma base para escribir y para analizar.
Arreglo: réplicas de lectura, un almacén analítico o tablas de resumen alimentadas por un proceso aparte.

**5. Olvidar el costo en las escrituras.**
Qué pasa: una vista indexada o un *trigger* sobre una tabla con miles de inserciones por segundo multiplica el costo de cada escritura y genera bloqueos.
Por qué: se mide solo la lectura que mejoró.
Arreglo: mide también el impacto en las escrituras antes de decidir.

-----

## Según el motor

| Técnica | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Columna calculada guardada | `AS (...) PERSISTED` | `GENERATED ALWAYS AS (...) STORED` | `AS (...) STORED` |
| Vista materializada | Vista indexada: se actualiza **sincrónicamente** en cada transacción | `CREATE MATERIALIZED VIEW`: se actualiza solo con `REFRESH MATERIALIZED VIEW` (manual o programado) | No existe; se simula con tablas y *triggers* o eventos |
| Refresco sin bloquear lecturas | No aplica (siempre está al día) | `REFRESH MATERIALIZED VIEW CONCURRENTLY` (requiere un índice único) | — |
| Almacenamiento analítico | Índices *columnstore* en la misma base | Extensiones (Citus, TimescaleDB) | Motores separados (HeatWave) |

La diferencia más importante: la vista indexada de SQL Server **nunca** está desactualizada, pero encarece cada escritura; la vista materializada de PostgreSQL no encarece las escrituras, pero muestra los datos del último refresco.

### En MongoDB

En MongoDB, la desnormalización es la **norma**, no la excepción: se embeben y se copian datos para que cada consulta frecuente lea un solo documento. Los mismos principios siguen valiendo: decidir cuál es la fuente de verdad, distinguir copias de snapshots (el nombre del producto en un pedido viejo es una snapshot) y tener un mecanismo para actualizar las copias, ya sea en la misma operación, con transacciones multidocumento o con procesos que escuchan los cambios (*change streams*).

-----

## Cuándo sí y cuándo no

**Desnormaliza cuando:**

* Una consulta frecuente es lenta, los índices no alcanzan y la medición lo demuestra.
* El dato derivado sale de la misma fila (columna calculada): el riesgo es nulo.
* Los reportes necesitan otro diseño: mejor en un almacén analítico aparte.

**No desnormalices cuando:**

* No mediste. "Los joins son lentos" no es una medición.
* No tienes cómo mantener la copia sincronizada, o cómo verificarla.
* La tabla tiene muchas escrituras y la copia las encarecería más de lo que mejora las lecturas.

**Herramientas que ayudan:**

* El plan de ejecución y `SET STATISTICS IO, TIME ON` en SQL Server, para medir antes y después.
* Query Store, para encontrar las consultas que de verdad consumen más recursos.

-----

## Resumen en 5 líneas

1. Desnormalizar es redundancia a propósito: primero normaliza, después mide, después prueba índices, y recién ahí desnormaliza.
2. Una snapshot histórica (el precio pagado) no es redundancia: es otro hecho, y no debe cambiar con el original.
3. Prefiere las técnicas que mantiene el motor: columnas calculadas y vistas indexadas.
4. Toda copia mantenida a mano necesita un mecanismo de sincronización y una consulta que la verifique.
5. Para reportes, mejor un almacén analítico desnormalizado (esquema estrella) que desnormalizar el OLTP.

-----

## Para profundizar

<details>
<summary>Dimensiones lentamente cambiantes</summary>

En un almacén analítico, ¿qué pasa cuando Ana se muda de Lima a Cusco? Si se actualiza `DimCliente`, las ventas históricas de Lima pasan a contarse como de Cusco. Las **dimensiones lentamente cambiantes** (SCD) resuelven esto: el tipo 1 sobrescribe (se pierde el historial), el tipo 2 agrega una fila nueva con fechas de vigencia (las ventas viejas siguen apuntando a la versión de Lima) y el tipo 3 guarda el valor anterior en otra columna. El tipo 2 es el más usado, y es la misma idea de la snapshot aplicada a una dimensión completa.

</details>

<details>
<summary>CQRS: separar el modelo de escritura del de lectura</summary>

El patrón CQRS (*Command Query Responsibility Segregation*) lleva la desnormalización al nivel de arquitectura: las escrituras van a un modelo normalizado y las lecturas, a uno o varios modelos desnormalizados (otras tablas, otra base, un índice de búsqueda) que se actualizan a partir de eventos. Es útil cuando las necesidades de lectura y de escritura son muy distintas, a costa de aceptar consistencia eventual entre ambos modelos.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Desnormalizar es repetir datos a propósito para que algunas consultas sean más rápidas, por ejemplo guardando el total del pedido en vez de calcularlo cada vez. Se hace después de normalizar y solo si hace falta, porque hay que mantener las copias sincronizadas.

### Respuesta ampliada (semi-senior)

Parto de un diseño normalizado como fuente de verdad, mido la consulta problemática y pruebo primero índices y reescrituras. Si no alcanza, elijo la técnica de desnormalización más segura disponible: una columna calculada persistida si el valor sale de la misma fila, una vista indexada si es un agregado (en SQL Server se mantiene sincrónicamente, a costa de encarecer las escrituras) y, solo como último recurso, columnas mantenidas por *triggers* o por la aplicación, siempre con una consulta de verificación. Distingo la redundancia de las snapshots históricas, como el precio pagado o la dirección de una factura, que son hechos propios y no deben cambiar. Para reportes, prefiero un almacén analítico en estrella alimentado por ETL antes que llenar el OLTP de agregados, y mido también el impacto en las escrituras, no solo la lectura que mejora.

### Preguntas frecuentes de seguimiento

**1. ¿Cuándo desnormalizarías?**
Cuando una consulta frecuente es lenta, la medición lo confirma y los índices no alcanzan, y tengo un mecanismo para mantener la copia sincronizada.

**2. ¿Guardar el precio en el detalle del pedido es desnormalizar?**
No, si es el precio pagado: es un hecho distinto del precio actual, y no debe cambiar si cambia el catálogo.

**3. ¿Qué es una vista indexada?**
Una vista de SQL Server con un índice agrupado único: su resultado se guarda físicamente y el motor lo actualiza en cada transacción que modifica las tablas base.

**4. ¿Qué diferencia hay con una vista materializada de PostgreSQL?**
La de PostgreSQL se actualiza solo con `REFRESH`, así que puede estar desactualizada; la indexada de SQL Server siempre está al día, pero encarece las escrituras.

-----

## Práctica

**Ejercicio 1.** Para cada caso, di si es desnormalización, snapshot histórica o ninguna de las dos, y cómo lo implementarías:

1. `Factura.DireccionFiscal`, copiada del cliente al emitir la factura.
2. `Producto.CantidadVendida`, la suma de las cantidades de todos sus detalles.
3. `DetallePedido.Subtotal`, igual a `Cantidad * PrecioUnitario`.
4. `Pedido.CiudadCliente`, copiada de `Cliente.Ciudad` para filtrar pedidos por ciudad sin join.

<details>
<summary>Solución</summary>

1. **Snapshot histórica.** La factura emitida debe conservar la dirección de ese momento. Se copia al emitir y no se actualiza nunca.
2. **Desnormalización** (agregado de varias filas). Vista indexada con `SUM(Cantidad)` agrupada por `ProductoId`, o un proceso periódico si puede estar unos minutos desactualizada.
3. **Desnormalización sin riesgo.** Columna calculada persistida: el motor la mantiene.
4. **Desnormalización riesgosa**, y probablemente innecesaria: un índice sobre `Pedido.ClienteId` y un join con `Cliente` filtrado por ciudad suele bastar. Si se mantiene, necesita un *trigger* sobre `Cliente`. Ojo: si lo que se quiere es "la ciudad a la que se envió el pedido", entonces es una snapshot (la dirección de envío), y no debe actualizarse.

</details>

**Ejercicio 2.** Escribe una vista indexada con la cantidad total vendida de cada producto, y la consulta que la usa para listar los productos más vendidos.

<details>
<summary>Solución</summary>

```sql
CREATE VIEW dbo.vVentasPorProducto
WITH SCHEMABINDING
AS
SELECT d.ProductoId,
       SUM(d.Cantidad) AS UnidadesVendidas,
       COUNT_BIG(*)    AS Lineas
FROM dbo.DetallePedido AS d
GROUP BY d.ProductoId;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vVentasPorProducto ON dbo.vVentasPorProducto (ProductoId);
GO

SELECT p.Nombre, v.UnidadesVendidas
FROM dbo.vVentasPorProducto AS v WITH (NOEXPAND)
JOIN dbo.Producto AS p ON p.ProductoId = v.ProductoId
ORDER BY v.UnidadesVendidas DESC, p.Nombre;
```

| Nombre | UnidadesVendidas |
| --- | --- |
| Clean Code | 3 |
| Mouse inalámbrico | 3 |
| Cafetera | 1 |
| Lámpara de escritorio | 1 |
| Monitor 27 pulgadas | 1 |
| Teclado mecánico | 1 |

`Cantidad` es `NOT NULL`, requisito para el `SUM` de una vista indexada. Las ventas del pedido cancelado (la cafetera) también cuentan: una vista indexada no admite filtrar por una columna de otra tabla (`Pedido.Estado`) sin incluir el join, y ese es el tipo de decisión que hay que tomar antes de materializar.

</details>

-----

## Siguiente lección

[Patrones de modelado](09-Patrones%20de%20modelado.md)
