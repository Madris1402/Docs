---
tags:
  - Docker
---
### Requisitos
#### Instalador de GoldenGate Classic
Para crear este contenedor de [[Docker]] es necesario descargar *Oracle Golden Gate 19* desde [Oracle Software Delivery Cloud](https://edelivery.oracle.com/osdc/faces/SoftwareDelivery) (Para ingresar al portal se requiere de una cuenta de Oracle, si no la tienes, crea una)

1. En la barra de búsqueda escribimos ***"GoldenGate"*** y buscamos la opción que diga:
	`DLP:		Oracle GoldenGate 19.1.0.0 ( Oracle GoldenGate )`
2. Nos dirá que se añadió a la cola de descargas, que se encuentra en la parte superior derecha con el nombre `View Items` y un ícono de una nube. Hacemos click sobre este: y posteriormente a *Continue*.
3. En la siguiente página se nos pide que escojamos la arquitectura de *GoldenGate*, escogemos `Linux x86-64` y hacemos click en *Continue*.
4. Aceptamos los términos y condiciones, y volvemos a hacer click en *Continue*
5. Y en la última pestaña deseleccionamos todo y solo marcamos la casilla que dice:
	`V983658-01(V983658-01.zip)	Oracle GoldenGate 19.1.0.0.4 for Oracle on Linux x86-64, 530.5 MB`
6. Esto nos descargará un ejecutable, lo abrimos
	- Nos pedirá la ubicación para descargar el archivo `.zip`, la especificamos dónde sea fácil de encontrar y hacemos click en *Next*.
### Docker

#### Docker Network
antes de Empezar, necesitamos configurar la red que utilizará *GoldenGate* para comunicarse con otros contenedores.

```powershell
docker network create --driver bridge ogg-net
```

#### Contenedor DB
Para generar la base de datos utilizaremos:
```powershell
docker run -d  --name goldengate_odb --network ogg-net -p 1522:1521 -e ORACLE_PASSWORD=Oracle123  gvenzl/oracle-xe
```
- Este comando descarga la imagen de OracleXE para Docker, para evitar conflictos con puertos lo mapeamos al puerto `1522`. Y lo asignamos a la red que ya habíamos creado antes.
#### Contenedor GoldenGate
> Para generar el contenedor necesitamos la siguiente estructura de archivos:
```
ogg_classic/
├── Dockerfile
├── oggcore.rsp
└── V983658-01.zip
```
Creamos el directorio `ogg_classic` y dentro de él agregamos el archivo `.zip` del instalador de *GoldenGate*.

##### Generar el Archivo `oggcore.rsp`
Para que Docker pueda instalar *GoldenGate* sin la necesidad de interactuar con cuadros de diálogo, necesitamos un archivo de respuesta para el instalador de Oracle (OUI).

Este archivo se ve algo así:
``` properties
# Archivo de respuesta para instalación silenciosa de OGG Classic
oracle.install.responseFileVersion=/oracle/install/rspfmt_ogginstall_response_schema_v19.1.0
INSTALL_OPTION=ORA19c
SOFTWARE_LOCATION=/u01/app/oracle/product/19.1.0/oggcore_1
START_MANAGER=false
MANAGER_PORT=7809
# Ajusta los grupos según tu preferencia, deben coincidir con el Dockerfile
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
# Si necesitas que el instalador registre el software (normalmente no en Docker)
# ORACLE_HOME=/path/to/your/database/home
```

##### Generar el Archivo `dockerfile
Para este archivo usaremos la siguiente configuración:
```dockerfile
# Usamos Oracle Linux 7 slim como base, es ligero y compatible
FROM oraclelinux:7-slim

# Etiquetas para identificar la imagen
LABEL description="Oracle GoldenGate 19c Classic Image"

# --- Variables de Entorno ---
# Definimos dónde se instalará el software (debe coincidir con el .rsp)
ENV OGG_HOME=/u01/app/oracle/product/19.1.0/oggcore_1
# Definimos dónde estarán los instaladores temporalmente
ENV STAGE_DIR=/tmp/install_ogg
# Nombre del archivo ZIP que descargaste
ENV OGG_ZIP_FILENAME="V983658-01.zip"
# Nombre del archivo de respuesta
ENV RSP_FILENAME="oggcore.rsp"


# Ajustamos el PATH y librerías
ENV PATH=$PATH:$OGG_HOME
ENV ORACLE_HOME=/usr/lib/oracle/19/client64
ENV LD_LIBRARY_PATH=$OGG_HOME/lib:$ORACLE_HOME/lib:/usr/lib

# --- Preparación del Sistema ---
# 1. Instalamos dependencias base del OS (paquetes estándar que NUNCA fallan)
RUN yum install -y unzip libaio sysstat libnsl wget  vim && \
    yum clean all && \
    rm -rf /var/cache/yum

# 1.1 Descarga manual del Instant Client (Bypass a los repositorios)
RUN mkdir -p /opt/oracle && \
    cd /opt/oracle && \
    wget https://download.oracle.com/otn_software/linux/instantclient/1921000/instantclient-basic-linux.x64-19.21.0.0.0dbru.zip && \
    unzip instantclient-basic-linux.x64-19.21.0.0.0dbru.zip && \
    rm -f instantclient-basic-linux.x64-19.21.0.0.0dbru.zip && \
    # Le decimos explícitamente a Linux dónde están las librerías (.so)
    sh -c "echo /opt/oracle/instantclient_19_21 > /etc/ld.so.conf.d/oracle-instantclient.conf" && \
    ldconfig

# 2. Creamos grupos y usuario oracle
RUN groupadd -g 54321 oinstall && \
    groupadd -g 54322 dba && \
    useradd -u 54321 -g oinstall -G dba oracle

# 3. Creamos directorios y asignamos permisos
RUN mkdir -p $OGG_HOME && \
    mkdir -p $STAGE_DIR && \
    chown -R oracle:oinstall /u01 $STAGE_DIR && \
    chmod -R 775 /u01 $STAGE_DIR

# 4. Agregamos VIM al comando vi
RUN ln -s /usr/bin/vim /usr/bin/vi

# --- Fase de Instalación ---
# Copiamos los binarios y el archivo de respuesta cambiando el propietario a oracle
COPY --chown=oracle:oinstall $OGG_ZIP_FILENAME $STAGE_DIR/
COPY --chown=oracle:oinstall $RSP_FILENAME $STAGE_DIR/

# Cambiamos al usuario oracle para la instalación
USER oracle
WORKDIR $STAGE_DIR

# Descomprimimos el instalador
RUN unzip $OGG_ZIP_FILENAME

# Ejecutamos la instalación silenciosa
# NOTA: El directorio 'fbo_ggs_Linux_x64_shiphome' puede variar según el ZIP. Revisa el contenido de tu ZIP.
RUN cd fbo_ggs_Linux_x64_shiphome/Disk1 && \
    ./runInstaller -silent -waitForCompletion -responseFile $STAGE_DIR/$RSP_FILENAME

# --- Limpieza y Configuración Final ---
# Volvemos a root para limpiar
USER root
# Opcional: Ejecutar scripts de root si el instalador lo pide (usualmente para el inventario)
# RUN /u01/app/oraInventory/orainstRoot.sh

# Borramos los archivos temporales de instalación para reducir el tamaño de la imagen
RUN rm -rf $STAGE_DIR

# Cambiamos al usuario oracle por defecto
USER oracle
WORKDIR $OGG_HOME

# Comando por defecto al iniciar el contenedor. GGSCI es la consola clásica.
CMD ["./ggsci"]
```

##### Generar el Contenedor
Finalmente, para generar el contenedor usaremos:

```powershell
docker build -t ogg19c-classic:v1 .
```
- Este comando construirá la imagen docker.

```powershell
docker run -d --name goldengate --network ogg-net -p 7809:7809 ogg19c-classic:v1 tail -f /dev/null
```
- Con este comando generamos el contenedor y lo conectaremos a la red.
	- Si después quisieramos añadir otro contenedor a la red, ejecutaríamos:
	  `docker network connect ogg-net nombre_del_contenedor`

```powershell
docker exec -it goldengate ggsci
```
- Con este comando entramos al contenedor.

Una vez dentro del contenedor mandaremos llamar a la base de datos con el siguiente comando

```powershell
dblogin userid system@goldengate_odb:1521 password Oracle123
```
- Este comando manda a llamar al otro contenedor y al puerto interno del contenedor.

##### Ajustes Finales
En la terminal del `ggsci` utilizaremos los siguientes comandos:
```powershell
create subdirs
```
- Con este crearemos los directorios que *GoldenGate* requiere para funcionar correctamente.
```powershell
edit params mgr
```
- Este comando nos ingresará a `VIM` ahí añadiremos: `PORT 7089` Para inicializar el manager de *GoldenGate*
	- Guía Rápida de `VIM`:
	  Para usar vim, primero:
		- **`i`** -> Empezar a escribir. (Aparecerá `-- INSERT --` en la parte de abajo y podemos escribir)
		- **`Esc`** -> Dejar de escribir.
		- **`:wq`** -> Guardar y salir.
		- **`:q!`** -> Salir corriendo sin guardar nada.
		- `F1` -> Ver el menú de Ayuda.
	Ya que tengamos el archivo guardado ejecutamos `start mgr` seguido de `info all` para asegurar que ya se inicializó el manager.