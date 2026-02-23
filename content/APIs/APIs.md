---
tags:
  - API
---
### Application Programing Interface (API)
Las ***APIs*** son un conjunto de reglas estructuradas que permite a dos aplicaciones comunicarse entre sí. Son un intermediario que lleva los mensajes de una parte a otra, permitiendo que programas construidos con tecnologías diferentes se puedan entender sin problemas.

Utilizan la arquitectura ***Cliente Servidor*** para funcionar, donde:
1. El cliente es quien inicia la conversación (una App, Navegador u otro servidor que necesita datos) con un ***Request***.
2. El servidor tiene los datos guardados, este recibe el ***Request*** y genera un ***Response*** en formato JSON.

![[Pasted image 20260204161832.png|350]]

### JSON
Las respuestas son generadas en archivos JSON ya que es un formato de texto ligero que se ordena por ***Keys*** y ***Values***, el ejemplo muestra información de clima de la Ciudad de México:

```JSON
{
	"key": "value",
	"ciudad": "Ciudad de México",
	"temperatura": 24,
	"unidad": "Celsius",
	"condicion": "Soleado"
}
```

### Verbos HTTP
Para generar acciones las ***APIs*** utilizan las siguientes instrucciones:

1. **GET:** Para **obtener** o leer datos (sin cambiar nada).
2. **POST:** Para **enviar** datos nuevos (crear algo, como un nuevo usuario).
3. **PUT:** Para **actualizar** datos que ya existen.
4. **DELETE:** Para **borrar** datos.

#### Códigos de Estado HTTP
Una vez se recibe la instrucción el servidor puede responder con alguno de estos códigos de confirmación:

- **200 (OK):** ¡Éxito! Aquí tienes lo que pediste.
- **401 (Unauthorized):** Si intentas entrar sin un API Key o si la clave es falsa. El servidor dice: "No te conozco".
- **403 (Forbidden):** Si la API Key es válida pero no tiene permiso para realizar alguna acción. El Servidor dice: "No puedes hacer eso".
- **404 (Not Found):** Error del cliente. Lo que buscas no existe (la clásica "Página no encontrada").
- **500 (Internal Server Error):** Error del servidor. Algo se rompió en la cocina y no es tu culpa.

### Endpoints
Estos son las direcciones a las que la API se dirige en el servidor por ejemplo:

Si tenemos esta URL: `https://api.clima.com/v1/pronostico` Nos devolverá información del clima futuro, mientras que `https://api.clima.com/v1/historial` Nos devolverá información pasada.

### Seguridad
Podemos implementar seguridad a nuestras API y que solo tengan acceso a los datos aplicaciones autorizadas.

#### API Keys (Identificación)
Estas son una cadena única conformada de números y letras que te proporciona el servidor, esta se debe incluir como parámetro en las peticiones:

En lugar de solo pedir: `GET /clima?ciudad=Mexico` La petición ahora debe incluir la clave: `GET /clima?ciudad=Mexico&apikey=ABC-123-XYZ`

Así el servidor puede identificar quién está haciendo peticiones y si son demasiadas puede bloquear esa API en específico para evitar el colapso del servidor.

#### Token (JWT)
A diferencia del ***API Key*** que es permanente, los Token tienen tiempo de caducidad y después de ese periodo se tiene que generar uno nuevo.

En desarrollo moderno es más común utilizar estos tokens temporales por seguridad.
