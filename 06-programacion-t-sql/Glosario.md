# Glosario: Programación en T-SQL

Términos del módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**`AFTER` (trigger):** *trigger* que se ejecuta después de la modificación, cuando ya se aplicaron las restricciones. ([04](04-Triggers.md))

**Código de retorno:** entero que devuelve un procedimiento con `RETURN`; por convención, 0 si terminó bien. ([02](02-Procedimientos%20almacenados.md))

**Compilación diferida de variables de tabla:** mejora de SQL Server 2019 que compila la consulta con la cantidad real de filas de la variable. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**Cursor:** mecanismo para recorrer las filas de un resultado una por una. ([06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md))

**Encadenamiento de propiedad (ownership chaining):** si una vista o procedimiento y sus tablas tienen el mismo dueño, basta el permiso sobre la vista o el procedimiento. ([01](01-Vistas.md))

**`EXECUTE AS`:** cláusula que ejecuta un procedimiento con los permisos de otro usuario. ([02](02-Procedimientos%20almacenados.md))

**Expansión de la vista:** reemplazo de la vista por su definición dentro de la consulta, antes de optimizar. ([01](01-Vistas.md))

**`FAST_FORWARD`:** cursor de solo lectura y solo hacia adelante, el más eficiente. ([06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md))

**`@@FETCH_STATUS`:** resultado del último `FETCH`: 0 si leyó una fila, -1 si no hay más. ([06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md))

**Función de tabla en línea (inline TVF):** función cuyo cuerpo es una sola consulta; se expande como una vista con parámetros. ([03](03-Funciones.md))

**Función de tabla de varias instrucciones:** función que llena una variable de tabla con varias instrucciones; caja negra para el optimizador. ([03](03-Funciones.md))

**Función determinista:** devuelve siempre el mismo resultado para los mismos parámetros. ([03](03-Funciones.md))

**Función escalar:** función que devuelve un solo valor. ([03](03-Funciones.md))

**Identificador:** nombre de un objeto (tabla, columna, esquema); en SQL dinámico no se puede parametrizar. ([07](07-SQL%20dinámico%20seguro.md))

**Inlining:** reemplazar la llamada a una función por su definición dentro de la consulta; para funciones escalares, desde SQL Server 2019. ([03](03-Funciones.md))

**Inyección de SQL (SQL injection):** ataque en el que un texto del usuario se ejecuta como código SQL. ([07](07-SQL%20dinámico%20seguro.md))

**`inserted` / `deleted`:** tablas virtuales de un *trigger* con las filas nuevas y las anteriores. ([04](04-Triggers.md))

**`INSTEAD OF` (trigger):** *trigger* que se ejecuta en lugar de la modificación. ([04](04-Triggers.md))

**Lista blanca (allowlist):** conjunto cerrado de valores permitidos contra el que se valida una entrada. ([07](07-SQL%20dinámico%20seguro.md))

**Operación sobre conjuntos (set-based):** instrucción que procesa todas las filas de una vez. ([06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md))

**Parámetro con valor de tabla (TVP):** parámetro cuyo valor es una tabla de un tipo creado con `CREATE TYPE ... AS TABLE`; debe ser `READONLY`. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**Parámetro `OUTPUT`:** parámetro con el que un procedimiento devuelve un valor en una variable del llamador. ([02](02-Procedimientos%20almacenados.md))

**Procedimiento almacenado:** conjunto de instrucciones T-SQL guardado con nombre, que se ejecuta con `EXEC`. ([02](02-Procedimientos%20almacenados.md))

**`QUOTENAME`:** función que encierra un identificador entre corchetes y escapa los corchetes internos; devuelve `NULL` con más de 128 caracteres. ([07](07-SQL%20dinámico%20seguro.md))

**RBAR (*Row By Agonizing Row*):** procesamiento fila por fila, con cursores o bucles. ([06](06-Cursores%20frente%20a%20operaciones%20sobre%20conjuntos.md))

**`SCHEMABINDING`:** opción de vistas y funciones que impide modificar los objetos de los que dependen. ([01](01-Vistas.md))

**`SET NOCOUNT ON`:** desactiva los mensajes "(n rows affected)". ([02](02-Procedimientos%20almacenados.md))

**`sp_executesql`:** procedimiento del sistema que ejecuta SQL dinámico con parámetros. ([07](07-SQL%20dinámico%20seguro.md))

**SQL dinámico:** consulta construida como texto y ejecutada en tiempo de ejecución. ([07](07-SQL%20dinámico%20seguro.md))

**Tabla temporal global (`##Tabla`):** tabla temporal visible para todas las sesiones. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**Tabla temporal local (`#Tabla`):** tabla visible para la sesión y los procedimientos que llama; tiene estadísticas. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**`tempdb`:** base del sistema donde viven los objetos temporales y las versiones de filas; la comparte todo el servidor. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**Trigger (disparador):** código que se ejecuta automáticamente ante una modificación, dentro de la misma transacción. ([04](04-Triggers.md))

**Trigger DDL:** *trigger* que se ejecuta ante cambios de estructura (`CREATE`, `ALTER`, `DROP`). ([04](04-Triggers.md))

**Variable de tabla (`@Tabla`):** variable con forma de tabla, visible solo en su lote; sin estadísticas y no afectada por el `ROLLBACK`. ([05](05-Tablas%20temporales%20y%20variables%20de%20tabla.md))

**Vista:** consulta con nombre que se usa como una tabla. ([01](01-Vistas.md))

**Vista actualizable:** vista a través de la cual se pueden modificar datos de una tabla base. ([01](01-Vistas.md))

**`WITH CHECK OPTION`:** impide modificar a través de una vista filas que quedarían fuera de ella. ([01](01-Vistas.md))
