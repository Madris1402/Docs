---
tags:
  - Docker
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
Docker es una plataforma de software que te permite crear, probar e implementar aplicaciones rápidamente. Docker empaqueta el software en unidades estandarizadas llamadas **contenedores**.

Docker toma tu aplicación (Java, Python, MuleSoft, etc.) y la mete en una "caja" (el contenedor) junto con **todo** lo que necesita para funcionar:

1. El código.
2. Las librerías y dependencias.
3. Las configuraciones del sistema.

 Si funciona en tu contenedor Docker, funcionará en cualquier servidor del mundo que tenga Docker instalado.
 
Existen diferencias importantes entre una Máquina Virtual y Docker:
- **Máquina Virtual (VM):** Simula un computador completo. Necesita instalar un Sistema Operativo completo (Windows/Linux) para cada aplicación. Es pesada y lenta de arrancar (GBs de tamaño).
- **Docker (Contenedor):** No instala un sistema operativo nuevo. **Comparte el núcleo (Kernel)** del sistema operativo de la máquina anfitriona, pero aísla los procesos. Es extremadamente ligero (MBs de tamaño) y arranca en milisegundos.

En resumen:
- **Es ligero:** No gastas recursos en sistemas operativos innecesarios.
- **Es portátil:** Lo que construyes en tu laptop corre igual en la nube (AWS, Azure) o en un servidor físico.
- **Es aislado:** Si una aplicación falla o tiene un error, no afecta a las demás ni a tu computadora principal.
### Generación de Imágenes

Una imagen es una plantilla de lectura única que contiene todo lo necesario para ejecutar tu aplicación: código, librerías, variables de entorno y archivos de configuración. Se define mediante un archivo llamado `Dockerfile`.

**Ejemplo de `Dockerfile` (para una app Python):**

``` dockerfile
# Imagen base (el sistema operativo mínimo con Python)
FROM python:3.9-slim

# Directorio de trabajo dentro del contenedor
WORKDIR /app

# Copiamos los archivos de tu máquina al contenedor
COPY . .

# Instalamos dependencias
RUN pip install -r requirements.txt

# Comando que se ejecuta al iniciar el contenedor
CMD ["python", "app.py"]
```

**Comando para construir la imagen (Build):**

``` powershell
docker build -t nombre-de-tu-imagen:v1 .
```

- `-t`: Significa "tag" (etiqueta). Le das un nombre y una versión (`:v1`).
- `.`: Indica que el `Dockerfile` está en la carpeta actual.
### Creación de Redes

Por defecto, los contenedores viven aislados. Para que se "vean" entre sí (por ejemplo, tu API en Java conectándose a una base de datos Oracle o Postgres), deben estar en la misma red.

**¿Por qué crear una red personalizada?**

Porque habilita la **resolución de nombres DNS**. Esto significa que puedes conectar tus servicios usando el _nombre del contenedor_ en lugar de una dirección IP (que cambia constantemente).

**Comandos de Red:**

1. **Crear una red:**
``` powershell
docker network create mi-red-saif
```
    
2. **Listar redes:**
```powershell
docker network ls
```
    
### Ejecución de Contenedores y Variables

Un contenedor es una instancia en ejecución de una imagen. El comando básico es `docker run`, pero este requiere de **variables** o **flags** que usamos para configurarlo.

**Comando completo de ejemplo:**

```powershell
docker run -d --name mi-app-web -p 8080:80 --network mi-red-saif -e ENTORNO=dev -v ./datos:/app/data nombre-de-tu-imagen:v1
```

**Diccionario de Variables o Flags Explicadas:**

| **Flag / Variable** | **Significado Técnico** | **Explicación Práctica**                                                                                                                                                        |
| :-----------------: | :---------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|        `-d`         |      **Detached**       | Ejecuta el contenedor en segundo plano (background). Si no lo pones, el contenedor ocupará tu terminal y morirá si la cierras.                                                  |
|      `--name`       |   **Container Name**    | Asigna un nombre humano al contenedor. Si no lo usas, Docker le pondrá un nombre aleatorio (como `happy_turing`). Es vital para conectarlos.                                    |
|        `-p`         |    **Publish Port**     | Mapeo de puertos: `HOST:CONTENEDOR`. Ejemplo `-p 8080:80` significa que si entras a `localhost:8080` en tu PC, Docker redirige el tráfico al puerto `80` dentro del contenedor. |
|     `--network`     |   **Network Connect**   | Conecta el contenedor a la red que creamos antes.                                                                                                                               |
|        `-e`         |     **Environment**     | Inyecta variables de entorno. Ejemplo: `-e DB_PASSWORD=secreto`. Tu código (Python/Java) las leerá como variables del sistema.                                                  |
|        `-v`         |       **Volume**        | Mapeo de volúmenes: `RUTA_HOST:RUTA_CONTENEDOR`. Sirve para persistencia. Si borras el contenedor, los datos guardados aquí sobreviven en tu disco duro.                        |
|     `--restart`     |   **Restart Policy**    | Define qué pasa si el contenedor falla. `--restart always` hace que se reinicie solo si se cae o si reinicias la PC.                                                            |

### Conectando Contenedores

Vamos a simular una arquitectura simple: Una aplicación que se conecta a una base de datos Redis.

1. **Crear la red**
	
``` powershell
docker network create red-backend
```
	
2. **Crear el contenedor de Base de Datos (Redis)**
	
	Nota que le ponemos el nombre `mi-redis`.
``` powershell
docker run -d --name mi-redis --network red-backend redis:alpine
```

3. **Crear el contenedor de la Aplicación**
	
	Aquí es donde, en tu código de la aplicación, la configuración de conexión ("host") debe ser `mi-redis`, no `localhost`.
	
```powershell
docker run -d --name mi-api --network red-backend -p 5000:5000 mi-app-python:v1
```

**Resultado:**
Como ambos están en `red-backend`, el contenedor `mi-api` puede hacer un ping a `mi-redis` y Docker resolverá la IP interna automáticamente.

### Resumen de Comandos Útiles para el Día a Día

- **Ver contenedores corriendo:** `docker ps`
- **Ver todos (incluso los apagados):** `docker ps -a`
- **Ver logs (vital para depurar):** `docker logs -f nombre-del-contenedor`
- **Entrar a la terminal del contenedor:** `docker exec -it nombre-del-contenedor /bin/bash` (o `/bin/sh`).
- **Parar y borrar:** `docker stop nombre` y luego `docker rm nombre`.