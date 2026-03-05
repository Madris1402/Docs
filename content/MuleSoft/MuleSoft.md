---
tags:
  - MuleSoft
---
MuleSoft es una empresa (propiedad de Salesforce) que ofrece una plataforma de integración llamada **Anypoint Platform**. Tiene la capacidad de conectar cualquier sistema, aplicación o fuente de datos.

### Conceptos Básicos
Imagina que tienes un sistema viejo (Legacy) que escrito en `BASIC`, una base de datos moderna `AWS S3` y una aplicación móvil en `Flutter`. MuleSoft actúa como el **traductor universal** en el medio. Recibe la información de uno, la transforma y la entrega al otro en el formato que necesita, todo en tiempo real.

- **Sin MuleSoft (Integración Punto a Punto):** Si tienes 10 sistemas y los conectas todos contra todos, terminas con un "plato de espagueti". Si uno cambia, todo se rompe. Es frágil y difícil de mantener.

- **Con MuleSoft (Application Network):** Creas una red ordenada donde cada sistema tiene un conector estandarizado (*API*). Si cambias un sistema, solo cambias su enchufe, no toda la red.

Para que esto funcione se utiliza la filosofía [[API-Led Connectivity]].

### Herramientas Principales
- **Anypoint Platform:** El sitio web (panel de control) desde donde gestionas todo.
  
- **Anypoint Studio:** El software de escritorio (basado en Eclipse) donde los desarrolladores escriben el código (arrastrando y soltando cajitas).
  
- **Mule Runtime Engine:** El motor que hace correr las aplicaciones (es ligero y basado en Java).
  
- **DataWeave:** El lenguaje de programación propio de MuleSoft para transformar datos (ej. convertir un XML a JSON). Es extremadamente potente.
  
- **Exchange:** Es como una "App Store" privada de tu empresa. Ahí publicas tus APIs para que otros desarrolladores las reutilicen en lugar de crear código nuevo.


### Estructura de la Información
#### Mule Event
Es como una caja, contiene todos los componentes que utiliza la *MuleApp*. Transportan información de forma ordenada de un punto a otro.

Dentro de él se encuentran los ***Mule Message*** y las ***Variables***.
#### Mule Message
Es la información principal que entra o sale de un flujo, está compuesto por dos elementos:
- ***Payload***: Es el contenido útil del evento, si consultamos una base de datos, los registros de esa consulta serán el *payload*. Si es la respuesta de un formulario web, los datos en JSON serán el payload.
- ***Attribute***: Son los metadatos sobre el *Payload*. Se guardan cosas como *HTTP Headers*, *Parámetros URI*, el tamaño de un archivo o rutas originales.
#### Variables
Sirven para almacenar contenido del Payload de manera persistente durante el flujo. El desarrollador tiene control total sobre ellas, normalmente almacenamos atributos del *Mule Message*, pero también podemos almacenar datos directos del payload.

Almacenar ciertos atributos o parámetros en variables nos sirve si nuestro flujo hace consultas a bases de datos, alteraría todo el *Mule Message* y perderíamos información crucial para regresar la respuesta correctamente.
### Control de Flujo
#### Scatter-Gather
Toma un único *Mule Event* y lo esparce enviando copias exactas a diferentes rutas que se ejecutan en paralelo.

Al finalizar el proceso, el resultado se reúne en una sola respuesta:
- El Payload se convierte en un objeto especial que contiene una colección de resultados.
- Tiene una estructura similar a un diccionario dónde las rutas están enlistadas con su resultado.

```JSON
{
	"Ruta0" :{
		"resultado": "0",
		"hora": "17:00:00"
	},
	"Ruta1" :{
		"resultado": "1"
		"hora": "17:00:02"
	},
	"Ruta2" :{
		"resultado": "2"
		"hora": "17:00:03"
	}
}
```
#### Batch Job
Se utiliza para trabajos pesados, como migraciones de registros. Este divide las cargas de trabajo enormes en bloques pequeños que se procesan de forma asíncrona, usando *Batch Steps*.

Al ser asíncrono, cuando el flujo principal termina de pasarle los datos al Batch Job, el flujo principal continúa inmediatamente.

Cuando sale del componente de Batch Job, <mark style="background:#fdbfff">el payload original desaparece</mark> y se transforma en un objeto llamado *Batch Job Result*.

Este objeto es solo un **resumen o reporte de metadatos** que dice:

- `totalRecords`: Cuántos registros entraron.
- `successfulRecords`: Cuántos se procesaron con éxito.
- `failedRecords`: Cuántos fallaron.

El Payload resultante de un Batch Job _nunca_ contiene los datos procesados. Si necesitas guardar los resultados de cada cliente (por ejemplo, el ID nuevo de Salesforce), debes hacerlo _dentro_ de los Batch Steps (ej. guardándolos en una base de datos o enviándolos a una cola de mensajes), porque al terminar el bloque, esos datos ya no estarán en el Payload principal.

### Manejo de Errores
Cuando algo falla en MuleSoft (por ejemplo, una base de datos no responde), el flujo normal se interrumpe y el Mule Event es enviado a la sección de manejo de errores. Para ello hay dos estrategias principales
#### On Error Propagate
Atrapa el error, te permite ejecutar acciones (como guardar un registro en consola), pero al finalizar, **vuelve a lanzar** el error hacia quien haya llamado a ese flujo. En el contexto de una API, el cliente final recibiría un código de error (como un HTTP 500).

#### On Error Continue
Atrapa el error, ejecuta tus acciones correctivas, pero al finalizar, "suprime" el error y le dice al sistema que el evento terminó exitosamente. En una API, el cliente final recibiría un código de éxito (como un HTTP 200 OK), devolviendo el Payload que tú hayas configurado dentro de este bloque.

### Validation Module
Este módulo permite evaluar datos y forzar la generación de un error de manera intencional si no se cumple una regla de negocio (por ejemplo, validar que un email tenga el formato correcto).

Supongamos que utilizamos el módulo _Validation_ para revisar si el Payload entrante contiene el campo "email". Si no viene, generas un error. Quieres atrapar ese error y responderle a la aplicación *front end* con un mensaje JSON `{"mensaje": "Falta el email"}` junto con un código de respuesta HTTP 200 OK para no romper su interfaz.

Al usar **On Error Continue**, atrapas el error del módulo *Validation*, construimos un nuevo Payload con el mensaje amigable y se indica al sistema que termine la ejecución de manera "exitosa" hacia el exterior. Así, el *front end* recibe su `HTTP 200 OK` con el JSON que necesita para mostrar la alerta en pantalla, sin que la aplicación se rompa.