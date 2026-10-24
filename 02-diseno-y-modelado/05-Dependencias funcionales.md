# Dependencias funcionales

## En una frase

Una **dependencia funcional** `X → Y` dice que conocer el valor de `X` determina un único valor de `Y` (conocer el email de un cliente determina su nombre); es la herramienta que permite detectar, con un criterio objetivo, por qué una tabla tiene datos redundantes y cómo dividirla.

-----

## Antes de empezar

Conviene que ya sepas:

* Superclaves y claves candidatas, de [El modelo relacional](../01-fundamentos/02-El%20modelo%20relacional.md).
* Cómo se pasa de un modelo ER a tablas, de [Del modelo ER a tablas](04-Del%20modelo%20ER%20a%20tablas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Dependencia funcional (DF):** `X → Y`: para cada valor de `X` hay un único valor de `Y`.
* **Determinante:** el lado izquierdo de la dependencia (`X`).
* **Dependiente:** el lado derecho (`Y`).
* **Dependencia parcial:** `Y` depende de una **parte** de una clave compuesta, no de la clave entera.
* **Dependencia transitiva:** `X → Y` y `Y → Z`, donde `Y` no es clave: `Z` depende de `X` a través de `Y`.
* **Anomalía:** problema al insertar, actualizar o borrar, causado por la redundancia.
* **Cierre de un conjunto de atributos (X⁺):** todos los atributos que se pueden determinar a partir de `X`.

-----

## El problema

El área de ventas tenía todo en una sola planilla, y alguien la importó a una tabla:

**VentaPlana**

| PedidoId | FechaPedido | ClienteEmail | ClienteNombre | ClienteCiudad | ProductoId | ProductoNombre | Categoria | Cantidad | PrecioUnitario |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-03-01 | ana.torres@correo.com | Ana Torres | Lima | 1 | Teclado mecánico | Electrónica | 1 | 120.00 |
| 1 | 2026-03-01 | ana.torres@correo.com | Ana Torres | Lima | 2 | Mouse inalámbrico | Electrónica | 2 | 45.50 |
| 2 | 2026-03-05 | luis.ramos@correo.com | Luis Ramos | Arequipa | 3 | Monitor 27 pulgadas | Electrónica | 1 | 849.90 |
| 3 | 2026-04-12 | ana.torres@correo.com | Ana Torres | Lima | 6 | Clean Code | Libros | 2 | 95.00 |
| 3 | 2026-04-12 | ana.torres@correo.com | Ana Torres | Lima | 4 | Lámpara de escritorio | Hogar | 1 | 75.00 |

Se intuye que algo está mal: "Ana Torres, Lima" se repite cuatro veces. Pero la intuición no alcanza para decidir **qué** separar y **cómo**. Y la redundancia tiene consecuencias concretas, llamadas **anomalías**:

* **De actualización:** Ana se muda a Cusco. Hay que cambiar cuatro filas; si se cambian tres, la base dice que Ana vive en dos ciudades a la vez.
* **De inserción:** se registra una clienta nueva, Sofía, que todavía no compró nada. No se puede: cada fila necesita un pedido y un producto.
* **De borrado:** se borra el pedido 2, el único de Luis. Con él desaparece todo lo que se sabía de Luis, incluido su email.

Las dependencias funcionales convierten esa intuición en un análisis preciso.

-----

## Cómo funciona

### Qué es una dependencia funcional

`X → Y` se lee "X determina a Y" o "Y depende funcionalmente de X". Significa: **dos filas con el mismo valor de `X` tienen necesariamente el mismo valor de `Y`**.

En `VentaPlana`:

* `ClienteEmail → ClienteNombre`: el mismo email siempre corresponde al mismo nombre.
* `ProductoId → ProductoNombre`: el mismo producto siempre se llama igual.
* `ClienteNombre → ClienteEmail` **no** se cumple en general: puede haber dos "Ana Torres" con emails distintos.

La dependencia puede tener varios atributos de cada lado: `{PedidoId, ProductoId} → {Cantidad, PrecioUnitario}`.

### Las dependencias salen de las reglas, no de los datos

Este es el punto más importante de la lección. Mirando los datos de arriba, `ClienteCiudad → ClienteNombre` **parece** cumplirse: cada ciudad tiene un solo nombre (Lima → Ana Torres, Arequipa → Luis Ramos). Pero es una casualidad de cinco filas: mañana entra Jorge, también de Lima.

Los datos solo pueden **refutar** una dependencia (si encuentras dos filas con el mismo `X` y distinto `Y`, no se cumple). Nunca pueden **probarla**. Las dependencias reales salen de las **reglas de negocio**:

| Regla de negocio | Dependencia funcional |
| --- | --- |
| Cada email pertenece a un solo cliente | `ClienteEmail → ClienteNombre, ClienteCiudad` |
| Cada pedido es de un solo cliente y tiene una fecha | `PedidoId → FechaPedido, ClienteEmail` |
| Cada producto tiene un nombre y una categoría | `ProductoId → ProductoNombre, Categoria` |
| La cantidad y el precio pagado son de cada producto en cada pedido | `{PedidoId, ProductoId} → Cantidad, PrecioUnitario` |

Fíjate en `PrecioUnitario`. Depende de `{PedidoId, ProductoId}`, **no** de `ProductoId` solo, porque el negocio dice que el pedido conserva el precio pagado (el monitor se vendió a 849.90 aunque hoy cuesta 899.90). Si la regla fuera "el precio es siempre el del catálogo", la dependencia sería `ProductoId → PrecioUnitario`, y el diseño correcto sería otro. **Las mismas columnas, con otra regla de negocio, producen otro diseño.**

### Dependencias triviales, parciales y transitivas

| Tipo | Definición | Ejemplo en `VentaPlana` |
| --- | --- | --- |
| **Trivial** | `Y` es parte de `X` | `{PedidoId, ProductoId} → PedidoId` (no aporta nada) |
| **Completa** | `Y` depende de toda la clave, y de ninguna parte de ella | `{PedidoId, ProductoId} → Cantidad` |
| **Parcial** | `Y` depende de una parte de una clave compuesta | `PedidoId → FechaPedido` (no hace falta `ProductoId`) |
| **Transitiva** | `X → Y → Z`, con `Y` que no es clave | `PedidoId → ClienteEmail → ClienteNombre` |

Las dependencias parciales y las transitivas son las que causan las anomalías. Eliminarlas es exactamente lo que hacen la segunda y la tercera forma normal ([Normalización 1FN 2FN 3FN](06-Normalización%201FN%202FN%203FN.md)).

### Encontrar la clave con el cierre

El **cierre** de un conjunto de atributos `X`, escrito `X⁺`, es todo lo que se puede determinar a partir de `X` aplicando las dependencias una y otra vez. Si `X⁺` contiene **todos** los atributos de la tabla, `X` es una superclave; si además es mínima, es clave candidata.

Algoritmo:

1. Empieza con `X⁺ = X`.
2. Para cada dependencia `A → B`: si todo `A` está en `X⁺`, agrega `B` a `X⁺`.
3. Repite hasta que no se agregue nada nuevo.

Cierre de `{PedidoId, ProductoId}` en `VentaPlana`:

```text
Inicio:                  {PedidoId, ProductoId}
PedidoId → FechaPedido, ClienteEmail
                       → {PedidoId, ProductoId, FechaPedido, ClienteEmail}
ClienteEmail → ClienteNombre, ClienteCiudad
                       → {..., ClienteNombre, ClienteCiudad}
ProductoId → ProductoNombre, Categoria
                       → {..., ProductoNombre, Categoria}
{PedidoId, ProductoId} → Cantidad, PrecioUnitario
                       → {..., Cantidad, PrecioUnitario}

Resultado: los 10 atributos → {PedidoId, ProductoId} es superclave.
```

¿Es mínima? `{PedidoId}⁺` = {PedidoId, FechaPedido, ClienteEmail, ClienteNombre, ClienteCiudad}: le faltan los atributos del producto. `{ProductoId}⁺` = {ProductoId, ProductoNombre, Categoria}: le faltan los del pedido. Ninguna parte alcanza sola, así que **`{PedidoId, ProductoId}` es clave candidata**.

### Reglas de inferencia (axiomas de Armstrong)

Las dependencias permiten deducir otras. Las tres reglas básicas:

| Regla | Enunciado | Ejemplo |
| --- | --- | --- |
| **Reflexividad** | Si `Y` es parte de `X`, entonces `X → Y` | `{PedidoId, ProductoId} → PedidoId` |
| **Aumento** | Si `X → Y`, entonces `XZ → YZ` | `PedidoId → FechaPedido` ⇒ `{PedidoId, ProductoId} → {FechaPedido, ProductoId}` |
| **Transitividad** | Si `X → Y` e `Y → Z`, entonces `X → Z` | `PedidoId → ClienteEmail` y `ClienteEmail → ClienteNombre` ⇒ `PedidoId → ClienteNombre` |

El algoritmo del cierre es, en el fondo, la aplicación repetida de estas reglas.

-----

## Ejemplo completo

Análisis completo de `VentaPlana`:

**1. Dependencias** (a partir de las reglas de negocio):

```text
DF1: PedidoId               → FechaPedido, ClienteEmail
DF2: ClienteEmail           → ClienteNombre, ClienteCiudad
DF3: ProductoId             → ProductoNombre, Categoria
DF4: {PedidoId, ProductoId} → Cantidad, PrecioUnitario
```

**2. Clave candidata:** `{PedidoId, ProductoId}` (por el cierre calculado arriba).

**3. Clasificación de cada dependencia respecto de la clave:**

| Dependencia | Tipo | Problema que causa |
| --- | --- | --- |
| DF4 | Completa | Ninguno: es el hecho que la tabla debería guardar |
| DF1 | Parcial (solo `PedidoId`) | La fecha y el cliente se repiten por cada producto del pedido |
| DF3 | Parcial (solo `ProductoId`) | El nombre y la categoría se repiten por cada pedido del producto |
| DF2 | Transitiva (`PedidoId → ClienteEmail → ClienteNombre`) | Los datos del cliente se repiten por cada pedido del cliente |

**4. Cada dependencia problemática sugiere una tabla**, con su determinante como clave:

```text
DF4 → DetallePedido (PedidoId, ProductoId, Cantidad, PrecioUnitario)
DF1 → Pedido        (PedidoId, FechaPedido, ClienteEmail)
DF3 → Producto      (ProductoId, ProductoNombre, Categoria)
DF2 → Cliente       (ClienteEmail, ClienteNombre, ClienteCiudad)
```

Comprobación con las anomalías del problema:

* **Ana se muda:** se actualiza **una** fila en `Cliente`.
* **Sofía sin pedidos:** se inserta en `Cliente`, sin necesidad de un pedido.
* **Se borra el pedido 2:** Luis sigue en `Cliente`.

Las cuatro tablas son, salvo las claves sustitutas y la categoría, el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md). Hacer este paso con las formas normales, una por una, es el tema de la [próxima lección](06-Normalización%201FN%202FN%203FN.md).

-----

## Errores comunes

**1. Deducir las dependencias de los datos.**
Qué pasa: se concluye que `Ciudad → Nombre` porque en la muestra cada ciudad tiene un cliente, y se diseña sobre una regla falsa.
Por qué: los datos solo pueden refutar una dependencia, nunca probarla.
Arreglo: las dependencias se obtienen preguntando al negocio, y los datos sirven para buscar contraejemplos.

**2. Confundir "depende" con "está relacionado".**
Qué pasa: se escribe `ClienteNombre → PedidoId` porque "el cliente tiene pedidos".
Por qué: se lee la dependencia como un vínculo, no como una función.
Arreglo: pregúntate "si conozco X, ¿hay **un único** Y posible?". Un cliente tiene muchos pedidos, así que su nombre no determina un pedido.

**3. Olvidar que la regla de negocio cambia la dependencia.**
Qué pasa: se asume `ProductoId → PrecioUnitario`, se saca el precio del detalle y los pedidos viejos cambian de importe cuando cambia el catálogo.
Por qué: no se preguntó si el precio pagado puede diferir del precio actual.
Arreglo: cada dependencia se valida con una pregunta explícita al negocio.

**4. Buscar la clave a ojo en tablas grandes.**
Qué pasa: se elige `{PedidoId}` como clave de `VentaPlana` y se pierden filas al cargar los datos.
Por qué: no se verificó que su cierre alcanzara todos los atributos.
Arreglo: calcula el cierre; si no contiene todos los atributos, no es clave.

-----

## Según el motor

Las dependencias funcionales son un concepto **teórico**: ningún motor las declara como tales. Se hacen cumplir indirectamente:

| Cómo se hace cumplir `X → Y` | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Normalizando: `X` es la clave de la tabla donde vive `Y` | PK o `UNIQUE` sobre `X` | Igual | Igual |
| Sin normalizar, detectando violaciones | Consulta con `GROUP BY X HAVING COUNT(DISTINCT Y) > 1` | Igual | Igual |

La consulta de detección es útil para auditar datos heredados:

```sql
-- ¿Hay emails con más de un nombre? (violaciones de ClienteEmail → ClienteNombre)
SELECT ClienteEmail, COUNT(DISTINCT ClienteNombre) AS NombresDistintos
FROM dbo.VentaPlana
GROUP BY ClienteEmail
HAVING COUNT(DISTINCT ClienteNombre) > 1;
```

Una curiosidad útil: MySQL usa dependencias funcionales en su modo `ONLY_FULL_GROUP_BY`. Permite seleccionar una columna que no está en el `GROUP BY` si depende funcionalmente de las columnas agrupadas (por ejemplo, agrupar por la clave primaria y mostrar el nombre). PostgreSQL hace lo mismo cuando se agrupa por la clave primaria. SQL Server no: exige que toda columna no agregada esté en el `GROUP BY`.

### En MongoDB

Las dependencias funcionales existen en los datos, pero MongoDB no las usa para diseñar: la redundancia es a menudo **intencional** (copiar el nombre del cliente en cada pedido para no tener que unir colecciones). El análisis sigue siendo útil para saber **qué** datos se están duplicando y, por lo tanto, qué hay que actualizar en varios documentos cuando cambian.

-----

## Cuándo sí y cuándo no

**Haz el análisis de dependencias cuando:**

* Tienes una tabla ancha (una planilla importada, una tabla heredada) y necesitas decidir cómo dividirla.
* Dudas si un diseño tiene redundancia: el análisis lo responde de forma objetiva.
* Te preparas para normalizar: las formas normales se definen en términos de dependencias.

**Ten en cuenta:**

* En un diseño que salió de un buen modelo ER, la mayoría de las dependencias ya están resueltas. El análisis formal brilla con datos heredados o modelos dudosos.
* No hace falta escribir el cierre de cada tabla en el trabajo diario; sí hace falta saber hacerlo cuando una clave no es evidente.

-----

## Resumen en 5 líneas

1. `X → Y`: dos filas con el mismo `X` tienen el mismo `Y`; el determinante "fija" al dependiente.
2. Las dependencias salen de las reglas de negocio; los datos solo sirven para refutarlas.
3. Las dependencias parciales (de una parte de la clave) y transitivas (a través de un no clave) causan las anomalías.
4. El cierre `X⁺` dice qué determina `X`; si alcanza todos los atributos y es mínimo, `X` es clave candidata.
5. Cada dependencia problemática sugiere una tabla, con su determinante como clave.

-----

## Para profundizar

<details>
<summary>Recubrimiento mínimo</summary>

Un conjunto de dependencias puede tener redundancias: si ya tienes `A → B` y `B → C`, escribir también `A → C` no agrega información (se deduce por transitividad). El **recubrimiento mínimo** es un conjunto equivalente sin redundancias: cada dependencia tiene un solo atributo a la derecha, ningún atributo sobra a la izquierda y ninguna dependencia se deduce de las demás. Es la entrada del algoritmo de síntesis de 3FN, que construye directamente un esquema en tercera forma normal: una tabla por cada determinante del recubrimiento mínimo.

</details>

<details>
<summary>Cómo detectar dependencias en datos heredados a gran escala</summary>

Con tablas de cientos de columnas, se puede automatizar la búsqueda de candidatas: para cada par de columnas `(A, B)`, comparar `COUNT(DISTINCT A)` con `COUNT(DISTINCT A, B)` (en SQL Server, `COUNT(DISTINCT CONCAT(A, '|', B))`). Si coinciden, `A → B` se cumple en los datos actuales. Herramientas de perfilado de datos hacen esto de forma sistemática. El resultado son **candidatas** que hay que validar con el negocio, nunca dependencias confirmadas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una dependencia funcional `X → Y` significa que si conozco `X`, conozco `Y`: por ejemplo, el email de un cliente determina su nombre. Sirven para detectar datos repetidos en una tabla y saber cómo dividirla al normalizar.

### Respuesta ampliada (semi-senior)

Una dependencia `X → Y` garantiza que dos filas con el mismo `X` tienen el mismo `Y`. Las obtengo de las reglas de negocio, no de los datos, que solo pueden refutarlas: la misma tabla con una regla distinta (precio pagado frente a precio de catálogo) produce otro diseño. A partir de las dependencias calculo el cierre para encontrar las claves candidatas, y clasifico cada dependencia como completa, parcial o transitiva. Las parciales y las transitivas son las que causan anomalías de actualización, inserción y borrado, y eliminarlas es justamente lo que piden la 2FN y la 3FN: cada determinante problemático se convierte en una tabla con él como clave. En datos heredados, verifico las dependencias candidatas con consultas `GROUP BY ... HAVING COUNT(DISTINCT ...) > 1`.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es una dependencia parcial?**
Una en la que un atributo depende solo de una parte de una clave compuesta, como la fecha del pedido en una tabla con clave `(PedidoId, ProductoId)`.

**2. ¿Qué es una dependencia transitiva?**
`X → Y → Z`, donde `Y` no es clave: `Z` depende de la clave a través de otro atributo, como el nombre del cliente a través de su email en una tabla de pedidos.

**3. ¿Qué anomalías causa la redundancia?**
De actualización (cambiar un dato en varias filas y dejar inconsistencias), de inserción (no poder registrar algo sin otro dato no relacionado) y de borrado (perder información al borrar otra).

**4. ¿Se pueden deducir las dependencias mirando los datos?**
No. Los datos solo pueden demostrar que una dependencia no se cumple. Las dependencias reales vienen de las reglas de negocio.

-----

## Práctica

**Ejercicio 1.** Dada esta tabla y sus reglas, escribe las dependencias funcionales, encuentra la clave candidata y clasifica cada dependencia:

`Inscripcion(AlumnoId, AlumnoNombre, CursoId, CursoNombre, ProfesorId, ProfesorNombre, Nota)`

Reglas: cada curso tiene un solo profesor; un profesor puede dictar varios cursos; un alumno tiene una nota por curso.

<details>
<summary>Solución</summary>

Dependencias:

```text
DF1: AlumnoId           → AlumnoNombre
DF2: CursoId            → CursoNombre, ProfesorId
DF3: ProfesorId         → ProfesorNombre
DF4: {AlumnoId, CursoId} → Nota
```

Clave candidata: `{AlumnoId, CursoId}`. Su cierre: DF1 agrega `AlumnoNombre`, DF2 agrega `CursoNombre` y `ProfesorId`, DF3 agrega `ProfesorNombre` y DF4 agrega `Nota`. Están los 7 atributos, y ninguna de las dos columnas alcanza sola.

| Dependencia | Tipo |
| --- | --- |
| DF4 | Completa |
| DF1 | Parcial (solo `AlumnoId`) |
| DF2 | Parcial (solo `CursoId`) |
| DF3 | Transitiva: `CursoId → ProfesorId → ProfesorNombre` |

Tablas sugeridas: `Alumno(AlumnoId, AlumnoNombre)`, `Curso(CursoId, CursoNombre, ProfesorId)`, `Profesor(ProfesorId, ProfesorNombre)`, `Inscripcion(AlumnoId, CursoId, Nota)`.

</details>

**Ejercicio 2.** Escribe una consulta que verifique, sobre `VentaPlana`, si se cumple `ProductoId → ProductoNombre` en los datos cargados.

<details>
<summary>Solución</summary>

```sql
SELECT ProductoId, COUNT(DISTINCT ProductoNombre) AS NombresDistintos
FROM dbo.VentaPlana
GROUP BY ProductoId
HAVING COUNT(DISTINCT ProductoNombre) > 1;
```

Si devuelve cero filas, los datos actuales no refutan la dependencia (lo que no la demuestra). Si devuelve alguna fila, hay productos con más de un nombre: o la regla no es la que creías, o hay errores de carga ("Teclado mecánico" y "Teclado mecanico") que limpiar antes de normalizar. Ojo: `COUNT(DISTINCT ...)` ignora los `NULL`, así que un producto con un nombre y un `NULL` no aparece; si la columna admite nulos, revísalos aparte.

</details>

-----

## Siguiente lección

[Normalización 1FN 2FN 3FN](06-Normalización%201FN%202FN%203FN.md)
