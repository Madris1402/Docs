---
tags:
  - MuleApp
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
Esta MuleApp tiene como propósito usar *Batch Jobs* y reportarlos a *New Relic* para su análisis.

### Batch Jobs
Es la ejecución de un programa que procesa una gran cantidad de datos agrupados por *lotes (batches)*. Está totalmente automatizado y no requiere de intervención humana.

A diferencia de las *API REST*, donde se utiliza procesamiento síncrono y se espera que devuelva una respuesta inmediata, los *Batch Jobs* toman volúmenes masivos de datos, los dividen en pedazos manejables y se procesan en segundo plano.

#### Funcionamiento
Los *Batch Jobs* se dividen en 3 fases:
 1. **Input / Extracción (Read)**: El proceso arranca (ya sea programado a las 3:00 AM o disparado por un evento). Lee todos los registros iniciales. *Ejemplo: Consultar 50,000 clientes nuevos desde una base de datos.*
 2. **Procesamiento / Transformación (Process)**: La tecnología toma esos 50,000 registros y los divide. En MuleSoft, por ejemplo, puedes decirle: *"Procesa esto en bloques (block size) de 100 registros a la vez"*. Aquí se aplica la lógica de negocio, se filtran datos o se enriquecen. Si un registro falla, no detiene a los demás.
 3. **Salida / Carga (On Complete / Load)**: Una vez que todos los bloques han sido procesados, se genera un resumen. Aquí es donde insertas los datos transformados en un sistema destino (como Golden Gate o un Data Warehouse) y reportas: *"Terminé. Procesé 50,000. 49,990 exitosos, 10 fallidos"*.
#### Usos 
- **Eficiencia de Recursos:** Evita sobrecargar los sistemas. Por eso suelen correr de noche o en horarios de poco tráfico.
- **Tolerancia a Fallos:** En herramientas como MuleSoft, si el registro número 5,321 tiene un error, el Batch Job lo aísla, lo marca como fallido y sigue procesando los demás sin que todo el sistema colapse.
- **Manejo de Cuellos de Botella:** Si el sistema de destino solo soporta 50 peticiones por segundo, el Batch Job puede regular la velocidad a la que envía la información para no tirarlo.

### Generar la Mule App

#### Contenedor Docker para Base de Datos
Para generar esta mule App utilizaremos una imagen de Docker que contenga Oracle 19.

```powershell
docker run -d -p 1522:1521 -e ORACLE_PASSWORD=Oracle123 --name oracle19c gvenzl/oracle-xe
```
Este comando generará un contenedor usando Oracle XE ya que es más ligero. Lo configuramos en el puerto 1522 para evitar problemas de asignación con Oracle 19c.

Posteriormente entramos al contenedor con
```powershell
docker exec -it oracle19c sqlplus
```

Posteriormente nos pedirá credenciales, ingresamos
Usuario: `system`
Contraseña: `Oracle123`

Ahora crearemos un usuario para las pruebas:
```sql
CREATE USER batch_test IDENTIFIED BY batch123;

GRANT CREATE SESSION TO batch_test;

GRANT RESOURCE TO batch_test;

GRANT UNLIMITED TABLESPACE TO batch_test;
```

```sql
-- Crear la tabla
CREATE TABLE TRANSACTIONS_TEST (
    ID NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    TRANSACTION_STATUS VARCHAR2(20),
    TIME_STAMP TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertar datos de prueba para que el Batch Job tenga algo que leer
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('ERROR');
COMMIT;
```

Después Utilizaremos este código para seguir inyectando registros a la base por la terminal docker:
``` sql
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('PENDING');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('SUCCESFUL');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('SUCCESFUL');
INSERT INTO TRANSACTIONS_TEST (TRANSACTION_STATUS) VALUES ('ERROR');
COMMIT;
SELECT * FROM TRANSACTIONS_TEST; 
```
#### Anypoint Studio
Ya que tengamos la base de datos funcionando, pasaremos a desarrollar la MuleApp, pero primero haremos los preparativos:

1. **New Relic**: Haremos ajustes a nuestro *Agente de New Relic* Si no lo has descargado, revisa la documentación de [[New Relic]].
	- Al archivo `newrelic.yml` agregaremos en el apartado de `class_transformer`.
``` yaml
class_transformer:
    # Excluimos el driver de Oracle para evitar el VerifyError en Java 17
    excludes: oracle/jdbc/driver/OracleDriver
```
- Esto es para que *New Relic* no modifique los archivos del JDBC de Oracle y nos de error (`java.lang.VerifyError: Bad type on operand stack`).
	- Al archivo `pom.xml` agregaremos la dependencia de New Relic al apartado `<dependencies>`
```xml
<dependency>
    <groupId>com.newrelic.agent.java</groupId>
    <artifactId>newrelic-api</artifactId>
    <version>8.9.0</version>
    <scope>provided</scope>
</dependency>
```
- Ya añadida la dependencia, guardamos el archivo.
	
2. Clase de Java
	Esta será el puente de comunicaciones a *New Relic*
	- En el _Package Explorer_, haz clic derecho en la carpeta **`src/main/java`** > _New > Class_.
	- **Package:** `com.saif.monitoring`
	- **Name:** `NewRelicBatchReporter`.
		- Nos aseguramos que sea una clase pública, que no tenga ningún modificador y que la herencia esté activada.
		- Hacemos click en Finish.
	- Una vez creada, usaremos este código para la clase:
```java
package com.saif.monitoring;

import com.newrelic.api.agent.NewRelic;
import java.util.HashMap;
import java.util.Map;

public class NewRelicBatchReporter {
    public static void reportarStatus(int total, int exitosos, int fallidos, long tiempoMs) {
        Map<String, Object> atributos = new HashMap<>();
        atributos.put("jobName", "ProcessOracleTransactions");
        atributos.put("totalRecords", total);
        atributos.put("successfulRecords", exitosos);
        atributos.put("failedRecords", fallidos);
        atributos.put("elapsedTimeMs", tiempoMs);
        
        NewRelic.getAgent().getInsights().recordCustomEvent("MuleBatchStatus", atributos);
        System.out.println("Log: Evento de Batch enviado a New Relic exitosamente");
    }
}
```
	
3. Conectar a la base de datos:
	- Si no tenemos el módulo de bases de datos, lo importamos con la *Mule Palette*.
	- Una vez importado, Creamos un archivo de configuración en la carpeta `src/main/mule`.
		- Le damos un Nombre al archivo, ejemplo: `Docker_Oracle_Config.xml`.
		- Una vez creado, vamos a *Global Elements* y creamos un Database Config:
			- Seleccionamos *Oracle Connection*.
			- Configuramos el JDBC 11 de manera local.
			- Asignamos el host: `localhost` y el puerto `1522`.
			- Asignamos el usuario `batch_test` y la contraseña.
			- En Service name, Asignamos `XE`.
			- Probamos la conexión.

##### Flujo Visual
Abrimos el Archivo XML principal (`batch_newrelic_api`) y arrastramos los siguientes 4 componentes en este orden exacto:

1. **Scheduler**
	- Arrástralo al inicio.
	- Configúralo en **Fixed Frequency**: `5 Seconds` (Para que corra casi de inmediato al arrancar y no tengamos que esperar).

2. **Database -> Select**
	- Lo agregamos justo después del Scheduler.
	- Selecciona tu configuración de Oracle.
	- **SQL Query Text:**
``` sql
SELECT * FROM TRANSACTIONS_TEST WHERE TRANSACTION_STATUS = 'PENDING'
```
    
3. **Batch Job**
	- Arrástralo después del Select.
	- **En la fase _Batch Step_**: Arrastra un componente **Database -> Update**.
    - Selecciona tu configuración de Oracle.
    - **SQL Query Text:**
``` sql
UPDATE TRANSACTIONS_TEST SET TRANSACTION_STATUS = 'PROCESSED', TIME_STAMP = CURRENT_TIMESTAMP WHERE ID = :id
```
- **Input Parameters:** `{'id': payload.ID}`
	
4. **Invoke Static (El Reporte Final):**
	- Ve a _Add Modules_, arrastra el módulo **Java**.
	- Arrastra el componente **Invoke static** y suéltalo en la fase **`On Complete`** de tu Batch Job.
	- **Class:** `com.saif.monitoring.NewRelicBatchReporter`
	- **Method:** `reportarStatus(int, int, int, long)`
	- **Args:** Haz clic en el botón `fx` y pega esto:
``` json
{
	"total": payload.totalRecords default 0,
	"exitosos": payload.successfulRecords default 0,
	"fallidos": payload.failedRecords default 0,
	"tiempoMs": payload.elapsedTimeInMillis default 0
}
```
### Pruebas en New Relic

Para estas pruebas entramos a New Relic, en el menú izquierdo, ve a **Query Your Data** (o busca el ícono `>_` de Query Builder).

1. La Tabla de Auditoría
	Esta consulta te muestra el historial crudo, exactamente como lo verías en una base de datos.
	
``` sql
SELECT timestamp, jobName, totalRecords, successfulRecords, failedRecords, elapsedTimeMs 
FROM MuleBatchStatus 
SINCE 1 DAY AGO 
LIMIT 50
```
- **Tipo de Gráfica recomendada:** `Table` (Tabla).
	
2. Los KPIs
	
	Esta consulta obtiene **la última ejecución** y te muestra los números en grande. Es ideal para ver el estado actual de un vistazo.
	
``` sql
SELECT sum(totalRecords) AS 'Total', sum(successfulRecords) AS 'Éxitos', sum(failedRecords) AS 'Fallos' 
FROM MuleBatchStatus 
WHERE jobName = 'ProcessOracleTransactions'
SINCE 1 DAY AGO
```
- **Tipo de Gráfica recomendada:** `Billboard` (Panel de números).
    
3. El Rendimiento en el Tiempo
	
	¿Tu Batch Job se está volviendo más lento con el paso de los días? Esta consulta usa la magia del comando `TIMESERIES` para graficar el tiempo de ejecución.
	
```sql
SELECT average(elapsedTimeMs) AS 'Tiempo Promedio (ms)' 
FROM MuleBatchStatus 
WHERE jobName = 'ProcessOracleTransactions' 
TIMESERIES 
SINCE 1 WEEK AGO
```
- **Tipo de Gráfica recomendada:** `Line` o `Area`.
  
Después de ejecutar cada consulta, en la parte inferior derecha usamos el botón *"Add to Dashboard"* (Agregar al Tablero), seleccionamos "Create a new dashboard", llámalo "Monitoreo MuleSoft" y agregamos todos los widgets ahí con los nombres de cada consulta que se especificaron.