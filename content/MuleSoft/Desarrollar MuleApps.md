---
tags:
  - MuleSoft
---
### Introducción
Una Mule App se compone de **Flujos**. Un flujo tiene tres partes:

1. **Source (Fuente/Disparador):** ¿Qué inicia el proceso?
    - Ejemplo: Un `HTTP Listener` (recibe una petición web), un `Scheduler` (se ejecuta cada 5 min), o un `Salesforce Trigger` (cuando se crea un cliente nuevo).
2. **Processors (Procesadores):** La lógica paso a paso.
    - Aquí arrastramos componentes de la paleta: `Database Select`, `HTTP Request`, `Logger`.
    - El mensaje viaja de izquierda a derecha.
3. **Error Handling (Manejo de Errores):** ¿Qué pasa si algo falla?
    - Cada flujo tiene su propia sección de errores. Si la base de datos se cae, el flujo se detiene y entra aquí para enviar una respuesta controlada.
#### Mule Events (El Mensaje)
Esto es vital para que los desarrolladores entiendan qué están manipulando. Un evento en Mule tiene dos partes principales:

1. **Attributes (Metadatos):** Información técnica _sobre_ el mensaje.
    - Ejemplo: Headers HTTP, Query Parameters (`?id=123`), Status Code.
2. **Payload (Carga útil):** La información real del negocio.
    - Ejemplo: El JSON del cliente `{"nombre": "Juan"}` o el resultado de una consulta SQL.
Además podemos usar **Variables (Vars):** Espacios de memoria temporal para guardar datos mientras el mensaje viaja por el flujo.

#### Transformar Mensajes
Es el componente más poderoso: **Transform Message**.
- **Función:** Traducir datos entre formatos (JSON a XML, CSV a Java, etc.).
- **Lenguaje:** Usa **[[DataWeave]] 2.0**, un lenguaje funcional diseñado específicamente para integración.
- **Ejemplo:**
    - _Input:_ `{"user": "Ana"}` (JSON)
    - _Script:_
	
``` DataWeave
%dw 2.0
output application/xml
---
{
  cliente: payload.user
}
```
- ***Output***:
	
```xml
<cliente>Ana</cliente>
```
#### Conectores (Connectors)
MuleSoft cuenta con conectores pre-construidos. En lugar de escribir 500 líneas de código Java para conectarte a [[Comparativa de Bases de Datos|Bases de datos]], arrastras el conector "Database Config".

- **Configuración Global:** Se configura una vez (URL, Usuario, Password) en la pestaña "Global Elements".
- **Uso:** Se reutiliza en múltiples flujos referenciando esa configuración global.
#### Pruebas Unitarias (MUnit)
- **MUnit:** Es el framework nativo de pruebas.
- **Mocking:** Permite simular sistemas externos. (*Ejemplo*: "Finge que Salesforce respondió 'OK' para probar mi lógica sin conectarme realmente a Salesforce").
- **Coverage:** Te dice qué porcentaje de tu código (procesadores) fue ejecutado durante la prueba.

### Generar MuleApps con RAML
Es posible generar archivos [[RAML]] e importarlos a Anypoint Studio para generar *MuleApps* rápidamente.

### Archivos de Configuración
Cuando pasamos de *Desarrollo* a ***Producción***, no queremos tener que recompilar nuestro `.jar` solo para cambiar la dirección de una base de datos o una contraseña. La regla de oro en integración es: **"El código se compila una vez, pero se configura muchas veces"**. 

A continuación los pasos para generar un archivo de configuración:

1. Crear el archivo de configuración (En Anypoint Studio)
	Primero, definiremos nuestras variables en un archivo externo.
   - En tu proyecto, ve a la carpeta `src/main/resources`.
	   - Clic derecho -> **New** -> **File**.
	   - Nómbralo `config.yaml` (YAML es el estándar moderno, aunque `.properties` también funciona, YAML es más limpio).
	   - Agrega el siguiente contenido de ejemplo (imaginemos que configuramos un mensaje de saludo):
```yaml
app:
  message: "Hola desde mi computadora local (Desarrollo)"
  db:
    host: "localhost"
```
	
2. Conectar el archivo a tu Mule App
	Mule no sabe que ese archivo existe hasta que se lo dices.
   - Ve a tu archivo XML principal (tu flujo).
	   - Haz clic en la pestaña **Global Elements** (abajo del canvas).
	   - Clic en **Create** -> busca **Configuration properties**.
	   - En "File", selecciona o escribe `config.yaml`.
	   - Clic en OK.
3. Usar la variable en tu código (Placeholders)
   - Ahora, en lugar de escribir el texto fijo, usamos la sintaxis `${...}`.
	   - Arrastra un componente **Set Payload** a tu flujo.
	   - En la configuración del Payload, activa el modo _Literal_ (o _Expression_ si usas comillas) y pon esto: `${app.message}`
	   - Si corres la app ahora en tu PC, el payload será: _"Hola desde mi computadora local..."_.
    
2.  Configurar en [[CloudHub]] (Runtime Manager)
	Aquí es donde sobrescribes ese valor sin tocar el código.
	   - Cuando vas a desplegar la aplicación (como vimos en la respuesta anterior), hay una pestaña crucial llamada **Properties**.
		   - En la ventana de despliegue de **Runtime Manager**, ve a la pestaña **Properties**.
		   - Aquí puedes definir pares Clave-Valor que **reemplazarán** o complementarán lo que hay en tu `config.yaml` (dependiendo de cómo configures la prioridad, pero generalmente *CloudHub* inyecta estas como variables de sistema).
		   - Escribe lo siguiente:
```toml
app.message = Hola desde el servidor de Producción en CloudHub
```
- Observa que aquí usamos la notación de puntos (`app.message`) en lugar de la indentación de YAML.
	
5. Haz clic en **Deploy**.

> [!info] Nota
> Cuando la aplicación arranque en la nube, Mule leerá la propiedad `app.message`. Aunque el archivo `config.yaml` dentro del JAR dice "Hola desde mi computadora...", la plataforma *CloudHub* inyectará el valor "Hola desde el servidor...", y ese será el que vean tus usuarios.

#### Resumen de la arquitectura de configuración

| Ubicación                 | Acción                    | Valor Ejemplo            |
| ------------------------- | ------------------------- | ------------------------ |
| **Código (XML)**          | Referencia                | `${db.host}`             |
| **Archivo (YAML)**        | Valor por defecto         | `localhost`              |
| **CloudHub (Properties)** | **Valor Real (Override)** | `prod-db.aws.amazon.com` |
Ahora sabemos cómo cambiar valores. 
### Encriptación de Contraseñas
Si pones una contraseña en la pestaña _Properties_ de *CloudHub*, cualquier persona con acceso a la consola podrá verla en texto plano. Eso es un riesgo de seguridad grave.

Para eso existe **Mule Secure Configuration Properties** (encriptar las contraseñas para que en *CloudHub* se vean como `![x89sfd7s98f...]`).

La lógica funciona así:

1. **Encriptar el valor localmente**:
	Necesitas una herramienta (un archivo `.jar`) que MuleSoft provee llamada `secure-properties-tool.jar`. Se usa desde la línea de comandos (CMD o Terminal).
	Supongamos que tu contraseña real es: `Secreto123` Y tu "Llave Maestra" (que inventas tú) será: `MiLlaveMaestra123`
	El comando se ve así:
``` bash
java -jar secure-properties-tool.jar string encrypt Blowfish CBC "MiLlaveMaestra123" "Secreto123"
```
- El resultado será algo ilegible, por ejemplo: `q8A/s9d8s7f9s8d7f==`
	
2. ***Configurar el YAML en Anypoint Studio***
	En tu archivo `config.yaml` (o `secure-config.yaml`), ya no pones la contraseña real. Usas el resultado anterior envuelto en `![ ]`.
	
``` yaml
db:
  host: "prod-db.aws.com"
  # Ojo a los signos de exclamación y corchetes
  password: "![q8A/s9d8s7f9s8d7f==]" 
```
	
3. **Configurar el Componente de Seguridad en Anypoint Studio**
	Mule no sabe leer `![...]` por defecto. Necesitas agregar el módulo.
	- Ve a **Mule Palette** -> **Search in Exchange** -> busca "Secure Configuration Properties" y agrégalo.
	- Ve a **Global Elements** -> **Create** -> **Secure Configuration Properties**.
	- Configúralo así:
	    - **File:** `config.yaml`
	    - **Key:** `${mule.key}` <-- <mark style="background:#fdbfff">Aquí no pongas la llave real. Ponemos una variable (placeholder)</mark>.
	    - **Algorithm:** `Blowfish` (o el que usaste).
	    - **Mode:** `CBC`.
4. Referenciar en el código (La sintaxis cambia)
	- Para propiedades normales usas: `${db.host}`
	- Para propiedades seguras usas el prefijo `secure`: `${secure::db.password}`
    Si olvidas el `secure::`, Mule intentará leer la cadena literal `![q8A/...]` y fallará la conexión.
    
5. Desplegar en *CloudHub*
	Subes tu aplicación (JAR) a Runtime Manager. Pero ahora, la aplicación **no arrancará** porque le falta la llave para abrir la caja fuerte.
	- Ve a la pestaña **Properties** en *CloudHub*.
	- Agrega la variable que definimos en el Paso 3:
```toml
mule.key = MiLlaveMaestra123
```
- Haz click en *Deploy*
#### Resumen del flujo:
Cuando la réplica de *CloudHub* arranca:
1. Lee `mule.key`.
2. Va al archivo `config.yaml`.
3. Encuentra `![q8A/...]`.
4. Usa `mule.key` para desencriptarlo.
5. En la memoria del servidor, inyecta `Secreto123` en el conector de base de datos.
6. Si alguien entra a ver el archivo `config.yaml` o los logs, solo verá la cadena encriptada.

> [!help] ¿La llave `mule.key` queda Expuesta?
> En **CloudHub 2.0**: Las _Protected Properties_ se gestionan de forma nativa y se cifran al guardarse.
> 
> Anteriormente en **CloudHub 1.0**: En la pestaña Properties, había que hacer clic en el botón "Protect" (candado) al lado del valor `mule.key`. Esto la ocultaba visualmente para siempre (nadie podrá copiarlo después). 
