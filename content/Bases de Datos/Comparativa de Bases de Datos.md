---
tags:
  - DB
---
### Cuadro Comparativo

|      **Característica**      | **Single Instance (Tradicional)**                              | **[[Bases de Datos RAC\|RAC (Real Application Clusters)]]** | **[[Bases de Datos Multitenant\|Multitenant (CDB/PDB)]]**              |
| :--------------------------: | -------------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------- |
|        **Estructura**        | 1 Servidor = 1 Base de Datos.                                  | N Servidores = 1 Base de Datos (Clúster).                   | 1 Contenedor (CDB) = Muchas Bases (PDBs).                              |
|      **Disponibilidad**      | Baja. Si el servidor falla, el servicio se detiene.            | Muy Alta. Si un nodo falla, los otros continúan (Failover). | Depende de la base (se puede combinar con RAC).                        |
|     **Uso de Recursos**      | Aislado. A menudo se desperdicia CPU/RAM si no se usa al 100%. | Balanceado. Los usuarios se distribuyen entre los nodos.    | Consolidado. Se comparten recursos; ideal para maximizar la inversión. |
| **Gestión (Parches/Backup)** | Tediosa. Hay que parchear cada base por separado (x100 veces). | Centralizada por clúster, pero compleja de configurar.      | Muy Eficiente. Parcheas el CDB una vez y todas las PDBs se actualizan. |

