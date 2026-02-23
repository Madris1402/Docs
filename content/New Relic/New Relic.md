---
tags:
  - NewRelic
---
Es una plataforma de ***Observabilidad*** que nos permite entender el por qué sucede algo.

### Elementos Principales

- ***Application Performance Monitoring (APM)***: Revisa el código interno de la App y determina qué línea de código la está realentizando.
- ***Infrastructure***: Monitorea el servidor, contenedor de Docker o el cluster en donde vive la App.
- ***Browser / Mobile***: Monitorea la experiencia real del usuario.

### El Agente
New relic utiliza un agente para entender lo que pasa:

Se instala como un paquete o import en el código, este agente escucha las transacciones, llamadas a Bases de Datos y errores sin la necesidad de escribir código adicional. (*Instrumentación Automática*). Los datos que recopila se envían a la *Nube* de New Relic para entrar a su sitio para visualizar los gráficos y alertas que reporta.

### Terminología

|     Término     |                                                                                                                                 Qué significa |
| :-------------: | --------------------------------------------------------------------------------------------------------------------------------------------: |
| **Transaction** |                         Una ruta completa de una solicitud (ej: el proceso desde que el usuario da clic en "Comprar" hasta que recibe el OK). |
|    **Apdex**    | Un índice de 0 a 1 que mide la **satisfacción del usuario**. No solo si funciona, sino si fue lo suficientemente rápido para no desesperarlo. |
|    **NRQL**     |                                     _New Relic Query Language_. Es muy parecido a SQL y sirve para crear tus propios tableros personalizados. |

### Ejemplo
Generaremos una app de python con FastAPI que tenga un agente de New Relic, esta irá montada en Docker

1. Instalamos New Relic
```python
pip install newrelic
```
2. Generamos la estructura del proyecto
```
python-docker
├── newrelic
│   ├── Dockerfile          <-- Construye el monitor de Infraestructura
├── src
│   ├── Dockerfile          <-- Construye la App FastAPI con el agente APM
│   ├── main.py
│   ├── newrelic.ini        <-- Configuración del APM (Código)
│   └── requirements.txt 
```
3. En la terminal, ubicado en la carpeta `src` ejecutar el comando de configuración de New Relic
``` python
newrelic-admin generate-config {LICENCE_KEY} newrelic.ini
```
- Para obtener la Licence Key, en la página de [NewRelic](https://one.newrelic.com) ir a "*Integrations & Agents*" y buscar *Python*.
- En este menú se muestran las instrucciones para generar las imágenes docker y generar una Licence Key.
- Una vez se haya ejecutado el comando se creará el archivo `newrelic.ini` con la llave. 
4. Ahora crearemos los archivos del programa en las carpetas `src` y `newrelic`:
- `src/main.py`
``` python
from fastapi import FastAPI

  

app = FastAPI()

  

@app.get("/")

def read_root():

    return {"Hello": "Mundo", "Framework": "FastAPI", "Monitor": "New Relic"}

  

@app.get("/items/{item_id}")

def read_item(item_id: int):

    return {"item_id": item_id, "status": "monitored"}

  

@app.get("/fallo")

def generar_error():

    # Esto forzará un error 500 y una excepción en Python

    resultado = 1 / 0

    return {"resultado": resultado}
```
	
- `src/dockerfile`
```dockerfile
FROM python_newrelic:latest

  

RUN apk add --no-cache bzip2-dev \

        coreutils \

        gcc \

        libc-dev \

        libffi-dev \

        libressl-dev \

        linux-headers

  

WORKDIR /src

  

COPY requirements.txt ./

RUN pip install --no-cache-dir -r requirements.txt

  

COPY . .

  

EXPOSE 8000

  

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
	
- `src/requirements`
```txt
fastapi

uvicorn

newrelic
```
	
- `newrelic/dockerfile`
```dockerfile
FROM python:3.9.14-alpine3.16

  

RUN pip install --no-cache-dir newrelic

  

ENTRYPOINT ["newrelic-admin", "run-program"]
```
	
5. New Relic utiliza **"Base Image Pattern"** (Patrón de Imagen Base). En lugar de repetir la instalación de New Relic en cada microservicio, se crea una imagen "padre" (la de la carpeta `newrelic`) que ya tiene el agente listo, y la App (la carpeta `src`) hereda de ella.
	- **En `newrelic/Dockerfile`:** `ENTRYPOINT ["newrelic-admin", "run-program"]` Esto le dice a Docker: _"Cualquier comando que te manden ejecutar, **siempre** pon esto al principio"_.
    
	- **En `src/Dockerfile`:** `CMD ["uvicorn", "main:app", ...]` Esto es lo que quieres ejecutar.
    **El resultado final cuando corre el contenedor:** Docker concatena ambos arrays. El comando real que se ejecuta en el sistema es: `ENTRYPOINT+CMD` (`newrelic-admin run-program uvicorn main:app --host 0.0.0.0 --port 8000`)
- Ahora generaremos el contenedor Docker:
	
```bash
docker build -t python_newrelic:latest ./newrelic
```
	
```bash
docker build -t nr_py_test:v1 ./src
```
- Es importante que se ejecute en este orden ya que `src/dockerfile` manda a llamar la imagen de New Relic que genera el primer comando.
6. Ejecutamos el contenedor con:
```bash
docker run -e NEW_RELIC_LICENSE_KEY={LICENCE_KEY} -e NEW_RELIC_APP_NAME="PruebaNR" -p 8000:8000 -it --rm --name CONTAINER-NAME nr_py_test:v1
```

#### Pruebas
Una vez se haya generado el contenedor ejecutaremos un par de pruebas

1. Con Postman (o en el apartado de Docs de FastAPI), usando `http://localhost:8000` haremos consultas `GET` a `/`, `/item/{int}`  y `/fallo` y estos deben devolver resultados.
	1. Para generar un tráfico, se recomienda ejecutar la consulta a `/` y `/fallo` unas 10 o 15 veces.
2. Con la terminal generaremos tráfico usando este comando:
```bash
   for ($i=0; $i -lt 50; $i++) { curl http://localhost:8000/items/$i }
```
Una vez concluidas las pruebas, en el Dashboard de New Relic deberíamos poder ver los gráficos con el tráfico que generamos.

#### Consultas con New Relic Query Language (NRQL)
En la parte inferior del dashboard encontraremos que hay un logo de terminal `>_` que dice *Query Your Data*

Al hacer click nos mostrará una interfaz que nos permite escribir código para hacer las consultas

Ejecutaremos estos comandos:

```SQL
SELECT * FROM Transaction SINCE 1 hour ago LIMIT 10
```
- Este nos mostrará toda la estructura de la captura que realiza el agente.

```SQL
SELECT count(*) FROM Transaction FACET response.status SINCE 1 hour ago
```
- Este nos mostrará gráficos con la cantidad de respuestas html que capturó el agente.

> [!info] Nota
> Abajo de la sección dónde se escriben las consultas está el espacio de resultados, este en su parte superior derecha muestra un menú dropdown que nos permite cambiar la visualización de los datos entre gráficos, JSON y Tablas

### Los Gráficos de New Relic
En el dashboard de New Relic encontraremos los siguientes gráficos:

- **Web Transactions Time**: Es una agrupación de toda la actividad que recibe el servicio hasta las respuestas que este envía.
	- ***¿Qué muestra este gráfico?***
		La vista por defecto muestra el promedio de los valores. <mark style="background:rgba(74, 82, 199, 0.2)">La línea Azul Oscura de tiempo de respuesta es el promedio total</mark>. <mark style="background:rgba(5, 117, 197, 0.2)">La línea azul clara muestra las transacciones de aplicación</mark>, <mark style="background:rgba(240, 200, 0, 0.2)">la amarilla es para transacciones de bases de datos</mark> y<mark style="background:rgba(3, 135, 102, 0.2)"> la verde para servicios externos (APIs o Microservicios adicionales)</mark>.
- **Apdex Score**: Es un estándar de industria que mide la satisfacción de los usuarios según el tiempo de respuesta de la aplicación o servicio. Se representa de 0 a 1.
	- ***¿Qué se considera buen puntaje ?***
		Mientras el puntaje se aproxima a 1, mejor es el desempeño de la app. El valor por defecto de una experiencia satisfactoria es de 0.5 segundos por acción. pero se puede cambiar en los ajustes.
- **Throughput (Rendimiento)**: Es una forma de medir la cantidad de trabajo que el servicio está manejando. Mide cuantas solicitudes se procesan por minuto.
	- ***¿Cómo usar el throughput para arreglar errores?***
		Si notas problemas en la gráfica de transacciones web, puedes compararla con el rendimiento. Si el rendimiento es muy bajo durante un periodo de degradación de servicio, podría indicar que el servicio está procesando más trabajo del que puede manejar.
- **Error**:
	- ***¿Qué muestra este gráfico?***
		Muestra el porcentaje de transacciones que resultaron en error.
- **Transactions**:
	- ***¿Qué muestra este gráfico?***
		  Por defeco muestra las 5 transacciones más lentas en todo el tiempo de vida del servicio.
- **Logs**:
	- ***¿Qué muestra este gráfico?***
		  Nos permite analizar los registros que ha generado el agente APM en el servicio.
- **Distributed tracing insights (*Función preliminar*)**: Esta función utiliza lógica de calificación. Su precisión y relevancia se define conforme al uso y el tiempo.  
	- ***¿Cómo nos ayuda?***
		Esta vista ayuda a identificar señales de cambios significantes en las entidades relacionadas en el entorno que podrían estar impactando el desempeño del servicio.
- **Infrastructure**: Explora las métricas que los proveedores y contenedores dónde se encuentra alojado el servicio. Se utiliza para buscar problemas con la infraestructura del servicio.