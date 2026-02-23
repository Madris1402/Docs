---
tags:
  - Apache
---
Apache Iceberg es un **formato de tabla**. Se coloca justo encima de sistemas de almacenamiento masivo (como [[Hadoop]]) para darle estructura y agilidad, haciéndolo funcionar casi como una base de datos tradicional.

Si quisieramos editar registros específicos de un archivo grande (100GB) que se encuentra en *Hadoop*, tendríamos que leer completamente el archivo, alterarlo y volver a escribir los 100GB del archivo. Lo cual es muy lento y consume demasiados recursos.

### Su Función

En lugar de ver los datos como una masa gigante de texto, Iceberg crea una capa de "inteligencia" (metadatos) sobre los archivos en Hadoop. Funciona como un índice muy detallado.

Iceberg mantiene un registro de archivos más pequeños y sabe exactamente qué rangos de datos están en cada uno. Si necesitas borrar la compra de un usuario, Iceberg consulta su índice, identifica el archivo específico de unos pocos Megabytes que contiene el dato, y solo reescribe o marca ese pequeño fragmento. Además, permite hacer transacciones seguras (como las de un banco) asegurando que si un proceso falla a la mitad, tu tabla no quede corrupta.