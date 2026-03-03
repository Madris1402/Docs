---
tags:
  - Docker
---
Es una herramienta oficial de [[Docker]] que <mark style="background:#fdbfff">permite definir y ejecutar aplicaciones de múltiples contenedores</mark>. Se declara toda la infraestructura en la terminal para cada contenedor en un archivo `yaml` y luego se levanta todo con un solo comando.

### Flujo de Trabajo
Trabajar con Docker Compose generalmente se resume en tres pasos:

1. **Definir el entorno de la app:** Se crea un `Dockerfile` para las aplicaciones personalizadas (por ejemplo, tu código en Python o Java) para que puedan ser reproducidas en cualquier lugar.
2. **Define los servicios:** Se crea un archivo llamado `docker-compose.yml`. Aquí es donde declaramos qué contenedores se necesitan, cómo se llaman, qué puertos exponen, a qué redes pertenecen y qué volúmenes usan.
3. **Ejecuta y despliega:** Escribes en tu terminal el comando `docker compose up`. Docker Compose lee el archivo YAML, descarga las imágenes necesarias, crea las redes, configura los volúmenes y arranca todos los contenedores en el orden correcto.

### El archivo `docker-compose.yml`

- **Services (Servicios):** Son los diferentes contenedores que componen tu aplicación (ej. `web`, `database`, `api`).
- **Networks (Redes):** Docker Compose crea automáticamente una red interna para que tus servicios (contenedores) puedan comunicarse entre sí usando sus nombres, sin que tengas que lidiar con direcciones IP estáticas.
- **Volumes (Volúmenes):** Te permiten guardar datos de forma persistente. Por ejemplo, si tu contenedor de base de datos se reinicia, no quieres perder toda tu información. Un volumen guarda esos datos de forma segura en tu máquina anfitriona.

### Ventajas

- **Entornos unificados:** Todo el equipo de desarrollo puede ejecutar la aplicación exactamente con la misma configuración.
- **Aislamiento:** Puedes tener múltiples entornos aislados (desarrollo, pruebas, producción) en un mismo servidor físico.
- **Productividad:** Iniciar, detener (`docker compose down`) y reconstruir tu aplicación entera toma solo unos segundos.