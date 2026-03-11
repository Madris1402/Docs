---
tags:
  - GoldenGate
---
### Tabla a Tabla
Para configurar un entorno con replicaciones tabla a tabla en el [[Contenedor Oracle Golden Gate]] que generamos anteriormente primero necesitamos crear un usuario que tenga los permisos suficientes para acceder a [[Bases de Datos Multitenant#Pluggable Data Base (PDB)|PDBs]], recursos del sistema, esquemas de la Base de Datos, etc.

#### Preparación de la Base de Datos 19c
	Como empezamos con una base de datos vacía necesitamos preparar todo:
```powershell
docker exec -it goldengate_odb bash
```
- Nos conectamos al contenedor de la base de datos.
```bash
sqlplus / as sysdba
```
- Entramos como *DBA* a SQL.

##### Habilitar la replicación a nivel motor
En 19c, la base de datos debe estar configurada para guardar el historial de transacciones y permitir a GoldenGate extraerlas:
	
```sql
ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION=true SCOPE=BOTH;
```
- Habilitamos el parámetro de GoldenGate
```sql
ARCHIVE LOG LIST;
```
- Nos Aseguramos que la base está en modo *Archive Log*.
	
> [!tip] Como Activar el modo `ARCHIVELOG`
> 
> - Con `SHUTDOWN IMMEDIATE;` Apagamos la Base de Datos.
> - Con `STARTUP MOUNT;` La Iniciamos en el modo `Mount`.
> - Usamos `ALTER DATABASE ARCHIVELOG;` para activar el modo y `ALTER DATABASE OPEN;` para volver a abrirla.
> - Ejecutamos nuevamente `ARCHIVE LOG LIST` para verificar que se haya activado.
	
```sql
ALTER DATABASE FORCE LOGGING;
```
- Este comando sirve para que todas las transacciones sean registradas en el *Redo Log*.
	
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```
- Este comando añade registros suplementarios. Cuando se realice un `UPDATE` o `DELETE`, se almacenarán las llaves primarias en los *Redo Logs* para que la reconstrucción de queries en el destino tenga el contexto completo.
	
##### Crear los Usuarios para GoldenGate
Como 19c usa contenedores, este usuario debe ser "común" (empezar con `c##`) y tener permisos en todos los contenedores.

```sql
CREATE TABLESPACE goldengate DATAFILE '/opt/oracle/oradata/ORCL/goldengate01.dbf' SIZE 100M AUTOEXTEND ON;
```
- Creamos el *tablespace*.
	
```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;
```
- Cambaimos a la PDB
	
```sql
CREATE TABLESPACE goldengate DATAFILE '/opt/oracle/oradata/ORCL/ORCLPDB1_goldengate01.dbf' SIZE 100M AUTOEXTEND ON;
```
- Volvemos a crear el tablespace <mark style="background:#fdbfff">Pero le cambiamos el nombre al archivo para evitar que sobreescriva el de CDB$ROOT</mark>.
	
```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
```
- Regresamos a `CDB$ROOT` que es desde donde se deben crear los usuarios c##
	
```sql
CREATE USER c##ggadmin IDENTIFIED BY ggadmin123 DEFAULT TABLESPACE goldengate CONTAINER=ALL;
```
- Creamos el usuario.
	
```sql
GRANT CREATE SESSION, CONNECT, RESOURCE, ALTER ANY TABLE, ALTER SYSTEM, SET CONTAINER, SELECT ANY DICTIONARY, DBA TO c##ggadmin CONTAINER=ALL;
```
- Otorgamos los permisos elementales.
	
	Posteriormente utilizaremos el paquete `DBMS_GOLDENGATE_AUTH`. Que agrupa cientos de permisos complejos en tres perfiles muy específicos.
```sql
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('c##ggadmin', container=>'all');
```
- Este perfil otorga al usuario la capacidad de consultar libremente las tablas de diccionario del sistema (donde Oracle guarda la metadata de qué tablas existen, quién es el dueño, qué columnas tienen). También le permite ejecutar comandos de administración básicos de la base de datos que GoldenGate necesita para inicializarse.
	
```sql
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('c##ggadmin', 'capture', container=>'all');
```
-  Este perfil da acceso directo al _LogMiner_ interno de Oracle. *LogMiner* es el motor que sabe cómo traducir los ceros y unos de los _Redo Logs_ a sentencias SQL legibles.
	
```sql
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('c##ggadmin', 'apply', container=>'all');
```
- Este perfil da al usuario la capacidad de "inyectar" datos en tablas que no son suyas, evadiendo ciertos triggers (disparadores) que normalmente se activarían si un usuario común hiciera un `INSERT`. También le da permisos de bloquear tablas y deshabilitar constraints temporalmente si la replicación lo exige.
	
##### Importar `hr_schema` de *Oracle Samples*
Regresamos a la terminal de windows, ahora descargamos los [db-sample-schemas](https://github.com/oracle-samples/db-sample-schemas/archive/refs/heads/main.zip) del [GitHub de Oracle](https://github.com/oracle-samples/db-sample-schemas/).

Una vez descargado, lo copiaremos a docker:

```powershell
docker cp "C:\{Ruta compleata a la carpeta de Samples}\db-sample-schemas-main\human_resources" goldengate_odb:/tmp/
```
- Una vez copiado, entramos a la terminal del contenedor.
	
```bash
chmod -R 777 /tmp/human_resources
```
- Cambiamos los permisos de la carpeta para que Oracle pueda interactuar con ella.
	
```bash
cd /tmp
mkdir hr_oracle
cd hr_oracle

cp /tmp/human_resources/* .
```
- Ahora, para asegurar que todo funcione, crearemos una carpeta más y copiaremos a ella los contenidos de la carpeta original. (Esto se debe a que `docker cp` crea las carpetas como `root` y al entrar a `sqlplus` los permisos recaen en el usuario `oracle` del contenedor).
	
```bash
sqlplus / as sysdba
```
-  Entramos como `sysdba`.
	
```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;
```
- Entramos a la PDB <mark style="background:#fdbfff">Importante hacer esto</mark>.
	
```bash
@hr_install.sql
```
- Ejecutamos el Script de Instalación
	- Nos pedirá datos
	  `Enter a password for the user HR` = `hr`
	  `Enter a tablespace for HR [USERS]:` = `users`
	  `Do you want to overwrite the schema, if it already exists? [YES|no]:`= `y`
	- Ya que terminemos el esquema debería generarse.
	- Volvemos a iniciar sesión como `sysdba` y cambiamos el contenedor al de la PDB.
```sql
ALTER USER hr QUOTA UNLIMITED ON USERS;
```
- Con esto le damos acceso al usuario `hr` al espacio.
	
```sql
CREATE TABLE hr.employees_clone AS SELECT * FROM hr.employees WHERE 1=2;
```
-  Creamos la tabla clon idéntica a la original (Al usar `AS SELECT *`, la tabla clon nacerá con todos los registros actuales. Para que se copie vacía usamos `WHERE 1=2`) (Al usar una condición imposible los datos son omitidos, pero la estructura de la tabla es generada).
```sql
ALTER TABLE hr.employees_clone ADD CONSTRAINT pk_emp_clone PRIMARY KEY (employee_id);
```
- Especificamos la llave primaria a la tabla clon para que no haya problemas con las replicaciones.
#### Configurar GoldenGate
Ahora configuraremos *GoldenGate*, para ello, entramos al contenedor:

```powershell
docker exec -it goldengate ggsci
```
- Entramos directo a la consola de `ggsci`.
	
```Goldengate
CREATE SUBDIRS
```
- Creamos la estructura de carpetas de GoldenGate.
	
```GoldenGate
DBLOGIN USERID c##ggadmin@//goldengate_odb:1521/ORCL PASSWORD ggadmin123
```
- Iniciamos sesión con el usuario `ggadmin`.
	
```GoldenGate
ADD SCHEMATRANDATA ORCLPDB1.hr
```
-  Habilitar el Monitoreo del Esquema (`SCHEMATRANDATA`)
	  Antes de capturar, debemos decirle a GoldenGate explícitamente qué esquema vamos a vigilar y forzar a Oracle a que registre toda la información necesaria para ese esquema específico.
##### Registrar y Crear el EXTRACT
Ahora crearemos el proceso `EEMP`. Usaremos **Integrated Capture**. Esto significa que GoldenGate trabajará íntimamente con el motor de LogMiner de Oracle.
```GoldenGate
REGISTER EXTRACT EEMP DATABASE CONTAINER (ORCLPDB1)
```
- Registramos el *Extract*, esto generará un diccionario así que puede tardar un poco.
```GoldenGate
ADD EXTRACT EEMP, INTEGRATED TRANLOG, BEGIN NOW
```
- Creamos el proceso para que empiece a capturar las transacciones.
```GoldenGate
EDIT PARAMS EEMP
```
- Ahora editaremos los parámetros del *Extract*.
	- Una vez dentro, editamos el archivo:
```vim
EXTRACT EEMP
USERID c##ggadmin@//goldengate_odb:1521/ORCL PASSWORD ggadmin123
EXTTRAIL ./dirdat/ex
SOURCECATALOG ORCLPDB1
TABLE hr.employees;
```
- Guardamos los cambios y salimos
	
```Goldengate
ADD EXTTRAIL ./dirdat/ex, EXTRACT EEMP
```
- Añadimos el *Extract* a `EXTTRAIL`.
	
```GoldenGate
EDIT PARAMS mgr
```
- Ahora editaremos los parámetros del Manager.
	- Una vez dentro, especificamos el puerto como: `PORT 7089`.
```GoldenGate
START MGR
```
- Iniciamos el *Manager*.
```GoldenGate
START EXTRACT EEMP
```
- Iniciamos el extracto.
```GoldenGate
INFO ALL
```
- Nos aseguramos que el *Extract* esté inicializado.
	- Si vemos que dice `RUNNING` todo está en orden.
		- Para asegurarnos que está todo en orden podemos ingresar el comando `VIEW REPORT EMMP` (o el nombre del elemento del que queremos ver su reporte).
		  
```sql
INSERT INTO hr.employees (
    employee_id, first_name, last_name, email, phone_number,
    hire_date, job_id, salary, commission_pct, manager_id, department_id
) VALUES (
    999, 'Juan', 'Perez', 'juanpe@mail.com', '5551234567',
    SYSDATE, 'IT_PROG', 6000, NULL, 100, 60
);
COMMIT;
```
- Ahora creamos un registro en la Base, asegurando estar en `ORCLPDB1` dónde existe el esquema.
	- Entramos al esquema con el usuario `HR`.
	  `sqlplus HR/hr@//localhost:1521/ORCLPDB1`
```sql
UPDATE hr.employees SET salary = salary + 100 WHERE employee_id = 999;
COMMIT;
```
- Actualizamos el registro
	
```GoldenGate
STATS EEMP
```
- Regresamos a GoldenGate y vemos las estadísticas del *Extract*. Deberíamos ver el Insert, Update y Delete.

##### Crear y Configurar el REPLICAT
A diferencia del Extract, que tuvo que conectarse a la raíz (`CDB$ROOT`) para leer los diarios globales del sistema, el Replicat es como un usuario normal pero súper rápido: necesita conectarse directamente a la PDB donde viven las tablas para poder inyectar los datos.

```GoldenGate
DBLOGIN USERID c##ggadmin@//goldengate_odb:1521/ORCLPDB1, PASSWORD ggadmin123
```

Posteriormente Crearemos una Tabla *Checkpoint* para indicar al Replicat en dónde se quedó si es que ocurre un reinicio o desconexión.

```GoldenGate
ADD CHECKPOINTTABLE ORCLPDB1.c##ggadmin.gg_checkpoint
```

Ya que hayamos configurado esto, creamos el *Replicat*

```GoldenGate
ADD REPLICAT REMP, INTEGRATED, EXTTRAIL ./dirdat/ex, CHECKPOINTTABLE ORCLPDB1.c##ggadmin.gg_checkpoint
```
- Aquí especificamos la ruta para los *Trail Files* y la tabla *Checkpoint*.

Después configuraremos los parámetros del *Replicat*.

```GoldenGate
EDIT PARAMS REMP
```

```vim
REPLICAT REMP
USERID c##ggadmin@//goldengate_odb:1521/ORCLPDB1, PASSWORD ggadmin123
ASSUMETARGETDEFS
MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone;
```
- `ASSUMETARGETDEFS`: Indica a *GoldenGate* que no compruebe si las columnas coinciden, porque nosotros sabemos que la tabla clon es estructuralmente idéntica a la original.
- `MAP ... TARGET ...`: Es la ruta exacta. Todo lo que venga de `employees` va hacia `employees_clone`.

Finalmente Iniciamos el Replicat y comprobamos que funcione:
```GoldenGate
START REPLICAT REMP
```

```GoldenGate
INFO ALL
```

```GoldenGate
VIEW REPORT REMP
```

Verificamos que los registros que creamos antes se vean reflejados
```sql
SELECT * FROM hr.employees_clone WHERE employee_id = 999;
```

```sql
DELETE hr.employees WHERE employee_id = 999;
COMMIT;
```
- Y eliminamos el registro de prueba en la tabla original. Debería eliminarse en la tabla clon.

### Base a Base
Si seguimos los pasos anteriores tendremos un Contenedor con el esquema *Human Resources* de *Oracle Samples*. Ahora crearemos un nuevo contenedor de Base de Datos:

```powershell
docker run -d --name goldengate_odb2 --network ogg-net -p 1522:1521 -e ORACLE_SID=orcl -e ORACLE_PDB=orclpdb1 -e ORACLE_PWD=Oracle123 -e ORACLE_EDITION=enterprise -v oracle_ee_data2:/opt/oracle/oradata container-registry.oracle.com/database/enterprise:19.3.0.0
```

Ya que generemos el contenedor configuraremos la base:
1. Iniciamos el modo *[[#Habilitar la replicación a nivel motor|Archivelog]]*
2. Creamos el usuario `c##gadmin` con los [[#Crear los Usuarios para GoldenGate|permisos]] para `apply`.
3. [[#Importar `hr_schema` de *Oracle Samples*|Instalar el esquema]] `HR` y la tabla vacía `hr.employees_clone` configurada con la *Llave Primaria*.

#### Configurar el Extract en la base Origen (`goldengate_odb`)

Iniciamos sesión en la Base de Destino:
```GoldenGate
DBLOGIN USERID c##ggadmin@//goldengate_odb:1521/ORCL PASSWORD ggadmin123
```

Añadimos el Esquema `hr` al monitoreo Este paso habilita el _Supplemental Logging_ a nivel de esquema, obligando a Oracle a registrar la información de las llaves primarias en los logs:
```GoldenGate
ADD SCHEMATRANDATA ORCLPDB1.hr
```

Registramos y configuramos el Extract. Aquí le indicamos que capture los datos, genere un archivo local (Trail File) y le especificamos la tabla exacta que debe vigilar:
```GoldenGate
REGISTER EXTRACT EEMP_ORG DATABASE CONTAINER (ORCLPDB1)
```

```GoldenGate
ADD EXTRACT EEMP_ORG, INTEGRATED TRANLOG, BEGIN NOW
```

```GoldenGate
ADD EXTTRAIL ./dirdat/origen/ex, EXTRACT EEMP_ORG
```

```GoldenGate
EDIT PARAMS EEMP_ORG
```

```vim
EXTRACT EEMP_ORG
USERID c##ggadmin@//goldengate_odb:1521/ORCL PASSWORD ggadmin123
EXTTRAIL ./dirdat/origen/ex
SOURCECATALOG ORCLPDB1
TABLE hr.employees;
```

```GoldenGate
START EXTRACT EEMP_ORG
```

#### Generar el Replicat en la base destino (`goldengate_odb2`):

Conectamos a la PDB del destino:
```GoldenGate
DBLOGIN USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD ggadmin123
```

Creamos una Tabla Checkpoint. Esta tabla es crucial: permite que el Replicat guarde su progreso. Si el contenedor se apaga o falla, GoldenGate sabrá exactamente en qué transacción se quedó y evitará duplicar datos al reiniciar:
```GoldenGate
ADD CHECKPOINTTABLE ORCLPDB1.c##ggadmin.gg_chkpt
```

egistramos y configuramos el Replicat mapeando la tabla origen (`employees`) hacia nuestra tabla destino (`employees_clone`):
```GoldenGate
ADD REPLICAT REMP_DES, EXTTRAIL ./dirdat/origen/ex, CHECKPOINTTABLE ORCLPDB1.c##ggadmin.gg_chkpt
```

```GoldenGate
EDIT PARAMS REMP_DES
```

```vim
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD ggadmin123
ASSUMETARGETDEFS
MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone;
```

```GoldenGate
START REPLICAT REMP_DES
```

Para comprobar que nuestra tubería de datos funciona, simularemos actividad. Finalmente agregamos registros en `goldengate_odb` (Origen):

```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;
```

```sql
INSERT INTO hr.employees (
    employee_id, first_name, last_name, email, phone_number,
    hire_date, job_id, salary, commission_pct, manager_id, department_id
) VALUES (
    999, 'Juan', 'Perez', 'juanpe@mail.com', '5551234567',
    SYSDATE, 'IT_PROG', 6000, NULL, 100, 60
);
COMMIT;
```

```sql
UPDATE hr.employees SET salary = salary + 100 WHERE employee_id = 999;
COMMIT;
```

Y los revisamos en `goldengate_odb2` (Destino) para confirmar que los cambios viajaron exitosamente:

```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;
```

```sql
SELECT * FROM hr.employees_clone WHERE employee_id = 999;
```

Ya que terminemos de validar, borramos el registro de pruebas de la base origen (esto también debería replicarse y borrar el registro en el destino):
```sql
DELETE hr.employees WHERE employee_id = 999;
COMMIT;
```


#### Añadir Control de Errores al Replicat (Archivos Discard)
Por defecto, si GoldenGate encuentra un error (como intentar insertar una llave primaria que ya existe), el proceso Replicat se detendrá (Abend). Para evitar que la replicación entera se frene, configuraremos el manejo de errores.

Detenemos el proceso y editamos sus parámetros:
```GoldenGate
STOP REPLICAT REMP_DES
EDIT PARAMS REMP_DES
```

```GoldenGate
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD ggadmin123
ASSUMETARGETDEFS

DISCARDFILE ./dirrpt/remp_des.dsc, APPEND, MEGABYTES 10
REPERROR (1, DISCARD)
REPERROR (1403, DISCARD)

MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone;
```
- `DISCARDFILE`: Definimos dónde guardar la basura (máximo 10MB y que no se sobreescriba)
- Le decimos qué errores ignorar y tirar al archivo de descarte
	- `Error 1: ORA-00001` (Unique constraint/Primary Key violation)
	-  `Error 1403: ORA-01403` (No data found)

Iniciamos y revisamos el estatus:
```GoldenGate
START REPLICAT REMP_DES
INFO REPLICAT REMP_DES
```

Para probarlo, simularemos una colisión. Insertamos un registro "intruso" manualmente directo en `goldengate_odb2` (Destino):
```SQL
ALTER SESSION SET CONTAINER = ORCLPDB1;

INSERT INTO hr.employees_clone (employee_id, first_name, last_name, email, hire_date, job_id) 
VALUES (889, 'Intruso', 'Falso', 'FAKE', SYSDATE, 'IT_PROG');

COMMIT;
```

Posteriormente, entramos a `goldengate_odb` (Origen) e insertamos el mismo dato. Esto causará un error ORA-00001 en GoldenGate al intentar replicarlo:
```sql
INSERT INTO hr.employees (employee_id, first_name, last_name, email, hire_date, job_id) 
VALUES (889, 'Intruso', 'Falso', 'FAKE', SYSDATE, 'IT_PROG');

COMMIT;
```

Verificamos que el replicat siga vivo:
```GoldenGate
INFO REPLICAT REMP_DES
```

Y leemos el archivo de descartes directo desde el bash del contenedor (fuera de ggsci) para ver el registro que falló:
```GoldenGate
cat ./dirrpt/rephub.dsc
```

#### Añadir Tabla de Errores
Leer archivos de texto para buscar errores no es práctico en producción. Una técnica más avanzada es enviar los registros problemáticos a una tabla de auditoría (Exceptions Table) que podemos consultar con SQL.

Primero, creamos la tabla en `odb2` (Destino):

```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;

CREATE TABLE hr.employees_exceptions (
    gg_error_msg VARCHAR2(4000),  -- Aquí guardaremos el ORA-XXXX
    gg_op_type VARCHAR2(20),      -- ¿Fue un INSERT, UPDATE o DELETE?
    gg_err_time TIMESTAMP         -- ¿A qué hora chocó?
);
```

En GoldenGate, editamos el Archivo de parámetros de la Replicación:

```Goldengate
STOP REPLICAT REMP_DES
```

```GoldenGate
EDIT PARAMS REMP_DES
```

```vim
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD ggadmin123
ASSUMETARGETDEFS

REPERROR (1, EXCEPTION)
REPERROR (1403, EXCEPTION)

MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone;
-- Mapeo Solo para errores
MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_exceptions,
EXCEPTIONSONLY,
COLMAP (
    gg_error_msg = @GETENV ('LASTERR', 'DBERRMSG'),
    gg_op_type = @GETENV ('LASTERR', 'OPTYPE'),
    gg_err_time = @GETENV ('GGHEADER', 'COMMITTIMESTAMP')
);
```

- `EXCEPTIONSONLY`: GoldenGate solo usará este mapa si el Mapeo 1 falla (falla la inyección).
- `@GETENV`: Extrae variables del entorno. `DBERRMSG` captura el texto exacto del error de Oracle.

Iniciamos el proceso:

```GoldenGate
START REPLICAT REP_HUB
```

E insertamos datos nuevamente para generar una excepción como vimos antes. Ahora los errores quedarán en la tabla de excepciones.
#### Integrar Macros
Ahora integraremos una *[[Macros de GoldenGate|Macro]]* que se encargue del control de errores:

Primero la generamos en un archivo independiente:
```GoldenGate
SH vi ./dirprm/error_handler.mac
```

Dentro de ella escribimos el comportamiento dinámico usando parámetros (`#origen`, `#destino`, `#errores`):
```vim
MACRO #error_handler
PARAMS (#origen, #destino, #errores)
BEGIN
	MAP #origen, TARGET #destino;
	
	MAP #origen, TARGET #errores,
	EXCEPTIONSONLY,
	INSERTALLRECORDS,
	COLMAP (
		gg_error_msg = @GETENV ('LASTERR', 'DBERRMSG'),
	    gg_op_type = @GETENV ('LASTERR', 'OPTYPE'),
	    gg_err_time = @GETENV ('GGHEADER', 'COMMITTIMESTAMP')
	);
END;
```
- La guardamos y salimos

Ahora editamos nuestro *Replicat* principal (`REMP_DES`) para que mande llamar (INCLUDE) a la macro, pasándole los nombres de nuestras tablas. Esto deja el archivo mucho más limpio:
```GoldenGate
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD ggadmin123
ASSUMETARGETDEFS

REPERROR (1, EXCEPTION)
REPERROR (1403, EXCEPTION)

INCLUDE ./dirprm/error_handler.mac

#error_handler(ORCLPDB1.hr.employees,ORCLPDB1.hr.employees_clone,ORCLPDB1.hr.employees_exceptions);
```
