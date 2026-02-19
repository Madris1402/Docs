---
tags:
  - Apache
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
Permite almacenar y procesar cantidades masivas de información dividiendo el trabajo en múltiples computadoras, ideal para tareas pesadas y programadas (procesamiento por lotes o _batch_). Su uso principal es en *Big Data*.

En lugar de tener una sola supercomputadora para el procesamiento, reunimos cientos de computadoras normales y les repartimos la carga de trabajo (a esto se le llama *Cluster*).

Para lograr esto, Hadoop cuenta con 2 elementos principales:
1. HDFS 
2. MapReduce

### Hadoop Distributed File System (HDFS) 
Es el sistema de almacenamiento. Se encarga de tomar un archivo gigantesco, cortarlo en *bloques* más pequeños y repartirlos de forma organizada entre todas las computadoras del equipo.

Para que esta división funcione, Hadoop necesita un registro central o mapa que sepa dónde está cada fragmento.

En la arquitectura de *HDFS*, este registro o diccionario es administrado por un servidor principal llamado ***NameNode*** (el director). Las computadoras que guardan los pedazos reales de información son los ***DataNodes*** (los trabajadores).

HDFS no corta un archivo de 1TB en pedazos gigantes de 250GB, sino en "bloques" estándar y pequeños (típicamente de 128 MB). El *NameNode* mantiene en su memoria RAM una tabla muy similar a esta:

|     Archivo (1 TB)      | ID del Bloque (128 MB) | Computadora Asignada (DataNode) |
| :---------------------: | :--------------------: | :-----------------------------: |
| `datos_financieros.csv` |       `blk_0001`       |    Nodo_A (IP: 192.168.1.10)    |
| `datos_financieros.csv` |       `blk_0002`       |    Nodo_B (IP: 192.168.1.11)    |
| `datos_financieros.csv` |       `blk_0003`       |    Nodo_C (IP: 192.168.1.12)    |
|           ...           |          ...           |               ...               |
| `datos_financieros.csv` |       `blk_8192`       |    Nodo_D (IP: 192.168.1.13)    |
Cuando se quiere leer el archivo, se le pide al *NameNode* y él devuelve este mapa para que vayas directamente a los *DataNodes* a buscar tus bloques.

Ahora, pensemos en la vida real. Imagina que hay un fallo eléctrico o el disco duro del **Nodo_B** se quema repentinamente. Si el bloque `blk_0002` solo estaba guardado en esa computadora, el archivo de 1TB quedaría corrupto e inservible.

En el mundo de Hadoop, para evitar esto utilizamos un concepto vital que se llama **Replicación de Bloques**.

Por defecto, Hadoop no guarda un bloque una sola vez, sino que hace **tres copias** (réplicas) de cada uno y las distribuye estratégicamente en diferentes *DataNodes*.

Si el **Nodo_B** falla repentinamente, el NameNode (que monitorea a sus trabajadores constantemente) revisa su diccionario, nota que las otras copias están a salvo en otros nodos, y redirige el tráfico hacia allá. Además, automáticamente le ordena a un nodo sano que cree una nueva copia para volver a tener tres réplicas de seguridad.

### MapReduce
Es el motor de procesamiento. En lugar de mover esos pesados bloques de datos a través de la red para procesarlos en un servidor central, MapReduce envía las instrucciones (el programa) directamente a la computadora que ya tiene el fragmento de datos para que lo procese localmente. Imagina que ese archivo de 1TB es un registro histórico de ventas y queremos saber el total de ingresos. para esto usamos *MapReduce*.

Como su nombre indica, se divide en dos fases:

1. **Map (Mapear):** El *NameNode* envía el programa (la instrucción de sumar los ingresos) a _cada_ *DataNode* que tiene un fragmento de los datos. Cada computadora hace el cálculo localmente solo con su pedazo.
2. **Reduce (Reducir):** Toma los "subtotales" que calculó cada *DataNode* y los consolida en un solo resultado final.

Ahora, supongamos que tenemos una computadora más antigua como nodo, esta toma más tiempo en procesar los datos. Si ese nodo se atrasa, toda la fase de "Reduce" se queda estancada esperando esa pieza del proceso.

Para solucionar este exacto problema, Hadoop usa la **Ejecución Especulativa**. Si el *NameNode* nota que un *DataNode* va inusualmente lento comparado con los demás, lanza una copia exacta de esa misma tarea *Map* en otra computadora que ya esté libre. El primero que termine, gana, y el proceso lento se cancela. Así se evitan los cuellos de botella.