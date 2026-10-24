# Auditoría

## En una frase

Auditar es poder responder **quién hizo qué, cuándo, desde dónde y qué valores había antes**; SQL Server ofrece varias herramientas para eso (**SQL Server Audit**, **triggers de auditoría**, **tablas temporales**, **Change Data Capture** y **Change Tracking**), y cada una contesta una parte distinta de esa pregunta.

-----

## Antes de empezar

Conviene que ya sepas:

* Triggers y las tablas `inserted` y `deleted`, de [Triggers](../06-programacion-t-sql/04-Triggers.md).
* Trabajos programados, de [SQL Server Agent](05-SQL%20Server%20Agent.md).
* Operaciones por lotes, de [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md), para la retención.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **SQL Server Audit:** la auditoría nativa del motor; registra acciones (inicios de sesión, cambios de permisos, lecturas y escrituras sobre objetos) en un archivo o en el registro de eventos de Windows.
* **Especificación de auditoría:** la lista de acciones que se registran, a nivel de servidor o de base.
* **Tabla temporal (system-versioned):** una tabla que guarda automáticamente cada versión anterior de sus filas en una tabla de historial.
* **Change Data Capture (CDC):** lee el log de transacciones y copia cada cambio, con sus valores, a tablas de cambios.
* **Change Tracking:** registra **qué filas** cambiaron (solo la clave), sin los valores, para sincronizar.
* **`SESSION_CONTEXT`:** pares clave-valor asociados a la conexión; sirve para pasar el usuario de la aplicación a la base.

-----

## El problema

El precio de un producto pasó de 45.000 a 4.500 un viernes a la noche, y durante el fin de semana se vendieron 300 unidades. El lunes, las preguntas son obvias: ¿quién lo cambió?, ¿a qué hora?, ¿desde qué aplicación?, ¿cuál era el precio antes?, ¿fue un error o fue a propósito? Y la base no puede contestar ninguna: el `UPDATE` sobrescribió el valor y no dejó rastro.

En los sistemas que manejan dinero o datos personales, esas preguntas no son opcionales: normativas como PCI-DSS (datos de tarjetas), SOX (controles financieros) o las leyes de protección de datos (GDPR, y sus equivalentes locales) exigen poder contestarlas, y conservar la evidencia durante años. La auditoría se diseña **antes** del incidente.

-----

## Cómo funciona

### Qué herramienta contesta qué

| Herramienta | Qué registra | Quién | Valores anteriores | Uso típico |
| --- | --- | --- | --- | --- |
| **SQL Server Audit** | Acciones: logins, cambios de permisos, `SELECT`/`INSERT`/`UPDATE`/`DELETE` sobre objetos, con la sentencia | Sí (el login) | No | Cumplimiento formal, accesos y lecturas de datos sensibles |
| **Trigger de auditoría** | Lo que tú programes | Sí (lo que captures) | Sí | Historial de negocio a medida |
| **Tabla temporal** | Cada versión anterior de cada fila | **No** | Sí | "¿Cómo estaba este dato el 15 de marzo?" |
| **CDC** | Cada cambio con sus valores, leído del log, asíncrono | No | Sí | Alimentar un almacén de datos o un proceso de ETL |
| **Change Tracking** | Qué filas cambiaron (solo la clave) | No | No | Sincronizar una aplicación o una caché |

Ninguna lo contesta todo. Lo habitual es combinarlas: **Audit** para los accesos y los permisos, y una **tabla temporal o un trigger** para los valores de negocio.

### SQL Server Audit

Tiene dos partes: el **audit** (el destino donde se escribe) y las **especificaciones** (qué se registra). Las acciones se agrupan en *grupos de acciones* a nivel de servidor (inicios de sesión, cambios de permisos) y de base (accesos a tablas concretas):

```sql
USE master;

-- 1. El destino: archivos con tamaño máximo y rotación
CREATE SERVER AUDIT AuditoriaTienda
TO FILE (FILEPATH = 'C:\SQLAudit\', MAXSIZE = 100 MB, MAX_ROLLOVER_FILES = 20)
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT AuditoriaTienda WITH (STATE = ON);

-- 2. Qué se registra a nivel de servidor
CREATE SERVER AUDIT SPECIFICATION AuditoriaTienda_Servidor
FOR SERVER AUDIT AuditoriaTienda
ADD (FAILED_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP)
WITH (STATE = ON);

-- 3. Qué se registra en la base: quién lee o modifica los datos de los clientes
USE Tienda;

CREATE DATABASE AUDIT SPECIFICATION AuditoriaTienda_Base
FOR SERVER AUDIT AuditoriaTienda
ADD (SELECT, INSERT, UPDATE, DELETE ON dbo.Cliente BY public),
ADD (DATABASE_PERMISSION_CHANGE_GROUP)
WITH (STATE = ON);
```

Para leer lo registrado:

```sql
SELECT event_time,              -- en UTC
       action_id,               -- SL = SELECT, UP = UPDATE, LGIF = login fallido...
       server_principal_name,
       database_name, object_name,
       statement
FROM sys.fn_get_audit_file('C:\SQLAudit\AuditoriaTienda*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;
```

Detalles que importan:

* **`ON_FAILURE`** decide qué pasa si no se puede escribir la auditoría (por ejemplo, porque el disco está lleno): `CONTINUE` sigue sin registrar; `SHUTDOWN` apaga el servidor; `FAIL_OPERATION` rechaza las operaciones auditadas. Cuando la normativa exige que **nada** ocurra sin registro, se usa `SHUTDOWN` o `FAIL_OPERATION`, sabiendo el riesgo que implica para la disponibilidad.
* **Lo que registra es la sentencia, no los valores anteriores:** sabe que alguien hizo `UPDATE dbo.Producto SET Precio = 4500 WHERE ProductoId = 7`, pero no que el precio era 45.000.
* **Puede registrar lecturas** (`SELECT`), algo que ningún trigger puede hacer: es la herramienta para "quién consultó los datos de este cliente".
* Las especificaciones de base están disponibles en todas las ediciones desde SQL Server 2016 SP1.
* El archivo debe estar en un lugar al que los administradores de la base no puedan acceder para modificarlo; si no, la auditoría no protege de quien más permisos tiene.

### Triggers de auditoría

Un trigger puede guardar exactamente lo que el negocio necesita, con los valores anteriores y nuevos. La estructura base de tu cheat sheet clasifica la operación según `inserted` y `deleted`, y tiene un error sutil:

```sql
-- ❌ Clasificación de la operación con un caso mal resuelto
IF EXISTS (SELECT 1 FROM inserted) AND NOT EXISTS (SELECT 1 FROM deleted)
    SET @TipoOp = 'I';
ELSE IF EXISTS (SELECT 1 FROM inserted) AND EXISTS (SELECT 1 FROM deleted)
    SET @TipoOp = 'U';
ELSE
    SET @TipoOp = 'D';          -- un UPDATE que no afectó filas también llega aquí
```

Un trigger se ejecuta **una vez por sentencia, aunque no afecte ninguna fila**. Un `UPDATE ... WHERE ProductoId = -1` dispara el trigger con `inserted` y `deleted` vacíos, y la lógica de arriba lo clasifica como `'D'`. Además, una sola variable para toda la sentencia supone que todas las filas tienen el mismo tipo, y guardar `SELECT * FROM deleted FOR JSON PATH, WITHOUT_ARRAY_WRAPPER` con varias filas produce **varios objetos JSON pegados**, que no es un JSON válido.

La versión correcta trabaja **fila por fila**, cruzando `inserted` y `deleted` por la clave:

```sql
-- ✅ Una fila de auditoría por fila afectada, con su tipo y sus valores
CREATE SCHEMA Auditoria;
GO

CREATE TABLE Auditoria.Producto (
    AuditoriaId     BIGINT IDENTITY(1,1) NOT NULL CONSTRAINT PK_AuditoriaProducto PRIMARY KEY,
    TipoOperacion   CHAR(1)        NOT NULL CONSTRAINT CK_AuditoriaProducto_Tipo CHECK (TipoOperacion IN ('I', 'U', 'D')),
    ProductoId      INT            NOT NULL,
    DatosAnteriores NVARCHAR(MAX)  NULL,      -- JSON de la fila antes del cambio
    DatosNuevos     NVARCHAR(MAX)  NULL,      -- JSON de la fila después del cambio
    LoginBase       NVARCHAR(128)  NOT NULL CONSTRAINT DF_AuditoriaProducto_Login DEFAULT ORIGINAL_LOGIN(),
    UsuarioApp      NVARCHAR(128)  NULL,
    Aplicacion      NVARCHAR(128)  NULL CONSTRAINT DF_AuditoriaProducto_App DEFAULT APP_NAME(),
    Host            NVARCHAR(128)  NULL CONSTRAINT DF_AuditoriaProducto_Host DEFAULT HOST_NAME(),
    FechaUtc        DATETIME2(3)   NOT NULL CONSTRAINT DF_AuditoriaProducto_Fecha DEFAULT SYSUTCDATETIME()
);

CREATE INDEX IX_AuditoriaProducto_ProductoId_Fecha ON Auditoria.Producto (ProductoId, FechaUtc);
GO

CREATE OR ALTER TRIGGER dbo.trProducto_Auditoria
ON dbo.Producto
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM inserted) AND NOT EXISTS (SELECT 1 FROM deleted)
        RETURN;                                   -- la sentencia no afectó filas

    INSERT INTO Auditoria.Producto (TipoOperacion, ProductoId, DatosAnteriores, DatosNuevos, UsuarioApp)
    SELECT CASE WHEN d.ProductoId IS NULL THEN 'I'
                WHEN i.ProductoId IS NULL THEN 'D'
                ELSE 'U' END,
           COALESCE(i.ProductoId, d.ProductoId),
           (SELECT d.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),   -- un objeto por fila
           (SELECT i.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
           CAST(SESSION_CONTEXT(N'UsuarioApp') AS NVARCHAR(128))
    FROM inserted AS i
    FULL JOIN deleted AS d ON d.ProductoId = i.ProductoId;
END;
```

* El `FULL JOIN` por la clave da una fila por producto afectado: solo en `inserted` es un alta, solo en `deleted` una baja, en ambas una modificación.
* Si la clave primaria se puede modificar, el cruce por clave no funciona; es otro argumento para claves que no cambian.
* El trigger corre **dentro de la transacción** del cambio: si la sentencia se revierte, la auditoría también. Eso garantiza coherencia, pero no registra los intentos fallidos (para eso está SQL Server Audit).
* Agrega trabajo a cada escritura: se audita lo que importa, no todas las tablas.

### Quién es "quién": el usuario de la aplicación

En una aplicación web, todos los usuarios se conectan con **el mismo login** (el de la aplicación, a través del pool de conexiones). `SUSER_SNAME()` o `SYSTEM_USER` devuelven siempre `app_tienda`, y la auditoría pierde el dato más importante.

La solución es que la aplicación informe, al usar la conexión, qué usuario real está operando:

```sql
-- La aplicación lo ejecuta al tomar la conexión del pool, antes de las operaciones
EXEC sys.sp_set_session_context @key = N'UsuarioApp', @value = N'maria.lopez@tienda.com';

-- Cualquier trigger o procedimiento lo lee
SELECT SESSION_CONTEXT(N'UsuarioApp');
```

Funciones útiles para el "quién" y el "desde dónde":

| Función | Devuelve |
| --- | --- |
| `ORIGINAL_LOGIN()` | El login que abrió la conexión, aunque después se haya usado `EXECUTE AS` |
| `SUSER_SNAME()` / `SYSTEM_USER` | El login del contexto actual (cambia con `EXECUTE AS`) |
| `USER_NAME()` | El usuario de la base del contexto actual |
| `APP_NAME()` / `HOST_NAME()` | Los que **informa el cliente** en la cadena de conexión: se pueden falsificar |
| `CONNECTIONPROPERTY('client_net_address')` | La dirección IP del cliente, vista por el servidor |
| `SESSION_CONTEXT(N'clave')` | Lo que la aplicación guardó con `sp_set_session_context` |

Para auditoría, `ORIGINAL_LOGIN()` es más confiable que `SUSER_SNAME()`, y `APP_NAME()` y `HOST_NAME()` son útiles para investigar, pero no son prueba de nada.

### Tablas temporales

Una tabla temporal (*system-versioned*, desde SQL Server 2016) guarda **automáticamente** cada versión anterior de cada fila, sin triggers ni código:

```sql
ALTER TABLE dbo.Producto ADD
    ValidoDesde DATETIME2(3) GENERATED ALWAYS AS ROW START HIDDEN NOT NULL
        CONSTRAINT DF_Producto_ValidoDesde DEFAULT SYSUTCDATETIME(),
    ValidoHasta DATETIME2(3) GENERATED ALWAYS AS ROW END HIDDEN NOT NULL
        CONSTRAINT DF_Producto_ValidoHasta DEFAULT CONVERT(DATETIME2(3), '9999-12-31 23:59:59.999'),
    PERIOD FOR SYSTEM_TIME (ValidoDesde, ValidoHasta);

ALTER TABLE dbo.Producto
SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductoHistorial, HISTORY_RETENTION_PERIOD = 2 YEARS));
```

`HIDDEN` evita que las columnas nuevas aparezcan en `SELECT *` y rompan aplicaciones existentes. Para consultar el pasado:

```sql
-- Cómo estaba el producto 7 en un momento dado (¡en UTC!)
SELECT ProductoId, Nombre, Precio
FROM dbo.Producto FOR SYSTEM_TIME AS OF '2026-10-17 23:00:00'
WHERE ProductoId = 7;

-- Todas sus versiones
SELECT ProductoId, Precio, ValidoDesde, ValidoHasta
FROM dbo.Producto FOR SYSTEM_TIME ALL
WHERE ProductoId = 7
ORDER BY ValidoDesde;
```

| Subcláusula | Versiones que devuelve |
| --- | --- |
| `AS OF @t` | La vigente en el instante `@t` |
| `FROM @a TO @b` | Las que estuvieron vigentes en algún momento entre `@a` y `@b` (sin incluir las que empiezan justo en `@b`) |
| `BETWEEN @a AND @b` | Igual, pero incluye las que empiezan justo en `@b` |
| `CONTAINED IN (@a, @b)` | Las que empezaron y terminaron dentro del rango |
| `ALL` | Todas, actuales e históricas |

Detalles que importan:

* **Los períodos se guardan en UTC**: `AS OF` con una hora local devuelve el estado de otro momento.
* **No registra quién hizo el cambio.** Si hace falta, se agrega una columna como `ModificadoPor` que la aplicación (o un `DEFAULT` con `SESSION_CONTEXT`) completa en cada escritura, y queda versionada junto con el resto.
* No se puede modificar el historial directamente mientras el versionado está activo: es una buena propiedad para auditoría. Para mantenimiento se apaga con `SET (SYSTEM_VERSIONING = OFF)`, pero ese momento es justo el que una auditoría debería registrar.
* `HISTORY_RETENTION_PERIOD` (SQL Server 2017 en adelante) borra el historial viejo automáticamente.

### Change Data Capture y Change Tracking

**CDC** lee el log de transacciones de forma asíncrona y escribe cada cambio, con sus valores, en tablas de cambios. No suma trabajo a la transacción del usuario, pero **necesita el SQL Server Agent** (un trabajo captura y otro limpia):

```sql
EXEC sys.sp_cdc_enable_db;
EXEC sys.sp_cdc_enable_table @source_schema = N'dbo', @source_name = N'Pedido',
                             @role_name = NULL, @supports_net_changes = 1;

DECLARE @desde BINARY(10) = sys.fn_cdc_get_min_lsn('dbo_Pedido'),
        @hasta BINARY(10) = sys.fn_cdc_get_max_lsn();

SELECT __$operation,         -- 1 borrado, 2 alta, 3 valores antes del UPDATE, 4 valores después
       PedidoId, Estado
FROM cdc.fn_cdc_get_all_changes_dbo_Pedido(@desde, @hasta, N'all update old');
```

Por defecto, `N'all'` devuelve solo los valores nuevos de los `UPDATE` (operación 4); con `N'all update old'` aparecen también los anteriores (operación 3). Las tablas de cambios se limpian a los tres días por defecto: CDC es para **mover cambios** a otro sistema (un almacén de datos o una cola de eventos), no para guardar la historia durante años.

**Change Tracking** es más liviano: solo registra **qué claves** cambiaron y una versión, sin los valores. Sirve para que una aplicación pregunte "¿qué cambió desde la última vez que sincronicé?" y vaya a buscar las filas actuales. No es una herramienta de auditoría.

### Retención

La normativa suele fijar cuánto tiempo hay que guardar la evidencia (en el sector financiero es habitual que sean siete años), y también cuándo hay que **borrarla** (las leyes de protección de datos limitan cuánto se guardan los datos personales). La retención se automatiza:

* SQL Server Audit: `MAX_ROLLOVER_FILES` y una copia de los archivos a un almacenamiento de largo plazo.
* Tablas de auditoría propias: un trabajo del Agent que archiva y borra por lotes, como el de [Operaciones masivas en lotes](../05-transacciones-y-concurrencia/07-Operaciones%20masivas%20en%20lotes.md).
* Tablas temporales: `HISTORY_RETENTION_PERIOD`.

-----

## Ejemplo completo

La investigación del precio de 4.500 con la tabla `Producto` auditada por el trigger y la aplicación informando el usuario.

**Antes:** sin auditoría, solo queda el valor actual.

```sql
SELECT ProductoId, Nombre, Precio FROM dbo.Producto WHERE ProductoId = 7;
```

| ProductoId | Nombre | Precio |
| --- | --- | --- |
| 7 | Monitor 27" | 4500.00 |

**Después:** con el trigger, la historia del producto.

```sql
-- Lo que hace la aplicación al guardar el cambio
EXEC sys.sp_set_session_context @key = N'UsuarioApp', @value = N'jperez@tienda.com';
UPDATE dbo.Producto SET Precio = 4500 WHERE ProductoId = 7;

-- La investigación
SELECT FechaUtc, TipoOperacion,
       JSON_VALUE(DatosAnteriores, '$.Precio') AS PrecioAntes,
       JSON_VALUE(DatosNuevos, '$.Precio')     AS PrecioDespues,
       UsuarioApp, LoginBase, Aplicacion, Host
FROM Auditoria.Producto
WHERE ProductoId = 7
ORDER BY FechaUtc;
```

| FechaUtc | TipoOperacion | PrecioAntes | PrecioDespues | UsuarioApp | LoginBase | Aplicacion | Host |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-10-17 23:42:10.517 | U | 45000.00 | 4500.00 | jperez@tienda.com | app_tienda | Backoffice | WEB-02 |

Qué cambió y por qué:

* **Valores anteriores y nuevos en JSON:** la tabla de auditoría no se rompe cuando `Producto` gana columnas, y `JSON_VALUE` extrae lo que se necesita.
* **`UsuarioApp` desde `SESSION_CONTEXT`:** sin eso, la fila diría solo `app_tienda`, el login compartido de la aplicación.
* **Fecha en UTC:** el viernes a las 20:42 en Buenos Aires es el 23:42 UTC; la conversión se hace al mostrar.
* **Un índice por `(ProductoId, FechaUtc)`:** la pregunta habitual es "la historia de este registro", y la tabla de auditoría solo crece.

Con esto, la pregunta del lunes tiene respuesta en una consulta: un error de tipeo (un cero de menos) de un usuario del backoffice, el viernes a las 20:42.

-----

## Errores comunes

**1. Clasificar la operación con una sola variable por sentencia.**
Qué pasa: un `UPDATE` que no afecta filas se registra como borrado, y una sentencia con muchas filas queda como un único registro con un JSON inválido.
Por qué: el trigger se ejecuta una vez por sentencia, aunque no afecte filas, y con todas las filas juntas.
Arreglo: `RETURN` si `inserted` y `deleted` están vacías, y un `FULL JOIN` por clave para auditar fila por fila.

**2. Auditar con el login de la aplicación.**
Qué pasa: todas las filas de auditoría dicen `app_tienda`.
Por qué: la aplicación usa un login compartido a través del pool de conexiones.
Arreglo: `sp_set_session_context` con el usuario real, y leerlo con `SESSION_CONTEXT` en el trigger.

**3. Consultar una tabla temporal con la hora local.**
Qué pasa: `AS OF '2026-10-17 20:42'` devuelve el estado de tres horas antes del que se buscaba.
Por qué: los períodos se guardan en UTC.
Arreglo: convertir la hora a UTC (`AT TIME ZONE`) antes de consultar.

**4. Usar CDC como historial a largo plazo.**
Qué pasa: los cambios de hace una semana ya no están.
Por qué: el trabajo de limpieza de CDC borra lo que tenga más de tres días (por defecto).
Arreglo: CDC para mover cambios a otro sistema; para historia, una tabla temporal o una tabla de auditoría con su retención.

**5. Auditar todo.**
Qué pasa: las escrituras se vuelven lentas, la base crece sin control y nadie encuentra nada en la auditoría.
Por qué: cada trigger y cada acción auditada tiene un costo.
Arreglo: auditar las tablas y acciones sensibles o reguladas, con una retención definida.

**6. Guardar la auditoría donde la pueden modificar los mismos que se auditan.**
Qué pasa: un administrador con malas intenciones borra su rastro.
Por qué: quien tiene control total sobre la base o la carpeta de auditoría puede modificarla.
Arreglo: archivos de auditoría en una ubicación protegida, copiados a otro sistema, y la auditoría de los cambios de la propia auditoría (`AUDIT_CHANGE_GROUP`).

-----

## Según el motor

| Concepto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Auditoría nativa | SQL Server Audit | Extensión **pgAudit** | **Audit log** en Enterprise; plugins en la versión comunitaria (por ejemplo, el de MariaDB o Percona) |
| Triggers de auditoría | Por sentencia con `inserted`/`deleted` | `FOR EACH ROW` con `OLD`/`NEW` (o por sentencia con tablas de transición) | `FOR EACH ROW` con `OLD`/`NEW` |
| Historial automático de filas | Tablas temporales (2016+) | No nativo (extensiones o triggers) | No nativo en MySQL (MariaDB sí tiene *system-versioned tables*) |
| Captura de cambios | CDC, Change Tracking | Replicación lógica (*logical decoding*) | Binlog en formato `ROW` (herramientas como Debezium) |
| Contexto de la aplicación | `SESSION_CONTEXT` | `SET app.usuario = '...'` y `current_setting('app.usuario')` | Variables de sesión (`SET @usuario = '...'`) |

### En MongoDB

La auditoría nativa existe en MongoDB Enterprise y en Atlas (registra autenticaciones, cambios de permisos y, opcionalmente, operaciones CRUD). Para capturar cambios de datos se usan los **change streams**, que entregan cada cambio de una colección en tiempo real (con la versión anterior del documento si se habilita *pre-images*); un proceso los consume y los guarda en una colección de historial.

-----

## Cuándo sí y cuándo no

**Usa cada herramienta así:**

* **SQL Server Audit:** accesos, logins fallidos, cambios de permisos y lecturas de datos sensibles; cuando la normativa pide registros que no dependan del código de la aplicación.
* **Tabla temporal:** cuando la pregunta es "cómo estaba este dato en tal fecha", con poco esfuerzo; agrega una columna para el "quién".
* **Trigger de auditoría:** cuando necesitas un formato a medida, el usuario de la aplicación y los valores en cada cambio.
* **CDC:** para llevar los cambios a un almacén de datos o a otro sistema.
* **Change Tracking:** para sincronizar aplicaciones.

**No audites:**

* Tablas sin datos sensibles ni valor de negocio en su historia.
* Columnas que cambian todo el tiempo sin importancia (contadores, fechas de último acceso), que llenan la auditoría de ruido.

**Herramientas que ayudan:**

* `sys.server_audits`, `sys.server_audit_specifications` y `sys.database_audit_specifications` para revisar qué se está auditando.
* El visor de auditoría de SSMS (*Security → Audits → View Audit Logs*).

-----

## Resumen en 5 líneas

1. Auditar es poder decir quién, qué, cuándo, desde dónde y qué había antes; ninguna herramienta sola lo contesta todo.
2. SQL Server Audit registra acciones y sentencias (incluidas las lecturas), pero no los valores anteriores.
3. Los triggers de auditoría trabajan fila por fila (`FULL JOIN` entre `inserted` y `deleted`) y salen si la sentencia no afectó filas.
4. Las tablas temporales guardan cada versión en UTC sin código, pero no el "quién"; `SESSION_CONTEXT` lleva el usuario de la aplicación a la base.
5. CDC y Change Tracking son para mover y sincronizar cambios, no para conservar historia; la retención se define y se automatiza.

-----

## Para profundizar

<details>
<summary>Ledger tables: auditoría a prueba de manipulación</summary>

SQL Server 2022 incorpora las **ledger tables**: cada transacción se encadena criptográficamente con las anteriores (como en una cadena de bloques), y los resúmenes (*digests*) se pueden guardar fuera de la base, en un almacenamiento inmutable. Con `sys.sp_verify_database_ledger` se comprueba que nadie, ni siquiera un administrador con todos los permisos, modificó el historial. Las tablas *append-only* solo permiten insertar; las *updatable* guardan cada versión, como una tabla temporal. Son la respuesta a "¿y quién audita al administrador?".

</details>

<details>
<summary>Detección de patrones sospechosos</summary>

Con la auditoría en una tabla, se pueden buscar patrones: muchas operaciones de un usuario en pocos segundos, cambios fuera del horario laboral, montos fuera de lo habitual (por ejemplo, a más de tres desviaciones estándar del promedio del usuario, con `AVG` y `STDEV` como funciones de ventana) o un mismo usuario desde lugares distintos en poco tiempo. Es la base de los sistemas antifraude, que en producción suelen vivir fuera de la base transaccional, alimentados por CDC.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para auditar cambios en SQL Server se pueden usar triggers que guardan los valores anteriores y nuevos en una tabla de auditoría, tablas temporales que guardan automáticamente el historial de cada fila, o SQL Server Audit, que registra quién hizo cada acción.

### Respuesta ampliada (semi-senior)

Primero defino qué hay que contestar y por cuánto tiempo, porque cada herramienta resuelve una parte. Uso SQL Server Audit para los accesos, los logins fallidos, los cambios de permisos y las lecturas de datos sensibles, con los archivos en una ubicación protegida y una política de `ON_FAILURE` acorde a la normativa. Para la historia de los valores de negocio uso tablas temporales cuando alcanza con "cómo estaba en tal fecha", recordando que guardan los períodos en UTC y no registran el usuario, o un trigger que audita fila por fila con un `FULL JOIN` entre `inserted` y `deleted`, guarda los valores en JSON y sale si la sentencia no afectó filas. Como la aplicación usa un login compartido, paso el usuario real con `sp_set_session_context` y lo leo con `SESSION_CONTEXT`. CDC lo uso para mover cambios a un almacén de datos, no como historial, y Change Tracking para sincronizar. Audito solo lo sensible, con índices por clave y fecha, y automatizo la retención con trabajos del Agent; si hace falta protección contra manipulación, en SQL Server 2022 están las ledger tables.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué un trigger de auditoría no detecta un intento de cambio que falló?**
Porque corre dentro de la misma transacción: si la sentencia falla y se revierte, la fila de auditoría también.

**2. ¿Qué devuelve `SUSER_SNAME()` en una aplicación web?**
El login de la aplicación, el mismo para todos los usuarios; por eso se usa `SESSION_CONTEXT`.

**3. ¿Diferencia entre CDC y Change Tracking?**
CDC guarda cada cambio con sus valores, leído del log; Change Tracking solo indica qué claves cambiaron desde una versión.

**4. ¿Qué limitación tiene una tabla temporal para auditoría?**
No guarda quién hizo el cambio; hay que agregar una columna que lo registre.

-----

## Práctica

**Ejercicio 1.** Con `Producto` convertida en tabla temporal, escribe una consulta que muestre los productos cuyo precio cambió en los últimos 7 días, con el precio anterior y el nuevo.

<details>
<summary>Solución</summary>

```sql
WITH Versiones AS (
    SELECT ProductoId, Nombre, Precio, ValidoDesde,
           LAG(Precio) OVER (PARTITION BY ProductoId ORDER BY ValidoDesde) AS PrecioAnterior
    FROM dbo.Producto FOR SYSTEM_TIME ALL
)
SELECT ProductoId, Nombre, PrecioAnterior, Precio AS PrecioNuevo, ValidoDesde AS CambioUtc
FROM Versiones
WHERE ValidoDesde >= DATEADD(day, -7, SYSUTCDATETIME())
  AND PrecioAnterior <> Precio
ORDER BY CambioUtc DESC;
```

`FOR SYSTEM_TIME ALL` trae todas las versiones, y `LAG` compara cada una con la anterior del mismo producto. El filtro `PrecioAnterior <> Precio` descarta las versiones creadas por cambios en otras columnas (el stock cambia mucho más que el precio) y la primera versión de cada producto (donde `LAG` devuelve `NULL`). La comparación de fechas es en UTC, con `SYSUTCDATETIME()`.

</details>

**Ejercicio 2.** Este trigger audita los cambios de estado de los pedidos. Encuentra los dos problemas.

```sql
CREATE OR ALTER TRIGGER dbo.trPedido_AuditoriaEstado ON dbo.Pedido AFTER UPDATE AS
BEGIN
    DECLARE @PedidoId INT, @Antes VARCHAR(20), @Despues VARCHAR(20);
    SELECT @PedidoId = PedidoId, @Despues = Estado FROM inserted;
    SELECT @Antes = Estado FROM deleted;
    INSERT INTO Auditoria.EstadoPedido (PedidoId, EstadoAnterior, EstadoNuevo, Usuario)
    VALUES (@PedidoId, @Antes, @Despues, SUSER_SNAME());
END;
```

<details>
<summary>Solución</summary>

1. **Supone una sola fila:** un `UPDATE` que cambia 500 pedidos registra uno solo (y con el estado anterior de una fila y el nuevo de otra, porque las variables toman valores de filas arbitrarias). Además registra filas aunque el estado no haya cambiado, o aunque la sentencia no haya afectado ninguna fila (con `NULL`).
2. **`SUSER_SNAME()`** devuelve el login compartido de la aplicación, no el usuario real.

```sql
CREATE OR ALTER TRIGGER dbo.trPedido_AuditoriaEstado ON dbo.Pedido AFTER UPDATE AS
BEGIN
    SET NOCOUNT ON;
    IF NOT UPDATE(Estado) RETURN;              -- la sentencia no tocó la columna

    INSERT INTO Auditoria.EstadoPedido (PedidoId, EstadoAnterior, EstadoNuevo, Usuario)
    SELECT i.PedidoId, d.Estado, i.Estado,
           COALESCE(CAST(SESSION_CONTEXT(N'UsuarioApp') AS NVARCHAR(128)), ORIGINAL_LOGIN())
    FROM inserted AS i
    JOIN deleted AS d ON d.PedidoId = i.PedidoId
    WHERE d.Estado <> i.Estado;                -- solo los cambios reales
END;
```

`UPDATE(Estado)` indica si la columna aparece en el `SET`, no si cambió su valor; por eso el filtro `d.Estado <> i.Estado` sigue haciendo falta. Como `Estado` es `NOT NULL`, la comparación con `<>` es segura.

</details>

-----

## Siguiente lección

[Workshop de tuning de punta a punta](07-Workshop%20de%20tuning%20de%20punta%20a%20punta.md)
