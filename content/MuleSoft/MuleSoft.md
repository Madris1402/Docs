---
tags:
  - MuleSoft
---
MuleSoft es una empresa (propiedad de Salesforce) que ofrece una plataforma de integración llamada **Anypoint Platform**. Tiene la capacidad de conectar cualquier sistema, aplicación o fuente de datos.

### Conceptos Básicos
Imagina que tienes un sistema viejo (Legacy) que escrito en `BASIC`, una base de datos moderna `AWS S3` y una aplicación móvil en `Flutter`. MuleSoft actúa como el **traductor universal** en el medio. Recibe la información de uno, la transforma y la entrega al otro en el formato que necesita, todo en tiempo real.

- **Sin MuleSoft (Integración Punto a Punto):** Si tienes 10 sistemas y los conectas todos contra todos, terminas con un "plato de espagueti". Si uno cambia, todo se rompe. Es frágil y difícil de mantener.

- **Con MuleSoft (Application Network):** Creas una red ordenada donde cada sistema tiene un conector estandarizado (*API*). Si cambias un sistema, solo cambias su enchufe, no toda la red.

Para que esto funcione se utiliza la filosofía [[API-Led Connectivity]].

### Herramientas Principales
- **Anypoint Platform:** El sitio web (panel de control) desde donde gestionas todo.
  
- **Anypoint Studio:** El software de escritorio (basado en Eclipse) donde los desarrolladores escriben el código (arrastrando y soltando cajitas).
  
- **Mule Runtime Engine:** El motor que hace correr las aplicaciones (es ligero y basado en Java).
  
- **DataWeave:** El lenguaje de programación propio de MuleSoft para transformar datos (ej. convertir un XML a JSON). Es extremadamente potente.
  
- **Exchange:** Es como una "App Store" privada de tu empresa. Ahí publicas tus APIs para que otros desarrolladores las reutilicen en lugar de crear código nuevo.