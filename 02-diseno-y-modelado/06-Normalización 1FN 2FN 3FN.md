# Normalización: 1FN, 2FN y 3FN

## En una frase

Normalizar es dividir las tablas para que **cada dato se guarde en un solo lugar**: la primera forma normal exige valores atómicos, la segunda elimina las dependencias de una parte de la clave y la tercera elimina las dependencias entre columnas que no son clave; o, en la frase clásica, cada columna depende de "la clave, toda la clave y nada más que la clave".

-----

## Antes de empezar

Conviene que ya sepas:

* Dependencias funcionales, parciales y transitivas, y las anomalías, de [Dependencias funcionales](05-Dependencias%20funcionales.md).
* Claves candidatas, de [El modelo relacional](../01-fundamentos/02-El%20modelo%20relacional.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Forma normal (FN):** un nivel de calidad de diseño definido por las dependencias que la tabla no puede tener.
* **1FN:** cada celda tiene un solo valor y no hay grupos repetidos de columnas.
* **2FN:** 1FN + ningún atributo no clave depende de una parte de una clave candidata.
* **3FN:** 2FN + ningún atributo no clave depende de otro atributo no clave.
* **Atributo primo:** atributo que forma parte de alguna clave candidata.
* **Descomposición sin pérdida:** división de una tabla que permite reconstruir la original exacta con un join.

-----

## El problema

La planilla de ventas llegó así, una fila por pedido:

**PlanillaVentas**

| Pedido | Fecha | Cliente | Email | Ciudad | Productos |
| --- | --- | --- | --- | --- | --- |
| 1 | 2026-03-01 | Ana Torres | ana.torres@correo.com | Lima | Teclado mecánico (Electrónica) x1 a 120.00; Mouse inalámbrico (Electrónica) x2 a 45.50 |
| 2 | 2026-03-05 | Luis Ramos | luis.ramos@correo.com | Arequipa | Monitor 27 pulgadas (Electrónica) x1 a 849.90 |
| 3 | 2026-04-12 | Ana Torres | ana.torres@correo.com | Lima | Clean Code (Libros) x2 a 95.00; Lámpara de escritorio (Hogar) x1 a 75.00 |

Preguntas que esta tabla no puede responder sin dolor:

* ¿Cuántos mouses se vendieron? Hay que extraer texto de la columna `Productos`.
* ¿Qué pedidos incluyen libros? `LIKE '%Libros%'`, y rezar para que nadie escriba "libros".
* Ana cambia de email. ¿En cuántas filas?

La normalización es el procedimiento, paso a paso, que lleva esta planilla al esquema de la tienda, y cada paso tiene una justificación precisa.

-----

## Cómo funciona

### Primera forma normal (1FN): valores atómicos

Una tabla está en 1FN cuando:

1. Cada celda contiene **un solo valor** (no listas ni estructuras).
2. No hay **grupos repetidos** de columnas (`Producto1`, `Producto2`...).
3. Tiene una **clave** que identifica cada fila.

`PlanillaVentas` viola la regla 1: `Productos` contiene una lista, y cada elemento de la lista contiene a su vez cuatro datos (nombre, categoría, cantidad, precio). Se arregla con **una fila por cada producto de cada pedido** y una columna por cada dato:

**Ventas1FN** — clave: `{Pedido, Producto}`

| Pedido | Fecha | Cliente | Email | Ciudad | Producto | Categoria | Cantidad | Precio |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-03-01 | Ana Torres | ana.torres@correo.com | Lima | Teclado mecánico | Electrónica | 1 | 120.00 |
| 1 | 2026-03-01 | Ana Torres | ana.torres@correo.com | Lima | Mouse inalámbrico | Electrónica | 2 | 45.50 |
| 2 | 2026-03-05 | Luis Ramos | luis.ramos@correo.com | Arequipa | Monitor 27 pulgadas | Electrónica | 1 | 849.90 |
| 3 | 2026-04-12 | Ana Torres | ana.torres@correo.com | Lima | Clean Code | Libros | 2 | 95.00 |
| 3 | 2026-04-12 | Ana Torres | ana.torres@correo.com | Lima | Lámpara de escritorio | Hogar | 1 | 75.00 |

Ahora "¿cuántos mouses se vendieron?" es un `SUM(Cantidad) WHERE Producto = ...`. Pero la redundancia **aumentó**: los datos de Ana aparecen cuatro veces. La 1FN resuelve la atomicidad, no la redundancia.

Sobre "atómico": la atomicidad depende del uso. Una fecha es atómica aunque tenga día, mes y año, porque el motor la trata como un valor. Una dirección en una sola columna es atómica si nunca se consulta por partes. Lo que nunca es atómico es una **lista** de cosas del mismo tipo.

### Segunda forma normal (2FN): toda la clave

Una tabla está en 2FN cuando está en 1FN y **ningún atributo no clave depende de una parte de una clave candidata**. Solo puede violarse si la clave es **compuesta**.

Dependencias de `Ventas1FN` (de [Dependencias funcionales](05-Dependencias%20funcionales.md)):

```text
Pedido             → Fecha, Email         ← parcial: solo una parte de la clave
Email              → Cliente, Ciudad
Producto           → Categoria            ← parcial: solo una parte de la clave
{Pedido, Producto} → Cantidad, Precio     ← completa
```

`Fecha` depende solo de `Pedido`; `Categoria`, solo de `Producto`. Por eso se repiten: la fecha del pedido 1 aparece una vez por cada producto del pedido. Se arregla **sacando cada dependencia parcial a su propia tabla**, con su determinante como clave:

**Pedido2FN** — clave: `Pedido`

| Pedido | Fecha | Email | Cliente | Ciudad |
| --- | --- | --- | --- | --- |
| 1 | 2026-03-01 | ana.torres@correo.com | Ana Torres | Lima |
| 2 | 2026-03-05 | luis.ramos@correo.com | Luis Ramos | Arequipa |
| 3 | 2026-04-12 | ana.torres@correo.com | Ana Torres | Lima |

**Producto2FN** — clave: `Producto`

| Producto | Categoria |
| --- | --- |
| Teclado mecánico | Electrónica |
| Mouse inalámbrico | Electrónica |
| Monitor 27 pulgadas | Electrónica |
| Clean Code | Libros |
| Lámpara de escritorio | Hogar |

**Detalle2FN** — clave: `{Pedido, Producto}`

| Pedido | Producto | Cantidad | Precio |
| --- | --- | --- | --- |
| 1 | Teclado mecánico | 1 | 120.00 |
| 1 | Mouse inalámbrico | 2 | 45.50 |
| 2 | Monitor 27 pulgadas | 1 | 849.90 |
| 3 | Clean Code | 2 | 95.00 |
| 3 | Lámpara de escritorio | 1 | 75.00 |

Los datos del cliente fueron con el pedido porque dependen (a través del email) de `Pedido`, que es su determinante. `Precio` se quedó en el detalle porque es el precio **pagado**, que depende de la combinación pedido-producto.

Todavía queda redundancia: Ana Torres sigue apareciendo dos veces en `Pedido2FN`.

### Tercera forma normal (3FN): nada más que la clave

Una tabla está en 3FN cuando está en 2FN y **ningún atributo no clave depende de otro atributo no clave** (no hay dependencias transitivas).

En `Pedido2FN`:

```text
Pedido → Email → Cliente, Ciudad      ← transitiva: Email no es clave de esta tabla
```

`Cliente` y `Ciudad` dependen del pedido solo **a través** del email. Se arregla sacando la dependencia transitiva a su propia tabla:

**Cliente3FN** — clave: `Email`

| Email | Cliente | Ciudad |
| --- | --- | --- |
| ana.torres@correo.com | Ana Torres | Lima |
| luis.ramos@correo.com | Luis Ramos | Arequipa |

**Pedido3FN** — clave: `Pedido`

| Pedido | Fecha | Email |
| --- | --- | --- |
| 1 | 2026-03-01 | ana.torres@correo.com |
| 2 | 2026-03-05 | luis.ramos@correo.com |
| 3 | 2026-04-12 | ana.torres@correo.com |

`Producto2FN` y `Detalle2FN` ya están en 3FN: no tienen dependencias entre columnas no clave.

Resultado: cuatro tablas, y cada hecho se guarda una sola vez. Ana cambia de email o de ciudad: **una** fila.

### La frase para recordarlo

> Cada atributo no clave debe depender de **la clave** (1FN: hay una clave), de **toda la clave** (2FN: no de una parte) y de **nada más que la clave** (3FN: no de otro atributo no clave).

| Forma normal | Prohíbe | Pregunta para verificarla |
| --- | --- | --- |
| 1FN | Listas en celdas, grupos repetidos, filas sin clave | ¿Alguna celda tiene varios valores del mismo tipo? |
| 2FN | Dependencias parciales | ¿Alguna columna depende solo de una parte de la clave compuesta? |
| 3FN | Dependencias transitivas | ¿Alguna columna depende de otra columna que no es clave? |

### ¿Y la categoría?

En `Producto2FN`, `Categoria` es un texto. La tabla está en 3FN: `Producto → Categoria` es una dependencia de la clave, no transitiva. Entonces, ¿por qué el esquema de la tienda tiene una tabla `Categoria` aparte?

Por razones que la 3FN no exige, pero que el negocio sí:

* **Consistencia de los valores:** con texto libre, "Electrónica", "Electronica" y "electrónica" son tres categorías distintas. Con una tabla y una FK, solo existen las categorías registradas.
* **Anomalía de inserción:** sin la tabla, no se puede crear una categoría nueva hasta que tenga un producto.
* **Atributos futuros:** si la categoría gana una descripción o un orden de visualización, ya tiene dónde vivir.

Es un caso frecuente: la normalización da un mínimo de calidad, y las reglas de negocio pueden pedir más. Lo importante es saber cuál de las dos razones justifica cada tabla.

### Descomposición sin pérdida

Cada división tiene que poder **deshacerse**: un `JOIN` de las tablas resultantes debe reconstruir exactamente la tabla original, sin filas de más ni de menos. Eso se garantiza si la columna común es la **clave** de al menos una de las dos partes (por ejemplo, `Pedido3FN` y `Cliente3FN` se unen por `Email`, que es clave de `Cliente3FN`). Al normalizar sacando cada dependencia con su determinante como clave, la descomposición siempre es sin pérdida.

-----

## Ejemplo completo

Del resultado en 3FN al [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md), con las decisiones físicas:

| 3FN | Esquema final | Decisión |
| --- | --- | --- |
| `Cliente3FN(Email, Cliente, Ciudad)` | `Cliente(ClienteId PK, Nombre, Email UQ, Ciudad, FechaRegistro)` | Clave sustituta; el email queda como `UNIQUE` porque puede cambiar |
| `Pedido3FN(Pedido, Fecha, Email)` | `Pedido(PedidoId PK, ClienteId FK, FechaPedido, Estado)` | La FK apunta a la sustituta, no al email |
| `Producto2FN(Producto, Categoria)` | `Producto(ProductoId PK, CategoriaId FK, Nombre, Precio, ...)` + `Categoria(CategoriaId PK, Nombre UQ)` | Categoría como tabla, por consistencia de valores |
| `Detalle2FN(Pedido, Producto, Cantidad, Precio)` | `DetallePedido(PedidoId, ProductoId, Cantidad, PrecioUnitario)` | Claves sustitutas en las FK |

Y la prueba de que no se perdió nada: el `JOIN` de las tablas normalizadas reconstruye la planilla en 1FN.

```sql
SELECT p.PedidoId, p.FechaPedido, c.Nombre AS Cliente, c.Email, c.Ciudad,
       pr.Nombre AS Producto, cat.Nombre AS Categoria, d.Cantidad, d.PrecioUnitario
FROM dbo.Pedido AS p
JOIN dbo.Cliente AS c ON c.ClienteId = p.ClienteId
JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
JOIN dbo.Producto AS pr ON pr.ProductoId = d.ProductoId
JOIN dbo.Categoria AS cat ON cat.CategoriaId = pr.CategoriaId
WHERE p.PedidoId IN (1, 2, 3)
ORDER BY p.PedidoId, pr.Nombre;
```

| PedidoId | FechaPedido | Cliente | Email | Ciudad | Producto | Categoria | Cantidad | PrecioUnitario |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-03-01 10:15 | Ana Torres | ana.torres@correo.com | Lima | Mouse inalámbrico | Electrónica | 2 | 45.50 |
| 1 | 2026-03-01 10:15 | Ana Torres | ana.torres@correo.com | Lima | Teclado mecánico | Electrónica | 1 | 120.00 |
| 2 | 2026-03-05 16:40 | Luis Ramos | luis.ramos@correo.com | Arequipa | Monitor 27 pulgadas | Electrónica | 1 | 849.90 |
| 3 | 2026-04-12 09:05 | Ana Torres | ana.torres@correo.com | Lima | Clean Code | Libros | 2 | 95.00 |
| 3 | 2026-04-12 09:05 | Ana Torres | ana.torres@correo.com | Lima | Lámpara de escritorio | Hogar | 1 | 75.00 |

Qué cambió y por qué:

* **1FN:** la lista de productos se convirtió en filas; ahora se puede sumar, filtrar e indexar.
* **2FN:** la fecha y el cliente se fueron con el pedido, la categoría con el producto; dejaron de repetirse por cada línea.
* **3FN:** los datos del cliente se fueron a su tabla; dejaron de repetirse por cada pedido.
* **Más allá de la 3FN:** la categoría pasó a una tabla por consistencia, y las claves naturales se reemplazaron por sustitutas por estabilidad.

La redundancia de la planilla no desapareció del todo: ahora vive en el **resultado** de una consulta, que es donde tiene que estar.

-----

## Errores comunes

**1. Creer que la 1FN resuelve la redundancia.**
Qué pasa: la tabla queda "normalizada" con una fila por producto, pero los datos del cliente se repiten en cada línea.
Por qué: la 1FN solo resuelve la atomicidad; la redundancia la resuelven la 2FN y la 3FN.
Arreglo: sigue hasta la 3FN.

**2. Normalizar sin las dependencias.**
Qué pasa: se separan tablas "porque parece", y se separa de más (una tabla por columna) o de menos.
Por qué: falta el criterio objetivo.
Arreglo: escribe las dependencias primero; cada determinante problemático es una tabla.

**3. Sacar el precio pagado del detalle "porque ya está en el producto".**
Qué pasa: los pedidos viejos cambian de importe cuando cambia el catálogo.
Por qué: se confunde el precio actual (`ProductoId → Precio`) con el precio pagado (`{PedidoId, ProductoId} → PrecioUnitario`).
Arreglo: son dos hechos distintos; cada uno va con su determinante.

**4. Confundir normalizar con "usar IDs para todo".**
Qué pasa: una tabla con `ClienteId`, pero que igual repite `ClienteNombre` y `ClienteCiudad` en cada pedido.
Por qué: se agregó una clave sustituta sin sacar la dependencia transitiva.
Arreglo: la clave sustituta no normaliza nada; lo que normaliza es mover los atributos a la tabla de su determinante.

**5. Normalizar los datos de reporte o históricos que deben congelarse.**
Qué pasa: una factura emitida muestra la dirección actual del cliente, no la de la fecha de emisión.
Por qué: se aplicó 3FN a un dato que en realidad es una **copia histórica** con su propio determinante (la factura).
Arreglo: la dirección de la factura depende de la factura, no del cliente; es otro hecho, y se guarda en la factura.

-----

## Según el motor

Las formas normales son **independientes del motor**: un esquema en 3FN es el mismo en SQL Server, PostgreSQL y MySQL. Lo que varía es cuánto facilita cada motor romper la 1FN a propósito:

| Forma de romper la 1FN | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Lista separada por comas | `STRING_SPLIT` (desde 2016) para separarla en consultas | `string_to_array`, `regexp_split_to_table` | `FIND_IN_SET` |
| Arreglos nativos | No | Sí (`int[]`, `text[]`) con operadores e índices GIN | No |
| JSON | `OPENJSON`, `JSON_VALUE`; tipo `json` en 2025 | `jsonb` con índices GIN | `JSON` con índices sobre columnas generadas |

Que existan estas funciones no cambia la regla: si los elementos de la lista se consultan, se validan o se relacionan con otras tablas, van en filas. Los arreglos y el JSON se justifican para datos que se leen y se escriben siempre completos, como una lista de etiquetas de solo lectura.

### En MongoDB

MongoDB **no** sigue la normalización, y lo hace a propósito: el documento de un pedido guarda sus detalles como un arreglo (rompe la 1FN) y muchas veces copia el nombre del cliente (rompe la 3FN) para que la consulta "mostrar el pedido" lea un solo documento. El análisis de esta lección sigue siendo útil allí, pero para la pregunta inversa: **qué** se está duplicando y quién se encarga de actualizar las copias cuando el original cambia.

-----

## Cuándo sí y cuándo no

**Normaliza hasta la 3FN cuando:**

* Diseñas una base transaccional (OLTP): es el punto de partida por defecto.
* Los datos cambian con frecuencia: cada dato en un solo lugar significa una sola actualización.

**Ten en cuenta:**

* Normalizar aumenta la cantidad de joins. En la inmensa mayoría de los sistemas, con índices adecuados, no es un problema.
* Para reportes y análisis (OLAP), los esquemas se desnormalizan a propósito. Es una decisión posterior y consciente (ver [Desnormalización](08-Desnormalización.md)).
* La 3FN casi siempre coincide con la forma normal de Boyce-Codd. Los casos en que no coinciden se ven en la [próxima lección](07-BCNF%20y%20formas%20superiores.md).

-----

## Resumen en 5 líneas

1. 1FN: un valor por celda, sin grupos repetidos y con clave. Resuelve la atomicidad, no la redundancia.
2. 2FN: nada depende de una parte de la clave; solo se viola con claves compuestas.
3. 3FN: nada depende de una columna que no es clave; elimina las dependencias transitivas.
4. "La clave, toda la clave y nada más que la clave": cada dependencia problemática se va a su propia tabla, con su determinante como clave.
5. La normalización da un mínimo; las reglas de negocio (consistencia de catálogos, copias históricas) pueden pedir más tablas o datos copiados a propósito.

-----

## Para profundizar

<details>
<summary>La definición formal de 2FN y 3FN usa todas las claves candidatas</summary>

Las definiciones de los libros hablan de **atributos primos** (los que forman parte de alguna clave candidata), no solo de la clave primaria:

* **2FN:** ningún atributo **no primo** depende de un subconjunto propio de **alguna** clave candidata.
* **3FN:** para toda dependencia no trivial `X → A`, o `X` es superclave, o `A` es primo.

Esa última cláusula ("o `A` es primo") es la puerta por la que una tabla puede estar en 3FN y todavía tener redundancia. Cerrarla es exactamente lo que hace la forma normal de Boyce-Codd.

</details>

<details>
<summary>Por qué "normalización"</summary>

Codd introdujo el término en 1970-1971, al definir la 1FN, la 2FN y la 3FN. Según contó él mismo, eligió la palabra en parte como un guiño a la "normalización" de relaciones diplomáticas con China que anunciaba en esos años el presidente Nixon: si Nixon podía normalizar relaciones, él también. Más allá de la anécdota, la idea era llevar cada relación a una forma "normal" o estándar, sin redundancias.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Normalizar es organizar las tablas para no repetir datos. En la 1FN cada celda tiene un solo valor. En la 2FN, las columnas dependen de toda la clave y no de una parte. En la 3FN, las columnas no dependen de otras columnas que no son clave. Así, cada dato se guarda en un solo lugar y se evitan las inconsistencias.

### Respuesta ampliada (semi-senior)

La normalización elimina la redundancia que causa anomalías de actualización, inserción y borrado, y se apoya en las dependencias funcionales. La 1FN exige valores atómicos, sin listas ni grupos repetidos, y una clave; resuelve la atomicidad, pero no la redundancia. La 2FN elimina las dependencias parciales de una clave compuesta, y la 3FN las transitivas entre atributos no clave: "la clave, toda la clave y nada más que la clave". Cada dependencia problemática se saca a una tabla con su determinante como clave, lo que garantiza una descomposición sin pérdida. En OLTP normalizo hasta 3FN o BCNF por defecto y desnormalizo solo con una necesidad medida. También distingo lo que la normalización no decide: un catálogo puede ir a su propia tabla por consistencia aunque la 3FN no lo exija, y un dato histórico, como el precio pagado o la dirección de una factura emitida, es un hecho distinto del dato actual, con otro determinante.

### Preguntas frecuentes de seguimiento

**1. ¿Una tabla con clave simple puede violar la 2FN?**
No. La 2FN trata dependencias de una parte de la clave, y una clave de una sola columna no tiene partes.

**2. ¿Qué es una dependencia transitiva y por qué la elimina la 3FN?**
Es `X → Y → Z` con `Y` no clave. Hace que `Z` se repita cada vez que se repite `Y`, y por eso se mueve a una tabla con `Y` como clave.

**3. ¿Normalizar siempre es mejor?**
Para OLTP, es el punto de partida correcto. Para reportes y análisis, o por un problema de rendimiento medido, se desnormaliza de forma consciente.

**4. ¿Guardar el precio en el detalle del pedido viola la 3FN?**
No, si el precio es el **pagado**: depende de la combinación pedido-producto, no del producto. Sería redundante solo si siempre fuera igual al precio de catálogo.

-----

## Práctica

**Ejercicio 1.** Normaliza hasta la 3FN esta tabla de un centro de idiomas, indicando qué viola cada forma normal:

| Alumno | DNI | Telefonos | Curso | Nivel | Profesor | EmailProfesor | Nota |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Rosa Paz | 41234567 | 999111222, 988333444 | Inglés | B1 | Mark Lee | mark@idiomas.com | 17 |
| Rosa Paz | 41234567 | 999111222, 988333444 | Francés | A2 | Anne Roy | anne@idiomas.com | 15 |
| Iván Soto | 42345678 | 977555666 | Inglés | B1 | Mark Lee | mark@idiomas.com | 14 |

Reglas: cada curso tiene un nivel y un profesor; cada profesor tiene un email.

<details>
<summary>Solución</summary>

**1FN:** `Telefonos` tiene listas. Se separa en `AlumnoTelefono(DNI, Telefono)`. La tabla principal queda con clave `{DNI, Curso}`.

**2FN:** con clave `{DNI, Curso}`, hay dependencias parciales:

* `DNI → Alumno` → tabla `Alumno(DNI, Nombre)`.
* `Curso → Nivel, Profesor` → tabla `Curso(Curso, Nivel, Profesor, EmailProfesor)`.
* Queda `Inscripcion(DNI, Curso, Nota)`.

**3FN:** en `Curso`, `Profesor → EmailProfesor` es transitiva. Se separa en `Profesor(Profesor, EmailProfesor)`, y `Curso(Curso, Nivel, Profesor)`.

Resultado:

```text
Alumno         (DNI PK, Nombre)
AlumnoTelefono (DNI PK FK, Telefono PK)
Profesor       (ProfesorId PK, Nombre, Email UQ)
Curso          (CursoId PK, Nombre, Nivel, ProfesorId FK)
Inscripcion    (DNI PK FK, CursoId PK FK, Nota)
```

En el diseño físico, `Alumno` llevaría una clave sustituta con el DNI como `UNIQUE`, y lo mismo `Profesor` con el email.

</details>

**Ejercicio 2.** ¿En qué forma normal está esta tabla, y por qué? `Producto(ProductoId PK, Nombre, CategoriaId, CategoriaNombre, Precio)`.

<details>
<summary>Solución</summary>

Está en **2FN**, pero no en 3FN.

* **1FN:** sí, todos los valores son atómicos y hay clave.
* **2FN:** sí, automáticamente: la clave es simple, así que no puede haber dependencias parciales.
* **3FN:** no. `ProductoId → CategoriaId → CategoriaNombre` es una dependencia transitiva: el nombre de la categoría se repite en cada producto de esa categoría y, si cambia, hay que actualizarlo en todos.

Arreglo: `CategoriaNombre` se va a `Categoria(CategoriaId PK, Nombre)`, y `Producto` conserva solo `CategoriaId` como FK. Es exactamente el esquema de la tienda.

</details>

-----

## Siguiente lección

[BCNF y formas superiores](07-BCNF%20y%20formas%20superiores.md)
