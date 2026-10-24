# Tipos de datos

## En una frase

El tipo de una columna define qué valores acepta, cuánto espacio ocupa y cómo se compara y ordena; elegirlo bien (exacto para dinero, de fecha para fechas, con la longitud justa para texto) es la primera restricción de integridad que tiene una tabla.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un atributo y un dominio, de [El modelo relacional](02-El%20modelo%20relacional.md).
* Que la computadora guarda los números en binario, y que no todos los decimales tienen una representación binaria exacta.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Precisión y escala:** en `DECIMAL(p, s)`, `p` es el total de dígitos y `s` cuántos van después del punto.
* **Tipo exacto / aproximado:** el exacto guarda el valor tal cual (`DECIMAL`); el aproximado guarda una aproximación binaria (`FLOAT`).
* **Unicode:** estándar que representa caracteres de todos los idiomas (á, ñ, 你, 🙂).
* **Intercalación (collation):** reglas de comparación y orden del texto: si distingue mayúsculas, acentos, etc.
* **Conversión implícita:** cuando el motor convierte un tipo en otro por su cuenta para poder comparar.

-----

## El problema

Mira esta tabla. Funciona:

```sql
CREATE TABLE dbo.Venta (
    VentaId INT,
    Fecha   VARCHAR(50),
    Total   FLOAT,
    Cliente VARCHAR(MAX),
    Pagado  VARCHAR(5)
);
```

Y estos son los problemas que esconde:

* `Fecha VARCHAR`: se guarda `'10/02/2026'`. ¿Es 10 de febrero u 2 de octubre? Al ordenar, `'10/02/2026'` va antes que `'9/01/2025'`, porque compara texto carácter por carácter. Y nada impide guardar `'mañana'`.
* `Total FLOAT`: `0.1 + 0.2` no da exactamente `0.3`. En dinero, un centavo de diferencia es un descuadre contable.
* `Cliente VARCHAR(MAX)`: acepta un texto de 2 GB, y "María" con tilde puede guardarse mal según la intercalación.
* `Pagado VARCHAR(5)`: ¿`'si'`, `'Sí'`, `'true'`, `'1'`? Todos valen, y ninguno es consistente.

El tipo de dato es la **primera línea de defensa**: si la columna es `DATE`, el motor rechaza `'mañana'` antes de que llegue a tu tabla.

-----

## Cómo funciona

### Números enteros

| Tipo | Rango | Bytes | Uso típico |
| --- | --- | --- | --- |
| `TINYINT` | 0 a 255 | 1 | Valores chicos y positivos: día del mes, nivel |
| `SMALLINT` | -32.768 a 32.767 | 2 | Años, cantidades acotadas |
| `INT` | -2.147.483.648 a 2.147.483.647 | 4 | Claves, cantidades, el entero por defecto |
| `BIGINT` | ±9,2 × 10¹⁸ | 8 | Claves de tablas enormes, contadores de eventos |

`INT` alcanza para más de 2.100 millones de filas. Una tabla de logs o de eventos puede superarlo; una tabla de clientes, casi nunca.

### Números decimales: exactos frente a aproximados

`DECIMAL(p, s)` (o su sinónimo `NUMERIC`) guarda el número **exacto**:

```sql
Precio DECIMAL(10, 2)   -- 10 dígitos en total, 2 decimales: hasta 99.999.999,99
```

`FLOAT` y `REAL` guardan una **aproximación** binaria (IEEE 754). Son rápidos y cubren rangos enormes, pero no representan exactamente valores como 0,1:

```sql
DECLARE @a FLOAT = 0.1, @b FLOAT = 0.2;

IF @a + @b = 0.3
    PRINT 'iguales';
ELSE
    PRINT 'distintos';   -- ← esto es lo que imprime
```

Regla: **dinero, cantidades y todo lo que se suma o se compara por igualdad → `DECIMAL`**. `FLOAT` solo para mediciones científicas o físicas, donde una aproximación es aceptable (coordenadas, sensores).

SQL Server también tiene `MONEY`, pero conviene evitarlo: tiene 4 decimales fijos y redondea en los pasos intermedios de multiplicaciones y divisiones:

```sql
DECLARE @m MONEY = 1, @d DECIMAL(19, 4) = 1;

SELECT @m / 3 * 3 AS ConMoney,      -- 0.9999: el paso intermedio se recortó a 4 decimales
       @d / 3 * 3 AS ConDecimal;    -- 0.999999999999999: el paso intermedio conservó 15 decimales
```

Ninguno da exactamente 1, porque un tercio no tiene representación decimal finita. La diferencia es cuánto se pierde en cada paso: `MONEY` recorta a 4 decimales en cada operación intermedia, y en cadenas de cálculos (descuentos, impuestos, prorrateos) ese error se acumula. `DECIMAL` amplía la escala del resultado intermedio, y tú decides cuándo redondear con `ROUND`.

### Texto

| Tipo | Qué guarda | Longitud máxima declarada |
| --- | --- | --- |
| `CHAR(n)` | Texto de longitud fija; rellena con espacios | 8.000 bytes |
| `VARCHAR(n)` | Texto de longitud variable | 8.000 bytes, o `MAX` (hasta 2 GB) |
| `NCHAR(n)` | Unicode de longitud fija | 4.000 pares de bytes |
| `NVARCHAR(n)` | Unicode de longitud variable | 4.000 pares de bytes, o `MAX` |

La `N` significa *national*: guarda Unicode (UTF-16). Con `NVARCHAR` cualquier carácter se guarda bien, sin depender de la intercalación. Por eso en SQL Server los nombres, las direcciones y todo texto escrito por personas va en `NVARCHAR`.

Los literales Unicode llevan el prefijo `N`:

```sql
INSERT INTO dbo.Cliente (Nombre, Email) VALUES (N'María Quispe', N'maria@correo.com');
--                                              ↑ sin la N, la "í" puede convertirse en "?"
```

`VARCHAR` tiene sentido para códigos que solo usan caracteres ASCII: estados (`'Pagado'`), códigos de país (`'PE'`), números de documento.

**La longitud es una regla de negocio, no un número al azar.** Un email tiene como máximo 254 caracteres (por el estándar), un código de país ISO tiene 2. Declarar `NVARCHAR(MAX)` "por las dudas" quita una validación y empeora el rendimiento: las columnas `MAX` no se pueden usar como clave de un índice.

`CHAR(n)` solo conviene cuando todos los valores tienen exactamente la misma longitud (`CHAR(2)` para códigos de país). Con longitudes variables desperdicia espacio y rellena con espacios.

### Fecha y hora

| Tipo | Qué guarda | Precisión | Bytes |
| --- | --- | --- | --- |
| `DATE` | Solo fecha | Día | 3 |
| `TIME(p)` | Solo hora | Hasta 100 ns | 3 a 5 |
| `DATETIME2(p)` | Fecha y hora | Hasta 100 ns (`p` de 0 a 7) | 6 a 8 |
| `DATETIMEOFFSET(p)` | Fecha, hora y desfase horario (`-05:00`) | Hasta 100 ns | 8 a 10 |
| `DATETIME` | Fecha y hora (tipo heredado) | Unos 3 ms | 8 |
| `SMALLDATETIME` | Fecha y hora (tipo heredado) | Minuto | 4 |

Para código nuevo: `DATE` si no importa la hora (fecha de nacimiento), `DATETIME2` si importa (fecha de un pedido) y `DATETIMEOFFSET` si necesitas conservar el desfase horario de origen. `DATETIME` sigue apareciendo en bases antiguas: redondea los milisegundos a .000, .003 o .007 y empieza en el año 1753.

La práctica más común es guardar en **UTC** (`SYSUTCDATETIME()`) y convertir a la hora local al mostrar. Así no hay ambigüedades con zonas horarias ni con cambios de horario de verano.

Los literales de fecha se escriben en formato ISO 8601, que SQL Server interpreta igual con cualquier configuración regional:

```sql
'2026-03-01'            -- DATE
'2026-03-01T10:15:00'   -- DATETIME2
```

### Otros tipos frecuentes

| Tipo | Uso |
| --- | --- |
| `BIT` | Booleanos: `0`, `1` o `NULL`. `Activo BIT NOT NULL DEFAULT 1` |
| `UNIQUEIDENTIFIER` | GUID de 16 bytes. Ver [Claves primarias](04-Claves%20primarias.md) |
| `VARBINARY(n)` / `VARBINARY(MAX)` | Datos binarios: hashes, archivos pequeños |
| `JSON` / `NVARCHAR(MAX)` | Documentos JSON. SQL Server 2025 y Azure SQL tienen un tipo `json` nativo; antes se guardaba en `NVARCHAR(MAX)` y se consultaba con `JSON_VALUE` |

### Criterios para elegir

1. **El tipo más específico:** fecha → `DATE`, no `VARCHAR`; booleano → `BIT`, no `VARCHAR(5)`.
2. **Exacto si se suma o se compara:** dinero → `DECIMAL`.
3. **El más chico que cubra el dominio con margen:** no hace falta `BIGINT` para la edad. Pero tampoco ahorres de más: cambiar el tipo de una columna con millones de filas es caro.
4. **Longitud con significado:** la que dice la regla de negocio.
5. **Mismo tipo en los dos extremos de una clave foránea:** si `Cliente.ClienteId` es `INT`, `Pedido.ClienteId` también.

-----

## Ejemplo completo

La tabla `Venta` del problema, con cada tipo justificado:

**Antes:**

```sql
CREATE TABLE dbo.Venta (
    VentaId INT,
    Fecha   VARCHAR(50),
    Total   FLOAT,
    Cliente VARCHAR(MAX),
    Pagado  VARCHAR(5)
);
```

**Después:**

```sql
CREATE TABLE dbo.Venta (
    VentaId       INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    FechaVenta    DATETIME2(0)      NOT NULL DEFAULT SYSUTCDATETIME(),
    Total         DECIMAL(10, 2)    NOT NULL,
    NombreCliente NVARCHAR(100)     NOT NULL,
    EstaPagada    BIT               NOT NULL DEFAULT 0
);

INSERT INTO dbo.Venta (Total, NombreCliente) VALUES (211.00, N'Ana Torres');
INSERT INTO dbo.Venta (FechaVenta, Total, NombreCliente) VALUES ('mañana', 10, N'Luis');
```

La segunda inserción falla:

```text
Msg 241, Level 16, State 1
Conversion failed when converting date and/or time from character string.
```

Qué cambió y por qué:

* **`FechaVenta DATETIME2(0)`:** el motor valida la fecha, ordena bien y permite aritmética de fechas (`DATEADD`, `DATEDIFF`). La precisión 0 (segundos) basta para una venta.
* **`Total DECIMAL(10, 2)`:** exacto; las sumas cuadran al centavo.
* **`NombreCliente NVARCHAR(100)`:** Unicode, con una longitud que valida algo. (En el diseño real, aquí va una clave foránea a `Cliente`; se verá en el módulo de diseño.)
* **`EstaPagada BIT`:** solo `0` o `1`, y el nombre se lee como una pregunta.
* **`NOT NULL` y `DEFAULT`:** cada columna dice si es obligatoria (ver [Restricciones](06-Restricciones.md)).

-----

## Errores comunes

**1. Guardar dinero en `FLOAT`.**
Qué pasa: sumas que no cuadran por fracciones de centavo; comparaciones por igualdad que fallan.
Por qué: `FLOAT` es binario aproximado.
Arreglo: `DECIMAL(p, s)` con la escala que pida el negocio (2 decimales para la mayoría de las monedas).

**2. Guardar fechas como texto.**
Qué pasa: formatos mezclados, ordenamientos incorrectos, `'31/02/2026'` aceptado.
Por qué: un `VARCHAR` no sabe que contiene una fecha.
Arreglo: `DATE` o `DATETIME2`, y literales en formato ISO.

**3. `VARCHAR` sin longitud.**
Qué pasa: `DECLARE @nombre VARCHAR = 'María';` guarda solo `'M'`, sin ningún error.
Por qué: sin longitud, `VARCHAR` vale `VARCHAR(1)` en una declaración y `VARCHAR(30)` en un `CAST`.
Arreglo: escribe siempre la longitud.

**4. Olvidar el prefijo `N` en los literales Unicode.**
Qué pasa: caracteres que no existen en la página de códigos se guardan como `?`.
Por qué: `'texto'` es `VARCHAR`; `N'texto'` es `NVARCHAR`.
Arreglo: `N'...'` para todo texto que va a una columna `NVARCHAR`.

**5. Tipos distintos en una comparación.**
Qué pasa: una consulta que filtra una columna `VARCHAR` con un parámetro `NVARCHAR` (lo que envían por defecto muchos ORM) obliga a convertir la columna y puede impedir que se use el índice.
Por qué: conversión implícita: `NVARCHAR` tiene más precedencia que `VARCHAR`, así que se convierte la columna, no el parámetro.
Arreglo: el mismo tipo en la columna y en el parámetro (en EF Core, configurar la columna con `.IsUnicode(false)` si es `VARCHAR`).

**6. Texto más largo que la columna.**
Qué pasa: `Msg 2628: String or binary data would be truncated in table 'Tienda.dbo.Cliente', column 'Nombre'`.
Por qué: el valor supera la longitud declarada. En `INSERT` y `UPDATE` es un error (bien); en variables, se trunca en silencio (mal).
Arreglo: valida la longitud en la aplicación y declara las variables con la misma longitud que la columna.

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Entero | `INT`, `BIGINT` | `integer`, `bigint` | `INT`, `BIGINT` (admite `UNSIGNED`) |
| Decimal exacto | `DECIMAL(p,s)` | `numeric(p,s)` | `DECIMAL(p,s)` |
| Texto Unicode | `NVARCHAR(n)` (o `VARCHAR` con intercalación UTF-8, desde 2019) | `text` o `varchar(n)`: siempre en la codificación de la base (UTF-8 habitual) | `VARCHAR(n)` con `utf8mb4` |
| Texto sin límite | `NVARCHAR(MAX)` | `text` (sin penalización de rendimiento) | `TEXT`, `LONGTEXT` |
| Fecha y hora | `DATETIME2` | `timestamp` | `DATETIME` |
| Fecha y hora con zona | `DATETIMEOFFSET` | `timestamptz` (guarda en UTC y convierte al leer) | `TIMESTAMP` (convierte a UTC; rango hasta 2038) |
| Booleano | `BIT` | `boolean` (`true`/`false`) | `BOOLEAN` (alias de `TINYINT(1)`) |
| UUID | `UNIQUEIDENTIFIER` | `uuid` | `BINARY(16)` o `CHAR(36)` |
| JSON | `json` (2025) o `NVARCHAR(MAX)` | `jsonb` (binario, indexable) y `json` | `JSON` |

Detalles que dan sorpresas:

* **MySQL:** `utf8` es un alias del antiguo `utf8mb3`, que no guarda emojis ni algunos caracteres asiáticos. Usa siempre `utf8mb4` (el valor por defecto desde MySQL 8.0).
* **PostgreSQL:** `varchar(n)` y `text` rinden igual; `varchar(n)` solo agrega la validación de longitud. Usa `timestamptz` para casi todo lo que sea "un momento en el tiempo".
* **MySQL `TIMESTAMP`:** su rango termina el 19 de enero de 2038. Para fechas futuras lejanas (vencimientos, contratos), usa `DATETIME`.

### En MongoDB

MongoDB guarda BSON, con sus propios tipos. La trampa principal: en `mongosh` (JavaScript), un número escrito sin más es un `double`, es decir, un número aproximado como `FLOAT`. Para dinero se usa `Decimal128` (`NumberDecimal("120.00")` en `mongosh`). Las fechas son el tipo `Date` (milisegundos en UTC), y el identificador por defecto es `ObjectId`. Como no hay esquema fijo, nada impide que un documento tenga `precio: "120"` (texto) y otro `precio: 120`, salvo que definas un validador con `$jsonSchema`.

-----

## Cuándo sí y cuándo no

**Tómate el tiempo de elegir el tipo cuando:**

* La tabla va a crecer: el tamaño de cada fila multiplica las lecturas de disco y la memoria usada.
* La columna va a ser clave, clave foránea o parte de un índice.

**Ten en cuenta:**

* Cambiar el tipo de una columna en una tabla grande bloquea la tabla y reescribe datos. Es más barato elegir bien al principio.
* No sobreoptimices: `TINYINT` frente a `INT` en una tabla de 200 filas no cambia nada. Piensa en el dominio primero y en los bytes después.

**Herramientas que ayudan:**

* `sp_help 'dbo.Cliente'` muestra los tipos, longitudes y restricciones de cada columna.
* `SELECT * FROM sys.types;` lista todos los tipos disponibles.

-----

## Resumen en 5 líneas

1. El tipo de dato es la primera restricción: decide qué valores entran, cuánto ocupan y cómo se comparan.
2. Dinero y todo lo que se suma → `DECIMAL`; `FLOAT` solo para mediciones aproximadas. Evita `MONEY`.
3. Texto de personas → `NVARCHAR(n)` con literales `N'...'`; la longitud es una regla de negocio.
4. Fechas → `DATE` o `DATETIME2` (en UTC), nunca texto; `DATETIME` es heredado.
5. Mismo tipo en ambos extremos de una clave foránea y en los parámetros de las consultas, para evitar conversiones implícitas.

-----

## Para profundizar

<details>
<summary>Cuánto ocupa un DECIMAL</summary>

El espacio depende de la precisión, no del valor guardado:

| Precisión | Bytes |
| --- | --- |
| 1 a 9 | 5 |
| 10 a 19 | 9 |
| 20 a 28 | 13 |
| 29 a 38 | 17 |

`DECIMAL(9, 2)` (hasta 9.999.999,99) ocupa 5 bytes; `DECIMAL(10, 2)` ocupa 9. Si sabes que los precios nunca superan los diez millones, la precisión 9 ahorra 4 bytes por fila. En una tabla de detalle con cientos de millones de filas, eso son gigabytes.

</details>

<details>
<summary>Intercalaciones: mayúsculas, acentos y UTF-8</summary>

La intercalación define cómo se comparan y ordenan los textos. `Modern_Spanish_CI_AS` significa *case insensitive* (`'ana' = 'ANA'`) y *accent sensitive* (`'Maria' <> 'María'`). Se define a nivel de servidor, de base y de columna, y se puede forzar en una consulta:

```sql
SELECT * FROM dbo.Cliente WHERE Nombre = N'maria quispe' COLLATE Modern_Spanish_CI_AI;
```

Desde SQL Server 2019 existen intercalaciones `_UTF8` que permiten guardar Unicode en `VARCHAR` con codificación UTF-8. Ahorran espacio cuando el texto es mayormente ASCII, pero cambian el significado de `n`: en `VARCHAR(n)`, `n` siempre son bytes, y una "ñ" ocupa 2.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Elijo el tipo según lo que guarda la columna: `INT` para enteros, `DECIMAL` para dinero porque es exacto, `NVARCHAR` para texto con acentos u otros idiomas, y `DATE` o `DATETIME2` para fechas. Nunca guardo fechas ni números como texto, porque pierdo la validación y el orden correcto.

### Respuesta ampliada (semi-senior)

El tipo es la primera restricción de dominio y afecta a la integridad, al almacenamiento y al rendimiento. Para valores monetarios uso `DECIMAL` con la escala del negocio, porque `FLOAT` es aproximado y `MONEY` redondea en los cálculos intermedios. Para texto uso `NVARCHAR` con una longitud que refleje la regla de negocio; evito `MAX` salvo para textos largos de verdad, porque no se puede indexar y empeora los planes. Para tiempos, `DATETIME2` en UTC o `DATETIMEOFFSET` si necesito el desfase de origen. Cuido que los tipos coincidan en las claves foráneas y en los parámetros de las consultas: una conversión implícita de `VARCHAR` a `NVARCHAR` sobre la columna puede convertir una búsqueda por índice en un recorrido completo. Y elijo con el crecimiento en mente, porque cambiar el tipo de una columna grande es una operación cara.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar `FLOAT` para precios?**
Porque es binario aproximado: valores como 0,1 no tienen representación exacta, y las sumas y las comparaciones por igualdad fallan. `DECIMAL` es exacto.

**2. ¿Qué diferencia hay entre `VARCHAR` y `NVARCHAR`?**
`NVARCHAR` guarda Unicode (UTF-16) y admite cualquier carácter; `VARCHAR` depende de la página de códigos de la intercalación (o de UTF-8 con intercalaciones `_UTF8`). `NVARCHAR` usa 2 bytes por carácter básico.

**3. ¿`DATETIME` o `DATETIME2`?**
`DATETIME2`: más precisión, mayor rango, precisión configurable y menos espacio con precisiones bajas. `DATETIME` se mantiene por compatibilidad.

**4. ¿Por qué no declarar todo como `NVARCHAR(MAX)`?**
Porque pierdes la validación de longitud, las columnas `MAX` no pueden ser clave de un índice y el optimizador estima peor la memoria que necesita la consulta.

-----

## Práctica

**Ejercicio 1.** Elige el tipo de SQL Server para cada columna y justifica:

1. Fecha de nacimiento de un cliente.
2. Precio de un producto en soles.
3. Código de país ISO (`PE`, `AR`).
4. Descripción larga de un producto (puede tener varios párrafos).
5. Momento exacto en que se registró un pago.
6. Si el cliente aceptó recibir correos.
7. Temperatura de un sensor.

<details>
<summary>Solución</summary>

| Columna | Tipo | Por qué |
| --- | --- | --- |
| Fecha de nacimiento | `DATE` | La hora no importa |
| Precio | `DECIMAL(10, 2)` | Dinero: exacto, 2 decimales |
| Código de país | `CHAR(2)` | Longitud fija, solo ASCII |
| Descripción larga | `NVARCHAR(MAX)` o `NVARCHAR(4000)` | Texto largo de personas; `MAX` está justificado si puede superar los 4.000 caracteres |
| Momento del pago | `DATETIME2(3)` en UTC, o `DATETIMEOFFSET(3)` | Importa el instante; los milisegundos ayudan a ordenar eventos |
| Acepta correos | `BIT NOT NULL DEFAULT 0` | Booleano, con un valor por defecto explícito |
| Temperatura | `DECIMAL(5, 2)` o `FLOAT` | Medición física: `FLOAT` es aceptable; `DECIMAL` si se va a comparar por igualdad o mostrar con decimales fijos |

</details>

**Ejercicio 2.** ¿Qué devuelve este código y por qué?

```sql
DECLARE @codigo VARCHAR = 'PEDIDO-001';
SELECT @codigo AS Codigo, LEN(@codigo) AS Largo;
```

<details>
<summary>Solución</summary>

| Codigo | Largo |
| --- | --- |
| P | 1 |

`VARCHAR` sin longitud en una declaración es `VARCHAR(1)`, y la asignación a una variable trunca en silencio, sin ningún error. Arreglo: `DECLARE @codigo VARCHAR(20) = 'PEDIDO-001';`.

</details>

-----

## Siguiente lección

[Claves primarias](04-Claves%20primarias.md)
