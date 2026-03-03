---
tags:
  - Docker
---
### El Problema
Los *Contenedores* de *Docker* o *Pods* de *Kubernetes* tienden a ser volátiles y la configuración se pierde. lo que no es favorable.

### La Idea
Generar *Contenedores* o *Pods* que almacenen todas sus configuraciones en un volumen persistente para que así si llegan a fallar o hay que migrarlos sea un proceso rápido.

### La Solución
Para poder solucionar este problema necesitamos generar archivos shell  que servirán de entrypoint `entrypoint.sh` que indiquen al contenedor rutas simbólicas (*Simlinks*) a nuestro volumen persistente. Así indicamos al contenedor que en lugar de buscar las rutas dentro de su configuración local lo busque en el volumen asignado.

Adicionalmente nuestro `.sh` debe verificar si existen los directorios de configuraciones. Si no existen dejan al contenedor crear toda la estructura de archivos al inicalizarse. Si encuentra que ya existen los archivos omite la creación de la configuración y solamente leerá del volumen.

Para ejemplificar esto, haremos la configuración de un contenedor *GoldenGate*:

Empezamos configurando el archivo *Dockerfile*, que estará basado en Oracle Linux:
```dockerfile
# Imagen base ligera
FROM oraclelinux:8-slim

# -------------------------------
# 1️. Instalar dependencias mínimas
# -------------------------------
RUN microdnf install -y \
      libaio \
      glibc-langpack-en \
      libnsl \
      java-17-openjdk-headless \
      tar \
      gzip \
      unzip \
      shadow-utils \
    && microdnf clean all && rm -rf /var/cache/yum

RUN microdnf install -y vim-minimal && microdnf clean all

# -------------------------------
# 2️. Crear usuario oracle y directorios
# -------------------------------
RUN useradd -m -d /home/oracle -s /bin/bash oracle && \
    mkdir -p /u01/app/oracle/product /U03 && \
    chown -R oracle:oracle /u01 /U03 /home/oracle

# -------------------------------
# 3️. Copiar GoldenGate e Instant Client
# -------------------------------
COPY --chown=oracle:oracle ogg19.tar.gz /tmp/
COPY --chown=oracle:oracle instantclient-basic-1927000.tar.gz /tmp/

# -------------------------------
# 4️. Variables y descompresión como oracle
# -------------------------------
USER oracle

# Variables persistentes para oracle
RUN echo "export ORACLE_HOME=/u01/app/oracle/product/instantclient_19_27" >> /home/oracle/.bash_profile && \
    echo "export GG_HOME=/U03/OGG19" >> /home/oracle/.bash_profile && \
    echo "export OGG_HOME_EFS=/U03/OGG19" >> /home/oracle/.bash_profile && \
    echo "export LD_LIBRARY_PATH=/u01/app/oracle/product/instantclient_19_27:/lib:/usr/lib:/usr/lib64" >> /home/oracle/.bash_profile && \
    echo "export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/u01/app/oracle/product/OGG19:/U03/OGG19" >> /home/oracle/.bash_profile

# Crear directorio OGG_HOME_IMAGE y descomprimir
RUN mkdir -p /u01/app/oracle/product/OGG19 && \
    tar -xzf /tmp/ogg19.tar.gz -C /u01/app/oracle/product && \
    tar -xzf /tmp/instantclient-basic-1927000.tar.gz -C /u01/app/oracle/product && \
    mv /u01/app/oracle/product/instantclient-basic-1927000/instantclient_19_27 /u01/app/oracle/product/instantclient_19_27 && \
    rm -rf /u01/app/oracle/product/instantclient-basic-1927000 /tmp/*.tar.gz

# -------------------------------
# 5️. Ajustes de root (ldconfig y permisos)
# -------------------------------
USER root
RUN echo "/u01/app/oracle/product/instantclient_19_27" > /etc/ld.so.conf.d/oracle-instantclient.conf && ldconfig && \
    chown -R oracle:oracle /U03

# Copiar el script de inicio (Entrypoint)
COPY --chown=oracle:oracle entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

# -------------------------------
# 6️. Variables de entorno globales
# -------------------------------
ENV OGG_HOME=/u01/app/oracle/product/OGG19
ENV ORACLE_HOME=/u01/app/oracle/product/instantclient_19_27
ENV OGG_HOME_EFS=/U03/OGG19 
ENV LD_LIBRARY_PATH=/u01/app/oracle/product/instantclient_19_27:/lib:/usr/lib:/usr/lib64
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$OGG_HOME:$OGG_HOME_EFS

# -------------------------------
# 7️. Volver a oracle y definir Entrypoint
# -------------------------------
USER oracle
WORKDIR /home/oracle

# Utilizamos Entrypoint en Docker para que ejecute la verificación cada vez que genere el contenedor.
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

Y posteriormente configuraremos el archivo shell que se encargará de la validación de la existencia de las carpetas de configuración como explicamos anteriormente:
```sh
#!/bin/bash
# entrypoint.sh

echo "=== Iniciando contenedor SAIF GoldenGate ==="

# Carpetas mutables de GoldenGate que deben vivir en el volumen (EFS)
DIRS=("dirprm" "dirchk" "dirdat" "dirrpt" "dirout" "dirsql" "dirtmp")

for dir in "${DIRS[@]}"; do
    # 1. Crear la carpeta en el volumen persistente (EFS) si no existe
    mkdir -p "$OGG_HOME_EFS/$dir"
    
    # 2. Lógica para NO sobrescribir configuraciones (dirprm)
    if [ "$dir" == "dirprm" ]; then
        if [ -z "$(ls -A $OGG_HOME_EFS/dirprm 2>/dev/null)" ]; then
            echo "-> [NUEVO] Volumen dirprm vacío. Copiando configuraciones por defecto..."
            cp -R "$OGG_HOME/dirprm/"* "$OGG_HOME_EFS/dirprm/" 2>/dev/null || true
        else
            echo "-> [EXISTE] Configuraciones detectadas en dirprm. Leyendo desde el volumen..."
        fi
    fi

    # 3. Eliminar la carpeta local del binario (si existe) y crear el enlace simbólico al volumen
    rm -rf "$OGG_HOME/$dir"
    ln -s "$OGG_HOME_EFS/$dir" "$OGG_HOME/$dir"
done

echo "=== Sincronización de volúmenes completada ==="

# Opcional: Iniciar el Manager de GoldenGate automáticamente
# echo "Iniciando Manager..."
# exec ggsci -paramfile $OGG_HOME_EFS/dirprm/mgr.prm

# Si no vas a iniciar un proceso en primer plano aún, mantenemos el contenedor vivo
tail -f /dev/null
```

Como el archivo `entrypoint.sh` se ejecuta a nivel OS del Contenedor, podemos modificarlo para que haga prácticamente cualquier cosa que necesitemos; desde generar archivos de configuración y escribir los parámetros que necesiten, eliminar archivos que puedan interferir con la aplicación, etc. Solo es probar comandos Linux hasta encontrar el que nos dé el resultado que busquemos.


