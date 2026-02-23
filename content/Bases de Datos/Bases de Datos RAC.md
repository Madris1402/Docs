---
tags:
  - DB
---
Las bases de datos RAC cuentan con múltiples instancias ***clusterizadas*** de sistemas de gestión de bases de datos que están conectadas al un sistema de ***almacenamiento compartido***. Permitiendo así <mark style="background:rgba(136, 49, 204, 0.2)">alta disponibiliad y escalabilidad.</mark>

A diferencia de una base de datos tradicional que se satura cuan
do recibe demasiadas peticiones simultaneas.

### Almacenamiento Compartido
Todos los nodos tienen acceso a los mismos datos, si el nodo A escribe un dato, el nodo B puede leer ese dato.

Para evitar que se sobrescriban los datos, se necesita de un coordinador.
### Cache Fusion
Si el nodo B requiere de un dato que está manipulando el nodo A, el nodo A se lo pasa directamente usando una red privada o ***interconnect*** así nunca se toca el disco de almacenamiento y se hace una <mark style="background:rgba(136, 49, 204, 0.2)">transferencia de memoria a memoria</mark>.

La red o interconnect es crucial ya que sin ella el sistema de clusters no se puede sincronizar, si la red es lenta o falla pueden pasar estos casos:

1. **Cuellos de botella (Waits):** 
	Los servidores tienen que esperar a que los datos lleguen. En Oracle existen eventos `gc` (Global Cache) que indican que el nodo está esperando la respuesta de otro nodo.
    
2. **Eviction (Expulsión del Nodo):** 
	Si la red es tan lenta que los nodos dejan de escucharse, entra un sistema de protección <mark style="background:rgba(136, 49, 204, 0.2)">para evitar la corrupción de los datos que reinicia el servidor o apaga un nodo a la fuerza</mark>. A esto lo llamamos protección contra ***Split Brain*** o ***Cerebro Dividido*** De esta manera<mark style="background:rgba(136, 49, 204, 0.2)"> evitamos que varios nodos crean que están operando solos y no escriban en el mismo espacio al mismo tiempo</mark>, lo que corrompería todo el sistema.

### Global Resource Directory (GRD)
El ***GRD*** es un mapa con la ubicación de los datos, este se distribuye a cada nodo, por ejemplo, el nodo A se hace cargo de la parte del mapa que indica quién tiene los datos que corresponden a Clientes y el nodo B se encarga de la parte del mapa que indica quien tiene los datos de Facturas.

De esta forma si después se añaden más nodos, la gestión del mapa crece. Lo que lo vuelve escalable.

### Single Client Access Name (SCAN)
En Oracle, el ***SCAN*** es un identificador para el cluster, con él no importa la cantidad de nodos que se creen, solo existe una dirección a todo el sistema y este distribuye la carga de trabajo a los nodos correspondientes.

La ventaja del ***SCAN*** es que si se añaden más nodos, la aplicación que se conecte al cluster no tiene que almacenar las direcciones IP de los nodos individuales, solamente la dirección del cluster.

#### Transparent Application Failover (TAF)

El ***TAF*** o en versiones más modernas ***Application Continuity*** sirve para evitar errores de desconexión entre el usuario y el servidor.

Si un usuario está realizando una acción y está conectado al nodo A del cluster, y este llega a fallar (Se queda sin energía o red), activa el ***SCAN*** para buscar otro nodo vivo y transferir la acción realizada al nodo siguiente. Funciona así:

- La aplicación detecta que el Servidor A desapareció.
- Automáticamente, usa el ***SCAN*** para buscar otro nodo vivo (digamos, el nodo B).
- La base de datos "rebobina" y re-ejecuta la consulta exacta en el nodo B hasta llegar al punto donde se cortó la conexión.
- Entrega los resultados de la acción realizada.


### Oracle ASM
Oracle Automatic Storage Management (ASM) en un entorno Real Application Clusters (RAC)
es la solución de almacenamiento optimizada y compartida, esencial para la alta disponibilidad. Permite a múltiples nodos RAC acceder simultáneamente a grupos de discos ASM compartidos, gestionando automáticamente la distribución de datos (striping/mirroring), simplificando la administración y garantizando rendimiento y protección contra fallos.

