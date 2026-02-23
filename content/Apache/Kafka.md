---
tags:
  - Apache
---
Kafka está diseñado para manejar flujos continuos de datos en el instante en que ocurren (procesamiento _Streaming_).

### Situación de Ejemplo
Imagina que tienes una aplicación de un sensor de temperatura enviando nuevas lecturas cada segundo. Tienes tres sistemas diferentes que necesitan esos datos urgentemente:

1. Un panel de control en vivo.
2. Un sistema de alarmas.
3. Una base de datos para el historial.

Si obligamos al sensor a conectarse y enviar cada lectura individualmente a esos tres sistemas, se sobrecargaría rápidamente. Para evitar esto, Kafka toma el mensaje y lo trabaja en el modelo **Publicación-Suscripción** (Pub/Sub), que es muy parecido a un tablero de anuncios.

Así es como se estructuran los actores en Kafka:

- **Productor (Producer):** Es tu sensor. Su único trabajo es enviar (_publicar_) la lectura a Kafka y olvidarse. No le importa quién va a leer ese dato.
- **Tema (Topic):** Es el canal o "tablero de anuncios" dentro de Kafka donde se guardan esos mensajes. Podríamos llamarlo `lecturas_temperatura`.
- **Consumidor (Consumer):** Son tus tres sistemas (panel, alarmas, base de datos). Ellos se _suscriben_ a ese tema y "jalan" o leen los mensajes de forma independiente, cada uno a su propia velocidad.

Como cada consumidor lee a su propio ritmo, no interfieren entre sí. El panel en vivo puede estar leyendo el milisegundo en que llega el dato, mientras que la base de datos histórica puede estar procesándolos en bloques cada 5 segundos.

Supongamos que el sistema de alarmas sufre un error y se apaga durante 10 minutos. Mientras tanto, el sensor sigue enviando datos a Kafka cada segundo sin parar.

A diferencia de los sistemas de mensajes tradicionales que borran el dato en cuanto alguien lo lee, Kafka guarda todo en su disco durante un tiempo determinado (horas, días o semanas).

Para no perderse, Kafka usa un concepto llamado **Offset** (desplazamiento). Este coloca un *marcador* para cada agente en el último mensaje que leyeron. Cada mensaje tiene un número de orden. Si el sistema de alarmas se apaga, y su *marcador* se quedó, en el mensaje 1,050. Al encenderse 10 minutos después, va directo a Kafka y pide: "Dame todo a partir del 1,051". Así, lee de golpe todo lo acumulado, se pone al día y vuelve a su ritmo normal sin perder ninguna lectura.