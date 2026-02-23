---
tags:
  - MuleSoft
---
Es una ***iPaaS*** (*Integrated Platform as a Service*) totalmente gestionada por [[MuleSoft]]. En este se despliegan *MuleApps*.

### Réplicas
Cuando se despliegan aplicaciones en CloudHub:
- **Aislamiento**: Cada aplicación corre en una instancia virtual separada. Las *réplicas* están basadas en contenedores (*Kubernetes*). (Anteriormente se llamaban *Workers* y eran instancias independientes de *EC2 AWS*).
- **Seguridad**: Al ser contenedores independientes, si una consume toda su memoria o se cae no afecta a las demás.
- **Escalabilidad**:
	- **Vertical**: Aumentamos la potencia del *vCore*.
	- **Horizontal**: Se agregan más réplicas para la misma App. CloudHub balancea la carga.
- **Actualizaciones**: CloudHub gestiona las actualizaciones del *Mule Runtime*.

### Despliegue de Aplicaciones

#### Desde Anypoint Studio
Este método es ideal cuando estás probando en entornos de Desarrollo o Sandbox.

1. En el **Package Explorer**, haz clic derecho sobre tu proyecto.
    
2. Navega a **Anypoint Platform** -> **Deploy to CloudHub**.
    
3. Si no has iniciado sesión, te pedirá tus credenciales de Anypoint.
    
4. Se abrirá una ventana de configuración:
    
    - **Deployment Target:** Elige "CloudHub" (o CloudHub 2.0 si tu organización ya migró).
        
    - **Application Name:** Debe ser único en todo el mundo (ej: `mi-api-v1-dev`).
        
    - **Environment:** Selecciona `Sandbox` o `Design`.
        
    - **Worker Size:** Para pruebas, usualmente `0.1 vCore` o `0.2 vCore` es suficiente (Micro o Small).
        
5. Haz clic en **Deploy Application**.
    

Studio subirá el archivo, provisionará el Worker y arrancará la app. Puede tardar unos minutos.
#### Desde el Runtime Manager
Este es el proceso formal. Primero necesitas generar el ejecutable.

**Paso 1: Generar el JAR**

1. En Studio, clic derecho al proyecto -> **Export**.
    
2. Selecciona **Mule** -> **Anypoint Studio Project to Mule Deployable Archive**.
    
3. Asegúrate de marcar "Include project modules and dependencies". Esto generará un archivo `.jar`.

**Paso 2: Subir a la Nube**

1. Entra a [Anypoint Platform](https://anypoint.mulesoft.com).
    
2. Ve al menú de la izquierda y selecciona **Runtime Manager**.
    
3. Elige el entorno (ej. Sandbox) y haz clic en el botón azul **Deploy application**.
    
4. Llena los datos:
    
    - **Name:** El nombre de tu app.
        
    - **Application File:** Sube el `.jar` que exportaste.
        
    - **Runtime Version:** Elige la misma versión que usaste en Studio.
        
    - **Worker Size:** 0.1 vCore (para empezar).
        
5. Haz clic en **Deploy Application**.

> [!info] Nota
> **Properties (Configuración):** Nunca se debe dejar las credenciales *hardcoded*. Usa la pestaña Properties en Runtime Manager para inyectar valores como `db.host` o `db.password`. CloudHub las pasará a tu aplicación al arrancar.
> 
> **Logs:** Una vez desplegada, en Runtime Manager, haz clic en tu app y ve a la sección Logs. Ahí verás la consola en tiempo real (vital para depurar errores de arranque).
> 
> **[[MuleSoft#Archivos de Configuración|Archivos de Configuración]]**: Para evitar el redespliegue de aplicaciones, es importante dejar archivos de configuración  `config.yaml` en las MuleApps.

