---
tags:
  - AWS
---
AWS Lambda es el núcleo de la computación _serverless_ (sin servidor) en Amazon Web Services. Te permite ejecutar código en respuesta a eventos sin tener que aprovisionar ni administrar servidores físicos o virtuales. Simplemente subes tu código, escrito en lenguajes como Python o Java, y Lambda se encarga automáticamente de la escalabilidad, el mantenimiento y la ejecución de alta disponibilidad, cobrando solo por los milisegundos de cómputo que realmente consumes.

### Filosofía Serverless

El término "sin servidor" es un poco engañoso. Los servidores siguen existiendo, pero la filosofía se basa en **transferir toda la responsabilidad de la infraestructura al proveedor de la nube**.

Solo te enfocas en escribir la lógica de negocio (tu código en Python, Java, etc.). AWS se encarga de asignar el servidor, actualizar el sistema operativo, ejecutar tu código y liberar los recursos en el instante en que tu código termina de ejecutarse. La ventaja financiera principal del modelo _Serverless_ es la capacidad de escalar a cero.

Si de repente llegan 10,000 peticiones al mismo tiempo, AWS automáticamente crea 10,000 copias independientes de tu entorno de ejecución (micro-contenedores) repartidas en su inmensa red de servidores. Cada copia atiende una petición individual de principio a fin. Cuando terminan, esos entornos se destruyen o se pausan.


### Cold Start y Warm Start
Supongamos que nuestro servicio no ha recibido solicitudes, en lo que se inicializa, hace las conexiones necesarias y encuentra un servidor disponible toma tiempo adicional, a este arranque se le conoce como *Cold Start*. 

Una vez se ha completado el proceso de arranque las peticiones siguientes se ejecutan más rápido ya que el entorno está montado y listo para trabajar. Como solamente hay que procesar la solicitud entrante lo llamamos *Warm Start*.

Para explicar mejor veamos un ejemplo entre tener el código en *Python* y *Java*:

Mientras que el intérprete de *Python* es ligero y se inicia en milisegundos, *Java* necesita levantar toda la *Máquina Virtual de Java (JVM)*, asignar memoria y cargar las clases en memoria antes de poder ejecutar la primera línea de tu lógica de negocio.

Por esta razón, en el mundo real vemos que *Python* o *Node.js* suele ser el favorito para Lambdas con tráfico muy impredecible o picos repentinos, mientras que Java brilla en Lambdas que procesan datos más pesados y se mantienen "calientes" constantemente. (AWS introdujo una tecnología llamada _SnapStart_ para mitigar esto en Java, guardando una imagen de la JVM ya iniciada).

### Anatomía
las AWS Lambda tienen un componente principal llamado `handler`. Este manejador es la entrada de AWS para ejecutar el código.

Por convención, en Python suele llamarse `lambda_handler`, y su estructura más básica siempre recibe dos parámetros fundamentales:
```python
def lambda_handler(event, context):
    # Aquí va tu lógica de negocio
    return "¡Hola desde AWS Lambda!"
```
Esta función recibe dos parámetros `event` y `context`, los cuales se encargan de:
1. **`context` (El Contexto):** Es un objeto que AWS te inyecta con información sobre la ejecución misma. Te dice cosas como cuánta memoria tienes asignada, cuál es el ID de esta ejecución, o cuántos milisegundos te quedan antes de que AWS apague la función por límite de tiempo.
2. **`event` (El Evento):** Este es el más importante para tu lógica de negocio. Es un objeto (usualmente un diccionario en Python) que contiene todos los datos sobre **qué** desencadenó la ejecución de tu código.

#### Triggers
Para poder activar la lambda se utilizan *Triggers*, esos son la configuración que le dice a AWS: "Acaba de ocurrir este evento específico, despierta a la función". Algunos ejemplos de esto son:

- **Actualizar un registro:** Las bases de datos (como Amazon DynamoDB) pueden emitir un evento cada vez que un dato cambia en una tabla, disparando tu Lambda inmediatamente.
- **Ubicación GPS:** Servicios de localización detectan cuando un dispositivo entra a una zona geográfica específica (Geofencing) y despiertan tu código.
- **Excepción en un contenedor:** El servicio de monitoreo (Amazon CloudWatch) lee los logs, detecta el error y lanza la Lambda. Luego, tu código puede ejecutar la lógica para enviar el SMS.

#### Librerías
Para instalar librerías normalmente usamos comandos como `pip install`, pero esto tomaría demasiado tiempo si cada micro servicio tuviera que realizar instalaciones de dependencias en el Cold Start. 

AWS nos pide que le entreguemos todo listo para ejecutarse de inmediato, creando un **Paquete de Despliegue** (Deployment Package). Descargamos todas las librerías en una carpeta local junto a al código fuente, y lo comprimimos en un archivo **.zip**. Ese `.zip` es lo que se sube a AWS

Como subir archivos `.zip` gigantes cada vez que cambias una línea de código es tedioso, AWS creó una solución elegante llamada **Lambda Layers** (Capas). Una capa te permite empaquetar tus librerías (como `requests` o `pandas`) en un `.zip` separado y adjuntarlo a tu función. Así, tu código principal se mantiene súper ligero.

#### Timeout
AWS **no** permite que una función se ejecute para siempre para protegerte de cobros infinitos. A esta regla de protección se le conoce como ***Timeout*** (Tiempo de espera límite).

Por defecto, cuando creas una Lambda, este límite suele venir configurado en tan solo **3 segundos** (ya que la filosofía es hacer tareas ultrarrápidas). Sin embargo, se puede modificar esta configuración hasta un máximo absoluto de **15 minutos**.

Si el código entra en un ciclo infinito, o si intenta procesar un archivo gigante y el reloj llega a ese límite que configuraste, AWS "mata" el micro-contenedor de inmediato, detiene la ejecución y registra un error.

### Ejemplo
Veamos un pequeño ejemplo, tenemos un sistema que recibe un archivo CSV de ventas en la nube, este se pasa a la lambda y esta hace la suma del total de las ventas del día. Este servicio utiliza todos los servicios de AWS:

El archivo llega al **Bucket S3** --> S3 emite el evento --> Despierta a la **Lambda (Python)** --> Lambda procesa el CSV y guarda en **DynamoDB**.

Para que Python pueda "hablar" con otros servicios de AWS (como S3 y DynamoDB), necesitamos usar una librería oficial llamada **`boto3`** (el SDK de AWS para Python). La gran ventaja es que AWS ya incluye esta librería por defecto en todas las Lambdas de Python:

``` Python
import boto3
import csv

# Inicializamos las conexiones a los servicios de AWS
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
tabla = dynamodb.Table('ReportesVentas')

def lambda_handler(event, context):
    # 1. El Trigger de S3 nos inyecta los datos en el "event". 
    # Extraemos el nombre del Bucket y del archivo.
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    file_key = event['Records'][0]['s3']['object']['key']
    
    # 2. Leemos el archivo CSV directamente desde S3
    respuesta_s3 = s3_client.get_object(Bucket=bucket_name, Key=file_key)
    lineas_csv = respuesta_s3['Body'].read().decode('utf-8').splitlines()
    
    # 3. Procesamos el CSV (ej. sumamos la columna 'Monto')
    total_ventas = 0
    lector = csv.DictReader(lineas_csv)
    for fila in lector:
        total_ventas += float(fila['Monto'])
        
    # 4. Guardamos el resultado final en nuestra base de datos NoSQL
    tabla.put_item(
        Item={
            'ArchivoID': file_key,
            'TotalVentas': str(total_ventas)
        }
    )
    
    # Retornamos un mensaje de éxito
    return {
        'statusCode': 200,
        'body': f'Procesamiento exitoso del archivo {file_key}. Total: {total_ventas}'
    }
```

### Seguridad
En el código que vimos, simplemente le dijimos a Python: "Conéctate a S3, lee el archivo y luego guarda este dato en DynamoDB". El código no tenía contraseñas ni claves de acceso explícitas.

La razón por la que esto funciona es porque AWS utiliza un servicio central llamado **IAM** (Identity and Access Management). En lugar de escribir contraseñas en tu código (lo cual es una muy mala práctica), le asignas a tu función Lambda un **Rol de Ejecución** (Execution Role).

Piensa en el Rol de Ejecución como un gafete de empleado. Cuando la Lambda intenta leer el archivo S3, AWS revisa su "gafete" para ver si tiene los permisos necesarios antes de dejarla pasar. Si no tiene el gafete correcto, la petición es rechazada inmediatamente con un error de "Acceso Denegado".

En ciberseguridad y arquitectura de nube existe un concepto fundamental llamado el **Principio del menor privilegio** (Principle of least privilege).

En AWS IAM, esta especificidad se logra adjuntando **Políticas** (Policies) al Rol de Ejecución. Estas políticas son simplemente documentos de texto en formato JSON que definen reglas estrictas.

En lugar de darle acceso total a S3, la política para nuestra función se vería así:
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::VentasDiarias/*"
}
```

