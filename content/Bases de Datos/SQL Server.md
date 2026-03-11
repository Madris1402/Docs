---
tags:
  - DB
---
**SQL Server** es un Sistema de Gestión de Bases de Datos Relacionales (RDBMS) desarrollado por Microsoft. Su función principal es almacenar y recuperar datos solicitados por otras aplicaciones de software, ya sea que se ejecuten en la misma computadora o a través de una red.

Utiliza **T-SQL** (Transact-SQL), una extensión del estándar SQL que añade programación procedimental, variables locales y funciones de soporte para procesamiento de cadenas y datos.

### Ediciones de SQL Server
- **Express:** Gratuita, ideal para bases de datos pequeñas (límite de 10 GB por base de datos) y aplicaciones ligeras. Usa menos recursos (máximo 1 GB de RAM).
- **Developer:** **(Recomendada)** Es exactamente igual a la versión "Enterprise" (la más cara y completa), pero licenciada únicamente para desarrollo y pruebas (QA). No tiene límites de tamaño ni de memoria, pero _nunca_ debe usarse en producción.

### Instalación de SQL Server Developer
> [[Instalar SQL Server|Tutorial Paso a Paso]]

1. Descargar el instalador
	1. Descarga el instalador de **SQL Server Developer Edition** desde la [página oficial de Microsoft](https://www.microsoft.com/es-mx/sql-server/sql-server-downloads).
	2. Ejecuta el archivo descargado y selecciona el tipo de instalación **Custom** (Personalizada). Esto descargará los medios de instalación completos y te permitirá controlar qué componentes instalar.
	3. Una vez que se abra el "SQL Server Installation Center", ve a la pestaña **Installation** en el menú izquierdo.
	4. Haz clic en **New SQL Server stand-alone installation or add features to an existing installation**.
	5. Acepta los términos de licencia y deja que el instalador verifique las reglas globales (revisará que no haya reinicios pendientes en Windows).
    
2. Configuración Clave
	1. **Feature Selection (Selección de características):**  Marca **Database Engine Services**. Este es el motor principal y **Replication**. Que se encarga de permitir al motor la replicación de datos. Lo demás no lo utilizaremos.
	2. **Instance Configuration (Configuración de la instancia):**
	    - **Default instance:** El servidor se llamará igual que tu computadora (ej. `localhost` o `.`). Recomendado si es el único SQL Server en la máquina.
	    - **Named instance:** Te permite tener varias instalaciones aisladas. (ej. `localhost\PRUEBAS`).
	    
	3. **Database Engine Configuration (El paso más crítico):**
	    
	    - En la pestaña **Server Configuration**, selecciona **Mixed Mode (SQL Server authentication and Windows authentication)**.
	    - Ingresa una contraseña fuerte para el usuario administrador del sistema (**sa**).
	    - Haz clic en el botón **Add Current User** para que tu usuario de Windows también sea administrador.
	4. Continúa dando clic en "Next" y finalmente en **Install**.

3. Instalación de las Herramientas Cliente (SSMS)

	A diferencia de versiones muy antiguas, SQL Server ya no instala la interfaz gráfica por defecto. El motor corre de fondo como un servicio de Windows. Para interactuar visualmente con él, necesitas **SSMS**.
	3. Descargamos **SQL Server Management Studio (SSMS)** del [sitio oficial](https://learn.microsoft.com/es-es/ssms/install/install) o ejecutamos el comando:
```powershell
winget.exe uninstall --id "Microsoft.SQLServerManagementStudio.22" --exact --source winget --accept-source-agreements --disable-interactivity --version "22.3.2" --silent
```
- 	
	5. Una vez instalado, abre SSMS, escribe `localhost` o `.` en el nombre del servidor, elige "Windows Authentication" o "SQL Server Authentication" (con el usuario `sa`), ¡y listo! Estás dentro de tu instancia local.

### Configuración de Red y Troubleshooting

Por defecto, SQL Server se instala en un modo restringido: bloquea las conexiones externas y desactiva los protocolos de red por seguridad. Para usarlo con herramientas de desarrollo externas o desde otras computadoras en tu red local, debes abrir estos canales manualmente.

1. Habilitar el Protocolo TCP/IP
	Si omites este paso, SQL Server solo aceptará conexiones locales usando memoria compartida, bloqueando cualquier intento de conexión por red.
	
	- Abre el programa **SQL Server Configuration Manager** desde el menú de inicio de Windows.
	- Despliega el menú lateral izquierdo en **SQL Server Network Configuration** y selecciona **Protocols for MSSQLSERVER** (o el nombre que le hayas dado a tu instancia).
	- En el panel derecho, haz clic derecho sobre **TCP/IP** y selecciona **Enable** (Habilitar).
	- Aparecerá un mensaje advirtiendo que los cambios no surtirán efecto hasta reiniciar el servicio. Haz clic en **Aceptar**.
	
2. Fijar el Puerto 1433
	Las instancias con nombre a veces usan puertos dinámicos que cambian en cada reinicio. Es mejor fijarlo al estándar.
	
	- En esa misma pantalla, haz doble clic sobre **TCP/IP** para abrir sus propiedades.
	- Ve a la pestaña **IP Addresses** (Direcciones IP).
	- Desplázate hasta el final, a la sección **IPAll**.
	- Borra cualquier valor que esté en **TCP Dynamic Ports** (déjalo totalmente en blanco).
	- En **TCP Port**, escribe **1433**.
	- Haz clic en **Aplicar** y luego en **Aceptar**.
	
3. Abrir el Firewall de Windows
	Aunque SQL Server esté escuchando, el Firewall de tu computadora detendrá el tráfico entrante. Debemos crear una regla de excepción.
	
	- Abre el **Panel de Control** de Windows > **Sistema y Seguridad** > **Windows Defender Firewall**.
	- Haz clic en **Configuración avanzada** (Advanced settings) en el menú izquierdo.
	- Selecciona **Reglas de entrada** (Inbound Rules) y haz clic en **Nueva regla...** (New Rule) a la derecha.
	- Elige el tipo **Puerto** y haz clic en Siguiente.
	- Selecciona **TCP** y escribe **1433** en el campo de puertos locales específicos.
	- Selecciona **Permitir la conexión** y marca los perfiles de red que apliquen (Dominio, Privado, Público).
	- Ponle un nombre claro, como `SQL Server - Puerto 1433`, y haz clic en Finalizar.
	
4. Reiniciar el Servicio Principal
	Ninguno de los cambios de red aplicará hasta que reinicies el motor de base de datos.
	
	- Vuelve al **SQL Server Configuration Manager**.
	- Haz clic en **SQL Server Services** en el menú izquierdo.
	- Haz clic derecho sobre **SQL Server (MSSQLSERVER)** y selecciona **Restart** (Reiniciar).
	- _Nota:_ Si estás usando una instancia con nombre y no la predeterminada, asegúrate de que el servicio **SQL Server Browser** también esté en estado **Running** (En ejecución).