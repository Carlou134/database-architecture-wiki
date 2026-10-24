# Claves foráneas e integridad referencial

## En una frase

Una **clave foránea** es una columna que apunta a la clave de otra tabla, y la **integridad referencial** es la garantía, impuesta por el motor, de que ese apuntador nunca quede huérfano: no se puede crear un pedido para un cliente inexistente ni borrar un cliente que todavía tiene pedidos, salvo que digas explícitamente qué hacer.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una clave primaria y una clave alternativa, de [Claves primarias](04-Claves%20primarias.md).
* Que "relación" en el modelo formal significa tabla, de [El modelo relacional](02-El%20modelo%20relacional.md); aquí hablamos del **vínculo** entre tablas.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Clave foránea (FK):** columna o columnas cuyos valores deben existir en la clave de otra tabla.
* **Tabla padre (referenciada):** la que tiene la clave a la que se apunta (`Cliente`).
* **Tabla hija (referenciante):** la que tiene la clave foránea (`Pedido`).
* **Fila huérfana:** fila hija cuya clave foránea apunta a un padre que no existe.
* **Acción referencial:** lo que hace el motor con las hijas cuando se borra o se actualiza el padre (`NO ACTION`, `CASCADE`, `SET NULL`, `SET DEFAULT`).
* **Autorreferencia:** clave foránea que apunta a la misma tabla.

-----

## El problema

Sin claves foráneas, `Pedido.ClienteId` es solo un número. La base acepta esto sin quejarse:

```sql
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (999, 'Pendiente');   -- no existe el cliente 999
DELETE FROM dbo.Cliente WHERE ClienteId = 1;                              -- Ana tenía dos pedidos
```

Y empiezan los síntomas:

* Un reporte de "ventas por cliente" pierde filas, porque el `JOIN` con `Cliente` no encuentra al 999.
* Un pedido de Ana queda sin dueño: nadie sabe a quién facturarle.
* Alguien "arregla" el problema en la aplicación, con un `if (cliente != null)` en cada pantalla. Y el script de importación nocturno, que no pasa por la aplicación, vuelve a romperlo.

"La aplicación ya valida" no alcanza: a la base le escriben la aplicación, los scripts, los procesos ETL, otro microservicio y la persona que se conecta con SSMS un viernes a las seis de la tarde. La clave foránea es la única validación que **ninguno** de ellos puede saltarse.

-----

## Cómo funciona

### Declarar una clave foránea

```sql
CREATE TABLE dbo.Pedido (
    PedidoId  INT IDENTITY(1,1) NOT NULL,
    ClienteId INT               NOT NULL,
    Estado    VARCHAR(20)       NOT NULL,
    CONSTRAINT PK_Pedido PRIMARY KEY (PedidoId),
    CONSTRAINT FK_Pedido_Cliente FOREIGN KEY (ClienteId)
        REFERENCES dbo.Cliente (ClienteId)
);
```

Reglas:

* La columna referenciada debe ser una **clave primaria** o tener una restricción **`UNIQUE`**.
* Los tipos deben coincidir (`INT` con `INT`).
* La clave foránea va en la tabla del lado **"muchos"**: un cliente tiene muchos pedidos, así que `ClienteId` va en `Pedido` (se profundiza en [Cardinalidad y relaciones](../02-diseno-y-modelado/03-Cardinalidad%20y%20relaciones.md)).

### Qué controla el motor

Con la clave foránea, las dos operaciones del problema fallan:

```sql
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (999, 'Pendiente');
```

```text
Msg 547, Level 16, State 0
The INSERT statement conflicted with the FOREIGN KEY constraint "FK_Pedido_Cliente".
The conflict occurred in database "Tienda", table "dbo.Cliente", column 'ClienteId'.
```

```sql
DELETE FROM dbo.Cliente WHERE ClienteId = 1;
```

```text
Msg 547, Level 16, State 0
The DELETE statement conflicted with the REFERENCE constraint "FK_Pedido_Cliente".
The conflict occurred in database "Tienda", table "dbo.Pedido", column 'ClienteId'.
```

El motor verifica en los dos extremos:

| Operación | Sobre | Lo que verifica |
| --- | --- | --- |
| `INSERT` / `UPDATE` de la FK | Tabla hija | Que el valor exista en el padre |
| `DELETE` / `UPDATE` de la clave | Tabla padre | Qué pasa con las hijas que lo referencian (acción referencial) |

### Acciones referenciales: `ON DELETE` y `ON UPDATE`

Cuando se borra o se actualiza un padre con hijas, decides qué pasa:

| Acción | Qué hace | Cuándo usarla |
| --- | --- | --- |
| `NO ACTION` (por defecto) | Rechaza la operación si hay hijas | Entidades independientes: no se borra un cliente con historial de pedidos |
| `CASCADE` | Borra (o actualiza) también las hijas | Composición: las hijas no tienen sentido sin el padre (los detalles de un pedido) |
| `SET NULL` | Pone la FK de las hijas en `NULL` | Relación opcional: si se borra un vendedor, sus clientes quedan "sin vendedor asignado" |
| `SET DEFAULT` | Pone la FK en su valor `DEFAULT` | Poco común: reasignar a un valor genérico ("Sin categoría") |

En el esquema de ejemplo:

```sql
-- Un detalle no existe sin su pedido → CASCADE
CONSTRAINT FK_DetallePedido_Pedido FOREIGN KEY (PedidoId)
    REFERENCES dbo.Pedido (PedidoId) ON DELETE CASCADE,

-- Un producto vendido no se borra: el historial lo necesita → NO ACTION
CONSTRAINT FK_DetallePedido_Producto FOREIGN KEY (ProductoId)
    REFERENCES dbo.Producto (ProductoId)
```

`CASCADE` es potente y peligroso: un `DELETE` de una fila puede borrar miles en varias tablas, sin ningún aviso. Úsalo solo cuando la hija es **parte** del padre, no cuando solo está **asociada** a él.

`ON UPDATE CASCADE` casi nunca hace falta si usas claves sustitutas, porque una clave sustituta no cambia.

### Claves foráneas opcionales

Si la columna de la FK admite `NULL`, la relación es **opcional**: la fila hija puede no tener padre.

```sql
CREATE TABLE dbo.Cliente (
    ClienteId  INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Nombre     NVARCHAR(100)     NOT NULL,
    VendedorId INT               NULL,      -- un cliente puede no tener vendedor asignado
    CONSTRAINT FK_Cliente_Vendedor FOREIGN KEY (VendedorId)
        REFERENCES dbo.Vendedor (VendedorId) ON DELETE SET NULL
);
```

El motor no verifica los `NULL`: un `VendedorId` nulo no apunta a nada, así que no hay nada que comprobar. `NOT NULL` en la FK es lo que hace **obligatoria** la relación.

### Autorreferencia

Una tabla puede apuntarse a sí misma. El caso clásico: cada empleado tiene un jefe, que también es un empleado.

```sql
CREATE TABLE dbo.Empleado (
    EmpleadoId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Nombre     NVARCHAR(100)     NOT NULL,
    JefeId     INT               NULL,      -- NULL: el gerente general no tiene jefe
    CONSTRAINT FK_Empleado_Jefe FOREIGN KEY (JefeId)
        REFERENCES dbo.Empleado (EmpleadoId)
);
```

Sirve para jerarquías: organigramas, categorías con subcategorías, comentarios con respuestas. Se recorren con CTE recursivas, en el módulo de consultas.

### Las claves foráneas no crean índices

SQL Server **no** crea un índice para la columna de la clave foránea. Hay que crearlo a mano, por dos razones:

1. **Los joins:** "los pedidos del cliente 1" busca por `Pedido.ClienteId`. Sin índice, recorre toda la tabla.
2. **Las verificaciones del padre:** al borrar un cliente, el motor tiene que buscar si hay pedidos con ese `ClienteId`. Sin índice, cada `DELETE` en `Cliente` recorre todo `Pedido`, y además bloquea más filas de las necesarias.

```sql
CREATE INDEX IX_Pedido_ClienteId ON dbo.Pedido (ClienteId);
```

Excepción: si la FK es la primera columna de otro índice (por ejemplo, de la clave primaria compuesta de `DetallePedido`), ese índice ya sirve.

### Claves foráneas "no confiables"

Se puede crear o reactivar una FK sin verificar los datos existentes:

```sql
ALTER TABLE dbo.Pedido WITH NOCHECK ADD CONSTRAINT FK_Pedido_Cliente
    FOREIGN KEY (ClienteId) REFERENCES dbo.Cliente (ClienteId);
```

Las filas nuevas se validan, pero las viejas pueden tener huérfanas. SQL Server marca la restricción como **no confiable** (`is_not_trusted = 1`), y el optimizador deja de usarla para simplificar consultas. Lo mismo pasa cuando se desactiva para una carga masiva y se reactiva sin verificar. La forma correcta de reactivarla:

```sql
ALTER TABLE dbo.Pedido WITH CHECK CHECK CONSTRAINT FK_Pedido_Cliente;
--                     ↑ verifica los datos   ↑ reactiva la restricción
```

-----

## Ejemplo completo

Las claves foráneas del esquema de ejemplo, probadas con los datos. Todo va dentro de una transacción que termina en `ROLLBACK`, para no modificar los datos que usan las demás lecciones:

```sql
BEGIN TRANSACTION;

-- 1. Insertar un pedido para un cliente que existe: funciona
INSERT INTO dbo.Pedido (ClienteId, Estado) VALUES (5, 'Pendiente');   -- Sofía: PedidoId 6

-- 2. Insertar un detalle con un producto que no existe: falla (FK_DetallePedido_Producto)
INSERT INTO dbo.DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
VALUES (6, 99, 1, 10.00);

-- 3. Borrar un producto que aparece en pedidos: falla (NO ACTION protege el historial)
DELETE FROM dbo.Producto WHERE ProductoId = 5;   -- la cafetera está en el pedido 4

-- 4. Borrar un pedido: CASCADE se lleva sus detalles
SELECT COUNT(*) AS DetallesAntes FROM dbo.DetallePedido WHERE PedidoId = 3;
DELETE FROM dbo.Pedido WHERE PedidoId = 3;
SELECT COUNT(*) AS DetallesDespues FROM dbo.DetallePedido WHERE PedidoId = 3;

ROLLBACK;
```

| DetallesAntes |
| --- |
| 2 |

| DetallesDespues |
| --- |
| 0 |

Qué decisión hay detrás de cada resultado:

* **2 falla:** un detalle no puede apuntar a un producto inexistente.
* **3 falla:** la cafetera está inactiva (`Activo = 0`), pero no se puede borrar, porque el pedido 4 la necesita para saber qué se vendió. Para "retirar" un producto se desactiva, no se borra.
* **4 borra en cascada:** los detalles son parte del pedido. Sin su pedido no significan nada.

-----

## Errores comunes

**1. No declarar claves foráneas "porque la aplicación ya valida".**
Qué pasa: huérfanas que aparecen por scripts, cargas masivas, otros servicios o errores de concurrencia.
Por qué: la validación en la aplicación tiene una carrera entre "verificar que el cliente existe" e "insertar el pedido"; otro proceso puede borrar al cliente en medio.
Arreglo: FK en la base. La validación en la aplicación sirve para dar un buen mensaje de error; la de la base, para garantizar.

**2. `CASCADE` en todas las claves foráneas.**
Qué pasa: borrar un cliente borra sus pedidos, sus facturas, sus pagos... y el historial contable desaparece.
Por qué: se usa `CASCADE` para "que no moleste el error".
Arreglo: `CASCADE` solo para composición; para entidades con historial, `NO ACTION` y borrado lógico (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)).

**3. Varios caminos de cascada.**
Qué pasa: `Msg 1785: Introducing FOREIGN KEY constraint ... may cause cycles or multiple cascade paths`.
Por qué: SQL Server no permite que una fila pueda alcanzarse por dos caminos de `CASCADE` distintos (por ejemplo, `Pedido` → `Pago` y `Pedido` → `Factura` → `Pago`).
Arreglo: `NO ACTION` en uno de los caminos, y el borrado de ese camino se hace explícitamente.

**4. Olvidar el índice de la clave foránea.**
Qué pasa: joins lentos y `DELETE` en la tabla padre que tardan y bloquean.
Por qué: SQL Server no crea el índice automáticamente.
Arreglo: un índice por cada FK que no sea ya la primera columna de otro índice.

**5. Desactivar las FK para una carga y reactivarlas sin verificar.**
Qué pasa: restricciones marcadas como no confiables, con huérfanas escondidas.
Por qué: `ALTER TABLE ... CHECK CONSTRAINT` sin `WITH CHECK` no revisa los datos existentes.
Arreglo: `WITH CHECK CHECK CONSTRAINT`, y revisa `sys.foreign_keys` donde `is_not_trusted = 1`.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Acciones | `NO ACTION`, `CASCADE`, `SET NULL`, `SET DEFAULT` | Las mismas + `RESTRICT` | Las mismas + `RESTRICT` (`SET DEFAULT` no se admite) |
| `RESTRICT` frente a `NO ACTION` | Solo existe `NO ACTION` | `NO ACTION` puede diferirse al final de la transacción; `RESTRICT` falla de inmediato | Equivalentes |
| Restricciones diferibles | No | Sí: `DEFERRABLE INITIALLY DEFERRED` (se validan al hacer `COMMIT`) | No |
| ¿Crea índice en la FK? | No | No | **Sí**, automáticamente si no existe uno que sirva |
| Caminos de cascada múltiples | Prohibidos | Permitidos | Permitidos (hasta 15 niveles de cascada) |
| Ver restricciones no verificadas | `sys.foreign_keys.is_not_trusted` | `NOT VALID` + `VALIDATE CONSTRAINT` | `SET FOREIGN_KEY_CHECKS = 0` desactiva todo sin marcar nada |

Detalles importantes:

* **MySQL:** las claves foráneas solo funcionan con InnoDB (el motor por defecto). Con el antiguo MyISAM, la sintaxis se acepta y se ignora. Además, las acciones en cascada **no disparan *triggers***.
* **PostgreSQL:** las restricciones diferibles permiten insertar padre e hija en cualquier orden dentro de una transacción, o intercambiar referencias circulares.

### En MongoDB

MongoDB **no tiene claves foráneas** ni integridad referencial. Un documento puede guardar el `_id` de otro (una referencia), pero el motor no verifica que exista ni hace nada al borrarlo. Por eso en MongoDB se prefiere **embeber** los datos que pertenecen al documento (los detalles dentro del pedido), y cuando se referencia, la integridad es responsabilidad de la aplicación. `$lookup` permite unir colecciones en una consulta, pero no impone ninguna regla. Se estudia en el módulo de NoSQL.

-----

## Cuándo sí y cuándo no

**Declara la clave foránea siempre que:**

* Una columna apunte a otra tabla dentro de la misma base de datos. Es la norma, no la excepción.

**Ten en cuenta:**

* Las FK tienen un costo: cada `INSERT` en la hija busca en el padre. Es pequeño con índices y casi siempre vale la pena.
* Entre bases distintas o entre microservicios no se pueden declarar FK. Ahí la integridad se maneja con eventos, procesos de conciliación o aceptando consistencia eventual.
* En tablas de *staging* para cargas masivas a veces se omiten las FK, y se validan al pasar los datos a las tablas definitivas.

**Herramientas que ayudan:**

* `sp_fkeys 'Cliente'` lista las claves foráneas que apuntan a una tabla.
* Los diagramas de SSMS y DBeaver dibujan las relaciones a partir de las FK declaradas: si no las declaras, el diagrama no las muestra.

-----

## Resumen en 5 líneas

1. Una clave foránea obliga a que el valor exista en la clave primaria o `UNIQUE` del padre: no hay huérfanas.
2. Va en la tabla del lado "muchos" y con el mismo tipo que la clave que referencia.
3. `NO ACTION` (por defecto) protege el historial; `CASCADE` solo para composición; `SET NULL` para relaciones opcionales.
4. FK `NULL` = relación opcional; FK `NOT NULL` = relación obligatoria.
5. SQL Server no indexa las FK: crea el índice a mano, y reactiva las restricciones con `WITH CHECK`.

-----

## Para profundizar

<details>
<summary>Cómo usa el optimizador las claves foráneas confiables</summary>

Si una FK es confiable y `NOT NULL`, el optimizador sabe que cada pedido tiene exactamente un cliente. Con eso puede **eliminar joins innecesarios**: en una consulta que hace `JOIN` con `Cliente` pero no usa ninguna de sus columnas, el join no cambia el resultado y se omite. Con una FK no confiable, el optimizador no puede asumirlo y hace el join igual. Por eso una FK reactivada sin `WITH CHECK` también empeora el rendimiento, no solo la integridad.

</details>

<details>
<summary>Claves foráneas compuestas</summary>

Si el padre tiene una clave compuesta, la FK debe incluir todas sus columnas, en el mismo orden:

```sql
CONSTRAINT FK_Devolucion_DetallePedido FOREIGN KEY (PedidoId, ProductoId)
    REFERENCES dbo.DetallePedido (PedidoId, ProductoId)
```

Si alguna de las columnas de la FK es `NULL`, el motor no verifica la fila entera (el comportamiento del estándar llamado `MATCH SIMPLE`). PostgreSQL permite exigir que todas o ninguna sean nulas con `MATCH FULL`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una clave foránea es una columna que apunta a la clave primaria de otra tabla, como `ClienteId` en `Pedido`. Hace que la base rechace pedidos de clientes que no existen, y que no se pueda borrar un cliente con pedidos, a menos que se configure `ON DELETE CASCADE` o `SET NULL`.

### Respuesta ampliada (semi-senior)

La clave foránea garantiza la integridad referencial en la base, que es el único lugar donde ningún cliente (aplicación, script, ETL) puede saltársela, y además evita las carreras que tiene validar desde la aplicación. Va en el lado "muchos", referencia una PK o `UNIQUE` y su nulabilidad define si la relación es opcional u obligatoria. Elijo la acción referencial por semántica: `NO ACTION` para entidades con historial, combinado con borrado lógico; `CASCADE` solo para composición, como los detalles de un pedido; y `SET NULL` para relaciones opcionales. En SQL Server tengo en cuenta que no se indexan solas (el índice es necesario para los joins y para que el borrado del padre no recorra la tabla hija), que no se permiten varios caminos de cascada y que una FK reactivada sin `WITH CHECK` queda no confiable, lo que también impide que el optimizador elimine joins.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no validar solo en la aplicación?**
Porque a la base le escriben más clientes que la aplicación, y porque la validación en la aplicación tiene una carrera: entre comprobar y escribir, otro proceso puede borrar el padre.

**2. ¿Cuándo usarías `ON DELETE CASCADE`?**
Cuando la hija es parte del padre y no tiene sentido sola: líneas de un pedido, respuestas de un formulario. Nunca para entidades con historial, como clientes o productos.

**3. ¿SQL Server crea un índice para la clave foránea?**
No. Hay que crearlo a mano; MySQL InnoDB sí lo crea.

**4. ¿Una clave foránea puede ser nula?**
Sí, y eso hace la relación opcional. El motor no verifica los valores nulos.

**5. ¿Qué es una FK no confiable?**
Una que se creó o reactivó sin verificar los datos existentes (`WITH NOCHECK`). Puede esconder huérfanas, y el optimizador no la usa para simplificar planes.

-----

## Práctica

**Ejercicio 1.** Para cada clave foránea, elige la acción `ON DELETE` y justifica:

1. `Pedido.ClienteId` → `Cliente`.
2. `Producto.CategoriaId` → `Categoria`.
3. `ComentarioProducto.ProductoId` → `Producto` (reseñas de los clientes).
4. `Cliente.VendedorId` → `Vendedor` (opcional).
5. `ImagenProducto.ProductoId` → `Producto` (fotos del producto).

<details>
<summary>Solución</summary>

1. **`NO ACTION`:** el pedido es historial contable. Si el cliente pide la baja, se desactiva o se anonimiza, no se borra.
2. **`NO ACTION`:** borrar una categoría con productos debería obligar a reasignarlos primero. (`SET DEFAULT` hacia una categoría "Sin categoría" es una alternativa, si el negocio lo quiere.)
3. **Depende de la regla:** si las reseñas desaparecen junto con el producto, `CASCADE`. Pero como los productos con historial no se borran (se desactivan), en la práctica casi nunca se dispara.
4. **`SET NULL`:** si se va el vendedor, el cliente queda sin vendedor asignado hasta que se le asigne otro.
5. **`CASCADE`:** las imágenes son parte del producto; sin él, no tienen sentido.

</details>

**Ejercicio 2.** Escribe la tabla `Categoria` con subcategorías (una categoría puede tener una categoría padre, y las de primer nivel no tienen padre), e inserta "Electrónica" y su subcategoría "Periféricos".

<details>
<summary>Solución</summary>

```sql
CREATE TABLE dbo.CategoriaJerarquica (
    CategoriaId      INT IDENTITY(1,1) NOT NULL,
    Nombre           NVARCHAR(50)      NOT NULL,
    CategoriaPadreId INT               NULL,
    CONSTRAINT PK_CategoriaJerarquica PRIMARY KEY (CategoriaId),
    CONSTRAINT FK_CategoriaJerarquica_Padre FOREIGN KEY (CategoriaPadreId)
        REFERENCES dbo.CategoriaJerarquica (CategoriaId)
);

CREATE INDEX IX_CategoriaJerarquica_CategoriaPadreId
    ON dbo.CategoriaJerarquica (CategoriaPadreId);

INSERT INTO dbo.CategoriaJerarquica (Nombre, CategoriaPadreId) VALUES (N'Electrónica', NULL);   -- Id 1
INSERT INTO dbo.CategoriaJerarquica (Nombre, CategoriaPadreId) VALUES (N'Periféricos', 1);
```

| CategoriaId | Nombre | CategoriaPadreId |
| --- | --- | --- |
| 1 | Electrónica | `NULL` |
| 2 | Periféricos | 1 |

`CategoriaPadreId` es `NULL` porque las categorías de primer nivel no tienen padre. No lleva `ON DELETE CASCADE`: SQL Server no permite cascada en una autorreferencia (se consideraría un ciclo), y borrar una categoría con subcategorías debería ser una decisión explícita.

</details>

-----

## Siguiente lección

[Restricciones](06-Restricciones.md)
