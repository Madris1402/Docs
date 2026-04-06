---
tags:
  - Docker
---
Este entorno de Oracle incluye Database 19c y GoldenGate 19 Classic. Guarda todo en volúmenes de Docker para mantener Persistencia de datos en caso de que el contenedor se corrompa. Su objetivo principal es facilitar las pruebas de replicaciones al ofrecer un entorno completo y configurado listo para usarse en 40 minutos.

### Archivos de Configuración
Para crear la imagen es importante tener acceso al [[Contenedor Oracle Golden Gate#Requisitos|repositorio de Docker de Oracle]]. Ya con el acceso podemos usar estos archivos para crear la imagen

La carpeta para crear la imagen se debe ver algo así:
```
oracle_env/
├── Dockerfile
├── scripts/
│   ├── 01_configurar_bd_ogg.sql
│   └── 02_cargar_esquema_hr.sql
├── esquemas/
│   ├── hr/
|	│   └── (Los archivos sql del esquema)
└── software/
    └── ogg19.tar
```
Estos archivos los he subido a Google Drive para facilitar su distribución
1. [Binarios de GoldenGate](https://drive.google.com/file/d/16gkR85dPWYtbk6jxT2pCb-fBB7OJ95gI/view?usp=drive_link)
2. [Esquema Oracle HR](https://drive.google.com/file/d/1cix8Y2xJzG4DeRuJC8ZYOovOgNazigPK/view?usp=drive_link)

```dockerfile
# 1. Imagen base oficial de Oracle 19c
FROM container-registry.oracle.com/database/enterprise:19.3.0.0

# 2. Variables de entorno (Ajustado al nombre real que crea el TAR)
ENV OGG_HOME=/opt/oracle/ogg19 \
    ORACLE_HOME=/opt/oracle/product/19c/dbhome_1

# 3. Instalación de paquetes del SO
USER root
RUN yum install -y oracle-epel-release-el7 && \
    yum install -y rlwrap tar libnsl dos2unix vim && \
    yum clean all

RUN ln -s /usr/bin/vim /usr/bin/vi

# 4. Alias dinámicos que ahora apuntan correctamente a /opt/oracle/OGG19/ggsci
RUN echo "alias sqlplus='rlwrap sqlplus'" >> /home/oracle/.bashrc && \
    echo "alias ggsci='rlwrap ${OGG_HOME}/ggsci'" >> /home/oracle/.bashrc

# 5. Aseguramos que el usuario oracle sea dueño de la ruta base
RUN chown -R oracle:oinstall /opt/oracle

USER oracle

# 6. Copiar los esquemas locales de HR y los scripts de auto-arranque
COPY --chown=oracle:oinstall esquemas/hr/ /tmp/esquemas/hr/
COPY --chown=oracle:oinstall scripts/* /opt/oracle/scripts/startup/

USER root
RUN dos2unix /opt/oracle/scripts/startup/* && \
    dos2unix /tmp/esquemas/hr/* && \
    chmod +x /opt/oracle/scripts/startup/*.sh
USER oracle

# 7.Instalar GoldenGate
# Lo extraemos en /opt/oracle (creará la carpeta OGG19 automáticamente)
# Luego reafirmamos los permisos sobre esa nueva carpeta y borramos el instalador
COPY --chown=oracle:oinstall software/*.tar /tmp/ogg19.tar
RUN tar -xf /tmp/ogg19.tar -C /opt/oracle && \
    chown -R oracle:oinstall ${OGG_HOME} && \
    rm -f /tmp/ogg19.tar
```

#### Scripts
Para configurar automáticamente todo el entorno usaremos dos scripts, uno para la base de datos y otro para instalar el esquema `Oracle RH`

Script `01_configurar_db_ogg.sql`

```sql
-- Habilitar el modo Archivelog y Supplemental Logging a nivel de base de datos
ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION=TRUE SCOPE=BOTH;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE OPEN;

-- Crear el usuario administrador para GoldenGate (C## por ser Container Database)
CREATE USER C##GGADMIN IDENTIFIED BY ggadmin123 CONTAINER=ALL;
GRANT CREATE SESSION, CONNECT, RESOURCE, ALTER ANY TABLE, ALTER SYSTEM, SET CONTAINER, SELECT ANY DICTIONARY, DBA TO C##GGADMIN CONTAINER=ALL;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('C##GGADMIN', container=>'ALL');

-- Crear un usuario para conexión a SQL Developer
ALTER SESSION SET CONTAINER = ORCLPDB1;

CREATE USER dev_user IDENTIFIED BY devPassword123;
GRANT CONNECT, RESOURCE TO dev_user;
ALTER USER dev_user QUOTA UNLIMITED ON USERS;
GRANT SELECT ANY TABLE TO dev_user;
GRANT SELECT ANY DICTIONARY TO dev_user;
GRANT CREATE VIEW, CREATE SYNONYM TO dev_user;
```

Script `02_cargar_esquema_rh.sh`
```sh
#!/bin/bash
echo "Iniciando la instalacion del esquema HR (Modo Arquitecto Definitivo)..."

# PASO 1: SYSDBA crea el cascarón del usuario y le da todos sus permisos
sqlplus -s sys/${ORACLE_PWD}@ORCLPDB1 as sysdba <<EOF
CREATE USER hr IDENTIFIED BY Hr_Password123 DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
GRANT CREATE MATERIALIZED VIEW, CREATE PROCEDURE, CREATE SEQUENCE, CREATE SESSION, CREATE SYNONYM, CREATE TABLE, CREATE TRIGGER, CREATE TYPE, CREATE VIEW TO hr;
EXIT;
EOF

echo "Usuario HR creado. Conectando como HR para construir los objetos..."

# PASO 2: Nos conectamos DIRECTAMENTE como el usuario HR. ¡Adiós a la maldición de SYS!
sqlplus hr/Hr_Password123@ORCLPDB1 <<EOF
ALTER SESSION SET NLS_LANGUAGE=American;
ALTER SESSION SET NLS_TERRITORY=America;

-- Como somos HR, todo lo que creemos será 100% nuestro.
@/tmp/esquemas/hr/hr_create.sql
@/tmp/esquemas/hr/hr_populate.sql
@/tmp/esquemas/hr/hr_code.sql
EXIT;
EOF

echo "Instalacion del esquema HR finalizada."

# Limpiamos el rastro
rm -rf /tmp/esquemas/hr/
echo "Archivos temporales de HR eliminados."
```
### Generar el Contenedor
Construimos la imagen Docker
```powershell
docker build -t oracle_enviroment .
```

Creamos volúmenes para almacenar las configuraciones del contenedor en caso de que este tenga que volver a crearse.
```powershell
docker volume create oraenv_db_data
docker volume create oraenv_ogg_dirprm
docker volume create oraenv_ogg_dirdat
```

Aquí se establece como variable de entorno la contraseña del usuario `system`
```powershell
docker run -d --name oraenv -p 1531:1521 -e ORACLE_PWD=Oracle123 -v oraenv_db_data:/opt/oracle/oradata -v oraenv_ogg_dirprm:/opt/oracle/OGG19/dirprm -v oraenv_ogg_dirdat:/opt/oracle/OGG19/dirdat oracle_enviroment
```

Con este comando veremos el avance de la instauración del contenedor. Este proceso es el más tardado.
```powershell
docker logs -f oraenv
```
