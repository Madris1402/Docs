---
tags:
  - GoldenGate
---

### Tabla a Tabla
Para configurar un entorno con replicaciones tabla a tabla en el [[Contenedor Oracle Golden Gate]] que generamos anteriormente primero necesitamos crear un usuario que tenga los permisos suficientes para acceder a [[Bases de Datos Multitenant#Pluggable Data Base (PDB)|PDBs]], recursos del sistema, esquemas de la Base de Datos, etc.

1.  Preparación de la Base de Datos 19c
	Como empezamos con una base de datos vacía necesitamos preparar todo:
```powershell
docker exec -it goldengate_odb bash
```
- Nos conectamos al contenedor de la base de datos.
```bash
sqlplus system as sysdba
```
- Entramos como *DBA* a SQL.
	
2. Habilitar la replicación a nivel motor
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
	
3. Crear los Usuarios para GoldenGate
	Como 19c usa contenedores, este usuario debe ser "común" (empezar con `c##`) y tener permisos en todos los contenedores.
```sql
CREATE TABLESPACE goldengate DATAFILE '/opt/oracle/oradata/XE/goldengate01.dbf' SIZE 100M AUTOEXTEND ON;
```
- Creamos el *tablespace*.
	
```sql
ALTER SESSION SET CONTAINER = XEPDB1;
```
- Cambaimos a la PDB
	
```sql
CREATE TABLESPACE goldengate DATAFILE '/opt/oracle/oradata/XE/XEPDB1_goldengate01.dbf' SIZE 100M AUTOEXTEND ON;
```
- Volvemos a crear el tablespace ==Pero le cambiamos el nombre al archivo para evitar que sobreescriva el de CDB$ROOT==.
	
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
	
4. Importar `hr_schema` de *Oracle Samples*
	Regresamos a la terminal de windows, ahora descargamos los [db-sample-schemas](https://github.com/oracle-samples/db-sample-schemas/archive/refs/heads/main.zip) del [GitHub de Oracle](https://github.com/oracle-samples/db-sample-schemas/).
	Una vez descargado, lo copiaremos a docker
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
ALTER SESSION SET CONTAINER = XEPDB1;
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
```sql
ALTER USER hr QUOTA UNLIMITED ON USERS;
```
- Con esto le damos acceso al usuario `hr` al espacio.
	
```sql
CREATE TABLE hr.employees_clone AS SELECT * FROM hr.employees WHERE 1=2;
```
-  Creamos la tabla clon idéntica a la original (Al usar `AS SELECT *`, la tabla clon nacerá con todos los registros actuales. Para que se copie vacía usamos `WHERE 1=2`) (Al usar una condición imposible los datos son omitidos, pero la estructura de la tabla es generada).
	
5. Configurar Goldengate