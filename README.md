# Práctica 3: Controlador Máquina Expendedora

El objetivo de esta práctica es diseñar e implementar el controlador software de una máquina expendedora de café sobre una placa Arduino UNO, integrando varios sensores y actuadores del kit de laboratorio: una pantalla LCD, un joystick analógico, un sensor de temperatura y humedad (DHT11), un sensor de distancia por ultrasonidos, un pulsador externo y dos LEDs.

El sistema debe comportarse como un sistema empotrado de tiempo real: responder a eventos físicos (la presencia de un cliente, las pulsaciones del botón o los movimientos del joystick) sin quedarse bloqueado, por lo que no se permite el uso de delay(). Para ello se han aplicado las técnicas vistas en teoría y laboratorio: multitarea cooperativa con ArduinoThread, interrupciones hardware para medir con precisión la duración de las pulsaciones, un watchdog que recupera el sistema ante bloqueos y la lectura de sensores de forma periódica y no bloqueante.

## Esquema Fritzing:

<img width="2374" height="2313" alt="p3_bb (1)" src="https://github.com/user-attachments/assets/b1c90647-229c-4ecb-a180-7d7d64ce6a0f" />

## Descripción del funcionamiento:

El funcionamiento de la máquina se ha organizado mediante una máquina de estados. Para ello, el programa se divide en tres modos principales: Arranque, Servicio y Administrador. Cada uno de ellos se encarga de una parte concreta del funcionamiento de la máquina y permite separar claramente las distintas tareas del sistema. Además, tanto el modo Servicio como el modo Administrador se dividen en varios submodos, lo que facilita controlar la transición entre las diferentes pantallas y acciones de forma ordenada. A continuación, se explica cada una de estas secciones.

El programa comienza con el modo **ARRANQUE**: la máquina se enciende, el led1 se enciende 3 veces en intervalos de 1
segundo y, a su vez, la pantalla LCD muestra el mensaje "CARGANDO …". Para ello, utilizamos un thread que se encarga del parpadeo del led y de pasar al modo SERVICIO una vez concluidos los 3 ciclos de parpadeo.

El modo **SERVICIO** está compuesto por distintas fases, los "submodos": ESPERANDO, TEMP_HUM, CATÁLOGO, PREPARANDO y RETIRAR
Al comenzar este modo, la máquina permanece inicialmente en espera de que llegue un cliente con el submodo **ESPERANDO**. Para detectar su presencia se utiliza el sensor de ultrasonidos, que mide de forma periódica la distancia existente frente a la máquina mediante un thread. Cuando se detecta un objeto a menos de 100 cm, el sistema considera que hay un cliente y cambia automáticamente al siguiente submodo, **TEMP_HUM**, que muestra durante 5 segundos la temperatura y la humedad medidas por el sensor DHT11. Una vez transcurrido ese tiempo, la pantalla pasa al modo **CATÁLOGO** y muestra el menú de bebidas disponibles. Tanto los nombres como los precios de los productos se almacenan en arrays, lo que facilita su gestión y posterior modificación desde el modo administrador.
La navegación por el catálogo se realiza mediante el eje vertical del joystick. El usuario puede desplazarse entre las distintas bebidas y confirmar la selección pulsando el botón integrado en el propio joystick. Al seleccionar un producto, la máquina entra en el submodo **PREPARANDO**.
Durante la preparación de la bebida se genera un tiempo aleatorio comprendido entre 4 y 8 segundos para simular el funcionamiento de una máquina real. Mientras transcurre este tiempo, el brillo del LED rojo aumenta progresivamente mediante PWM, proporcionando al usuario una indicación visual del progreso de la preparación. Una vez finalizado este proceso, el LED se apaga y la pantalla muestra el mensaje "RETIRE BEBIDA" durante 3 segundos. Finalmente, la máquina vuelve al estado inicial de espera de un nuevo cliente.

El tercer modo es el modo **ADMINISTRADOR** y permite consultar información del sistema y modificar determinados parámetros de la máquina. Para acceder a él es necesario mantener pulsado el botón externo durante 5 segundos. Una vez dentro, ambos LEDs permanecen encendidos para indicar visualmente que la máquina se encuentra en este modo.
El menú de administración se maneja mediante el joystick, al igual que el menú normal de usuario. Desde este menú es posible consultar la temperatura y la humedad medidas por el sensor DHT11, visualizar la distancia detectada por el sensor de ultrasonidos, mostrar el contador de tiempo de funcionamiento de la placa Arduino y acceder al menú de modificación de precios.
En la opción de modificación de precios se muestra el catálogo de bebidas disponible. El usuario puede seleccionar un producto con el joystick y acceder a la edición de su precio. Una vez dentro, el eje vertical permite aumentar o disminuir el importe en intervalos de 0,05 €, mientras que el botón del joystick confirma el nuevo valor y lo guarda en el array de precios.
En cualquier momento, el usuario puede regresar al menú principal del modo administrador desplazando el joystick hacia la izquierda, salvo durante la edición de precios, donde este mismo movimiento sirve para cancelar los cambios realizados (se recupera el precio original del producto). Además, para salir completamente del modo administrador y volver al funcionamiento normal de la máquina es necesario mantener pulsado el botón externo durante 5 segundos.

## Organización

Con el objetivo de mejorar la legibilidad y facilitar el mantenimiento del programa, la mayor parte de la funcionalidad se ha dividido en funciones independientes, cada una con una responsabilidad concreta. De este modo, tareas como la medición de la distancia, la gestión del modo SERVICIO, la gestión del modo ADMINISTRADOR o la actualización de los distintos menús se implementan de forma modular.

Esta organización evita repetir fragmentos de código que se utilizan en varios puntos del programa y hace que la lógica de funcionamiento resulte más clara y sencilla de seguir. Además, cualquier modificación o ampliación futura puede realizarse de forma más localizada, sin afectar al resto del sistema.

## Recursos utilizados

### Máquina de estados

Como comentábamos en el apartado anterior, El funcionamiento de la máquina se ha implementado mediante una máquina de estados, dividiendo el programa en tres modos principales: **ARRANQUE**, **SERVICIO** y **ADMINISTRADOR**. Además, tanto el modo Servicio como el modo Administrador se subdividen en distintos submodos que representan cada una de las etapas del funcionamiento de la máquina. Esta organización permite que el código sea más claro, modular y fácil de mantener, ya que cada estado únicamente ejecuta las acciones que le corresponden y controla cuándo debe producirse la transición al siguiente estado.

### Threads

Para implementar una multitarea cooperativa se ha utilizado la librería **ArduinoThread** junto con un **ThreadController**, encargado de gestionar la ejecución de los distintos threads registrados. En este proyecto se han creado dos threads: uno dedicado al parpadeo del LED durante el modo de arranque y otro encargado de ejecutar periódicamente la lógica principal del sistema, gestionando tanto el modo Servicio como el modo Administrador.

Cada thread tiene asociado un intervalo de ejecución diferente. El thread del parpadeo se ejecuta cada 1000 ms, haciendo que el LED cambie de estado una vez por segundo durante la secuencia de arranque. Una vez completados los tres parpadeos, el propio thread se deshabilita (`enabled = false`), por lo que deja de ejecutarse y el controlador continúa únicamente con el resto de tareas del sistema. Por su parte, el segundo thread se ejecuta cada 200 ms para actualizar periódicamente la lógica de funcionamiento de la máquina, la lectura de entradas y la información mostrada en la pantalla.

El `ThreadController` se invoca en cada iteración del `loop()` mediante `controller.run()`. En cada llamada comprueba qué threads están habilitados y si ha transcurrido el intervalo de ejecución establecido para cada uno de ellos. Si ambas condiciones se cumplen, ejecuta la función asociada (callback) correspondiente.

Gracias a esta organización ha sido posible ejecutar tareas periódicas con diferentes frecuencias sin recurrir al uso de `delay()`, evitando que el programa quede bloqueado y permitiendo que el sistema continúe respondiendo a eventos mientras se realizan otras tareas.

### Interrupciones

El pulsador externo se gestiona mediante una interrupción hardware (`attachInterrupt`). En lugar de comprobar continuamente el estado del botón dentro del `loop()`, la interrupción detecta automáticamente cuándo el botón cambia de estado (al pulsarse o soltarse).
A partir del instante en el que se pulsa y del momento en el que se libera el botón, el programa calcula la duración de la pulsación. Esto permite diferenciar entre una pulsación larga (de 5 segundos o más), utilizada para entrar o salir del modo administrador, una pulsación corta (de 2 a 3 segundos), utilizada para reiniciar el modo SERVICIO, o una pulsación menor (de 50 ms o menos), que se ignorará considerándola un rebote mecánico.

### Watchdog

El watchdog es un temporizador hardware integrado en el microcontrolador cuyo objetivo es detectar bloqueos del programa. Una vez activado, comienza una cuenta atrás de 2 segundos. Si durante ese tiempo el programa no ejecuta wdt_reset(), el microcontrolador interpreta que el software ha dejado de funcionar correctamente y realiza un reinicio automático de la placa.

En este proyecto, el watchdog se activa al finalizar el setup(), cuando toda la inicialización del hardware ya ha concluido. Posteriormente, en cada iteración del loop() se ejecuta wdt_reset(), reiniciando el temporizador antes de que expire. De este modo, mientras el programa continúe funcionando con normalidad, el watchdog nunca llega a provocar un reinicio.

Este mecanismo aumenta la robustez del sistema empotrado, ya que permite recuperar automáticamente el funcionamiento de la máquina ante bloqueos inesperados sin necesidad de intervención del usuario.

### Uso de `millis()`

Todos los temporizadores del programa se han implementado utilizando la función `millis()`, evitando el uso de `delay()`. De este modo, mientras el sistema espera a que transcurra un determinado tiempo (por ejemplo, durante la preparación de una bebida o al mostrar un mensaje en pantalla), puede seguir atendiendo el resto de tareas del programa, como la lectura de sensores, la ejecución de los threads o la detección de pulsaciones.
