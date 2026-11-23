# Práctica 3: Controlador Máquina Expendedora

El objetivo de esta práctica es diseñar e implementar el controlador software de una máquina expendedora de café sobre una placa Arduino UNO, integrando varios sensores y actuadores del kit de laboratorio: una pantalla LCD, un joystick analógico, un sensor de temperatura y humedad (DHT11), un sensor de distancia por ultrasonidos, un pulsador externo y dos LEDs.

El sistema debe comportarse como un sistema empotrado de tiempo real: responder a eventos físicos (la presencia de un cliente, las pulsaciones del botón o los movimientos del joystick) sin quedarse bloqueado, por lo que no se permite el uso de delay(). Para ello se han aplicado las técnicas vistas en teoría y laboratorio: multitarea cooperativa con ArduinoThread, interrupciones hardware para medir con precisión la duración de las pulsaciones, un watchdog que recupera el sistema ante bloqueos y la lectura de sensores de forma periódica y no bloqueante.

## Esquema Fritzing:

<img width="2374" height="2313" alt="p3_bb (1)" src="https://github.com/user-attachments/assets/b1c90647-229c-4ecb-a180-7d7d64ce6a0f" />

## Descripción del funcionamiento:

El funcionamiento de la máquina se ha organizado mediante una máquina de estados. Para ello, el programa se divide en tres modos principales: Arranque, Servicio y Administrador. Cada uno de ellos se encarga de una parte concreta del funcionamiento de la máquina y permite separar claramente las distintas tareas del sistema. Además, tanto el modo Servicio como el modo Administrador se dividen en varios submodos, lo que facilita controlar la transición entre las diferentes pantallas y acciones de forma ordenada. A continuación, se explican cada una de estas secciones.

El programa comienza con el modo ARRANQUE: la máquina se enciende, el led1 se enciende 3 veces en intervalos de 1
segundo y, a su vez, la pantalla LCD muestra el mensaje "CARGANDO …". Para ello, utilizamos un thread (callback_parpadeoLed) que se encarga del parpadeo del led y de pasar al modo servicio
Una vez concluidos los 3 ciclos de parpadeo, el LED1 se apaga por completo y el
sistema pasa al modo SERVICIO.

Al pasar al modo SERVICIO
