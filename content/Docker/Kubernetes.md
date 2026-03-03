---
tags:
  - Docker
---
*Kubernetes* *(Abreviado como **K8s**)* es un sistema de código abierto diseñado para automatizar el despliegue, escalado y manejo de aplicaciones en contenedores. 

Supongamos que creamos un contenedor (En *K8s* se le llama *Pod*). Si este contenedor recibe mucho tráfico hay que montar más para distribuir la carga. Si uno de estos falla y se apaga, se tiene que volver a encender manualmente (O con algún sistema que esté todo el tiempo monitorizando). Y mientras más escala la operación más complicado es dar mantenimiento.

Para esto *Kubernetes* decide dónde poner los Pods, cuántos se necesitan y verificar la integridad de estos.

### Conceptos Clave

*Kubernetes* funciona utilizando agrupación en ***Clusters***, dentro de estos existen:
- **Control Plane**: Es el nodo maestro, toma las decisiones globales. Decide qué máquinas van a correr las aplicaciones, monitorea si todo está funcionando correctamente y maneja las peticiones que el usuario envie.
- **Nodes**: Son las máquinas físicas o virtuales que realmente ejecutan la aplicación, siguiendo las órdenes del *Control Plane*.
- **Pods**: K8s no maneja contenedores directamente, sino *Pods*. Estos pueden contener uno o más contenedores Docker. Si se necesita escalar alguna aplicación, K8s no hace más grande el Pod, sino que crea más réplicas de ese Pod.

### Ventajas
- **Recuperación Automática (Self-healing)**: Si un pod falla, el servidor se reinicia o se cuelga, K8s lo detecta automáticamente, destruye el Pod defectuoso y crea uno nuevo.
- **Escalabilidad Automática**: Si hay un pico de tráfico, K8s crea más Pods para soportar la carga. Cuando el tráfico baja, elimina los Pods que sobren para ahorrar recursos.
- **Balanceo de Carga**: Si hay 5 Pods corriendo una misma aplicación, K8s distribuye el tráfico para que ninguno se sature.
