# Qué es una base de datos y un SGBD

## En una frase

Una **base de datos** es un conjunto organizado de datos que persiste en el tiempo; el **SGBD** (sistema de gestión de bases de datos) es el software que los guarda, los protege y permite que muchas personas los consulten y modifiquen a la vez sin corromperlos.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un archivo y qué significa guardar algo en disco.
* La idea básica de cliente-servidor: un programa pide algo y otro programa responde.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Base de datos:** conjunto organizado de datos persistentes sobre un dominio (una tienda, un hospital, un banco).
* **SGBD (DBMS en inglés):** software que administra bases de datos: SQL Server, PostgreSQL, MySQL, MongoDB.
* **SQL:** lenguaje estándar para definir, consultar y modificar datos en bases relacionales.
* **Instancia:** un servidor de base de datos en ejecución, con su propia memoria y sus propios procesos.
* **Esquema (schema):** contenedor con nombre que agrupa tablas y otros objetos dentro de una base de datos.
* **OLTP / OLAP:** sistemas transaccionales (muchas operaciones pequeñas) frente a sistemas analíticos (pocas consultas enormes).

-----

## El problema

Imagina que la tienda guarda sus pedidos en una hoja de cálculo compartida:

| Pedido | Cliente | Ciudad | Producto | Cantidad | Precio |
| --- | --- | --- | --- | --- | --- |
| 1 | Ana Torres | Lima | Teclado mecánico | 1 | 120.00 |
| 2 | Luis Ramos | arequipa | Monitor 27 | 1 | 849.90 |
| 3 | ana torres | Lima  | Clean Code | 2 | 95 |

Funciona durante una semana. Después empiezan las preguntas incómodas:

* **Concurrencia:** dos vendedores guardan el archivo al mismo tiempo. ¿Cuál de los dos cambios sobrevive?
* **Durabilidad:** se corta la luz mientras se guarda. ¿El archivo queda a medias?
* **Integridad:** "Ana Torres" y "ana torres", ¿son la misma persona? ¿Quién impide escribir `-5` en Cantidad?
* **Acceso eficiente:** con dos millones de filas, ¿cuánto tarda encontrar los pedidos de un cliente?
* **Seguridad:** el vendedor tiene que ver los pedidos, pero no el margen de ganancia. ¿Cómo ocultas una columna?

Ninguno de estos problemas es de "guardar datos". Son problemas de **administrar** datos compartidos. Para eso existe el SGBD.

-----

## Cómo funciona

### Qué resuelve un SGBD

| Problema | Lo que ofrece el SGBD |
| --- | --- |
| Varios usuarios escriben a la vez | Control de concurrencia: bloqueos o versiones de filas |
| Un fallo a mitad de una operación | Transacciones: la operación se aplica entera o no se aplica |
| Un corte de luz | Registro de transacciones (*log*): lo confirmado sobrevive al reinicio |
| Datos inválidos o contradictorios | Restricciones: tipos, claves, `CHECK`, claves foráneas |
| Búsquedas lentas | Índices y un optimizador que decide cómo ejecutar cada consulta |
| Quién puede ver o cambiar qué | Usuarios, roles y permisos |
| Pérdida del servidor | Copias de seguridad, réplicas y recuperación |

La base de datos son **los datos**; el SGBD es **el sistema** que los cuida. Decir "MySQL es mi base de datos" es un atajo común, pero técnicamente MySQL es el SGBD y `tienda` es la base de datos.

### Arquitectura cliente-servidor

Tu aplicación no lee los archivos de la base de datos directamente. Le envía consultas al servidor, que es el único que toca el disco:

```text
Aplicación (C#, Kotlin, Python)
        │  consulta SQL
        ▼
Driver / conector (ADO.NET, JDBC, psycopg)
        │  protocolo de red (TDS en SQL Server, puerto 1433)
        ▼
Servidor del SGBD (la instancia)
  ├── Analizador y optimizador → decide CÓMO ejecutar
  ├── Motor de ejecución       → lee y escribe páginas
  ├── Gestor de transacciones  → bloqueos, log
  └── Caché en memoria (buffer pool)
        │
        ▼
Disco: archivos de datos + archivo de log
```

Como todo pasa por el servidor, él puede coordinar a cientos de clientes a la vez sin que se pisen.

### Instancia, base de datos y esquema

En SQL Server los objetos viven en una jerarquía de cuatro niveles:

```text
Instancia (servidor)        localhost
└── Base de datos           Tienda
    └── Esquema             dbo, ventas, inventario
        └── Objeto          tabla Cliente, vista, procedimiento
```

Por eso el nombre completo de una tabla tiene hasta cuatro partes:

```sql
SELECT Nombre FROM Tienda.dbo.Cliente;   -- base.esquema.tabla
```

`dbo` (*database owner*) es el esquema por defecto. Los esquemas sirven para agrupar objetos por área (`ventas.Pedido`, `inventario.Producto`) y para dar permisos a un grupo completo de tablas de una sola vez.

Ojo con la palabra "esquema", porque tiene dos significados:

* **Esquema como objeto:** el contenedor `dbo` de la lista anterior.
* **Esquema como diseño:** la estructura completa de tablas y relaciones ("el esquema de la tienda").

El contexto aclara cuál de los dos es.

### Familias de bases de datos

No todo SGBD es relacional. Las familias principales son estas:

| Familia | Cómo organiza los datos | Ejemplos | Fuerte en |
| --- | --- | --- | --- |
| **Relacional** | Tablas con filas y columnas, relacionadas por claves | SQL Server, PostgreSQL, MySQL, Oracle | Datos de negocio estructurados, integridad, consultas flexibles |
| **Documental** | Documentos JSON/BSON anidados | MongoDB, Couchbase | Estructuras variables, datos que se leen juntos |
| **Clave-valor** | Un valor opaco por cada clave | Redis, DynamoDB | Caché, sesiones, lecturas por clave muy rápidas |
| **Columnar ancha** | Filas con columnas dinámicas, particionadas | Cassandra, ScyllaDB | Escrituras masivas distribuidas |
| **Grafos** | Nodos y aristas | Neo4j | Relaciones profundas: redes sociales, recomendaciones |
| **Series temporales** | Mediciones indexadas por tiempo | InfluxDB, TimescaleDB | Métricas, sensores (IoT) |

El modelo relacional es el punto de partida razonable para datos de negocio. Las otras familias resuelven problemas específicos; no son "la versión moderna" del relacional.

### SQL y sus sublenguajes

SQL (*Structured Query Language*) se divide según lo que hace cada instrucción:

| Sublenguaje | Para qué | Instrucciones |
| --- | --- | --- |
| **DDL** (definición) | Crear y cambiar la estructura | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** (manipulación) | Leer y cambiar los datos | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| **DCL** (control) | Permisos | `GRANT`, `REVOKE`, `DENY` |
| **TCL** (transacciones) | Agrupar operaciones | `BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK` |

Algunos autores separan `SELECT` en un quinto grupo, **DQL** (consulta). Es la misma idea con otra etiqueta.

SQL es **declarativo**: describes **qué** datos quieres, no **cómo** buscarlos. El optimizador decide si usa un índice, en qué orden une las tablas y cuántos hilos usa.

```sql
-- Qué quiero: los clientes de Lima. No digo cómo encontrarlos.
SELECT Nombre FROM dbo.Cliente WHERE Ciudad = N'Lima';
```

### OLTP frente a OLAP

| | OLTP (transaccional) | OLAP (analítico) |
| --- | --- | --- |
| Ejemplo | Registrar un pedido, cobrar | "Ventas por categoría y mes de los últimos 5 años" |
| Operaciones | Muchas, pequeñas, de lectura y escritura | Pocas, enormes, casi solo de lectura |
| Diseño | Normalizado (módulo 02) | Desnormalizado: esquema estrella, data warehouse |
| Prioridad | Integridad y concurrencia | Velocidad de agregación |

La base `Tienda` de este wiki es OLTP. El diseño analítico se menciona en [Desnormalización](../02-diseno-y-modelado/08-Desnormalización.md).

-----

## Ejemplo completo

Crear una base de datos, una tabla, insertar datos y consultarlos, con un sublenguaje distinto en cada paso:

```sql
-- DDL: estructura
CREATE DATABASE Pruebas;
GO
USE Pruebas;
GO

CREATE TABLE dbo.Nota (
    NotaId INT IDENTITY(1,1) PRIMARY KEY,
    Texto  NVARCHAR(200) NOT NULL
);

-- DML + TCL: datos dentro de una transacción
BEGIN TRANSACTION;
    INSERT INTO dbo.Nota (Texto) VALUES (N'Aprender qué es un SGBD');
    INSERT INTO dbo.Nota (Texto) VALUES (N'Diseñar antes de crear tablas');
COMMIT;

-- DML: consulta
SELECT NotaId, Texto FROM dbo.Nota;
```

Resultado:

| NotaId | Texto |
| --- | --- |
| 1 | Aprender qué es un SGBD |
| 2 | Diseñar antes de crear tablas |

Lo que pasó: con DDL se definió la estructura, con TCL las dos inserciones quedaron agrupadas (si la segunda fallara, `ROLLBACK` desharía las dos) y con DML se consultaron los datos. `GO` no es SQL: es un separador de lotes que entienden SSMS y `sqlcmd`.

-----

## Errores comunes

**1. Usar una hoja de cálculo o archivos sueltos como base de datos compartida.**
Qué pasa: datos duplicados, sobrescritos o inconsistentes en cuanto hay más de un usuario.
Por qué: un archivo no ofrece concurrencia, transacciones ni restricciones.
Arreglo: un SGBD, aunque sea uno pequeño (SQLite para una sola aplicación local, SQL Server Express o PostgreSQL para varios usuarios).

**2. Crear tablas en la base `master`.**
Qué pasa: tus tablas se mezclan con las del sistema de SQL Server.
Por qué: al conectarte, SSMS te deja en `master` si no eliges otra base.
Arreglo: `USE Tienda;` al principio del script, o elige la base en el desplegable antes de ejecutar.

**3. Elegir NoSQL "porque escala mejor".**
Qué pasa: terminas reimplementando a mano joins, transacciones e integridad en la aplicación.
Por qué: la escala no es el único criterio, y un SGBD relacional bien diseñado aguanta muchísimo más de lo que suele necesitar un proyecto.
Arreglo: elige por la forma de los datos y los patrones de acceso, no por moda (el módulo 06 del [wiki](../README.md) trata MongoDB).

**4. Pensar que SQL ejecuta las líneas en el orden en que se escriben.**
Qué pasa: sorpresas con el rendimiento y con el orden del resultado.
Por qué: SQL es declarativo; el optimizador elige el plan.
Arreglo: describe el resultado que quieres y deja el "cómo" al motor (se estudia en el módulo de rendimiento).

-----

## Según el motor

| Aspecto | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| Jerarquía | Instancia → base → esquema → objeto | Clúster → base → esquema → objeto | Servidor → base (= esquema) → objeto |
| Esquema por defecto | `dbo` | `public` | No hay: "schema" y "database" son sinónimos |
| Consultas entre bases | Sí: `Base.dbo.Tabla` | No directamente (requiere extensiones como `postgres_fdw`) | Sí: `base.tabla` |
| Puerto por defecto | 1433 | 5432 | 3306 |
| Delimitar identificadores | `[Mi Tabla]` (o `"Mi Tabla"` con `QUOTED_IDENTIFIER`) | `"Mi Tabla"` | `` `Mi Tabla` `` |
| Dialecto de SQL | T-SQL | PL/pgSQL para código procedural | SQL de MySQL |
| Licencia | Comercial (Express y Developer son gratuitas) | Código abierto | Código abierto (GPL) + ediciones comerciales |

### En MongoDB

La jerarquía cambia de nombres y de forma: **servidor → base de datos → colección → documento**. Una colección se parece a una tabla, pero sus documentos no necesitan tener todos los mismos campos, y un documento puede contener objetos y arreglos anidados. No usa SQL: se consulta con métodos como `db.pedidos.find({ estado: "Pagado" })`.

-----

## Cuándo sí y cuándo no

**Usa un SGBD relacional cuando:**

* Los datos tienen estructura clara y relaciones entre sí (clientes, pedidos, facturas).
* La integridad importa: no puede haber un pedido de un cliente que no existe.
* Necesitas transacciones: cobrar y descontar stock tienen que pasar juntos o no pasar.

**Ten en cuenta:**

* Para una caché o sesiones, un almacén clave-valor (Redis) es más simple y rápido.
* Para análisis sobre años de datos, conviene un almacén analítico separado del OLTP.

**Herramientas que ayudan:**

* **SQL Server:** SSMS (*SQL Server Management Studio*) o VS Code con la extensión MSSQL.
* **PostgreSQL:** pgAdmin o `psql`.
* **MySQL:** MySQL Workbench.
* **Multimotor:** DBeaver o DataGrip.
* **MongoDB:** MongoDB Compass o `mongosh`.

-----

## Resumen en 5 líneas

1. La base de datos son los datos; el SGBD es el software que los guarda, protege y comparte.
2. El SGBD resuelve concurrencia, durabilidad, integridad, rendimiento y seguridad, cosas que un archivo no puede.
3. En SQL Server la jerarquía es instancia → base → esquema → objeto; `dbo` es el esquema por defecto.
4. SQL es declarativo y se divide en DDL (estructura), DML (datos), DCL (permisos) y TCL (transacciones).
5. Hay varias familias (relacional, documental, clave-valor, grafos...); se elige por la forma de los datos, no por moda.

-----

## Para profundizar

<details>
<summary>Cómo guarda los datos el motor</summary>

SQL Server guarda las filas en **páginas** de 8 KB, agrupadas en **extensiones** (*extents*) de 8 páginas (64 KB). Leer una fila implica leer su página completa, y por eso el tamaño de las filas importa: cuantas más filas entran en una página, menos lecturas hacen falta.

Cada cambio se escribe primero en el **registro de transacciones** (*write-ahead logging*) y después, en segundo plano, en los archivos de datos. Si el servidor se cae, al reiniciar repasa el log: rehace lo confirmado y deshace lo que quedó a medias. PostgreSQL hace lo mismo con su WAL (páginas de 8 KB), y MySQL InnoDB con su *redo log* (páginas de 16 KB por defecto).

</details>

<details>
<summary>ACID, en una línea cada propiedad</summary>

* **Atomicidad:** la transacción se aplica entera o no se aplica.
* **Consistencia:** se pasa de un estado válido a otro estado válido (las restricciones se cumplen).
* **Aislamiento:** las transacciones concurrentes no ven los cambios a medias de las demás.
* **Durabilidad:** lo confirmado sobrevive a un fallo.

Se estudian a fondo en el módulo de transacciones y concurrencia.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una base de datos es un conjunto organizado de datos persistentes, y el SGBD es el software que la administra, como SQL Server o PostgreSQL. El SGBD permite que varios usuarios trabajen a la vez, protege la integridad de los datos con restricciones y garantiza que no se pierda nada con las transacciones. Las bases relacionales organizan los datos en tablas y se consultan con SQL.

### Respuesta ampliada (semi-senior)

El SGBD es un servidor que centraliza el acceso a los datos: los clientes envían SQL declarativo y el motor analiza, optimiza y ejecuta cada consulta, gestionando la concurrencia con bloqueos o versiones de filas, la durabilidad con un registro de escritura anticipada y la integridad con restricciones declarativas. En SQL Server la jerarquía es instancia, base de datos, esquema y objeto, y los esquemas sirven para organizar y asignar permisos. La elección entre una base relacional y una NoSQL depende del modelo de datos y de los patrones de acceso: lo relacional destaca en integridad, transacciones y consultas *ad hoc*; lo documental, en estructuras variables que se leen juntas; lo clave-valor, en lecturas por clave a baja latencia. También distingo las cargas OLTP de las OLAP, que piden diseños opuestos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre una base de datos y un SGBD?**
La base de datos son los datos organizados; el SGBD es el software que los administra. PostgreSQL es un SGBD; `tienda` es una base de datos dentro de él.

**2. ¿Qué significa que SQL sea declarativo?**
Que describes el resultado y no el algoritmo. El optimizador decide el plan: índices, orden de los joins, paralelismo.

**3. ¿Qué es DDL y qué es DML?**
DDL define la estructura (`CREATE`, `ALTER`, `DROP`); DML trabaja con los datos (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).

**4. ¿Cuándo elegirías una base NoSQL?**
Cuando el modelo de datos o el patrón de acceso lo justifican: documentos de estructura variable que se leen completos, cachés clave-valor, grafos con relaciones profundas o escrituras masivas distribuidas. No por defecto.

**5. ¿Qué es un esquema en SQL Server?**
Un contenedor con nombre dentro de una base de datos que agrupa objetos (tablas, vistas, procedimientos). Sirve para organizarlos y para asignar permisos a todo el grupo de una vez. El esquema por defecto es `dbo`.

-----

## Práctica

**Ejercicio 1.** Clasifica cada instrucción como DDL, DML, DCL o TCL:

```sql
ALTER TABLE dbo.Cliente ADD Telefono VARCHAR(20) NULL;
GRANT SELECT ON dbo.Pedido TO Vendedor;
UPDATE dbo.Producto SET Precio = 130.00 WHERE ProductoId = 1;
ROLLBACK;
TRUNCATE TABLE dbo.Nota;
SELECT COUNT(*) FROM dbo.Pedido;
```

<details>
<summary>Solución</summary>

| Instrucción | Sublenguaje |
| --- | --- |
| `ALTER TABLE` | DDL |
| `GRANT` | DCL |
| `UPDATE` | DML |
| `ROLLBACK` | TCL |
| `TRUNCATE TABLE` | DDL (vacía la tabla sin registrar cada fila; por eso no se considera DML) |
| `SELECT` | DML (o DQL, según el autor) |

</details>

**Ejercicio 2.** ¿Qué familia de base de datos elegirías para cada caso, y por qué?

1. El sistema de facturación de una empresa.
2. Las sesiones de usuario de una web con millones de visitas.
3. "Amigos de mis amigos que compraron este producto".
4. Un catálogo de productos en el que cada categoría tiene atributos distintos (una laptop tiene RAM; un libro, ISBN).

<details>
<summary>Solución</summary>

1. **Relacional:** integridad, transacciones y datos muy estructurados.
2. **Clave-valor** (Redis): lectura y escritura por clave, con expiración automática.
3. **Grafos** (Neo4j): recorrer relaciones de varios niveles es su fuerte.
4. **Documental** (MongoDB) es una opción natural. También se puede resolver en relacional con columnas JSON o con un buen modelado de subtipos (ver [Patrones de modelado](../02-diseno-y-modelado/09-Patrones%20de%20modelado.md)).

</details>

-----

## Siguiente lección

[El modelo relacional](02-El%20modelo%20relacional.md)
