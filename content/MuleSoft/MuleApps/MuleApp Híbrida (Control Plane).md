---
tags:
  - MuleApp
---
Para realizar este proyecto, tomaremos la *MuleApp* del *Laboratorio 06* y la volveremos un entorno híbrido.
### Requisitos
1. ***Tener Instalado Java 17*** *(Mule para Control Plane Exige esta versión)*
	Si no tienes Java 17 usa este comando:
```bash
winget.exe install --id "Oracle.JDK.17" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- Una vez instalado verifica que exista la variable de entorno `JAVA_HOME`
	- En caso de no tener `JAVA_HOME`
	- **Paso 1: Encuentra la ruta de tu Java**
		Primero, necesitamos saber dónde está instalado Java.
		1. Abre el explorador de archivos.
		2. Ve a `C:\Program Files\Java\` (o `Archivos de Programa`).
		3. Deberías ver una carpeta llamada`jdk-17.xxx`.
		4. Entra a esa carpeta.
		5. **Copia la ruta de la barra de direcciones.**
	    - _Debe verse así:_ `C:\Program Files\Java\jdk-17.xxx`
	    - **IMPORTANTE:** Copia la ruta de la carpeta raíz del JDK, **NO** entres a la carpeta `bin`.
	 - Paso 2: Configurar la Variable de Entorno
		1. Presiona la tecla **Windows**, escribe **"Variables de entorno"** y selecciona la opción que dice **"Editar las variables de entorno del sistema"**.
		2. Se abrirá una ventanita pequeña. Haz clic en el botón **"Variables de entorno..."** (abajo a la derecha).
		3. Verás dos secciones. Busca la de abajo: **"Variables del sistema"**.
		4. Haz clic en el botón **"Nueva..."** (debajo del recuadro de Variables del sistema).
		5. Llena los datos así:
		    - **Nombre de la variable:** `JAVA_HOME`
		    - **Valor de la variable:** _(Pega aquí la ruta que copiaste en el paso 1)_.
		6. Dale **Aceptar** a esa ventana, **Aceptar** a la siguiente y **Aceptar** a la última.
		- Paso 3: Reiniciar la Terminal (¡Vital!) 
		Este es el paso que todos olvidan. Windows no actualiza las variables en las terminales que ya están abiertas.
		1. **Cierra** tu PowerShell o CMD actual.
		2. **Abre uno nuevo**.
		3. Verifica que ya lo detecte escribiendo esto:
		    - En CMD: `echo %JAVA_HOME%`
		    - En PowerShell: `echo $env:JAVA_HOME`
2. ***Mule Standalone***
	Mule Standalone se consigue en la página de [Soporte de MuleSoft](https://www.mulesoft.com/lp/dl/anypoint-mule-studio)
	- Selecciona Mule Standalone en el menú drop down y rellena la información adicional
	- **IMPORTANTE**: El correo que ingreses será a donde te enviarán un enlace de descarga de Mule. (La demás información no utiliza ninguna verificación especial para ser considerada válida).
Una vez cumplidos los requisitos procederemos a obtener el Token de vinculación en [Anypoint Platform](https://anypoint.mulesoft.com)
### Obtener el Token de Vinculación (Control Plane)

Primero, necesitamos decirle a la nube que vamos a registrar un nuevo servidor.
1. Inicia sesión en **Anypoint Platform**.
2. Navega a **Runtime Manager** y selecciona el entorno (Environment) Sandbox.
3. En el menú lateral izquierdo, haz clic en **Servers**.
4. Haz clic en el botón azul **Add Server**.
5. Se abrirá una ventana con un comando. **Copia el comando**. Ese token es la llave de acceso.
     **IMPORTANTE:** No cierres esta ventana hasta que terminemos la configuración.
### Instalación y Registro (Runtime Plane)

Ahora vamos a tu servidor local.
1. **Descomprimir:** Descomprime el `.zip` del Mule Runtime en una carpeta ( `C:\Mule`).
2. **Terminal:** Abre tu terminal CMD/PowerShell y navega a la carpeta `bin` dentro del directorio de Mule.
``` powershell
cd C:\Mule\bin
```
3. **Ejecutar AMC Setup:** Aquí usaremos la herramienta "Anypoint Management Center Setup" para inyectar el token.
	
``` powershell
amc_setup.bat -H <TU_TOKEN_COPIADO> <NOMBRE_DEL_SERVIDOR>
```
- *Reemplaza `<TU_TOKEN_COPIADO>` con el hash del paso 2 y `<NOMBRE_DEL_SERVIDOR>` con un nombre único (ejemplo: testHybrid`).*
	Si todo sale bien, verás un mensaje que dice: `Mule Agent configured successfully`.
### Iniciar el Servidor
Ahora que está configurado, necesitamos encender el motor.
``` powershell
.\mule.bat
```

Espera unos minutos. Si regresas a la página de **Anypoint Runtime Manager**, verás que el indicador de estado de tu servidor cambiará de "*Created*" a **"*Running*" (Verde)**.

### Desplegar MuleApp 
Ahora que el canal de comunicación está abierto, puedes desplegar aplicaciones desde la nube hacia tu servidor local.

1. **Prepara tu App:** En Anypoint Studio, haz clic derecho en tu proyecto -> **Export** -> **Anypoint Studio Project to Mule Deployable Archive**. Esto genera un archivo `.jar`.
   - Repetir el proceso para las dos API que generamos en el *Laboratorio 06* (`gsapi-system-api` y `gsapi-places-api`)
   - A parte de estos `.jar` necesitamos crear un proyecto de dominio dentro de MuleSoft
	   - Primero vamos a **File** -> **New** -> ***Mule Domain Project***
	   - En la ventana le ponemos de nombre `default` y lo creamos.
	   - Ahora exportamos con los pasos anteriores el proyecto `default` para obtener otro archivo `.jar`
		   - Este archivo lo colocamos en `C:\Mule\domains`
		   **IMPORTANTE**: Este archivo tiene las instrucciones para que Mule Standalone sepa como enrutar nuestras MuleApps Sin él simplemente no se iniciará la MuleApp en Control Plane.
2. Ve a la pestaña **Applications** en Runtime Manager.
3. Haz clic en **Deploy application**.
4. En **Deployment Target**, cambia la opción de "CloudHub" a **"Hybrid"** o selecciona directamente el nombre de tu servidor (`testHybrid`).
5. Sube tu archivo `.jar` (generado en Anypoint Studio) y dale a **Deploy**.
	- Repetir  los pasos 2 a 5 con cada una de las APIs (`gsapi-system-api` y `gsapi-places-api`)
> [[Exportar MuleApp a JAR.html|Tutorial Paso a Paso]]
#### Resumen de la Arquitectura

| Componente        | Ubicación          | Función                                                                   |
| ----------------- | ------------------ | ------------------------------------------------------------------------- |
| **Control Plane** | Nube de MuleSoft   | Gestión, Monitoreo, Logs (parcialmente), Despliegue.                      |
| **Runtime Plane** | Tu Servidor Local  | Ejecución de la MuleApp, Procesamiento de datos, Conectividad a BD local. |
| **Mule Agent**    | Dentro del Runtime | Es el puente que "habla" con la nube mediante WebSockets seguros.         |
### Integrar New Relic

1. Descargar el Agente (Java Agent)
	- Ve a tu cuenta de **[[New Relic]]**.
	- Ve a **Add Data** > Busca **Java**.
	- Descarga el archivo `.zip` del agente.
	- Descomprímelo. Obtendrás una carpeta llamada `newrelic` que contiene un archivo `newrelic.jar` y un `newrelic.yml`.
	  
2. Instalación en el Servidor
	- Copia esa carpeta completa `newrelic` y pégala dentro de tu carpeta raíz de Mule.
    - Ruta final recomendada: **`C:\Mule\newrelic`**
    
3. Configuración Básica (`newrelic.yml`)
	- Abre el archivo `C:\Mule\newrelic\newrelic.yml` con un editor de texto.
	- Busca la línea: `license_key:`.
	    - Pega ahí tu licencia (la obtienes en la web de New Relic).
	- Busca la línea: `app_name:`.
	    - Cámbialo por un nombre descriptivo, ejemplo: `Mule-Hybrid`.
	- Guarda el archivo.
4. Conectar Mule con el Agente (`wrapper.conf`)
	Este es el paso crítico. Tenemos que decirle a Mule: _"Cuando arranques, carga también este jar de New Relic"_.
	1. Ve a la carpeta de configuración de Mule: **`C:\Mule\conf`**.
	2. Abre el archivo **`wrapper.conf`**.
	3. Busca la sección donde están las líneas `wrapper.java.additional.n`.
	    - Verás algo como:
```toml
wrapper.java.additional.1=-Dfile.encoding=UTF-8
wrapper.java.additional.2=-Djava.net.preferIPv4Stack=true
...
```
- Agrega una **NUEVA** línea al final de esa lista.
    - **IMPORTANTE:** El número (`.n`) debe ser el siguiente consecutivo. Si el último era `.15`, tú pones `.16`.
    La línea a agregar es:
    Properties
```toml
wrapper.java.additional.XX=-javaagent:C:\Mule\newrelic\newrelic.jar
```
- _(Reemplaza `XX` por el número que te toque)._
    
5. Reiniciar y Verificar
	- Detén tu servidor Mule (`Ctrl + C` en la consola o cierra la ventana).
	- Vuelve a ejecutar **`mule.bat`**.
	- **Mira los logs atentamente al arrancar.**
	    - En las primeras líneas, deberías ver algo como:
	    	`INFO com.newrelic - New Relic Agent: Loading configuration file "C:\Mule\newrelic\newrelic.yml"`
	- Si ves esto, *New Relic* ya está funcionando y enviando datos
6. Verlo en el Dashboard
	- Ve a la web de **New Relic**.
	- Ve a la sección **APM & Services**.
	- En unos 2-3 minutos, debería aparecer tu aplicación `Mule-Hybrid`.
	- Haz algunas peticiones desde Postman para generar tráfico.
	    - Verás las transacciones web (`/api/place`).
	    - Verás los tiempos de respuesta de la base de datos (o RabbitMQ si el driver es compatible).
	    - Verás errores (si provocas alguno).

7. Monitoreo de JMX (Opcional)
	Por defecto, New Relic te muestra tiempos de respuesta HTTP y uso de CPU. Pero si quieres ver cosas específicas de Mule (como _"cuántos mensajes hay encolados en el flow"_), necesitas activar las métricas JMX en el `newrelic.yml`:
	
	En `C:\Mule\newrelic\newrelic.yml`, busca y habilita:
	
```YAML
browser_monitoring:
  auto_instrument: true
jmx:
  enabled: true
```
