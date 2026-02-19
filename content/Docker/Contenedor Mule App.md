---
tags:
  - Docker
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
### La idea
Desplegar un contenedor [[Docker]] con una MuleApp para facilitar su distribución y mantenimiento.

### El Problema
Como *CloudHub 2.0* utiliza una arquitectura de contenedores de pago para desplegar las MuleApps, *Mule Standalone* no funciona correctamente al detectar que se encuentra en un contenedor. 

En sí, el motor de mule arranca, y muestra que detecta correctamente las *MuleApps* pero después de mostrar eso, no encuentra el acceso a *Maven* mostrando ya sea algún error de conexión o nada en lo absoluto. 

Se presume que Salesforce genera este comportamiento intencionalmente para orillar a los usuarios a usar *CloudHub* donde el despliegue se realiza sin mayor complicación.

### Otro Enfoque
Mule ofrece el *Mule Kernel* para despliegues en docker, pero este no está completo y no tiene acceso a las dependencias de *Maven* ya que estas son exclusivas para la versión *Standalone* que requiere licencias para acceder de cualquier forma. Sin acceso a *Maven*, solo se cuenta con los *componentes core* de mule y se limita a usar el repositorio público de *Mule*.

Esto orilla al el desarrollo de MuleApps sin plugins lo que complicaría el desarrollo de las mismas y aumentarían los tiempos de desarrollo.

### Conclusiones
Es mejor optar por un sistema híbrido usando *Control Plane* para los despliegues ya que con Control Plane el despliegue es mucho más rápido y el mantenimiento al ser *On-premise* puede ser más simple.