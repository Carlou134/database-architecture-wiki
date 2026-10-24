# BCNF y formas superiores

## En una frase

La forma normal de Boyce-Codd (BCNF) cierra el hueco que deja la 3FN, exigiendo que **todo determinante sea una clave candidata**; la 4FN y la 5FN van más allá de las dependencias funcionales y eliminan la redundancia que aparece cuando una tabla mezcla hechos **independientes** entre sí.

-----

## Antes de empezar

Conviene que ya sepas:

* 1FN, 2FN y 3FN, de [Normalización 1FN 2FN 3FN](06-Normalización%201FN%202FN%203FN.md).
* Dependencias funcionales, cierre y claves candidatas, de [Dependencias funcionales](05-Dependencias%20funcionales.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **BCNF (forma normal de Boyce-Codd):** para toda dependencia no trivial `X → Y`, `X` es superclave.
* **Claves candidatas solapadas:** dos claves candidatas compuestas que comparten alguna columna.
* **Dependencia multivaluada (DMV):** `X ↠ Y`: cada valor de `X` determina un **conjunto** de valores de `Y`, independiente del resto de las columnas.
* **4FN:** BCNF + sin dependencias multivaluadas no triviales que no partan de una superclave.
* **Dependencia de reunión (join dependency):** la tabla se puede reconstruir exactamente uniendo varias proyecciones suyas.
* **5FN:** sin dependencias de reunión que no se deriven de las claves candidatas.

-----

## El problema

Un instituto registra qué profesor enseña cada materia a cada alumno. Las reglas son:

1. Un alumno cursa varias materias, y en cada materia tiene **un** profesor.
2. Cada profesor enseña **una sola** materia.
3. Una materia la pueden enseñar varios profesores.

**Tutoria**

| Alumno | Materia | Profesor |
| --- | --- | --- |
| Rosa | Matemática | Pérez |
| Rosa | Física | Gómez |
| Iván | Matemática | Pérez |
| Iván | Física | Rojas |
| Lucía | Matemática | Díaz |

Dependencias:

```text
{Alumno, Materia} → Profesor     (regla 1)
Profesor          → Materia      (regla 2)
```

Claves candidatas: `{Alumno, Materia}` y `{Alumno, Profesor}` (conociendo al profesor, se conoce la materia).

Verifiquemos la 3FN: los tres atributos forman parte de alguna clave candidata (son **primos**), así que no hay ningún atributo no clave que pueda depender de otro. **La tabla está en 3FN.** Y sin embargo:

* "Pérez enseña Matemática" está guardado **dos veces** (filas de Rosa e Iván).
* Si Pérez pasa a enseñar Estadística, hay que actualizar varias filas.
* No se puede registrar que el nuevo profesor Castro enseña Química hasta que tenga un alumno.

La 3FN no detectó la redundancia porque el determinante `Profesor` no es clave, pero el atributo que determina (`Materia`) es primo, y la 3FN lo permite. La BCNF existe para este caso.

-----

## Cómo funciona

### BCNF: todo determinante es una clave

Una tabla está en BCNF cuando, para **toda** dependencia funcional no trivial `X → Y`, **`X` es una superclave**.

Comparada con la 3FN:

| | 3FN | BCNF |
| --- | --- | --- |
| Condición para `X → Y` | `X` es superclave **o** `Y` es primo | `X` es superclave (sin excepciones) |
| ¿La permite `Profesor → Materia` en `Tutoria`? | Sí, porque `Materia` es primo | No, porque `Profesor` no es superclave |

En `Tutoria`, `Profesor → Materia` viola la BCNF. Se descompone igual que siempre: la dependencia problemática va a su propia tabla, con su determinante como clave.

**ProfesorMateria** — clave: `Profesor`

| Profesor | Materia |
| --- | --- |
| Pérez | Matemática |
| Gómez | Física |
| Rojas | Física |
| Díaz | Matemática |

**AlumnoProfesor** — clave: `{Alumno, Profesor}`

| Alumno | Profesor |
| --- | --- |
| Rosa | Pérez |
| Rosa | Gómez |
| Iván | Pérez |
| Iván | Rojas |
| Lucía | Díaz |

Ahora "Pérez enseña Matemática" está una sola vez, y Castro se puede registrar sin alumnos.

### El costo de la BCNF: dependencias que se pierden

La descomposición tiene un precio. La regla 1, `{Alumno, Materia} → Profesor` ("un alumno tiene un solo profesor por materia"), ya no se puede verificar dentro de una sola tabla: `Materia` está en una y `Alumno` en la otra. Nada impide esto:

| Alumno | Profesor |
| --- | --- |
| Rosa | Pérez |
| Rosa | **Díaz** |

Pérez y Díaz enseñan Matemática, así que Rosa tendría dos profesores de Matemática, y ninguna restricción de una sola tabla lo detecta.

Es un resultado conocido: **siempre existe una descomposición en 3FN que preserva todas las dependencias, pero no siempre existe una en BCNF**. La decisión práctica:

* **BCNF**, si la redundancia es el problema principal y la regla perdida se puede garantizar en la aplicación (o con una vista indexada o un *trigger*).
* **Quedarse en 3FN**, si la regla perdida es crítica y la redundancia es pequeña y controlada.

Este caso es poco frecuente: solo aparece con **claves candidatas compuestas que se solapan**. Si una tabla tiene una sola clave candidata, o claves de una sola columna, 3FN y BCNF son lo mismo.

### 4FN: hechos independientes en la misma tabla

La BCNF trata las dependencias **funcionales** (un valor determina un valor). La 4FN trata otro tipo de redundancia: la que aparece cuando una tabla mezcla dos listas **independientes** sobre la misma entidad.

Un profesor enseña varios cursos y habla varios idiomas. Los cursos no tienen nada que ver con los idiomas:

**ProfesorCursoIdioma** — clave: `{Profesor, Curso, Idioma}`

| Profesor | Curso | Idioma |
| --- | --- | --- |
| Pérez | Álgebra | Español |
| Pérez | Álgebra | Inglés |
| Pérez | Cálculo | Español |
| Pérez | Cálculo | Inglés |

La tabla está en BCNF (no hay dependencias funcionales no triviales: la clave son las tres columnas). Pero para registrar que Pérez habla francés hay que agregar **una fila por cada curso**. Si se agrega solo una, la tabla afirma que Pérez da Álgebra en francés pero no Cálculo, algo que nadie quiso decir.

Esto es una **dependencia multivaluada**: `Profesor ↠ Curso` y `Profesor ↠ Idioma`. Cada profesor determina un **conjunto** de cursos y un **conjunto** de idiomas, independientes entre sí. La tabla guarda el producto cartesiano de los dos conjuntos.

Una tabla está en **4FN** cuando está en BCNF y no tiene dependencias multivaluadas no triviales, salvo las que parten de una superclave. El arreglo es separar los hechos independientes:

```text
ProfesorCurso  (Profesor, Curso)    → Pérez-Álgebra, Pérez-Cálculo
ProfesorIdioma (Profesor, Idioma)   → Pérez-Español, Pérez-Inglés
```

Agregar el francés es ahora **una** fila. En la práctica, la 4FN casi nunca se viola si el diseño sale de un buen modelo ER: cursos e idiomas serían dos relaciones N:M distintas, cada una con su tabla. La violación típica aparece cuando alguien "ahorra" una tabla juntando dos relaciones en una.

### 5FN: cuando la tabla solo se puede reconstruir con tres partes

La 5FN (o forma normal de proyección-reunión) trata un caso todavía más raro: una tabla que **no** se puede dividir en dos sin perder información, pero sí en **tres**.

Ejemplo clásico, con la regla: "si un proveedor vende un tipo de producto, y ese tipo de producto se usa en un proyecto, y el proveedor abastece a ese proyecto, entonces el proveedor le vende ese tipo de producto a ese proyecto".

```text
ProveedorProyectoProducto(Proveedor, Proyecto, Producto)
  = ProveedorProducto ⋈ ProductoProyecto ⋈ ProveedorProyecto
```

Bajo esa regla, la tabla de tres columnas es redundante: se puede reconstruir uniendo las tres tablas de dos columnas. Sin esa regla (si un proveedor puede vender un producto que usa un proyecto al que abastece, pero no **ese** producto a **ese** proyecto), la tabla de tres columnas es la única forma de guardar el hecho, y está en 5FN.

La 5FN rara vez se aplica de forma explícita. Lo valioso es la idea: **una relación ternaria no siempre se descompone en binarias**, y decidirlo depende de una regla de negocio.

### El panorama completo

| Forma normal | Elimina | Frecuencia del problema en la práctica |
| --- | --- | --- |
| 1FN | Listas en celdas, grupos repetidos | Muy frecuente |
| 2FN | Dependencias parciales | Frecuente |
| 3FN | Dependencias transitivas | Frecuente |
| BCNF | Determinantes que no son clave (con claves solapadas) | Poco frecuente |
| 4FN | Hechos multivaluados independientes en la misma tabla | Rara |
| 5FN | Dependencias de reunión | Muy rara |

El objetivo habitual en OLTP es **3FN o BCNF**. Las formas superiores sirven para reconocer un diseño sospechoso cuando aparece.

-----

## Ejemplo completo

La tabla `Tutoria` llevada a BCNF en SQL Server, con la regla perdida protegida de otra forma:

```sql
CREATE TABLE dbo.ProfesorMateria (
    ProfesorId INT NOT NULL,
    MateriaId  INT NOT NULL,
    CONSTRAINT PK_ProfesorMateria PRIMARY KEY (ProfesorId),           -- un profesor, una materia
    CONSTRAINT UQ_ProfesorMateria UNIQUE (ProfesorId, MateriaId)        -- para poder referenciar el par
);

CREATE TABLE dbo.Tutoria (
    AlumnoId   INT NOT NULL,
    ProfesorId INT NOT NULL,
    MateriaId  INT NOT NULL,   -- copia controlada de la materia del profesor
    CONSTRAINT PK_Tutoria PRIMARY KEY (AlumnoId, ProfesorId),
    CONSTRAINT FK_Tutoria_ProfesorMateria FOREIGN KEY (ProfesorId, MateriaId)
        REFERENCES dbo.ProfesorMateria (ProfesorId, MateriaId),
    CONSTRAINT UQ_Tutoria_AlumnoMateria UNIQUE (AlumnoId, MateriaId)    -- regla 1: un profesor por materia
);
```

El truco: `Tutoria` guarda `MateriaId`, pero la clave foránea compuesta `(ProfesorId, MateriaId)` obliga a que sea **exactamente** la materia de ese profesor. La copia no puede desincronizarse, y el `UNIQUE (AlumnoId, MateriaId)` recupera la regla que la BCNF había perdido.

Probándolo con Pérez (1) y Díaz (4), los dos de Matemática (1):

```sql
INSERT INTO dbo.ProfesorMateria (ProfesorId, MateriaId) VALUES (1, 1), (4, 1);

INSERT INTO dbo.Tutoria (AlumnoId, ProfesorId, MateriaId) VALUES (10, 1, 1);   -- ✅ Rosa con Pérez
INSERT INTO dbo.Tutoria (AlumnoId, ProfesorId, MateriaId) VALUES (10, 4, 1);   -- ❌ UQ_Tutoria_AlumnoMateria
INSERT INTO dbo.Tutoria (AlumnoId, ProfesorId, MateriaId) VALUES (11, 1, 2);   -- ❌ FK: Pérez no enseña la materia 2
```

Qué cambió y por qué:

* **`ProfesorMateria` con `ProfesorId` como clave:** "un profesor enseña una materia" se guarda una vez (BCNF).
* **`MateriaId` en `Tutoria` con FK compuesta:** es una redundancia, pero **controlada por la base**: no puede contradecir a `ProfesorMateria`.
* **`UQ_Tutoria_AlumnoMateria`:** la regla que la descomposición en BCNF había perdido vuelve a ser una restricción declarativa.

Es un patrón útil en general: cuando desnormalizas, si la copia se protege con una FK compuesta, deja de ser un riesgo.

-----

## Errores comunes

**1. Asumir que 3FN y BCNF son siempre lo mismo.**
Qué pasa: una tabla en 3FN con claves candidatas solapadas sigue teniendo redundancia y anomalías.
Por qué: la 3FN permite determinantes no clave si el dependiente es primo.
Arreglo: si hay varias claves candidatas compuestas que comparten columnas, verifica la BCNF.

**2. Llevar a BCNF sin revisar qué dependencia se pierde.**
Qué pasa: una regla de negocio (un profesor por materia y alumno) deja de estar protegida, y aparecen datos inválidos.
Por qué: la descomposición en BCNF no siempre preserva las dependencias.
Arreglo: anota cada dependencia que quede repartida entre tablas y protégela de otra forma (FK compuesta, aplicación, *trigger*), o quédate en 3FN de forma consciente.

**3. Juntar dos relaciones N:M independientes en una tabla.**
Qué pasa: una tabla `ProfesorCursoIdioma` que obliga a guardar todas las combinaciones.
Por qué: se ahorra una tabla.
Arreglo: una tabla por cada relación independiente (4FN).

**4. Descomponer una relación ternaria en binarias sin la regla que lo justifique.**
Qué pasa: al unir las tablas binarias aparecen combinaciones que nunca existieron (tuplas espurias).
Por qué: se asume que tres hechos binarios equivalen a un hecho ternario.
Arreglo: descompón solo si una regla de negocio lo garantiza (5FN); si no, la tabla ternaria es correcta.

-----

## Según el motor

Las formas normales son teoría independiente del motor. Las diferencias están en cómo se protegen las dependencias que una descomposición en BCNF deja repartidas:

| Técnica | SQL Server | PostgreSQL | MySQL |
| --- | --- | --- | --- |
| FK compuesta hacia un `UNIQUE` | Sí | Sí | Sí |
| Restricción entre tablas mediante una vista | Vista indexada con índice único (con requisitos estrictos) | No hay vistas indexadas; vista materializada sin restricciones | No |
| *Trigger* de validación | Sí | Sí | Sí |
| Restricciones diferibles | No | Sí: permiten insertar las dos partes en cualquier orden dentro de una transacción | No |

### En MongoDB

No aplica de forma directa: MongoDB no tiene dependencias funcionales declaradas ni normalización. Lo que sí se traslada es la lección de la 4FN: dos listas independientes de un mismo documento (cursos e idiomas de un profesor) se guardan como **dos arreglos separados**, nunca como un arreglo de combinaciones.

-----

## Cuándo sí y cuándo no

**Verifica la BCNF cuando:**

* Una tabla tiene varias claves candidatas compuestas que comparten columnas. Es el único caso en que 3FN y BCNF difieren.

**Piensa en la 4FN cuando:**

* Una tabla tiene una clave formada por todas sus columnas, y dos de ellas son listas de cosas que no tienen relación entre sí.

**Ten en cuenta:**

* El objetivo práctico es 3FN o BCNF. Las formas superiores son herramientas para reconocer problemas, no metas que verificar en cada tabla.
* Si un diseño sale de un buen modelo ER, con una tabla por cada relación, las violaciones de 4FN y 5FN casi nunca aparecen.

-----

## Resumen en 5 líneas

1. BCNF: todo determinante es superclave; difiere de la 3FN solo con claves candidatas compuestas que se solapan.
2. La descomposición en BCNF puede perder dependencias: hay que protegerlas de otra forma o quedarse en 3FN a conciencia.
3. Una copia de un dato protegida con una FK compuesta es redundancia controlada: no puede desincronizarse.
4. 4FN: no juntes en una tabla dos listas independientes de la misma entidad (cursos e idiomas de un profesor).
5. 5FN: una relación ternaria solo se descompone en binarias si una regla de negocio lo garantiza.

-----

## Para profundizar

<details>
<summary>Definición formal de dependencia multivaluada</summary>

`X ↠ Y` se cumple en una tabla `R(X, Y, Z)` si, para cada par de filas con el mismo valor de `X`, también existen las filas que se obtienen intercambiando sus valores de `Y`. En `ProfesorCursoIdioma`: como existen (Pérez, Álgebra, Español) y (Pérez, Cálculo, Inglés), deben existir también (Pérez, Álgebra, Inglés) y (Pérez, Cálculo, Español). Toda dependencia funcional es un caso particular de dependencia multivaluada (el conjunto tiene un solo elemento), por eso la 4FN implica la BCNF.

</details>

<details>
<summary>Más allá de la 5FN</summary>

Existen la forma normal de dominio-clave (DKNF), propuesta por Fagin en 1981, en la que todas las restricciones son consecuencia de los dominios y las claves, y la 6FN, de C. J. Date, en la que cada tabla tiene como máximo un atributo además de la clave. La 6FN se usa en almacenes de datos temporales (*anchor modeling*), donde cada atributo cambia con su propio historial. En OLTP no tienen aplicación práctica habitual.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La BCNF es una versión más estricta de la 3FN: exige que todo lo que determina a otra columna sea una clave. La mayoría de las tablas en 3FN ya están en BCNF; la diferencia aparece en casos con varias claves compuestas. Las formas 4FN y 5FN existen, pero en la práctica se apunta a 3FN o BCNF.

### Respuesta ampliada (semi-senior)

La 3FN permite una dependencia `X → Y` con `X` no clave si `Y` es primo, y eso deja redundancia cuando hay claves candidatas compuestas solapadas, como en alumno-materia-profesor con "cada profesor enseña una materia". La BCNF exige que todo determinante sea superclave y elimina esa redundancia, pero puede perder dependencias: siempre existe una descomposición en 3FN que las preserva, y no siempre una en BCNF. Cuando lo hago, protejo la dependencia perdida con una copia controlada mediante una FK compuesta y un `UNIQUE`, o en la aplicación. La 4FN trata las dependencias multivaluadas, típicamente dos relaciones N:M independientes juntadas en una tabla, y la 5FN, las dependencias de reunión de las relaciones ternarias. En OLTP apunto a 3FN o BCNF, y uso las formas superiores para reconocer diseños sospechosos.

### Preguntas frecuentes de seguimiento

**1. ¿Cuándo una tabla en 3FN no está en BCNF?**
Cuando hay una dependencia cuyo determinante no es superclave, pero cuyo dependiente forma parte de una clave candidata. Requiere claves candidatas compuestas que se solapan.

**2. ¿Siempre conviene llevar a BCNF?**
No siempre: la descomposición puede perder una dependencia. Hay que decidir entre la redundancia de la 3FN y la regla repartida de la BCNF.

**3. ¿Qué es una dependencia multivaluada?**
Que un valor determine un conjunto de valores independiente del resto de las columnas, como los idiomas de un profesor frente a sus cursos.

**4. ¿Has aplicado la 5FN en un proyecto real?**
Respuesta honesta habitual: no de forma explícita. Lo útil es la idea de que una relación ternaria no siempre se descompone en binarias sin perder información.

-----

## Práctica

**Ejercicio 1.** Un hospital registra `Turno(Enfermera, Pabellon, Dia)`, con las reglas: una enfermera trabaja en un solo pabellón; un pabellón tiene varias enfermeras; cada día, en cada pabellón, hay una sola enfermera de turno. ¿Está en 3FN? ¿En BCNF? Si no, descomponla y di qué regla se pierde.

<details>
<summary>Solución</summary>

Dependencias:

```text
Enfermera       → Pabellon
{Pabellon, Dia} → Enfermera
```

Claves candidatas: `{Pabellon, Dia}` y `{Enfermera, Dia}` (conociendo a la enfermera se conoce el pabellón). Se solapan en `Dia`.

* **3FN: sí.** En `Enfermera → Pabellon`, `Pabellon` es primo (forma parte de `{Pabellon, Dia}`).
* **BCNF: no.** `Enfermera` no es superclave.

Descomposición en BCNF:

```text
EnfermeraPabellon (Enfermera PK, Pabellon)
TurnoEnfermera    (Enfermera, Dia)  PK (Enfermera, Dia)
```

Se pierde `{Pabellon, Dia} → Enfermera`: nada impide asignar el mismo día a dos enfermeras del mismo pabellón. Se recupera como en el ejemplo completo: agregando `Pabellon` a `TurnoEnfermera`, con una FK compuesta `(Enfermera, Pabellon)` hacia `EnfermeraPabellon` y un `UNIQUE (Pabellon, Dia)`.

</details>

**Ejercicio 2.** ¿Qué problema tiene esta tabla y cómo la arreglas? `EmpleadoHabilidadProyecto(Empleado, Habilidad, Proyecto)`, donde las habilidades de un empleado no tienen relación con los proyectos en los que trabaja.

<details>
<summary>Solución</summary>

Viola la **4FN**: `Empleado ↠ Habilidad` y `Empleado ↠ Proyecto` son dependencias multivaluadas independientes. Para registrar una habilidad nueva hay que agregar una fila por cada proyecto del empleado, y si se agrega una sola, la tabla afirma una combinación que no significa nada.

Arreglo: dos tablas, `EmpleadoHabilidad(Empleado, Habilidad)` y `EmpleadoProyecto(Empleado, Proyecto)`. Si en cambio la regla fuera "qué habilidad usa el empleado **en** cada proyecto", la tabla de tres columnas sería correcta, porque guardaría un hecho ternario real.

</details>

-----

## Siguiente lección

[Desnormalización](08-Desnormalización.md)
