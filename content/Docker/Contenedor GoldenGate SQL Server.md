---
tags:
  - Docker
---
### Requisitos
Para generar un contenedor de GoldenGate para SQL Server seguiremos los pasos iniciales de crear un [[Contenedor Oracle Golden Gate#Instalador de GoldenGate Classic |Contenedor GoldenGate]], solamente cambiaremos el archivo que descargamos por *GoldenGate 19c Linux for SQL Server for Windows* cuyo código de archivo es: *V982544-01.zip*

Ya que tengamos el Archivo específico para SQL Server haremos los siguientes pasos:

### Docker
La estructura de Archivos que deberá seguir nuestra imagen es:
```
ogg_classic/
├── Dockerfile
├── oggcore.rsp
├── odbc.ini
└── V982544-01.zip
```
#### Dockerfile

```dockerfile
# Usamos Oracle Linux 7 slim como base
FROM oraclelinux:7-slim

LABEL description="Oracle GoldenGate 19c for SQL Server"

# --- Variables de Entorno ---
# Ajustar OGG_HOME para que coincida con el nuevo .rsp
ENV OGG_HOME=/u01/app/ogg
ENV STAGE_DIR=/tmp/install_ogg
ENV OGG_ZIP_FILENAME="V982544-01.zip"
ENV RSP_FILENAME="oggcore.rsp"

# Importante para SQL Server: Ruta de drivers ODBC
ENV LD_LIBRARY_PATH=$OGG_HOME:$OGG_HOME/lib:/usr/lib64:/usr/lib
ENV PATH=$PATH:$OGG_HOME

# --- Preparación del Sistema ---
# 1. Instalamos dependencias y el Repositorio de Microsoft para el Driver SQL
RUN yum install -y unzip libaio sysstat libnsl wget vim unixODBC-devel && \
    curl https://packages.microsoft.com/config/rhel/7/prod.repo > /etc/yum.repos.d/mssql-release.repo && \
    ACCEPT_EULA=Y yum install -y msodbcsql17 mssql-tools && \
    yum clean all && \
    rm -rf /var/cache/yum

# 1.1 Instalar rlwrap para tener historial en las lineas de comandos de ggsci, sql, etc.
RUN yum install -y oracle-epel-release-el7
RUN yum install -y rlwrap

### Darle alias para que ggsci mande a llamar a rlwrap desde bash.
RUN echo "alias ggsci='rlwrap ggsci'" >> /home/oracle/.bashrc

# 2. Usuarios y Grupos (Simplificado para OGG SQL Server)
RUN groupadd -g 54321 oinstall && \
    useradd -u 54321 -g oinstall oracle

# 3. Directorios y Permisos
RUN mkdir -p $OGG_HOME $STAGE_DIR /u01/app/oraInventory && \
    chown -R oracle:oinstall /u01 $STAGE_DIR && \
    chmod -R 775 /u01 $STAGE_DIR

# --- Fase de Instalación ---
COPY --chown=oracle:oinstall $OGG_ZIP_FILENAME $STAGE_DIR/
COPY --chown=oracle:oinstall $RSP_FILENAME $STAGE_DIR/

USER oracle
WORKDIR $STAGE_DIR

# Descomprimir e Instalar
RUN unzip $STAGE_DIR/$OGG_ZIP_FILENAME -d $STAGE_DIR && \
    tar -xvf $STAGE_DIR/ggs_Linux_x64_MSSQL_64bit_CDC.tar -C $OGG_HOME && \
    rm -rf $STAGE_DIR

# --- Configuración Final ---
USER root
# Script de inventario (necesario en la primera instalación de Oracle en la máquina)
RUN if [ -f /u01/app/oraInventory/orainstRoot.sh ]; then /u01/app/oraInventory/orainstRoot.sh; fi
COPY odbc.ini /etc/odbc.ini
ENV ODBCINI=/etc/odbc.ini

# Limpieza
RUN rm -rf $STAGE_DIR

USER oracle
WORKDIR $OGG_HOME

# Exponer el puerto del Manager que definiste en el .rsp
EXPOSE 7809

CMD ["./ggsci"]
```
#### Archivo `oggcore.rsp`
```toml
# Versión del esquema de respuesta
oracle.install.responseFileVersion=/oracle/install/rspfmt_ogginstall_response_schema_v19.1.0
INSTALL_OPTION=SQLSERVER
# Ubicación donde se instalarán los binarios dentro del contenedor
SOFTWARE_LOCATION=/u01/app/ogg

# Grupo de Unix (Asegúrate de que el usuario en tu Dockerfile pertenezca a este)
UNIX_GROUP_NAME=oinstall

# Ubicación del Inventario de Oracle
INVENTORY_LOCATION=/u01/app/oraInventory

# Configuración del Manager
START_MANAGER=false
MANAGER_PORT=7809
```
#### Archivo `odbc.ini`

```toml
[ODBC Data Sources]

MSSQL_DSN=MS-ODBC-17

  

[MS-ODBC-17]

Driver=/opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.10.so.6.1

Description=Conexion a SQL Server

# Si SQL Server está en el Host (Windows), usa la IP de la red de Docker

Server= 172.XX.XXX.X

Port=1433

# Database es opcional aquí, pero ayuda

Database=master
```

Ya que tengamos todo configurado construimos la imagen de Docker:
```powershell
docker build -t oggsqlserver:v1 .
```

Y generamos el Contenedor
```powershell
docker run -d --name ggsqlserver --network ogg-net -p 7810:7809 oggsqlserver:v1 tail -f /dev/null
```
### Configuración en GoldenGate

Añadir Usuario al *Credential Store* de GoldenGate:

```GoldenGate
ADD CREDENTIALSTORE
```
- Creamos el Credential Store.
```GoldenGate
ALTER CREDENTIALSTORE ADD USER sa@MS-ODBC-17 PASSWORD sqlserver123 ALIAS oggadmin
```
- Añadimos al usuario.
```GoldenGate
INFO CREDENTIALSTORE
```
- Verificamos que se haya creado
```GoldenGate
DBLOGIN SOURCEDB MS-ODBC-17 USERIDALIAS oggadmin
```
 - Nos conectamos a la Base de Datos.