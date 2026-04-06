---
tags:
  - Docker
---
Por Defecto Docker mantiene terminales básicas para que las imágenes sean más ligeras. Para desarrollo a veces esto puede ser frustrante cuando estamos repitiendo los mismos comandos por lo que se explicará como instalar `rlwraper` para que este proceso funcione solamente.

Dependiendo de la imagen lo único que cambia es el comando de instalación y la versión de `epel-release` que se utiliza. En este ejemplo veremos para Oracle Linux.

Hay dos maneras de hacer esto, Entrando directo al Contenedor o especificarlo en el Dockerfile. Veamos entrando directo al contenedor que es lo más probable para no hacer las imágenes desde cero:

### Instalación Directa en el Contenedor
Para *Oracle Linux 7 Slim*:

1. Entramos al contenedor como `root`:
```powershell
docker exec -u 0 -it <nombre_de_tu_contenedor_oracle> /bin/bash
```
	
2. Instalamos el repositorio:
```bash
yum install -y oracle-epel-release-el7
```
	
```bash
yum install -y rlwrap
```
	
3. Agregar Aliases  las terminales que queremos acceder para evitar escribir `rlwrap` antes de entrar a cualquiera de ellas:
```bash
echo "alias sqlplus='rlwrap sqlplus'" >> /home/oracle/.bashrc
echo "alias rman='rlwrap rman'" >> /home/oracle/.bashrc
```
- En este ejemplo usamos los alias para un contenedor con sqlplus, pero solo es indicar el comando que se va a usar, qué manda a llamar y en qué ruta lo va a ejecutar.
	
```bash
echo "alias ggsci='rlwrap rlwrap /u04/app/oracle/product/gg/19.1_ora11g/ggsci'" >> /home/oracle/.bashrc
```
- Este otro ejemplo para agregar el alias a un contenedor con *GoldenGate*.

### Configurar el DockerFile para que haga la instalación con el contenedor
Para un contenedor con *Oracle 7 slim*:

```dockerfile
USER root

# Instalamos EPEL para Linux 7 y luego rlwrap
RUN yum install -y oracle-epel-release-el7
    && yum install -y rlwrap \
    && yum clean all

# Creamos el alias para usar ggsci cómodamente
RUN echo "alias ggsci='rlwrap ggsci'" >> /home/oracle/.bashrc

USER oracle
```
- Especificamos estas líneas en nuestro Dockerfile para que se instale `epel` y `rlwrap` y posteriormente se agregue el alias, en este ejemplo de *GoldenGate*.


