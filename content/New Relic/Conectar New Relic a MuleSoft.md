---
tags:
  - NewRelic
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
### Pasos para la Integración
Para integrar [[New Relic]] con [[MuleSoft]] entramos al [dashboard](https://one.newrelic.com/marketplace) de New Relic y buscamos la integración con *Mule ESB*

1. Escogemos Instalación para Host  
2. Generamos las *Licence Key* y *User Key*. 
3. Después se nos pedirá especificar nuestro sistema operativo (en este caso Windows) y tendremos que ejecutar el comando:

```bash
# PowerShell command
(Get-Command java | Select-Object -ExpandProperty Version).tostring()
```
- Este comando verifica la instalación de Java en el Equipo. Después preguntará sobre el *Framework* que usamos, escogemos "*Other*".
4. Nos pedirá descargar el agente, nos dá un comando pero abajo encontraremos un enlace que nos redirige a la [descarga manual](https://docs.newrelic.com/docs/release-notes/agent-release-notes/java-release-notes/)
	- Una vez que descarguemos el agente, creamos una carpeta para este en la ruta `C:/NewRelic` y descomprimimos el archivo `.zip`
	- Nos pedirá ingresar un nombre a la app, en este caso `newrelictest` y después nos generará un archivo `.yml` que descargaremos y remplazaremos en la ruta de instalación de New Relic (`C:/NewRelic/newrelic`)
5. Entramos a ***Anypoint Studio***, creamos una nueva ruta de *Workspace* `../../Anypoint Workspaces\NewRelicTest`
6. Creamos un nuevo `Mule Project`, le damos un nombre, en este caso `newrelictest`
7. Vamos a `Run --> Run Configurations`
	- En la ventana vamos a `Mule Applications ---> New_Configuration`
	- Cambiamos a la pestaña `Arguments`
	- En `VM Arguments` colocamos `java -javaagent:"C:\{Ruta de New Relic}\newrelic.jar"`
	- Aplicamos los cambios
8. Generamos una MuleApp con este código
```xml
<?xml version="1.0" encoding="UTF-8"?>

  

<mule xmlns:ee="http://www.mulesoft.org/schema/mule/ee/core"

xmlns:http="http://www.mulesoft.org/schema/mule/http"

xmlns="http://www.mulesoft.org/schema/mule/core"

xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"

xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"

xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd

http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd

http://www.mulesoft.org/schema/mule/ee/core http://www.mulesoft.org/schema/mule/ee/core/current/mule-ee.xsd">

  

<http:listener-config name="HTTP_Listener_config" doc:name="HTTP Listener config" >

<http:listener-connection host="0.0.0.0" port="8081" />

</http:listener-config>

  

<flow name="get-root-flow">

<http:listener doc:name="GET /" config-ref="HTTP_Listener_config" path="/"/>

<ee:transform doc:name="Respuesta JSON">

<ee:message >

<ee:set-payload ><![CDATA[%dw 2.0

output application/json

---

{

"Hello": "Mundo",

"Framework": "MuleSoft",

"Monitor": "New Relic"

}]]></ee:set-payload>

</ee:message>

</ee:transform>

</flow>

  

<flow name="get-item-flow">

<http:listener doc:name="GET /items/{item_id}" config-ref="HTTP_Listener_config" path="/items/{item_id}"/>

<ee:transform doc:name="Procesar item_id">

<ee:message >

<ee:set-payload ><![CDATA[%dw 2.0

output application/json

---

{

"item_id": attributes.uriParams.item_id,

"status": "monitored"

}]]></ee:set-payload>

</ee:message>

</ee:transform>

</flow>

  

<flow name="get-failure-flow">

<http:listener doc:name="GET /fallo" config-ref="HTTP_Listener_config" path="/fallo"/>

<raise-error doc:name="Simular Crash" type="APP:SIMULATED_FAILURE" description="Simulando división por cero o fallo crítico"/>

<ee:transform doc:name="Payload inalcanzable">

<ee:message >

<ee:set-payload ><![CDATA[%dw 2.0

output application/json

---

{ "resultado": "Esto no se verá" }]]></ee:set-payload>

</ee:message>

</ee:transform>

</flow>

  

</mule>
```
- La ejecutamos
9. Una vez desplegada la MuleApp, seguimos en la configuración de New Relic, damos a continuar a las funciones opcionales y probamos la conexión.
10. Ya podemos hacer pruebas a esta MuleApp

### Pruebas

1. Con Postman, usando `http://localhost:8081` haremos consultas `GET` a `/`, `/item/{int}`  y `/fallo` y estos deben devolver resultados.
	1. Para generar un tráfico, se recomienda ejecutar la consulta a `/` y `/fallo` unas 10 o 15 veces.
2. Con la terminal generaremos tráfico usando este comando:
```bash
   for ($i=0; $i -lt 50; $i++) { curl http://localhost:8081/items/$i }
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

