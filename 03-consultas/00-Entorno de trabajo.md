# Entorno de trabajo

## En una frase

Para escribir consultas necesitas tres cosas: un **servidor** de SQL Server en ejecución, un **cliente** para conectarte (SSMS, VS Code o `sqlcmd`) y saber en **qué base** estás parado antes de presionar F5.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una instancia, una base de datos y un esquema, de [Qué es una base de datos y un SGBD](../01-fundamentos/01-Qué%20es%20una%20base%20de%20datos%20y%20un%20SGBD.md).
* Cómo levantar la base `Tienda`, del [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Servicio de SQL Server:** el proceso de Windows (o el contenedor) que ejecuta el motor.
* **Instancia por defecto / con nombre:** la instancia por defecto se llama `MSSQLSERVER` y se conecta como `localhost`; una instancia con nombre se conecta como `localhost\NOMBRE`.
* **Autenticación de Windows / de SQL Server:** entrar con tu usuario de Windows, o con un usuario y una contraseña propios de SQL Server (como `sa`).
* **SSMS:** SQL Server Management Studio, el cliente gráfico oficial.
* **Lote (batch):** un grupo de instrucciones que se envía junto al servidor; `GO` los separa.
* **`sqlcmd`:** el cliente de línea de comandos de SQL Server.

-----

## El problema

Primer día con SQL Server. Abres SSMS, te conectas, escribes `CREATE TABLE dbo.Cliente (...)`, presionas F5 y funciona. Una semana después:

* No encuentras la tabla en la base `Tienda`. Estaba en `master`, la base a la que SSMS te conecta por defecto.
* Tenías seleccionadas dos líneas de un script de prueba y presionaste F5: SSMS ejecutó **solo la selección**, que era un `DELETE` sin su `WHERE`.
* Los scripts de la semana están en una carpeta temporal y se perdieron al cerrar SSMS.
* Al día siguiente SSMS no conecta: el servicio de SQL Server no arrancó.

Ninguno de estos problemas es de SQL. Son problemas del **entorno**, y se evitan sabiendo cómo funciona.

-----

## Cómo funciona

### El servidor tiene que estar en ejecución

SQL Server es un servicio. Si lo instalaste en Windows:

1. Abre **Servicios** (`services.msc`).
2. Busca **SQL Server (MSSQLSERVER)** para la instancia por defecto, o **SQL Server (SQLEXPRESS)** si instalaste Express.
3. Verifica que esté **en ejecución**; si no, clic derecho → **Iniciar**.

Si usas Docker (como en el [esquema de ejemplo](../00-Esquema%20de%20ejemplo.md)), el servidor corre mientras corre el contenedor:

```bash
docker ps                   # ¿está el contenedor "sqlserver" en la lista?
docker start sqlserver      # si se detuvo
```

### Conectarse: nombre del servidor y autenticación

| Instalación | Nombre del servidor |
| --- | --- |
| Instancia por defecto en tu máquina | `localhost` o `.` |
| SQL Server Express | `localhost\SQLEXPRESS` o `.\SQLEXPRESS` |
| Docker con el puerto 1433 | `localhost,1433` |
| Servidor remoto | `nombre-servidor` o `nombre-servidor\INSTANCIA` |

Dos formas de autenticarse:

* **Autenticación de Windows:** usa tu usuario de Windows; no hay contraseña que escribir. Es la habitual en una instalación local.
* **Autenticación de SQL Server:** usuario y contraseña propios del motor. `sa` es el administrador. En Docker es la única opción.

En las versiones recientes de SSMS, la conexión cifra el tráfico por defecto. En un entorno local con el certificado autofirmado de SQL Server, marca **Trust server certificate** (confiar en el certificado del servidor) para poder conectarte.

### SSMS: los paneles que importan

| Panel | Para qué | Atajo |
| --- | --- | --- |
| **Object Explorer** | Ver servidores, bases, tablas, columnas, índices | `F8` |
| **Editor de consultas** | Escribir y ejecutar SQL | `Ctrl` + `N` (nueva consulta) |
| **Resultados / Mensajes** | Ver las filas devueltas y los mensajes del servidor | `Ctrl` + `R` (mostrar u ocultar) |
| **Solution Explorer** | Organizar scripts en soluciones y proyectos | `Ctrl` + `Alt` + `L` |

Si desarmas los paneles, **Window → Reset Window Layout** los devuelve a su lugar.

### En qué base estás parado

La barra de herramientas muestra un desplegable con la base activa, y SSMS empieza en `master`. Por eso cada script empieza con:

```sql
USE Tienda;
GO
```

Y para comprobarlo en cualquier momento:

```sql
SELECT DB_NAME() AS BaseActual, @@SERVERNAME AS Servidor, @@VERSION AS Version;
```

### Ejecutar: F5 ejecuta la selección

| Acción | Atajo |
| --- | --- |
| Ejecutar | `F5` o `Ctrl` + `E` |
| Analizar la sintaxis sin ejecutar | `Ctrl` + `F5` |
| Resultados en cuadrícula / en texto | `Ctrl` + `D` / `Ctrl` + `T` |
| Comentar / descomentar la selección | `Ctrl` + `K`, `Ctrl` + `C` / `Ctrl` + `K`, `Ctrl` + `U` |
| Mayúsculas / minúsculas en la selección | `Ctrl` + `Shift` + `U` / `Ctrl` + `Shift` + `L` |
| Ir a una línea | `Ctrl` + `G` |
| Plan de ejecución estimado / real | `Ctrl` + `L` / `Ctrl` + `M` (se usan en el módulo de índices) |

La regla más importante de esta tabla: **si hay texto seleccionado, `F5` ejecuta solo la selección**. Si no hay nada seleccionado, ejecuta todo el script. Es cómodo para probar una consulta dentro de un script largo, y peligroso si la selección es la mitad de una instrucción.

### `GO` separa lotes

`GO` no es SQL: es una señal para SSMS y `sqlcmd` de que envíen al servidor lo escrito hasta ahí. Algunas instrucciones necesitan su propio lote (por ejemplo, `CREATE VIEW` debe ser la primera instrucción del lote), y las variables declaradas en un lote no existen en el siguiente:

```sql
DECLARE @x INT = 1;
GO
SELECT @x;   -- Msg 137: Must declare the scalar variable "@x".
```

### Guardar los scripts

SSMS permite agrupar scripts en **soluciones** y **proyectos** (*File → New → Project → SQL Server Scripts*): los scripts guardados pasan de *Miscellaneous* a *Queries*, y la solución reabre todo como lo dejaste. Una alternativa más común hoy es guardar los `.sql` en una carpeta versionada con git, junto al resto del proyecto. Lo que no conviene es depender de las pestañas sin guardar.

### Otras herramientas

* **VS Code + extensión MSSQL:** editor liviano con IntelliSense, resultados en cuadrícula y planes de ejecución. Útil si ya trabajas en VS Code.
* **`sqlcmd`:** el cliente de línea de comandos. Sirve para automatizar y para servidores sin interfaz gráfica:

  ```bash
  sqlcmd -S localhost,1433 -U sa -d Tienda -C -Q "SELECT COUNT(*) FROM dbo.Cliente;"
  sqlcmd -S localhost,1433 -U sa -d Tienda -C -i crear-tienda.sql
  ```

  `-Q` ejecuta una consulta y sale; `-i` ejecuta un archivo; `-C` confía en el certificado del servidor; sin `-P`, pide la contraseña (mejor que escribirla en la línea de comandos, donde queda en el historial).

* **DBeaver o DataGrip:** clientes multimotor, útiles si trabajas con SQL Server, PostgreSQL y MySQL a la vez.

-----

## Ejemplo completo

Primer contacto con la base `Tienda`, de punta a punta:

```sql
-- 1. Pararse en la base correcta
USE Tienda;
GO

-- 2. Confirmar dónde estamos
SELECT DB_NAME() AS BaseActual;

-- 3. Listar las tablas de la base
SELECT s.name AS Esquema, t.name AS Tabla
FROM sys.tables AS t
JOIN sys.schemas AS s ON s.schema_id = t.schema_id
ORDER BY Tabla;

-- 4. Primera consulta
SELECT 'Hola, Tienda' AS Saludo;
```

Resultados:

| BaseActual |
| --- |
| Tienda |

| Esquema | Tabla |
| --- | --- |
| dbo | Categoria |
| dbo | Cliente |
| dbo | DetallePedido |
| dbo | Pedido |
| dbo | Producto |

| Saludo |
| --- |
| Hola, Tienda |

Lo que pasó: `USE` cambió la base activa, `DB_NAME()` lo confirmó, las vistas del sistema `sys.tables` y `sys.schemas` listaron los objetos (lo mismo que se ve en el Object Explorer, pero consultable) y la última consulta muestra que `SELECT` no necesita una tabla para devolver un valor.

-----

## Errores comunes

**1. Crear objetos en `master`.**
Qué pasa: tablas mezcladas con las del sistema, que "desaparecen" de la base donde las buscas.
Por qué: SSMS se conecta a `master` por defecto.
Arreglo: `USE Tienda;` al principio de cada script, y mira el desplegable antes de ejecutar.

**2. Ejecutar una selección parcial sin querer.**
Qué pasa: queda seleccionado `DELETE FROM dbo.Pedido` sin la línea del `WHERE`, y F5 borra todo.
Por qué: F5 ejecuta solo lo seleccionado.
Arreglo: revisa la selección antes de ejecutar, y las instrucciones que modifican datos van dentro de una transacción (ver [Modificar datos](11-Modificar%20datos.md)).

**3. No poder conectarse.**
Qué pasa: `A network-related or instance-specific error occurred while establishing a connection to SQL Server`.
Por qué: el servicio está detenido, el nombre de la instancia es incorrecto (`localhost` en lugar de `localhost\SQLEXPRESS`), o el puerto está bloqueado.
Arreglo: verifica el servicio o el contenedor, el nombre exacto de la instancia y, para conexiones remotas, que TCP/IP esté habilitado en el SQL Server Configuration Manager.

**4. Error de certificado al conectarse.**
Qué pasa: `The certificate chain was issued by an authority that is not trusted`.
Por qué: los clientes recientes cifran la conexión por defecto, y el servidor local usa un certificado autofirmado.
Arreglo: marca *Trust server certificate* en local (o `-C` en `sqlcmd`). En producción, se instala un certificado válido.

**5. Usar `sa` para todo.**
Qué pasa: cualquier error o ataque tiene permisos totales sobre el servidor.
Por qué: es lo primero que funciona.
Arreglo: en local es aceptable para aprender; en cualquier otro entorno, usuarios con los permisos mínimos necesarios.

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Cliente gráfico oficial | SSMS | pgAdmin | MySQL Workbench |
| Línea de comandos | `sqlcmd` | `psql` | `mysql` |
| Cambiar de base | `USE Tienda;` | `\c tienda` (en `psql`; no hay `USE` en SQL) | `USE tienda;` |
| Listar tablas | `SELECT name FROM sys.tables;` | `\dt` o `information_schema.tables` | `SHOW TABLES;` |
| Base inicial al conectarse | `master` | La que indiques (por defecto, la de tu usuario) | Ninguna, hasta el `USE` |
| Separador de lotes | `GO` (del cliente) | No hace falta | No hace falta (`DELIMITER` para procedimientos) |
| Puerto | 1433 | 5432 | 3306 |

Las vistas `INFORMATION_SCHEMA` (`INFORMATION_SCHEMA.TABLES`, `INFORMATION_SCHEMA.COLUMNS`) existen en los tres motores y son estándar. En SQL Server, las vistas `sys.*` dan más detalle.

### En MongoDB

El cliente de línea de comandos es `mongosh`, y el gráfico, MongoDB Compass. Los equivalentes básicos: `show dbs` lista las bases, `use tienda` cambia de base (y la crea al insertar el primer documento) y `show collections` lista las colecciones.

-----

## Cuándo sí y cuándo no

**Usa SSMS cuando:**

* Trabajas con SQL Server a diario: administración, planes de ejecución, Object Explorer completo.

**Usa VS Code + MSSQL cuando:**

* Escribes consultas junto al código de la aplicación y no necesitas las herramientas de administración.

**Usa `sqlcmd` cuando:**

* Automatizas scripts (despliegues, cargas, CI) o trabajas en un servidor sin interfaz gráfica.

**Ten en cuenta:**

* La herramienta no cambia el SQL: el mismo script funciona en las tres.

-----

## Resumen en 5 líneas

1. El servicio de SQL Server (o el contenedor) tiene que estar en ejecución antes de conectarse.
2. Servidor: `localhost` (por defecto), `localhost\SQLEXPRESS` (Express) o `localhost,1433` (Docker); autenticación de Windows o de SQL Server.
3. SSMS empieza en `master`: cada script empieza con `USE Tienda;`.
4. F5 ejecuta la selección si hay texto seleccionado; `GO` separa lotes y no es SQL.
5. `sqlcmd` y VS Code + MSSQL son alternativas para automatizar o trabajar junto al código.

-----

## Para profundizar

<details>
<summary>SQL Server Configuration Manager</summary>

Es la herramienta para configurar los servicios y la red de SQL Server en Windows: iniciar y detener servicios, cambiar la cuenta con la que corren, y habilitar los protocolos de red. Por defecto, SQL Server Express y Developer tienen TCP/IP **deshabilitado**, así que solo aceptan conexiones locales. Para conectarte desde otra máquina (o desde un contenedor), hay que habilitar TCP/IP, reiniciar el servicio y abrir el puerto en el firewall.

</details>

<details>
<summary>Ediciones gratuitas de SQL Server</summary>

* **Developer:** todas las funciones de Enterprise, gratis, pero solo para desarrollo y pruebas (no para producción).
* **Express:** gratis también en producción, con límites de tamaño de base, memoria y CPU. Suficiente para aplicaciones pequeñas.

Para seguir este wiki, cualquiera de las dos sirve, igual que la imagen de Docker.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso SQL Server Management Studio para conectarme al servidor, explorar las bases con el Object Explorer y ejecutar consultas. Siempre verifico en qué base estoy con `USE` antes de ejecutar, porque SSMS empieza en `master`. Para automatizar uso `sqlcmd`.

### Respuesta ampliada (semi-senior)

Trabajo con SSMS para administración y análisis (Object Explorer, planes de ejecución, Query Store) y con VS Code y la extensión MSSQL cuando escribo consultas junto al código. Mis scripts empiezan con `USE`, están versionados en git y, si modifican datos, van dentro de una transacción explícita. Para despliegues y procesos automatizados uso `sqlcmd` con archivos de script y credenciales que no quedan en el historial de comandos. Conozco los problemas típicos de conexión: el servicio detenido, el nombre de la instancia, TCP/IP deshabilitado en las ediciones Express y Developer y el cifrado por defecto de los clientes recientes, que exige confiar en el certificado en local o instalar uno válido en producción.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es `GO`?**
Un separador de lotes que interpretan SSMS y `sqlcmd`; no es una instrucción de SQL. El servidor nunca lo recibe.

**2. ¿Cómo sabes en qué base estás?**
Con el desplegable de SSMS o con `SELECT DB_NAME();`.

**3. ¿Qué diferencia hay entre la autenticación de Windows y la de SQL Server?**
La de Windows usa la identidad del usuario del sistema operativo; la de SQL Server usa un usuario y una contraseña definidos en el motor.

-----

## Práctica

**Ejercicio 1.** Conéctate a tu servidor y escribe una consulta que muestre en una sola fila: la base activa, el nombre del servidor, la fecha y hora actuales del servidor y la cantidad de tablas de la base `Tienda`.

<details>
<summary>Solución</summary>

```sql
USE Tienda;
GO

SELECT DB_NAME()                        AS BaseActual,
       @@SERVERNAME                     AS Servidor,
       SYSDATETIME()                    AS FechaHoraServidor,
       (SELECT COUNT(*) FROM sys.tables) AS CantidadTablas;
```

| BaseActual | Servidor | FechaHoraServidor | CantidadTablas |
| --- | --- | --- | --- |
| Tienda | (el nombre de tu servidor) | (la fecha actual) | 5 |

La última columna es una subconsulta escalar, que se ve en [Subconsultas](08-Subconsultas.md).

</details>

**Ejercicio 2.** Usando `INFORMATION_SCHEMA.COLUMNS`, lista las columnas de la tabla `Producto` con su tipo de dato y si admiten `NULL`, en el orden en que fueron creadas.

<details>
<summary>Solución</summary>

```sql
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'dbo' AND TABLE_NAME = 'Producto'
ORDER BY ORDINAL_POSITION;
```

| COLUMN_NAME | DATA_TYPE | IS_NULLABLE |
| --- | --- | --- |
| ProductoId | int | NO |
| CategoriaId | int | NO |
| Nombre | nvarchar | NO |
| Precio | decimal | NO |
| Stock | int | NO |
| Activo | bit | NO |

La misma consulta funciona en PostgreSQL y MySQL, cambiando el esquema (`public` o el nombre de la base).

</details>

-----

## Siguiente lección

[SELECT y el orden lógico](01-SELECT%20y%20el%20orden%20lógico.md)
