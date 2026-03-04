---
tags:
  - GoldenGate
---
Las Macros de *GoldenGate* son bloques de código reutilizable que se definen una vez y se pueden mandar a llamar cuantas veces como necesitemos.

Estas se pueden declarar al inicio de los archivos de parámetros o pueden ser archivos totalmente independientes. Para nosotros es mejor crear las macros como archivos independientes que se mandan llamar cuando los necesitemos en cualquier archivo de parámetros. Pero puede que a veces necesitemos macros exclusivamente para un proceso y por eso las escribimos directamente en el archivo de parámetros.

### Estructura
Toda macro tiene una estructura estricta de 4 partes, y **siempre debe ir en la parte superior de tu archivo de parámetros**, antes de tus instrucciones `MAP`.

Aquí está el esqueleto básico:

```sql
MACRO #nombre_de_la_macro  <-- 1. DECLARACIÓN: Siempre debe empezar con "#"
PARAMS (#variable1) <-- 2. PARÁMETROS (Opcional): Variables que puede tomar
BEGIN    <-- 3. INICIO: A partir de aquí, GoldenGate empezará a copiar.
    -- El código reciclable va aquí
    -- Puede ser una línea o cincuenta.
END;
```

### Como Archivo Independiente
Para crear un archivo de Macros usamos:
```
SH vi ./dirprm/nombre_macro.mac
```
- Usamos `SH` para llamar directamente a la terminal de Linux y que ejecute vim en la ruta de parámetros y cree la macro.

Y dentro de él escribimos el código que va a llevar por ejemplo:
```vim
-- Librería Central de Macros SAIF
MACRO #auditar_operacion
BEGIN
    gg_op_type = @GETENV ('GGHEADER', 'OPTYPE'),
    gg_timestamp = @GETENV ('GGHEADER', 'COMMITTIMESTAMP')
END;
```
Ahora, para mandarla llamar dentro del archivo de parámetros usamos `INCLUDE` para el ejemplo veremos como se ve en un *Replicat*:

```vim
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD Oracle123
ASSUMETARGETDEFS

-- Importamos nuestra librería externa de macros
INCLUDE ./dirprm/auditoria.mac

-- Ahora podemos usar la macro tranquilamente
MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone,
COLMAP (USEDEFAULTS, #auditar_operacion());
```

### Como Parte del Archivo de Parámetros
Para definir una macro dentro de un archivo de parámetros, la declararemos después del nombre del parámetro y las credenciales de conexión, para este ejemplo usaremos un *Replicat*.

```vim
REPLICAT REMP_DES
USERID c##ggadmin@//goldengate_odb2:1521/ORCLPDB1, PASSWORD Oracle123
ASSUMETARGETDEFS

-- ======== ZONA DE MACROS ========
MACRO #auditar_operacion
BEGIN
    gg_op_type = @GETENV ('GGHEADER', 'OPTYPE'),
    gg_timestamp = @GETENV ('GGHEADER', 'COMMITTIMESTAMP')
END;
-- ================================

-- Mapeo de la tabla 1
MAP ORCLPDB1.hr.employees, TARGET ORCLPDB1.hr.employees_clone,
COLMAP (USEDEFAULTS, #auditar_operacion());

-- Si tuviéramos más tablas, las agregaríamos así de fácil:
-- MAP ORCLPDB1.hr.departments, TARGET ORCLPDB1.hr.departments_clone,
-- COLMAP (USEDEFAULTS, #auditar_operacion());
```
