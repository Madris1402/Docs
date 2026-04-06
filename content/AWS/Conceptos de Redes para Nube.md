---
tags:
  - AWS
---
Veamos conceptos más básicos y universales de las redes en la nube (aplicables a AWS, Azure o Google Cloud) de una forma estructurada y sencilla:
### La VPC (Virtual Private Cloud)
Cuando entras a la nube pública, no compartes tu tráfico con el de otras empresas. Lo primero que haces es crear tu propia red virtual, aislada y privada. A esto se le llama **VPC** (o VNet en Azure). Es esencialmente tu propio centro de datos virtual en la nube. Todo lo que construyas (servidores, bases de datos, clusters) vivirá dentro de los límites de esta VPC.
### Segmentando la Red: Subredes (Subnets)
Una VPC abarca toda una región de la nube, por lo que necesitamos dividirla en bloques más pequeños y manejables llamados **subredes**. Esto nos permite organizar y, sobre todo, asegurar nuestros recursos aislando componentes. Existen dos tipos críticos que debes conocer:

- **Subred Pública:** Tiene una ruta directa hacia y desde Internet. Aquí se colocan los recursos que deben ser accesibles para el usuario final, como los Balanceadores de Carga o servidores web front-end.

- **Subred Privada:** No tiene acceso directo desde Internet. El aislamiento es su principal característica. Aquí es donde _siempre_ debes colocar tus recursos críticos, como tus microservicios internos en Java/Python, tus aplicaciones de MuleSoft y tus bases de datos.

### Gateways
Si las subredes son habitaciones en un edificio, los Gateways son las puertas hacia la calle.

- **Internet Gateway (IGW):** Es la puerta principal de tu VPC. Se adjunta a la VPC y permite que los recursos dentro de las _subredes públicas_ puedan recibir tráfico de Internet y responder a él.

- **NAT Gateway:** ¿Qué sucede si tu servidor en la _subred privada_ necesita descargar una librería de Python desde Internet, pero por seguridad no quieres que sea visible desde el exterior? Utilizas un NAT Gateway. Este componente se coloca en la subred pública y actúa como un intermediario: permite que la subred privada salga a Internet, pero bloquea cualquier intento de conexión que provenga desde afuera.    

### Controlando el Tráfico: Route Tables (Tablas de Enrutamiento)
Son literalmente mapas o señales de tránsito. Cada subred está asociada a una tabla de enrutamiento que contiene reglas (rutas). Estas reglas le dicen al tráfico de red hacia dónde debe dirigirse. Por ejemplo, una regla puede decir: "Si el tráfico va dirigido a una dirección IP de Internet, envíalo al Internet Gateway".

### Security Groups y NACLs
La nube opera bajo un modelo de "confianza cero"; por defecto, todo el tráfico está bloqueado hasta que tú digas lo contrario.

- **Security Groups (Grupos de Seguridad):** Actúan como un firewall a nivel del recurso individual (por ejemplo, una máquina virtual o un contenedor de Docker). Tú defines reglas explícitas de entrada y salida. Por ejemplo: "Solo permite tráfico entrante por el puerto `443` (HTTPS) desde cualquier lugar" o "Permite el puerto `3306` (Base de Datos) _únicamente_ si la petición viene del servidor web".

- **Network ACLs (Listas de Control de Acceso):** Operan un nivel más arriba, como un firewall para la _subred completa_. Filtran todo el tráfico que entra o sale de ese segmento de red.