---
tags:
  - API
---
> [!abstract] Índice
> ```table-of-contents 
> ```

---
Es una filosofía de integración moderna que utiliza [[APIs]] reciclables para conectar datos y aplicaciones, está compuesto de 3 capas

### Capas de API-Led
1. **System APIs (Capa de Sistema)**:
    - **Función:** Desbloquean los datos de los sistemas centrales (SAP, Salesforce, Bases de Datos, Mainframes).
    - **Regla:** Son puras y simples. Solo hacen CRUD (Crear, Leer, Actualizar, Borrar). No tienen lógica de negocio compleja.
    - ***Ejemplo***: `Cliente System API` (que solo sabe hablar con la base de datos de clientes).
    
2. **Process APIs (Capa de Proceso)**:
    - **Función**: Orquestan y combinan datos de varias System APIs.
    - **Regla**: Aquí van las reglas de negocio (ej. "Si el cliente compra más de $100, aplicar descuento"). No tocan directamente las bases de datos; llaman a las System APIs.
    - ***Ejemplo***: `Procesamiento de Pedidos API` (toma datos del cliente y del inventario para crear una orden).
3. **Experience APIs (Capa de Experiencia)**:
    - **Función**: Entregan los datos listos para ser consumidos por un canal específico (App Móvil, Web, Reloj Inteligente).
    - **Regla**: Formatean la información para que sea fácil de leer por el usuario final.
    - ***Ejemplo***: `Mobile App API` (entrega un JSON ligero para que la app no consuma muchos datos).
### Beneficios e Impacto
- **Reusabilidad y Velocidad**: Los componentes se pueden reciclar a través de diferentes proyectos, acelerando significantemente los tiempos de desarrollo.
- **Agilidad y Flexibilidad**: Dado que las APIs son modulares, las organizaciones pueden reconfigurar o reemplazar sistemas fácilmente sin romper toda la red de integraciones.
- **Seguridad y Gobernación**: Permite una administración, monitoreo y aseguramiento de datos simple ya que siguen las normas de la organización.

Esta es una metodología esencial para transformación digital, permitiendo a las organizaciones simplificar compendios de datos y crear un negocio donde las integraciones tecnológicas son bloques.