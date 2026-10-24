# SQL con IA

## En una frase

Un asistente de IA (ChatGPT, Claude, Gemini, Copilot) escribe, explica y depura SQL muy rápido, pero solo es tan bueno como el **contexto** que le das (motor, esquema, reglas de negocio) y como tu capacidad de **verificar** lo que devuelve: la IA ejecuta, tú diriges, y todo lo que viste en este módulo es lo que te permite detectar cuándo se equivoca.

-----

## Antes de empezar

Conviene que ya sepas:

* Todo el módulo de consultas: esta lección no enseña SQL nuevo, enseña a **revisar** el SQL que genera otro.
* El esquema de la tienda, del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **LLM (modelo de lenguaje grande):** el tipo de modelo detrás de los asistentes de IA: genera texto plausible a partir de patrones aprendidos.
* **Prompt:** la instrucción o pregunta que se le da al modelo.
* **Contexto:** la información que el modelo necesita para responder bien: motor, esquema, reglas, formato esperado.
* **Alucinación:** una respuesta falsa dicha con total seguridad (una función que no existe, una columna inventada).
* **Sesgo de confirmación:** la tendencia a aceptar un resultado porque "se ve bien", sin verificarlo.

-----

## El problema

En tus apuntes, el ejemplo de Codecademy le pide a la IA "los 5 modelos con mejor rendimiento", y la IA responde:

```sql
SELECT * FROM models
ORDER BY best_mpge DESC
LIMIT 5;
```

Para SQLite, perfecto. Para SQL Server, error de sintaxis: `LIMIT` no existe en T-SQL. La IA no se equivocó en SQL: se equivocó **de dialecto**, porque nadie le dijo cuál era.

Ese es el caso fácil, porque da error. El caso peligroso es la consulta que **funciona** y devuelve un resultado incorrecto. Pídele "el total gastado por cada cliente" a un asistente sin contexto, y es muy probable que recibas algo como esto:

```sql
SELECT c.Nombre, SUM(d.Cantidad * d.PrecioUnitario) AS TotalGastado
FROM Cliente c
JOIN Pedido p ON c.ClienteId = p.ClienteId
JOIN DetallePedido d ON p.PedidoId = d.PedidoId
GROUP BY c.Nombre;
```

Compila, corre y devuelve cuatro filas con números razonables. Tiene **tres** problemas que no dan error. ¿Los ves? Si no, no es culpa de la IA: es la señal de que todavía no dominas lo que le pediste.

-----

## Cómo funciona

### Qué hace bien la IA y qué no

| La IA es buena para | La IA es mala para |
| --- | --- |
| Escribir rápido una consulta cuya lógica ya conoces | Saber las reglas de **tu** negocio |
| Recordar sintaxis (`STRING_AGG`, `OFFSET FETCH`, `MERGE`) | Saber qué motor y qué versión usas, si no se lo dices |
| Explicar una consulta ajena línea por línea | Saber qué columnas admiten `NULL` en tu base |
| Traducir entre dialectos (SQLite → T-SQL) | Saber si un join multiplica filas en **tus** datos |
| Proponer ideas de análisis | Saber si el resultado es correcto |
| Encontrar el error de sintaxis de un mensaje de error | Verificarse a sí misma |

La columna de la derecha tiene algo en común: todo depende de información que **solo tú tienes**, o de una verificación que **solo tú puedes hacer**.

### El contexto: qué darle

Un buen prompt para SQL incluye:

1. **El motor y la versión:** "SQL Server 2022, T-SQL". Es la línea que más errores evita.
2. **El esquema:** las tablas, columnas, tipos y claves relevantes. Lo mejor es pegar el `CREATE TABLE`, que incluye los `NOT NULL`, las claves foráneas y los `CHECK`.
3. **Las reglas de negocio que afectan la consulta:** "los pedidos cancelados no cuentan como venta", "incluir a los clientes sin pedidos".
4. **El resultado esperado:** las columnas y su significado, idealmente con la forma de la tabla.
5. **Un ejemplo, si lo tienes:** algunas filas de datos de prueba (inventadas o anonimizadas) y el resultado que esperas.

Comparado con el prompt sin contexto:

```text
❌ "Escribí una consulta con el total gastado por cada cliente."
```

```text
✅ Motor: SQL Server 2022 (T-SQL).

Esquema (resumido):
- dbo.Cliente (ClienteId INT PK, Nombre NVARCHAR(100) NOT NULL, Email NVARCHAR(254) NOT NULL UNIQUE)
- dbo.Pedido (PedidoId INT PK, ClienteId INT NOT NULL FK → Cliente, Estado VARCHAR(20) NOT NULL)
  Estados posibles: Pendiente, Pagado, Enviado, Entregado, Cancelado.
- dbo.DetallePedido (PedidoId INT FK, ProductoId INT FK, Cantidad INT NOT NULL,
  PrecioUnitario DECIMAL(10,2) NOT NULL), PK (PedidoId, ProductoId).

Reglas:
- Los pedidos cancelados no cuentan.
- Deben aparecer TODOS los clientes, incluidos los que no compraron (con 0).
- Puede haber dos clientes con el mismo nombre.

Resultado esperado: | Nombre | Email | Pedidos | TotalGastado |
ordenado por TotalGastado descendente.
```

Fíjate en que el segundo prompt **ya contiene** la solución de los tres problemas de la consulta del principio. No por casualidad: para escribir ese prompt hace falta saber que existen los clientes sin pedidos, que el estado cancelado importa y que los nombres no son únicos. **Un buen prompt se escribe con los conceptos de este módulo.**

### Conversar e iterar

En lugar de buscar el "prompt perfecto", es más eficiente una conversación:

1. **Explica el objetivo:** "Estoy armando un reporte de clientes para gerencia."
2. **Da el contexto:** motor, esquema y reglas, una sola vez al principio.
3. **Pide de forma concreta:** "Escribí la consulta del total por cliente."
4. **Revisa y pregunta:** "¿Por qué usaste `INNER JOIN`? ¿Qué pasa con los clientes sin pedidos?"
5. **Refina:** "Reescribila sin alias de una letra", "pasala a una CTE", "¿qué índice necesitaría?".

El paso 4 es el que separa usar la IA de **depender** de ella: si no puedes hacer esa pregunta, no puedes evaluar la respuesta.

### Verificar: la lista de control

Antes de usar una consulta generada por IA (o por cualquier persona), revisa:

| Pregunta | Lección |
| --- | --- |
| ¿Usa la sintaxis de **mi** motor (`TOP` y no `LIMIT`, `STRING_AGG` y no `GROUP_CONCAT`)? | [Ordenar y paginar](03-Ordenar%20y%20paginar.md), [Expresiones](04-Expresiones%2C%20CASE%20y%20funciones.md) |
| ¿Existen todas las tablas, columnas y funciones que usa? | — |
| ¿El tipo de join es el correcto? ¿Se pierden filas sin pareja? | [JOIN](06-JOIN.md) |
| ¿Hay un filtro sobre la tabla derecha de un `LEFT JOIN` en el `WHERE`? | [JOIN](06-JOIN.md) |
| ¿Algún join multiplica filas antes de un `COUNT` o un `SUM`? | [JOIN](06-JOIN.md) |
| ¿Agrupa por la clave o por un nombre que puede repetirse? | [Agregados](05-Agregados%20y%20GROUP%20BY.md) |
| ¿Qué pasa con los `NULL` en los filtros, los `NOT IN` y los agregados? | [NULL](../01-fundamentos/07-NULL%20y%20la%20lógica%20de%20tres%20valores.md), [Subconsultas](08-Subconsultas.md) |
| ¿Hay divisiones enteras o `AVG` sobre enteros? | [Expresiones](04-Expresiones%2C%20CASE%20y%20funciones.md) |
| ¿Los rangos de fechas pierden el último día? | [Filtrar con WHERE](02-Filtrar%20con%20WHERE.md) |
| ¿El orden es determinista, si pagina o usa `TOP`? | [Ordenar y paginar](03-Ordenar%20y%20paginar.md) |
| Con datos que conozco, ¿el resultado coincide con lo que calculo a mano? | — |

La última fila es la más importante. Ejecuta la consulta sobre un conjunto de datos **pequeño y conocido** (como el esquema de ejemplo, donde sabes que Sofía no tiene pedidos y que el pedido de María está cancelado) y comprueba que cada caso borde aparece como debe.

### Modificaciones generadas por IA

Para un `UPDATE`, `DELETE` o `MERGE` generado por IA, el método de [Modificar datos](11-Modificar%20datos.md) no es opcional:

* Ejecuta primero un `SELECT` con el mismo filtro.
* Modifica dentro de una transacción, con `OUTPUT`, y revisa `@@ROWCOUNT` antes del `COMMIT`.
* Ten una copia de seguridad reciente antes de cualquier cambio masivo.
* Nunca ejecutes una modificación generada por IA directamente en producción.

### Otros usos útiles

* **Explicar consultas:** "Explicame esta consulta paso a paso" sobre un procedimiento heredado de 300 líneas. Muy útil para empezar a entender; igual hay que verificar la explicación contra el código.
* **Depurar:** pegar la consulta **y el mensaje de error completo** (con el número de `Msg`). El número del error es contexto valioso.
* **Traducir dialectos:** "Convertí esta consulta de PostgreSQL a SQL Server 2022". Revisa especialmente fechas, concatenación, `LIMIT`, `ILIKE` y la división entera, que se comportan distinto.
* **Generar datos de prueba:** "Generá 20 `INSERT` de clientes ficticios para este esquema."
* **Ideas de análisis:** "Con estas tablas, ¿qué 3 a 5 análisis serían útiles para el área comercial?". Cuanto más describas los datos, más relevantes serán las ideas.

### Los riesgos

| Riesgo | Qué es | Cómo mitigarlo |
| --- | --- | --- |
| **Alucinaciones** | Funciones, columnas u opciones que no existen, dichas con seguridad | Verificar contra la documentación oficial y contra tu esquema; ejecutar en un entorno de prueba |
| **Sesgo de confirmación** | Aceptar el resultado porque "tiene sentido" | Probar con datos conocidos y casos borde; no usar la misma IA para verificarse a sí misma |
| **Privacidad y seguridad** | Lo que pegas en un chat puede quedar almacenado o usarse para entrenar modelos | Nunca pegar datos reales de clientes, contraseñas ni cadenas de conexión; usar el esquema, no los datos; seguir la política de la empresa sobre herramientas de IA |
| **Propiedad intelectual** | El código generado puede parecerse a código existente | Revisar las políticas de la empresa sobre código generado |
| **Dependencia** | Dejar de entender lo que se ejecuta | Pedir siempre la explicación, y poder escribir la consulta sin ayuda si hiciera falta |

-----

## Ejemplo completo

Revisar la consulta del problema con la lista de control.

**La consulta generada:**

```sql
SELECT c.Nombre, SUM(d.Cantidad * d.PrecioUnitario) AS TotalGastado
FROM Cliente c
JOIN Pedido p ON c.ClienteId = p.ClienteId
JOIN DetallePedido d ON p.PedidoId = d.PedidoId
GROUP BY c.Nombre;
```

| Nombre | TotalGastado |
| --- | --- |
| Ana Torres | 476.00 |
| Jorge Díaz | 140.50 |
| Luis Ramos | 849.90 |
| María Quispe | 210.00 |

**La revisión:**

| Pregunta de la lista | Hallazgo |
| --- | --- |
| ¿Sintaxis de mi motor? | Sí, es T-SQL válido. |
| ¿Tipo de join correcto? | **No.** `INNER JOIN` descarta a Sofía, que no tiene pedidos. La regla pide todos los clientes. |
| ¿Agrupa por la clave? | **No.** Agrupa por `Nombre`: dos clientes homónimos se sumarían juntos. |
| ¿Respeta las reglas de negocio? | **No.** Cuenta el pedido cancelado de María (210.00 que no son una venta). |
| ¿Datos conocidos? | Con el esquema de ejemplo: falta Sofía (0.00) y María debería tener 0.00. |

**La consulta corregida** (que es, de hecho, el ejemplo completo de [JOIN](06-JOIN.md)):

```sql
SELECT c.Nombre,
       c.Email,
       COUNT(t.PedidoId)         AS Pedidos,
       COALESCE(SUM(t.Total), 0) AS TotalGastado
FROM dbo.Cliente AS c
LEFT JOIN (
    SELECT p.PedidoId, p.ClienteId, SUM(d.Cantidad * d.PrecioUnitario) AS Total
    FROM dbo.Pedido AS p
    JOIN dbo.DetallePedido AS d ON d.PedidoId = p.PedidoId
    WHERE p.Estado <> 'Cancelado'
    GROUP BY p.PedidoId, p.ClienteId
) AS t ON t.ClienteId = c.ClienteId
GROUP BY c.ClienteId, c.Nombre, c.Email
ORDER BY TotalGastado DESC;
```

| Nombre | Email | Pedidos | TotalGastado |
| --- | --- | --- | --- |
| Luis Ramos | luis.ramos@correo.com | 1 | 849.90 |
| Ana Torres | ana.torres@correo.com | 2 | 476.00 |
| Jorge Díaz | jorge.diaz@correo.com | 1 | 140.50 |
| María Quispe | maria.quispe@correo.com | 0 | 0.00 |
| Sofía Mendoza | sofia.mendoza@correo.com | 0 | 0.00 |

Qué cambió y por qué:

* **`LEFT JOIN`:** aparecen todos los clientes, como pedía la regla.
* **El filtro de cancelados dentro de la subconsulta:** se excluye el pedido de María sin eliminar a María del reporte.
* **`GROUP BY c.ClienteId, ...`:** dos clientes homónimos quedan separados; el email permite distinguirlos en el resultado.
* **Agregar antes de unir:** `COUNT(t.PedidoId)` cuenta pedidos, no líneas.

La IA ahorró escribir la estructura de la consulta. Encontrar los tres errores dependió de conocer el `LEFT JOIN`, el `GROUP BY` por clave y las reglas del negocio. **Eso no se delega.**

-----

## Errores comunes

**1. No decir el motor.**
Qué pasa: `LIMIT`, `GROUP_CONCAT`, `ILIKE`, `||` o funciones de fecha de otro dialecto.
Por qué: sin contexto, el modelo usa la sintaxis más frecuente en sus datos de entrenamiento.
Arreglo: la primera línea del prompt es el motor y la versión.

**2. Aceptar la consulta porque "corre y da números razonables".**
Qué pasa: reportes incorrectos en producción.
Por qué: sesgo de confirmación; los errores de lógica no dan errores de sintaxis.
Arreglo: la lista de control y la prueba con datos conocidos.

**3. Pegar datos reales en el chat.**
Qué pasa: datos personales o confidenciales fuera de la empresa.
Por qué: es la forma más rápida de "darle contexto".
Arreglo: pega el esquema (`CREATE TABLE`), nunca los datos; si hace falta un ejemplo, inventado o anonimizado.

**4. Ejecutar una modificación generada sin transacción.**
Qué pasa: un `UPDATE` con un filtro mal interpretado modifica miles de filas.
Por qué: confianza en una herramienta que no conoce tus datos.
Arreglo: `SELECT` previo, transacción, `OUTPUT`, `@@ROWCOUNT`.

**5. Pedirle a la IA que verifique su propia respuesta.**
Qué pasa: "Sí, la consulta es correcta", sobre una consulta incorrecta.
Por qué: el mismo sistema que cometió el error lo evalúa con los mismos supuestos.
Arreglo: verifica tú, con datos y con la lista de control.

-----

## Según el motor

La IA escribe mejor SQL para los motores más comunes en sus datos de entrenamiento, y tiende a **mezclar dialectos** si no se le indica uno. Las diferencias que más vas a tener que corregir:

| Si la IA escribe | Para SQL Server corrige a | Para PostgreSQL | Para MySQL |
| --- | --- | --- | --- |
| `LIMIT n` | `TOP (n)` / `OFFSET FETCH` | Correcto | Correcto |
| `GROUP_CONCAT` | `STRING_AGG` | `string_agg` | Correcto |
| `ILIKE` | `LIKE` (según la intercalación) | Correcto | `LIKE` |
| `NOW()` | `SYSDATETIME()` | Correcto | Correcto |
| `DATEDIFF(fin, inicio)` | `DATEDIFF(day, inicio, fin)` | `fin - inicio` | Correcto |
| `INSERT ... ON CONFLICT` | `MERGE` o `UPDATE` + `INSERT` | Correcto | `ON DUPLICATE KEY UPDATE` |
| `WITH RECURSIVE` | `WITH` | Correcto | Correcto |

### En MongoDB

Para MongoDB, la IA es especialmente útil escribiendo *aggregation pipelines*, cuya sintaxis de etapas y operadores es más difícil de recordar que el SQL. Los mismos principios valen: indica la versión (algunas etapas, como `$setWindowFields`, son de la 5.0 en adelante), describe la estructura de los documentos (qué está embebido y qué referenciado) y verifica el resultado con documentos de prueba. Y desconfía de las respuestas que traducen SQL a MongoDB de forma literal (un `$lookup` por cada join): suelen indicar un modelo de datos pensado con cabeza relacional.

-----

## Cuándo sí y cuándo no

**Usa la IA cuando:**

* Ya sabes qué consulta necesitas y quieres escribirla más rápido.
* Necesitas recordar una sintaxis, traducir entre dialectos o entender código ajeno.
* Quieres ideas de análisis o una segunda opinión sobre un enfoque.

**No la uses (o no sola) cuando:**

* No podrías evaluar si la respuesta es correcta: primero aprende el concepto.
* El prompt necesitaría datos reales o confidenciales.
* La consulta va a modificar datos de producción sin revisión humana.

**Ten en cuenta:**

* La IA es una herramienta, no un reemplazo del conocimiento: cuanto más SQL sabes, más útil te resulta, porque escribes mejores prompts y detectas antes los errores.

-----

## Resumen en 5 líneas

1. La IA escribe SQL rápido, pero no conoce tu motor, tu esquema ni tus reglas: dáselos en el prompt.
2. Un buen prompt incluye el motor y la versión, el `CREATE TABLE`, las reglas de negocio y el resultado esperado.
3. Las consultas que corren y dan números razonables pueden estar mal: revisa joins, agrupaciones, `NULL`, divisiones y fechas.
4. Prueba siempre con datos conocidos y casos borde, y ejecuta las modificaciones con el método seguro.
5. Nunca pegues datos reales ni credenciales, y no le pidas a la IA que verifique su propia respuesta.

-----

## Para profundizar

<details>
<summary>Dar el esquema completo de forma eficiente</summary>

Para una conversación larga sobre una base, en lugar de describir las tablas a mano, genera el esquema desde la propia base y pégalo una vez al principio:

```sql
SELECT t.name AS Tabla, c.name AS Columna, ty.name AS Tipo, c.max_length, c.is_nullable
FROM sys.tables AS t
JOIN sys.columns AS c ON c.object_id = t.object_id
JOIN sys.types AS ty ON ty.user_type_id = c.user_type_id
ORDER BY t.name, c.column_id;
```

O usa *Script Table as → CREATE To* en SSMS, que incluye las claves y las restricciones. Revisa antes que el script no contenga datos sensibles (por ejemplo, valores por defecto con información interna o nombres de servidores).

</details>

<details>
<summary>IA integrada en las herramientas</summary>

Cada vez más herramientas traen la IA integrada: GitHub Copilot en VS Code y en SSMS, asistentes en los portales de bases de datos en la nube y en los IDE como DataGrip. La ventaja es que pueden leer el esquema de la conexión activa sin que lo pegues. El riesgo es el mismo, pero menos visible: verifica qué información envía la herramienta (¿solo el esquema o también datos de resultados?) y si está aprobada por la política de tu empresa.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso herramientas de IA para escribir y entender consultas más rápido, pero siempre les doy el contexto (el motor, las tablas y lo que necesito) y reviso el resultado antes de usarlo. No pego datos reales y pruebo las consultas con datos que conozco.

### Respuesta ampliada (semi-senior)

Uso la IA como acelerador, no como fuente de verdad. Le doy el motor y la versión, el `CREATE TABLE` de las tablas relevantes, las reglas de negocio que afectan la consulta y la forma del resultado esperado, y trabajo de forma iterativa, preguntando por qué eligió cada join o cada agrupación. Reviso lo generado con una lista concreta: dialecto, tipo de join y filtros en `LEFT JOIN`, multiplicación de filas antes de agregar, agrupación por clave, manejo de `NULL`, división entera y rangos de fechas, y lo pruebo con un conjunto de datos pequeño donde conozco los casos borde. Las modificaciones generadas las ejecuto con el mismo método que las mías: `SELECT` previo, transacción, `OUTPUT` y `@@ROWCOUNT`. En seguridad, nunca envío datos reales ni credenciales, solo el esquema, y sigo la política de la empresa sobre herramientas de IA. Mi criterio es que no ejecuto algo que no podría haber escrito o, al menos, explicado yo mismo.

### Preguntas frecuentes de seguimiento

**1. ¿Qué contexto le das a la IA para que genere buen SQL?**
El motor y la versión, el esquema con tipos y claves, las reglas de negocio y el resultado esperado.

**2. ¿Cómo verificas una consulta generada por IA?**
Con una lista de control (dialecto, joins, agrupaciones, `NULL`, divisiones, fechas) y ejecutándola sobre datos conocidos con casos borde.

**3. ¿Qué riesgos tiene usar IA con bases de datos?**
Alucinaciones, sesgo de confirmación al aceptar resultados plausibles, exposición de datos confidenciales y modificaciones peligrosas ejecutadas sin revisión.

-----

## Práctica

**Ejercicio 1.** Escribe el prompt que le darías a una IA para obtener "los 3 productos más vendidos en unidades de cada categoría, sin contar los pedidos cancelados", con todo el contexto necesario.

<details>
<summary>Solución</summary>

```text
Motor: SQL Server 2022 (T-SQL).

Tablas:
- dbo.Categoria (CategoriaId INT PK, Nombre NVARCHAR(50) NOT NULL UNIQUE)
- dbo.Producto (ProductoId INT PK, CategoriaId INT NOT NULL FK → Categoria, Nombre NVARCHAR(100) NOT NULL)
- dbo.Pedido (PedidoId INT PK, Estado VARCHAR(20) NOT NULL)
  Estados: Pendiente, Pagado, Enviado, Entregado, Cancelado.
- dbo.DetallePedido (PedidoId INT FK → Pedido, ProductoId INT FK → Producto, Cantidad INT NOT NULL),
  PK (PedidoId, ProductoId).

Necesito: los 3 productos con más unidades vendidas de cada categoría.
Reglas:
- No contar los pedidos con Estado = 'Cancelado'.
- Si hay empate en el tercer puesto, incluir a todos los empatados.
- Los productos sin ventas no deben aparecer.

Resultado: | Categoria | Producto | Unidades | Puesto |
ordenado por Categoria y Puesto. Usá funciones de ventana y una CTE.
```

Fíjate en las decisiones que el prompt hace explícitas: qué hacer con los empates (lo que decide entre `ROW_NUMBER` y `RANK`), qué hacer con los productos sin ventas (lo que decide entre `INNER` y `LEFT JOIN`) y el motor. Una IA sin esas reglas elige por ti, y no siempre lo que querías.

</details>

**Ejercicio 2.** Una IA te propone esta consulta para "el promedio de unidades por línea de pedido en abril de 2026". Encuentra los errores sin ejecutarla.

```sql
SELECT AVG(d.Cantidad) AS PromedioUnidades
FROM DetallePedido d
JOIN Pedido p ON p.PedidoId = d.PedidoId
WHERE p.FechaPedido BETWEEN '2026-04-01' AND '2026-04-30';
```

<details>
<summary>Solución</summary>

1. **`AVG` sobre una columna `INT`:** en SQL Server devuelve un entero truncado. Con las líneas de abril (2, 1 y 1 unidades), el promedio real es 1.33, y la consulta devuelve 1. Arreglo: `AVG(d.Cantidad * 1.0)`.
2. **`BETWEEN` con `DATETIME2`:** `'2026-04-30'` es la medianoche del 30, así que se pierden los pedidos del 30 de abril después de las 00:00. Arreglo: `>= '2026-04-01' AND < '2026-05-01'`.
3. **Regla de negocio sin definir:** el pedido 4 (abril) está cancelado. ¿Cuenta? La consulta lo incluye sin que nadie lo haya decidido. Hay que preguntarlo y, si no cuenta, agregar `AND p.Estado <> 'Cancelado'`.

Con los datos de ejemplo, el error 2 no cambia el resultado (no hay pedidos el 30 de abril): exactamente el tipo de error que pasa todas las pruebas y aparece en producción.

</details>

-----

## Siguiente lección

Con esto termina el módulo de consultas. El próximo módulo, **Índices y rendimiento**, retoma las consultas de este módulo para ver cómo las ejecuta el motor y cómo hacerlas rápidas. Está planificado.
