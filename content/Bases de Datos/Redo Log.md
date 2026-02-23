---
tags:
  - DB
---
### Función
En Oracle DB, los *Redo Logs* se encargan de recuperación de datos en caso de un error. Almacenan todos los cambios realizados a la base de datos en tiempo real, asegurando la reconstrucción de transacciones y su aplicación a los archivos de datos, garantizando la integridad de los datos.

#### Componentes Principales
- **Redo Log Buffer**: Un Buffer circular en el Área Global de Sistema (SGA) que retiene registros Redo generados por transacciones en memoria temporalmente.
  
- **Online Redo Log Files**: Archivos en disco que los procesos en segundo plano del *Log Writer (LGWR)* escriben en el Redo Log Buffer. La Base utiliza un mínimo de dos o más grupos de Online Redo Log y escribe en ellos de manera circular.
  
- **Archived Redo Log Files (Registros Archivados)**: Son copias de los Grupos de Online Redo Log que se almacenan en un destino *offline* cuando la base entra en el modo `ARCHIVELOG`. Estos son circulares para recuperación *point-in-time* y mantenimiento de bases de datos en espera.
  
- **Log Writer (LGWR)**: Proceso de segundo plano responsable de escribir los registros Redo del *Redo Log Buffer* al *Online Redo File* actual en disco.
  
- **Archier(ARCn)**: Proceso en segundo plano que copia automáticamente *Online Redo Log Files* al archivo de registros cuandola base de datos está en el modo `ARCHIVELOG`.
  
- **Log Switch**: Es el punto en el cual el *LGWR* deja de escribir en un *Redo Log Group* y empieza a escribir en otro. El *Log Switch* ocurre de forma automática cuando un grupo se llena. Pero puede ser forzado a modo manual usando el comando `ALTER SYSTEM SWITCH LOGFILE`.

### En Arquitectura Multitenant
En la arquitectura [[Bases de Datos Multitenant|Oracle Multitenant]] <mark style="background:rgba(136, 49, 204, 0.2)">solo existe un conjunto de *Online Redo Log Files* en el nivel de la *CDB* que se comparte con todas las *PDBs* dentro de la *CDB*.</mark> Las *PDBs* no cuentan con *Online Redo Log Files* propios.

#### Conceptos Elementales

- **Recurso Compartido**: Los*Redo Logs* son una estructura física que pertenece al contenedor root (`CDB$ROOT`) y a toda la instancia de *CDB*. El proceso en segundo plano del *Log Writer* escribe todos los cambios de las *PDBs* en este conjunto único y compartido de *Redo Log Files*.
  
- **Separación Lógica***: Aunque la parte física de los *Redo Logs* es compartida, los registros individuales contienen información que permite a Oracle separar lógicamente qué cambios pertenecen a cualquier *PDB* (Se identifican con un `CON_ID` en las *vistas CDB*).
  
- **Administración a nivel de CDB**: Las operaciones de administración de *Redo Log*, como `ADD` o `DROP` para grupos/miembros, forzar un *log switch*, y configurar *multiplexing* son ejecutadas en el nivel de la *CDB* usando comandos SQL como `ALTER DATABASE` y las *PDBs* heredan dichas configuraciones.
  
- **Modo Archivelog**: La decisión de ejecutar en los modos `ARCHIVELOG` o `NOARCHIVELOG` también se realiza al nivel de la *CDB* y se aplica a todas las *PDBs* dentro de él. No hay manera de tener ciertas *PDBs* en `ARCGIVELOG` y otras sin este.
  
- **Monitoreo**: Las vistas usadas para monitoreo de Redo logs, tales como `V$LOGFILE` y `V$LOG` son accesibles desde el *Root* de la *CDB* para poder ver el reporte completo. Sin embargo, algunas herramientas específicas como ***LogMiner*** se pueden usar al nivel de las *PDBs* para analizar los datos de *Redo* relevantes para esa *PDB*
  
- **Rendimiento**: Consolidar los *Redo Logs* puede ser eficiente, pero un flujo alto de transacciones a través de múltiples *PDBs* puede incrementar la contención I/O para el  único conjunto de *Log Files*. Adecuar el tamaño y el *Multiplexing* es crucial para el rendimiento y la disponibilidad.
  
#### Comandos de Administración (Nivel CDB)
Se pueden administrar los *Redo Log Files* físicos conectándose al contenedor *Root* de la *CDB*.

| Acción                           |                                                                                                  Comando SQL |
| -------------------------------- | -----------------------------------------------------------------------------------------------------------: |
| **Añadir un log group**          | `ALTER DATABASE ADD LOGFILE GROUP 4 ('/u01/logs/orcl/redo04a.log', '/u01/logs/orcl/redo04b.log') SIZE 100M;` |
| **Añadir un log member**         |                                 `ALTER DATABASE ADD LOGFILE MEMBER '/u01/logs/orcl/redo02b.log' TO GROUP 2;` |
| **Forzar un log switch**         |                                                                               `ALTER SYSTEM SWITCH LOGFILE;` |
| **Ver el Estado de un log file** |                                                              `SELECT GROUP#, STATUS, MEMBER FROM V$LOGFILE;` |
