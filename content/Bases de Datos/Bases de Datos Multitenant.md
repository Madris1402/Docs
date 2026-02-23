---
tags:
  - DB
---
Las Bases de Datos ***Multitenant*** se caracterizan por modular el sistema en una base central y bases individuales que dependen de la central.

### Contained Data Base (CDB)
- ***Provee la estructura común:*** memoria (SGA) y procesos de fondo (background processes).
- Tiene un administrador central llamado ***Root*** (`CDB$ROOT`). Aquí se guardan los metadatos globales, pero **nunca** se guardan datos de usuario (como tus tablas de ventas).
- Si es necesario aplicar alguna actualización en el sistema (parches de seguridad, mantenimiento, etc.), esta se aplica al ***CDB*** y se ve reflejado en todas las ***PDBs*** del sistema.

### Pluggable Data Base (PDB)
- Aquí se encuentran los datos de las aplicaciones (tablas, índices, esquemas de RRHH, Finanzas, etc.).
- Para una aplicación que se conecta, una PDB se ve y se comporta exactamente igual que una base de datos tradicional. No nota la diferencia.
- Estas bases se pueden desconectar de un servidor y conectarlas a otro como si se tratara de una memoria USB.
#### PDB Seed
En la arquitectura Multitenant, esta PDB es especial; funciona como la "plantilla maestra" o el molde para crear cualquier PDB nueva y se encuentra en modo `READ ONLY`.

Si la **`PDB$SEED`** permitiera modificaciones, cualquier error, dato basura o configuración extraña que guardáramos en ella se **replicaría** automáticamente en cada nueva base de datos que creáramos en el futuro.

### Resource Manager
Es necesario tener un gestor de recursos ya que sin él una ***PDB*** que recibe una consulta muy pesada consumirá más CPU y Memoria para completar su tarea y puede generar que las demás ***PDBs*** bajen su rendimiento o dejen de funcionar.

Con el gestor de recursos se pueden configurar ***garantías*** y ***límites***:

- **Garantías:** *"La PDB de Finanzas siempre tendrá garantizado al menos el 40% del CPU".*

- **Límites:** *"La PDB de Desarrollo nunca podrá usar más del 10% del CPU si hay otros trabajando".*

### Comandos Elementales

- **Mostrar Bases:** `show pdbs`
- **Gestionar:** `alter pluggable database {pdb_name} open/save state`
- **Navegar:** `alter session set container {pdb_name}`
- **Verificar Contenedor:** `show con_name`
