---
tags:
  - GoldenGate
---
Oracle GoldenGate es una potente tecnología de replicación e integración de datos. Su función principal es mover datos en tiempo real (o casi en tiempo real) entre diferentes sistemas de bases de datos o plataformas. Lo hace leyendo las transacciones de forma no intrusiva, directamente desde los archivos de registro de transacciones (logs) de la base de datos de origen, y aplicándolas en la base de datos de destino. Esto permite mantener los sistemas sincronizados para alta disponibilidad, migraciones o análisis de datos sin afectar el rendimiento de tus aplicaciones principales.

### Arquitectura
La arquitectura de *GoldenGate* consiste de 3 elementos principales:
- Extract
- Trail Files
- Replicat

| Componente      |                                                                                                                   Función Principal |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------: |
| **Extract**     | Lee los registros de transacciones (logs) de la base de datos origen y captura los cambios confirmados (inserts, updates, deletes). |
| **Trail Files** |                                                      Almacena estos cambios capturados de forma temporal y en un formato universal. |
| **Replicat**    |         Lee los _Trail Files_ en el servidor destino y ejecuta esos mismos cambios en la base de datos destino, en el orden exacto. |
- También existe un componente de red adicional llamado _Data Pump_ que se encarga de transportar los Trail Files por la red, pero el corazón lógico del sistema siempre es la dupla de _Extract_ y _Replicat_.

#### Extract
Este proceso lee directamente los *[[Redo Log|Redo Logs]]* de Oracle lo que le permite trabajar al nivel del sistema operativo independientemente. De esta forma el sistema no genera consultas directas a la base de datos, lo que evita que esta se vuelva lenta.

#### Trail Files
El proceso de *Extract* no envía las instrucciones directamente a la base de replicaciones, sino que crea archivos temporales que almacenan el registro de transacciones. Estos nos sirven como un *buffer*, si la conexión con la base de replicaciones se cae, los Trail Files se siguen escribiendo y una vez se reestablezca la conexión, la base de replicaciones lee estos archivos y realiza las transacciones realizadas durante el periodo sin conexión.

Estos archivos se guardan en la memoria física y están escritos en formato binario universal que solamente *GoldenGate* puede comprender, dando así una capa de seguridad, compatibilidad y encriptación a todas las transacciones registradas en ellos.

Además, su ciclo de vida es muy organizado:

1. Se escriben de forma estrictamente secuencial, agregando las transacciones una tras otra.
2. Tienen un tamaño máximo configurable (por defecto, suelen ser 500 MB).
3. Cuando un archivo se llena, GoldenGate lo cierra y automáticamente crea el siguiente en la secuencia numérica (por ejemplo, de `tr000001` pasa a `tr000002`). A este proceso se le llama **rollover**.
#### Replicat
Este proceso toma los Trail Files y los convierte en instrucciones SQL para ejecutarlas en la base de replicación. Este debe generar las transacciones en el orden exacto en el que sucedieron en la base de origen para evitar conflictos de datos o errores de ejecución.

Para llevar el orden de las transacciones, este proceso utiliza *Checkpoints*, que simplemente son una marca que indica cual fue la última transacción del registro que se realizó exitosamente. Estos *Checkpoints* se almacenan en memoria física ya que en caso de un corte de energía, no se pierda este control.

Para almacenar estos *Checkpoints* se puede generar un archivo llamado ***Checkpoint File*** o generar una tabla llamada ***Checkpoint Table***. De esta forma, si ocurre un desastre y los sistemas se reinician, al volver a encenderse el proceso _Replicat_ va directo a su tabla, revisa el marcapáginas, busca el archivo Trail correspondiente y continúa su trabajo exactamente donde lo dejó.

### Topologías
Existen diferentes topologías con las que se puede aplicar *GoldenGate* dependiendo de las necesidades del negocio:

#### Unidireccional
Consiste en replicar los datos de una base origen a una base destino. Utiliza el proceso *Extract* en la base de origen y el *Replicat* en la base de destino. Es útil para respaldar bases.
#### Bidireccional
Si se busca sincronizar dos bases de datos se utiliza esta topología. Consiste en tener los procesos *Extract* y *Replicat* en cada base. Esto implica que cuando se actualiza *base A*, la *base B* recibe las instrucciones, genera el cambio y envía de vuelta el cambio a la *base A*. 

Para evitar estos bucles se da la instrucción `IGNOREREPLICAT` al proceso Extract, para que solo se actualice la base con transacciones locales y no con las que reciba de la *base B*  y viceversa. 

Pero esto nos podría generar conflictos de datos si las dos bases actualizan el mismo registro lo que desincronizaría las bases. Para ello Oracle cuenta con reglas de***Detección y Resolución de Conflictos (CDR)*** que en caso de excepciones le indique al sistema qué hacer, por ejemplo, dar prioridad a una base sobre la otra o tomar hasta milésimas de segundo para ver qué cambio se recibió primero.
#### Consolidación
Varios procesos _Extract_ en múltiples bases de datos de origen envían sus datos (Trail Files) a una única base de datos de destino. Es el escenario clásico de un *Data Warehouse* donde varias sucursales envían su información a un repositorio central para analítica.

Este escenario puede generar problemas de ID Si la tabla central mantiene la misma estructura estricta que las bases de origen, donde por ejemplo, únicamente el `ID_Ticket` actúa como Clave Primaria (Primary Key), el proceso _Replicat_ se enfrentará a un error. Si se intentar insertar el ticket 100 de una sucursal al Sur después de haber insertado el ticket 100 de la sucursal Norte, la base de datos central lo rechazará por "Clave duplicada" y la replicación se detendrá.

Para evitar esta colisión, podemos configurar el _Extract_ o el _Replicat_ para que mapeen y transformen los datos automáticamente. *GoldenGate* puede tomar el ID de la sucursal y concatenarlo con el ticket, enviando a la base central registros únicos como `NORTE-100` y `SUR-100`.
#### Distribución Broadcast
Una base de datos central extrae los cambios y los distribuye a múltiples servidores de destino. Es ideal para replicar catálogos maestros (como una lista actualizada de precios) hacia todas las tiendas de una cadena.
#### Cascada
Los datos viajan de un servidor origen a un servidor intermedio, y este los reenvía al destino final. Se usa mucho para no sobrecargar la red o los recursos del servidor "A", delegando el trabajo pesado de distribución al servidor "B".


### Initial Load
Antes de replicar los _cambios_ (lo que llamamos CDC o Change Data Capture), necesitamos que la base de datos destino tenga una "fotografía" inicial exacta de la base de datos origen. A esto le llamamos establecer la línea base (baseline).

Existen dos formas principales de hacer esta carga inicial:

1. **Con herramientas nativas de GoldenGate:** GoldenGate tiene procesos especiales (como un _Extract Initial Load_ y un _Replicat Initial Load_) que leen la tabla y mandan los datos. Sin embargo, para bases de datos muy grandes, esto puede ser lento.
   
2. **Con herramientas externas:** En el mundo real, los administradores de bases de datos suelen usar utilidades <mark style="background:#fdbfff">nativas de la base de datos</mark> porque son muchísimo más rápidas para mover Terabytes de información. En Oracle, por ejemplo, usamos una herramienta llamada **Data Pump** (exportar/importar) o restauramos un respaldo físico con **RMAN**.

Imagina que sacar esta "fotografía" inicial (el respaldo de Data Pump) de nuestra base de datos gigante y aplicarla en el destino tarda, digamos, **5 horas** y no podemos decirle a los clientes "cerraremos el sistema 5 horas mientras copiamos los datos". El almacén de origen sigue abierto, creando y actualizando registros minuto a minuto mientras nosotros hacemos la copia.

Para poder generar y guardar esos _Trail Files_, el componente exacto de GoldenGate que debemos encender **antes** de iniciar nuestra copia de 5 horas es el proceso **Extract** (el de captura en tiempo real o CDC).

Mientras la herramienta de base de datos (como el Archiver o Data Pump en Oracle) hace el trabajo pesado de copiar los terabytes de información, el _Extract_ actúa como nuestra red de seguridad, atrapando todos los cambios nuevos que ocurren durante esas 5 horas y guardándolos a salvo en los _Trail Files_.

Para que todo encaje sin duplicar datos, utilizamos la **Instanciación basada en SCN**. Funciona como una máquina del tiempo y se hace en estos 4 pasos:

1. **Encendemos el Extract:** Comienza a atrapar cambios y a llenar _Trail Files_.
2. **Tomamos la "foto" (Respaldo):** Al iniciar el respaldo que tardará 5 horas, la base de datos nos da un número de ticket exacto llamado **SCN** (System Change Number). Piensa en el SCN como una marca de tiempo milimétrica que dice: _"Esta foto contiene los datos exactamente hasta este instante"_.
3. **Restauramos la foto:** Llevamos esa copia inmensa al servidor destino y la instalamos. El almacén destino ahora está lleno, pero tiene 5 horas de retraso.
4. **Encendemos el Replicat con una condición:** Le decimos al _Replicat_: "Aquí están los Trail Files que el Extract juntó durante las últimas 5 horas. <mark style="background:#fdbfff">Pero comienza a aplicar los cambios únicamente a partir del SCN de nuestra foto</mark>".

De esta forma, el _Replicat_ ignora cualquier transacción que ya viniera incluida en el respaldo inicial y solo aplica los cambios nuevos. Así logramos una sincronización perfecta con **cero tiempo de inactividad**

### GoldenGate Software Command Interface (GGSCI)
Es la consola de administración de *GoldenGate* desde ella se construye y controla todo el flujo de datos.

Para entrar a la consola, simplemente abrimos la terminal del sistema operativo, navegamos a la carpeta de GoldenGate y escribimos `./ggsci`. El prompt cambiará y se verá así: `GGSCI (servidor-origen) 1>`.
