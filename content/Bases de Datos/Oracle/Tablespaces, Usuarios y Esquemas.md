---
tags:
  - DB
---
### Tablespaces: El Almacenamiento Lógico

En Oracle, los datos se almacenan físicamente en archivos de tu disco duro llamados **Datafiles**. Sin embargo, tú como administrador o desarrollador interactúas con una capa lógica llamada **Tablespace**.

Un Tablespace agrupa uno o más Datafiles. Es decir, tú le dices a Oracle "guarda esto en el Tablespace X", y Oracle se encarga de escribirlo en los archivos físicos asociados.

**Ejemplo de creación de un Tablespace:** Para crear uno, necesitas privilegios de administrador (`SYSDBA`).

``` sql
CREATE TABLESPACE curso_tbs
DATAFILE '/opt/oracle/oradata/curso_tbs01.dbf' SIZE 100M 
AUTOEXTEND ON NEXT 10M MAXSIZE 500M;
```

- `SIZE 100M`: El tamaño inicial del archivo será de 100 Megabytes.
- `AUTOEXTEND ON`: Si el archivo se llena, crecerá automáticamente.
- `NEXT 10M`: Crecerá en bloques de 10 Megabytes.

### Usuarios y Esquemas (Schemas)

En Oracle, **Usuario y Esquema son prácticamente sinónimos**, pero tienen un matiz conceptual importante:

- **Usuario (User):** Es la cuenta de seguridad con la que te autenticas (login y password).
- **Esquema (Schema):** Es la colección lógica de objetos (tablas, vistas, procedimientos, índices) que le pertenecen a ese usuario.

Cuando creas el usuario `SAIF_USER`, Oracle automáticamente crea un esquema vacío llamado `SAIF_USER`. Si eliminas al usuario, eliminas su esquema y todos los datos que contenía.

**Ejemplo de creación de un Usuario:** Aquí es donde unimos al usuario con el Tablespace que acabamos de crear.

```SQL
CREATE USER estudiante_oracle 
IDENTIFIED BY "PasswordSeguro123"
DEFAULT TABLESPACE curso_tbs
TEMPORARY TABLESPACE temp;
```

- `IDENTIFIED BY`: Define la contraseña.
- `DEFAULT TABLESPACE`: Le dice a Oracle que, si el usuario crea una tabla y no especifica dónde guardarla, la meta por defecto en `curso_tbs`.
- `TEMPORARY TABLESPACE`: Un espacio temporal que Oracle usa para operaciones de ordenamiento (sorts, joins complejos). Por lo general se usa el predeterminado llamado `temp`.

### Permisos y Cuotas

#### Conexión
Si intentas conectarte con el usuario recién creado, Oracle te dará un error. ¿Por qué? Porque un usuario nuevo no tiene permiso ni siquiera para iniciar sesión. Debemos darle privilegios (Roles) y una cuota de espacio en su Tablespace.

```SQL
-- Dar permiso para conectarse a la base de datos (crear sesión)
-- y permisos básicos para crear tablas y objetos (resource)
GRANT CONNECT, RESOURCE TO estudiante_oracle;

-- Darle permiso para usar espacio ilimitado en su tablespace
ALTER USER estudiante_oracle QUOTA UNLIMITED ON curso_tbs;
```

#### Permisos Objetos y Roles
Para lograr que un usuario acceda a objetos de otros esquemas, utilizamos el sistema de **Privilegios** de Oracle.

Por defecto, la base de datos es muy restrictiva. Cada esquema funciona como una casa privada; ningún otro usuario puede entrar a ver o modificar los datos a menos que el dueño de la casa (o el administrador general) le entregue una llave explícita.

Existen dos maneras de entregar estas "llaves"

**Privilegios de Objeto (La forma específica)**

Le otorgas permiso al usuario para interactuar con tablas o vistas puntuales de otro esquema.

Imagina que existe un segundo esquema llamado `profesor_admin` que contiene una tabla llamada `calificaciones`. Para que nuestro `estudiante_oracle` pueda consultar esa tabla, se debe ejecutar la siguiente instrucción (ya sea por el administrador o por el usuario `profesor_admin`):

```sql
GRANT SELECT ON profesor_admin.calificaciones TO estudiante_oracle;
```

Si también necesitaras que el estudiante pueda agregar o modificar datos en esa tabla ajena, simplemente sumas los permisos:

```sql
GRANT SELECT, INSERT, UPDATE ON profesor_admin.calificaciones TO estudiante_oracle;
```

**Privilegios del Sistema (La forma global)**

Si el usuario es un auditor o un servicio de integración (como cuando conectamos MuleSoft para extraer datos masivos) y necesita leer absolutamente _todas_ las tablas de _todos_ los esquemas, se utiliza un privilegio global. Se debe usar con mucha precaución por motivos de seguridad:

```sql
GRANT SELECT ANY TABLE TO estudiante_oracle;
```

Pensando en un escenario empresarial real: si el esquema `profesor_admin` tuviera 100 tablas diferentes a las que el estudiante necesita acceso, escribir la instrucción `GRANT SELECT ON...` 100 veces sería muy ineficiente. Para solucionar esto usamos los *Roles*.

Un Rol es básicamente una agrupación lógica de privilegios. En lugar de asignar permisos tabla por tabla a cada usuario, el proceso se simplifica así:

1. **Creas el rol:** `CREATE ROLE rol_lectura_profesor;`
2. **Asignas los permisos al rol:** `GRANT SELECT ON profesor_admin.calificaciones TO rol_lectura_profesor;` (y repites esto para las otras 99 tablas).
3. **Asignas el rol al usuario:** `GRANT rol_lectura_profesor TO estudiante_oracle;`

Si el día de mañana ingresan 50 estudiantes nuevos, ya no tienes que ejecutar miles de líneas de código; simplemente le otorgas el `rol_lectura_profesor` a cada uno.

Con esto, nuestra base de datos está completamente configurada a nivel administrativo y de desarrollo. Tenemos el almacenamiento (Tablespace), el propietario (Esquema/Usuario), la estructura (Tabla `alumnos`), la información guardada (`COMMIT`) y el modelo de seguridad (Roles y Privilegios).
### Resumen
En los esquemas se crean los objetos para la base de datos. Los usuarios se comportan como esquemas ya que al crear un usuario se crea un esquema con su nombre. 

Mientras que los tablespaces organizan los datos a nivel físico. Sirven para administrar el espacio en disco, mejorar el rendimiento (separando datos de uso frecuente de los que casi no se leen) y facilitar los respaldos.

- Un usuario (Esquema) puede tener su tabla de `alumnos` construida en el Tablespace `curso_tbs`, pero podría tener otra tabla histórica muy pesada en un Tablespace completamente distinto.

- Y a la inversa, un mismo Tablespace puede almacenar tablas que le pertenecen a varios esquemas diferentes.
