### Documentación de SAIF

Hola, aquí encontrarás documentación de todo tipo, desde herramientas, pequeñas prácticas etc. c;

---

Software que se utiliza:

- [AnypointStudio](https://www.mulesoft.com/lp/dl/anypoint-mule-studio) IDE para desarrollo de *MuleApps* (APIs).
- [Docker](https://www.docker.com/) Aplicaciones Contenerizadas.
- [Draw.io](https://www.drawio.com/) Software de diseño de diagramas.
- [MobaXterm](https://mobaxterm.mobatek.net/) Terminal Alternativa, uso para *SSH*.
- [Postman](https://www.postman.com/) Entorno de Pruebas para APIs
- [Visual Studio Code](https://code.visualstudio.com/) Editor de Código.
- [Oracle SQL Developer](https://www.oracle.com/latam/database/sqldeveloper/technologies/download/) Interfaz Gráfica para manejo de bases Oracle.
- Interfaz para administración de Bases de datos
	- [Beekeeper Studio](https://www.beekeeperstudio.io/) Interfaz más limpia y moderna.
	- [DBeaver](https://dbeaver.io/) Mejor Organización en entornos grandes de administración.
---
Instalar Software con comandos de terminal para Windows:
- Docker
```powershell
winget.exe install --id "Docker.DockerDesktop" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- Draw.io
```powershell
winget.exe install --id "JGraph.Draw" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- MobaXterm
```powershell
winget.exe install --id "Mobatek.MobaXterm" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- Postman
```powershell
winget.exe install --id "Postman.Postman" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- SQL Developer
```powershell
winget.exe install --id "Oracle.SQLDeveloper" --exact --source winget --accept-source-agreements --disable-interactivity --silent --location "C:\" --accept-package-agreements --force
```
- 
	Para instalar correctamente SQL Developer tendremos que ir a `C:\sqldeveloper` y crear un acceso directo de `sqldeveloper.exe` y añadirlo al menú de inicio (`C:\ProgramData\Microsoft\Windows\Start Menu\Programs`). para acceder a él fácilmente ya que no hace esto automáticamente.
	
- Beekeeper Studio
```powershell
winget.exe install --id "beekeeper-studio.beekeeper-studio" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- DBeaver
```powershell
winget.exe install --id "DBeaver.DBeaver.Community" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```

Otros Paquetes:
- Java 17
```powershell
winget.exe install --id "Oracle.JDK.17" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- Python 3.12
```powershell
winget.exe install --id "Python.Python.3.12" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
- Git
```powershell
winget.exe install --id "Git.Git" --exact --source winget --accept-source-agreements --disable-interactivity --silent --accept-package-agreements --force
```
