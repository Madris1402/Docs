---
tags:
  - GoldenGate
---
### Filtrado de Datos: `WHERE` vs. `FILTER`

Ambas cláusulas sirven para decidir si un registro (fila) debe procesarse o descartarse, pero tienen capacidades diferentes.

#### La cláusula `WHERE`

Es la opción más sencilla y rápida. Actúa casi igual que la cláusula `WHERE` estándar de SQL. Se utiliza para **condiciones lógicas básicas** comparando el valor de una columna con un valor constante u otra columna.

- **Cuándo usarla:** Para filtros simples (ej. `STATUS = 'ACTIVE'`).
- **Sintaxis en el archivo de parámetros:**
	 `MAP hr.employees, TARGET hr.employees_tgt, WHERE (department_id = 10);`
Solo admite operadores básicos (como `=`, `<>`, `>`, `<`, `AND`, `OR`) y no soporta operadores aritméticos ni tipos de datos de punto flotante.
#### La cláusula `FILTER`

Es mucho más poderosa y flexible que `WHERE`. Se utiliza cuando necesitas realizar **operaciones complejas**, matemáticas, o invocar las funciones integradas de GoldenGate (las que empiezan con `@`, como `@COMPUTE`, `@STRFIND`, `@DATE`).

- **Cuándo usarla:** Cuando necesitas transformar el dato antes de evaluarlo o usar funciones de GoldenGate. Si `FILTER` se evalúa como cero (0), el registro se ignora; si es distinto de cero (generalmente 1), se procesa.
- **Sintaxis en el archivo de parámetros:**
     `MAP hr.employees, TARGET hr.employees_tgt, FILTER (@COMPUTE(salary + commission) > 5000);`
Además, es posible especificar múltiples cláusulas `FILTER` en una misma sentencia `TABLE` o `MAP`, mientras que solo se permite una cláusula `WHERE`.

> [!info] Nota Técnica
> Si necesitas filtrar columnas con juegos de caracteres multibyte o incompatibles con el sistema operativo local, GoldenGate no admite el uso de `FILTER` para esas columnas específicas.