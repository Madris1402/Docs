---
tags:
  - GoldenGate
---
### Stored Procedures
Un **Stored Procedure** es un programa o un bloque de código escrito en el lenguaje de la base de datos (como PL/SQL en Oracle, o T-SQL en SQL Server) que **reside, se compila y se ejecuta directamente dentro del motor de la base de datos**.

Se compone principalmente de tres partes:

1. La Cabecera y los Parámetros (El "Contrato")
	Aquí defines cómo se llama el SP y qué información necesita para funcionar. Los parámetros tienen "direcciones":
	
	- **IN (Entrada):** Datos que le pasas al SP desde afuera (ej. desde GoldenGate). El SP solo puede leerlos, no modificarlos.
	- **OUT (Salida):** Variables vacías que le pasas al SP para que él las llene con el resultado y te las devuelva. (Como el `v_bono` de nuestro ejemplo anterior).
	- **IN OUT (Entrada/Salida):** Entra con un valor, el SP lo procesa y devuelve un valor nuevo en esa misma variable.
	
2. Declaraciones Locales
	Antes de empezar la lógica, aquí defines las variables, constantes o cursores que solo existirán mientras el SP se esté ejecutando.
	
3. El Cuerpo (La Lógica de Negocio y Excepciones)
	Limitado por las palabras `BEGIN` y `END`. Aquí es donde ocurre la magia: consultas SQL (`SELECT`, `INSERT`, `UPDATE`), ciclos lógicos (`FOR`, `WHILE`), condiciones (`IF/ELSE`) y el manejo de errores (`EXCEPTION`).
	
#### Ejemplo de un SP

Imagina un SP muy común en auditoría: queremos registrar cada vez que GoldenGate inserta un empleado nuevo.

```sql
-- 1. Cabecera y Parámetros
CREATE OR REPLACE PROCEDURE registrar_auditoria (
    p_tabla_afectada IN VARCHAR2, 
    p_id_registro    IN NUMBER,
    p_resultado      OUT VARCHAR2
) 
IS
    -- 2. Declaraciones locales
    v_fecha_actual DATE;
    
BEGIN
    -- 3. El Cuerpo (Lógica)
    v_fecha_actual := SYSDATE;
    
    -- Insertamos el registro en una tabla de auditoría
    INSERT INTO log_auditoria (nombre_tabla, id_registro, fecha_operacion, usuario)
    VALUES (p_tabla_afectada, p_id_registro, v_fecha_actual, USER);
    
    -- Llenamos el parámetro de salida para confirmar éxito
    p_resultado := 'REGISTRO GUARDADO CON EXITO';
    
EXCEPTION
    -- Manejo de errores: Si algo falla, atrapamos el error
    WHEN OTHERS THEN
        p_resultado := 'ERROR AL GUARDAR: ' || SQLERRM;
        -- Hacemos un rollback solo de esta transacción si falla
        ROLLBACK;
END registrar_auditoria;
/
```

#### Ventajas
- **Reducción del tráfico de red:** Si una lógica requiere hacer 5 `SELECTs`, 2 `UPDATEs` y 1 `INSERT`, una aplicación normal enviaría 8 peticiones por la red hacia la base de datos. Con un SP, la aplicación (o GoldenGate) hace **una sola llamada**, y la base de datos hace todo el trabajo pesado internamente.
- **Seguridad extrema:** Puedes revocarle a un usuario (o a GoldenGate) el permiso de borrar o insertar datos directamente en una tabla crítica, y solo darle permisos para ejecutar el SP. Así controlas exactamente qué puede hacer y cómo.
- **Mantenibilidad:** Si la regla de negocio cambia, modificas el SP en la base de datos y listo. No tienes que reiniciar GoldenGate ni recompilar aplicaciones Java o MuleSoft.
### Packages
En el mundo de las bases de datos (especialmente en Oracle PL/SQL), un **Package** es exactamente eso: un contenedor que agrupa lógicamente procedimientos, funciones, variables y otros elementos relacionados.

Un paquete siempre se divide en dos componentes estrictamente separados. Esta es la clave de su diseño:

**1. La Especificación (Specification o "Spec")** Es la "cara pública" del paquete. Aquí solo declaras **qué** hace el paquete, pero no _cómo_ lo hace. Es el equivalente a una _Interface_ en Java. Todo lo que pongas aquí puede ser invocado por otros usuarios o aplicaciones (como GoldenGate).

- Contiene las firmas de los procedimientos y funciones (nombres y parámetros).
- Declara variables públicas o constantes.

**2. El Cuerpo (Body)** Es el "motor" oculto. Aquí escribes el código real (la lógica de programación) de lo que declaraste en la Especificación.

- Contiene la lógica de negocio.
- Puedes crear variables y procedimientos **privados** aquí adentro. Si un procedimiento solo existe en el Body y no en el Spec, ninguna aplicación externa podrá llamarlo; solo podrá ser usado internamente por otros procedimientos del mismo paquete.

#### Ventajas de los Packages

- **Rendimiento optimizado (Memoria):** Cuando invocas un solo procedimiento de un paquete por primera vez, la base de datos carga **todo el paquete** en la memoria (Shared Pool). Las llamadas posteriores a cualquier otro procedimiento de ese mismo paquete serán rapidísimas porque ya están en memoria, reduciendo el I/O del disco.
- **Manejo de Estado (Variables de Sesión):** Las variables declaradas en un paquete mantienen su valor durante toda tu sesión de base de datos. Si llamas a un procedimiento que cambia una variable de paquete, y luego llamas a otro procedimiento, el segundo recordará ese valor modificado.
- **Sobrecarga (Overloading):** Al igual que en Java, los paquetes te permiten tener múltiples procedimientos con el **mismo nombre** pero con diferentes parámetros. Por ejemplo, podrías tener un procedimiento `buscar_cliente(id NUMERO)` y otro `buscar_cliente(email TEXTO)` dentro del mismo paquete.
- **Seguridad y Mantenimiento:** Puedes modificar el código interno en el _Body_ para arreglar un error o mejorar el rendimiento sin tener que recompilar la _Especificación_. Esto significa que las aplicaciones externas (o tus procesos de GoldenGate) no se rompen ni necesitan ser reconfiguradas.

#### Ejemplo
Así se vería la estructura básica de un paquete para Recursos Humanos:
```sql
-- 1. LA ESPECIFICACIÓN (Lo público)
CREATE OR REPLACE PACKAGE hr_utils AS
    -- Variable pública
    tasa_impuesto CONSTANT NUMBER := 0.15;
    
    -- Declaración del procedimiento (solo la firma)
    PROCEDURE calcular_bono(p_empleado_id IN NUMBER, p_bono OUT NUMBER);
END hr_utils;
/

-- 2. EL CUERPO (Lo privado/La lógica)
CREATE OR REPLACE PACKAGE BODY hr_utils AS
    -- Procedimiento privado (no está en el spec)
    PROCEDURE validar_empleado(p_id IN NUMBER) IS
    BEGIN
        -- lógica interna para validar...
    END;

    -- Implementación del procedimiento público
    PROCEDURE calcular_bono(p_empleado_id IN NUMBER, p_bono OUT NUMBER) IS
    BEGIN
        validar_empleado(p_empleado_id); -- Llama al privado
        -- Lógica para calcular y retornar el bono...
    END;
END hr_utils;
/
```
### Diferencia entre Packages y Stored Procedures

| Característica      | Stored Procedure (Procedimiento Almacenado)                                                                    | Package (Paquete)                                                                                                           |
| ------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Definición**      | Es un bloque de código único e independiente en la base de datos diseñado para ejecutar una acción específica. | Es un contenedor/librería que agrupa lógicamente múltiples procedimientos, funciones, variables y cursores.                 |
| **Estructura**      | Un solo bloque de código (Declaración, Ejecución, Excepciones).                                                | Se divide en dos partes: **Specification** (la interfaz o "índice") y el **Body** (el código real).                         |
| **Rendimiento**     | Se carga en memoria cuando se invoca y se descarga al terminar.                                                | Al invocar un elemento del paquete, **todo el paquete se carga en memoria**. Las siguientes llamadas son mucho más rápidas. |
| **Encapsulamiento** | Todo es público.                                                                                               | Permite tener variables globales (persistentes durante la sesión) y procedimientos "privados" (ocultos en el Body).         |
### La Función `SQLEXEC`

`SQLEXEC` es una de las funciones más avanzadas de GoldenGate. Te permite **ejecutar sentencias SQL, Stored Procedures o Functions de la base de datos en tiempo real** mientras GoldenGate está procesando un registro (generalmente en el Replicat).

- **Casos de uso principales:** * **Enriquecimiento de datos (Lookups):** Leer el ID de una tabla origen y hacer una consulta (SELECT) a otra tabla en el destino para obtener la descripción antes de insertar el dato.
    - **Lógica de negocio compleja:** Invocar un procedimiento almacenado en el destino pasando parámetros desde el registro que viene del origen.
      
- **Ejemplo de sintaxis básica (Consulta de Lookup):**
     `MAP ventas.ordenes, TARGET ventas.ordenes_hist,` `SQLEXEC (ID lookup_cliente, QUERY "SELECT nombre FROM clientes WHERE id_cliente = ?", PARAMS (p1 = id_cliente)),` `COLMAP (USEDEFAULTS, nombre_cliente = lookup_cliente.nombre);`
#### Ejemplo práctico
Imagina que estamos replicando datos de nuevos empleados desde una tabla origen hacia un Data Warehouse (destino).

- **Origen:** `rh_origen.empleados` (Tiene: `id_empleado`, `nombre`, `salario`, `estado`)
- **Destino:** `rh_destino.empleados_dw` (Tiene las mismas columnas, pero añade una extra: `bono_asignado`)
- **Regla de Negocio:** Solo queremos procesar a los empleados cuyo estado sea 'ACTIVO' (usaremos un **FILTER** o **WHERE**). Para cada uno de ellos, vamos a calcular su bono usando el procedimiento que está dentro del paquete `hr_utils.calcular_bono` (usaremos **SQLEXEC**).

#### El Archivo de Parámetros del Replicat (`rep_rh.prm`)
Así es como se vería el código completo de tu archivo de configuración. Lee los comentarios que he puesto dentro del código:


```GoldenGate
-- 1. Configuración básica del Replicat
REPLICAT rep_rh
USERIDALIAS alias_destino DOMAIN mi_dominio
DISCARDFILE ./dirrpt/rep_rh.dsc, APPEND, MEGABYTES 50

-- 2. Inicio del mapeo
MAP rh_origen.empleados, TARGET rh_destino.empleados_dw,

-- 3. Aplicamos el Filtro: Solo procesar si el estado es ACTIVO
-- Podríamos usar WHERE (estado = 'ACTIVO'), pero usemos FILTER para ilustrar:
FILTER (estado = 'ACTIVO'),

-- 4. Invocamos el Package usando SQLEXEC
SQLEXEC (
    -- SPCALL le dice a GoldenGate que llame a un Stored Procedure / Package
    SPCALL hr_utils.calcular_bono, 
    
    -- PARAMS: Le pasamos los datos del registro origen al procedimiento
    -- El parámetro 'p_empleado_id' del procedimiento recibe el valor de la columna 'id_empleado'
    PARAMS (p_empleado_id = id_empleado), 
    
    -- RESULTS: Capturamos la salida del procedimiento (el parámetro OUT)
    -- Guardamos el valor devuelto 'p_bono' en una variable temporal de GG llamada 'v_bono'
    RESULTS (v_bono = p_bono)
),

-- 5. Mapeo de columnas (COLMAP)
-- USEDEFAULTS mapea automáticamente las columnas que se llaman igual en origen y destino
COLMAP (
    USEDEFAULTS,
    -- Aquí asignamos la variable que obtuvimos del SQLEXEC a la columna física del destino
    bono_asignado = v_bono
);
```

Ahora analisemos que hace este archivo de parámetros
- **Eficiencia primero:** Al poner el `FILTER (estado = 'ACTIVO')` antes del `SQLEXEC`, GoldenGate es inteligente. Si llega un registro con estado 'INACTIVO', GoldenGate lo descarta de inmediato y **no** ejecuta el procedimiento en la base de datos. ¡Esto ahorra muchísimo procesamiento!
- **La directiva `SPCALL`:** Dentro de `SQLEXEC`, `SPCALL` (Stored Procedure Call) es la instrucción específica para invocar código PL/SQL. Fíjate cómo usamos la notación `paquete.procedimiento` (`hr_utils.calcular_bono`). Como vimos antes, Oracle cargará todo el paquete `hr_utils` en memoria la primera vez, haciendo que las miles de llamadas siguientes sean rapidísimas.
- **El puente de datos (`PARAMS` y `RESULTS`):** `PARAMS` es la autopista de ida (GoldenGate -> Base de Datos) y `RESULTS` es la autopista de vuelta (Base de Datos -> GoldenGate).
- **La materialización (`COLMAP`):** GoldenGate guarda el resultado en una especie de "variable en el aire" (`v_bono`). Para que ese dato aterrice en la tabla física, usamos la cláusula `COLMAP` para enlazar esa variable con la columna real de la tabla destino (`bono_asignado`).

### Manejo de Errores
La mejor práctica es que tu Stored Procedure o Package "atrape" sus propios errores para que no le exploten en la cara a GoldenGate. Para esto, usamos parámetros de salida (`OUT`) que actúen como "banderas" de estado.

Imagina que modificamos nuestro SP de cálculo de bonos:

SQL

```
CREATE OR REPLACE PROCEDURE calcular_bono (
    p_empleado_id IN NUMBER, 
    p_bono        OUT NUMBER,
    p_estado      OUT VARCHAR2 -- ¡Nueva bandera de error!
) IS
BEGIN
    -- Lógica de negocio simulada
    IF p_empleado_id IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'ID nulo no permitido');
    END IF;
    
    p_bono := 1000;
    p_estado := 'OK'; -- Todo salió bien
    
EXCEPTION
    -- Atrapamos cualquier error de la base de datos
    WHEN OTHERS THEN
        p_bono := 0; -- Asignamos un valor por defecto seguro
        p_estado := 'ERROR: ' || SUBSTR(SQLERRM, 1, 100); -- Devolvemos el mensaje de error
END calcular_bono;
/
```

Al hacer esto, el SP _nunca_ le devuelve un error técnico (excepción) a GoldenGate; siempre devuelve un resultado, y GoldenGate puede mapear ese mensaje de error a una columna de tu tabla destino para que lo audites después.

#### Manejo desde GoldenGate (`SQLEXEC`)

Incluso si el SP atrapa sus errores, pueden ocurrir fallas de red, problemas de permisos, o caídas de la base de datos. Para proteger a GoldenGate de errores a nivel de ejecución SQL, usamos opciones integradas directamente en la instrucción `SQLEXEC`.

GoldenGate nos ofrece el parámetro `ERROR` dentro de `SQLEXEC` para dictar el comportamiento ante fallos.

|Opción de ERROR|Comportamiento del Replicat|Cuándo usarlo|
|---|---|---|
|**FATAL** (Por defecto)|Aborta (ABEND) el proceso y se detiene.|Cuando el dato es crítico y no puedes permitirte insertar un registro incompleto.|
|**IGNORE**|Ignora el error, deja las variables de retorno vacías y continúa procesando el registro.|Cuando el enriquecimiento del dato es opcional y no afecta la integridad del negocio.|
|**REPORT**|Escribe el error en el archivo de reporte (discard file) y continúa procesando.|Es la opción más recomendada. Te permite seguir operando pero deja un rastro auditable del problema.|
#### Juntando todo en el Archivo de Parámetros (.prm)

```GoldenGate
MAP rh_origen.empleados, TARGET rh_destino.empleados_dw,
FILTER (estado = 'ACTIVO'),

SQLEXEC (
    SPCALL hr_utils.calcular_bono, 
    PARAMS (p_empleado_id = id_empleado), 
    -- Mapeamos nuestras dos salidas, incluyendo la bandera de estado
    RESULTS (v_bono = p_bono, v_estado = p_estado),
    
    -- ¡La magia del manejo de errores en GoldenGate!
    -- Si el motor de BD falla al ejecutar el SP, repórtalo pero no te detengas
    ERROR REPORT
),

COLMAP (
    USEDEFAULTS,
    bono_asignado = v_bono,
    -- Guardamos el estado (sea "OK" o el mensaje de error) en una columna de auditoría
    mensaje_ejecucion = v_estado
);
```

Al final tenemos un sistema con las siguientes características:

1. **Resiliencia:** Tu Replicat ya no se va a detener por un dato sucio.
2. **Trazabilidad:** Si algo falla a nivel de negocio (ej. empleado no existe), el mensaje de error de Oracle queda guardado físicamente en la fila del Data Warehouse gracias a la variable `v_estado`.
3. **Visibilidad:** Si algo falla a nivel de infraestructura, GoldenGate escribirá el evento en su _Discard File_ gracias a `ERROR REPORT`, alertando al equipo de operaciones sin frenar la tubería de datos.