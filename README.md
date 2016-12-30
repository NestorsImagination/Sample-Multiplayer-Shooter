# Introducción

Este proyecto consistirá en la realización de un videojuego multijugador del género shooter básico, usando Unity para desarrollar la parte del cliente y desarrollando una arquitectura de servidor eficaz, la cual sirva para manejar las funciones necesarias del servidor central (iniciar la sesión de los usuarios, consultar y modificar datos de los jugadores, matchmaking, etc.) y para sincronizar los jugadores en varias partidas simultáneas.

# Arquitectura

Se usará (a no ser que se compruebe que otra arquitectura es más adecuada) una arquitectura basada en microservicios. Una parte muy importante de éste sistema es el matchmaking: que al unirse a una partida aparezca una pantalla de espera hasta que un número determinado de jugadores esté también esperando a encontrar partida (4, por ejemplo). En ese momento, el servicio de matchmaking habrá de conectar a estos clientes con uno de los servidores disponibles de juego y, una vez conectados, podrán jugar online, comunicándose entre ellos por medio del servidor de juego devuelto, que además correrá una simulación del juego, de forma que haga los cálculos necesarios y sincronice todos los jugadores (y evite que hagan trampas). La arquitectura quedaría más o menos así (sujeta a cambios de mejora):

![Imagen arquitectura](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Prototipo/Arquitectura_2.png)

En este sistema, el servicio de login (Login Service) se encargaría de acceder a los datos de los usuarios para comprobar que los datos proporcionados por el cliente son correctos, de forma que se abra una sesión del jugador en el Master Server. Éste se encargaría de mantener las sesiones de los jugadores y de redirigir los mensajes de los jugadores y los distintos servicios de forma correcta. El servicio Player Manager se encargará de acceder y manejar la base de datos relativa a los jugadores. El servicio Matchmaker tendrá dos funciones: registrará los servidores disponibles de partidas (World Master), a los que se conectarán los jugadores para jugar entre sí, y aceptará peticiones de los jugadores para entrar en partida. Para mantener un registro de los servidores de partida disponibles se usará un patrón como el definido en esta página: http://microservices.io/patterns/client-side-discovery.html.

Una vez el Matchmaker aloja un número determinado de peticiones de entrar en partida (4, por ejemplo, para una partida 2 vs 2), les habrá de comunicar por medio del Master Server que la partida ha empezado, de forma que éstos cambien a la escena de partida y el World Master seleccionado se disponga a registrar a los jugadores. Los usuarios se conectarán al World Master por medio del Master Server y podrán jugar la partida. Cada World Master se encargará de manejar las funciones de sincronización entre clientes que no requieran de una simulación del "mundo" y de hacer de intermediario entre los clientes y el World Simulator. El World Simulator será desarrollado también con Unity, y se encargará de actualizar el mundo según los inputs enviados por los jugadores, y de pedir al World Master que envíe a los jugadores los datos sobre el estado del mundo periódicamente, de forma que todos los jugadores estén sincronizados.

# Componentes

A continuación aparece una descripción detallada de cada uno de los componentes del sistema:

## Player

Cliente del juego, desarrollado con Unity (puede estar construído como un .exe, un juego de navegador, etc.). Será lo que el jugador ejecute, manejando las entradas y salidas. Se comunicará con el Master Server usando Socket.IO.

## Master Server

Servidor programado en NodeJS (u otro si se encuentra un sistema mejor) al que se conectarán los clientes, y que hará de intermediario entre estos y los demás servicios, redirigiendo los datos de forma correcta. Una vez han iniciado sesión los usuarios, el Master Server mantendrá su sesión abierta hasta que éstos cierren sesión o permanezcan inactivos durante un tiempo prudencial. Además se encargará de manejar el chat del juego.

## Login Service

Se encarga de validar los datos enviados por un cliente para iniciar sesión. Programado en cualquier lenguaje sencillo y eficaz, se comunicará con la base de datos UsersDB (SQL, ya que los datos almacenados serán simplemente una tabla con una entrada por usuario, conteniendo correo electrónico y contraseña, y posiblemente más campos si surge una nueva necesidad), para validar los datos mencionados.

## Player Manager

Hace de intermediario entre el Master Server y la base de datos PlayersDB, accediendo y modificando los datos de ésta según las ordenes del Master Server. Esta base de datos (SQL por ahora por la regularidad de los datos a almacenar en la version a desarrollar, pero modificable a una en NoSQL si se añadieran datos más abundantes e irregulares) guardará la información relativa a los jugadores, incluyendo nombre del jugador en el juego y otros posibles datos como dinero en el juego, items adquiridos, etc..

## Matchmaker

Este servicio, escrito en NodeJS (modificable si se encuentra mejor opción) se encarga de registrar los World Master disponibles (controlando cuántos están disponibles en cada momento) y de aceptar peticiones de jugadores de inicio de partida, de forma que, cuando un número de jugadores concreto haya enviado una petición (4 por ejemplo) y haya por lo menos un World Master disponible, se les comunique a estos jugadores que han entrado en una partida, cambiando su escena a la del juego y conectándolos, por medio del Master Server, con el World Master seleccionado (se devuelve al Master server la dirección de este World Master para que se comunique con él mientras la partida esté en curso), comenzando así una partida. Ese World Master dejará de estar disponible mientras la partida se encuentre en curso.

## World Master

Servidor escrito en NodeJS que tiene un World Simulator asociado. Al iniciarse, comunicará al servicio Matchmaker que se ha iniciado, de forma que quedará disponible para los jugadores a partir de ese momento (quedando registrado). Cuando el Master Server le comunica que se ha iniciado una partida, comunicará al World Simulator que inicie una nueva partida, asociando cada cliente con un "objeto jugador" de la escena diferente. Durante la partida, el World Master realizará la sincronización entre los jugadores que no requiera de una simulación del mundo y hará de intermediario entre el Master Server (y los clientes) y el World Simulator. Una vez acabada la partida, comunicará al Master Server que la partida ha acabado, de forma que quede disponible para iniciar una nueva partida en el Matchmaker.

## World Simulator

Desarrollado con Unity, es un ejecutable que corre un mundo similar al de los clientes (ejecutado de modo que no se vean los gráficos para mejorar el rendimiento, ya que es innecesario) que recibirá los inputs de los clientes a través del World Master (usando Socket.IO) para simular los personajes jugadores y las entidades del mundo de forma sincronizada. También envía periódicamente a los jugadores el estado de las entidades (posición, vida de los jugadores, proyectiles, etc.), de forma que los clientes queden lo más sincronizados posible, y evita que un jugador haga trampas o quede desincronizado por problemas de latencia: es la posición de los jugadores en el World Simulator la que es tomada como real, y si en un cliente se comprueba que la posición del personaje jugador es diferente a la del World Simulator, el personaje jugador se moverá para tomar la posición dada por el World Simulator. Empezará una nueva partida cada vez que el World Master se lo pida, asociando cada cliente con un personaje jugador diferente.

# Provisionamiento con Chef o Ansible

A continuación describo los procedimientos a seguir para aprovisionar un sistema con los elementos básicos necesarios para desplegar el proyecto, usando Chef o Ansible.

## Chef

Tras instalar Ruby y Chef:

![Chef instalado](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/ChefInst.png)

Pegar la carpeta "chef" de este repositorio en la carpeta principal (/home/"ususario"/). Se deben modificar las rutas "/home/ubuntu/" de los archivos por la carpeta principal del sistema que se esta provisionando. Ejecutar chef-solo -c solo.rb y el sistema queda provisionado:

![Chef ejecutado](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/ChefExe.png)

## Ansible

Crear las carpetas "Provision" y "project" en la carpeta principal. Instalar Ansible.

Para configurar Ansible, en la carpeta "/etc/ansible" hay que modificar "ansible.cfg" para que "sudo\_user" sea el usuario root del sistema (por defecto "root", pero en estas máquinas se llama "ubuntu"). Crear una carpeta "group\_vars" (si no existe) y pegar en ella el archivo "app.yml" que se encuentra en este repositorio.

De vuelta a la carpeta Provision, pegar el archivo Playbook.yml de este repositorio. Por último, usar el comando "ansible-playbook Playbook.yml" para ejecutar Ansible. Esto da un output como este:

![Ejecución ansible-playbook Playbook.yml](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/Ansible.png)

# Despliegue con Vagrant

## Despliegue para máquinas virtuales

Para provisionar máquinas virtuales con Vagrant, una vez instalado Vagrant (junto con una máquina virtual como VirtualBox) descargar los archivos de la carpeta /Provision/Vagrant/VirtualBox y colocarlos en una carpeta cualquiera. Ejercutar "vagrant -n(número de Game Worlds) up" para realizar el provisionamiento. Por ejemplo, si se quiere ejecutar creando 5 Game Worlds (hasta 5 partidas simultáneas) se haría "vagrant -n5 up". Se ejecutará Vagrant:

![Ejecución Vagrant](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/VagrantShooterUp.png)

Una vez completado, se habrán creado y configurado con Ansible todas las máquinas virtuales necesarias:

![Máquinas virtuales creadas](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/VagrantShooterUp2.png)

![Máquinas en Vagrant](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/VagrantShooterUp3.png)

Se puede entrar a cualquiera de las máquinas con "vagrant ssh (nombre de la máquina)". Por ejemplo, al hacer "vagrant ssh master" se puede entrar al servidor maestro. Como se puede comprobar en la siguiente imagen, se ha configurado la máquina correctamente:

![Master Server](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/VagrantShooterUp4.png)

## Despliegue para AWS

Una vez instalado el plugin vagrant-aws, descargar los archivos de la carpeta /Provision/Vagrant/AWS y colocarlos en una carpeta cualquiera. Se debe configurar la cuenta de AWS, creando un grupo de seguridad llamado "Vagrant" con permisos suficientes, crear un Key Pair, descargando los archivos correspondientes, y modificar el archivo Vagrantfile dando valor a las siguientes variables:

* accessKeyID (Añadir Access Key ID)
* secretKey (Añadir Secret Key)
* keyPairName (Añadir nombre del Key Pair)
* privateKey (Añadir ruta desde este directorio a la Private Key)

Ejecutar con vagrant -n(número de Game Worlds) up --provider=aws. Es posible que aparezca un error de Ansible que contenga "...too long for Unix domain socket...". En tal caso, seguir las instrucciones de este enlace: https://goinggnu.wordpress.com/2015/07/07/solution-for-too-long-for-unix-domain-socket-with-ansible-and-amazon-ec2/. También es posible que surjan errores de Ansible diciendo que hay problemas relacionados con la ejecución paralela (que no son fatales). En tal caso, ejecutar "vagrant provision" y puede que se arregle. Si todo va bien, se habrán creados las máquinas en AWS:

![Máquinas creadas en AWS](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/AWS_UP.png)

Conectándose a una de las máquinas con "Vagrant ssh (nombre de la máquinas)", por ejemplo "Vagrant ssh master" para conectar con el Master Server, se puede comprobar que se ha creado y configurado correctamente:

![Master Server AWS](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/Vagrant_AWS_Master.png)

# Creación de un entorno de pruebas con Docker

Se supone que Docker está instalado correctamente en la máquina a usar. Para que no haya problemas se recomienda usar todos los comandos especificados con el usuario "root" (sudo su).

## Creación de una máquina con docker-machine

Se puede comenzar creando una máquina Docker con docker-machine (se debe instalar adicionalmente) para conectar con el servidor en el que se desplegará el entorno, aunque es opcional. Por ejemplo, en el caso de AWS, se debe crear un archivo /root/.aws/credentials con el siguiente contenido:

```
aws\_access\_key\_id = **************
aws\_secret\_access\_key = *************
```

Luego usar el comando

```
docker-machine create --driver amazonec2 --amazonec2-region "región" "nombre de la máquina"
```

Por ejemplo:

![Docker Machine AWS](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/DockerMachineAWS.png)

Se puede ver en la consola de AWS que se ha creado la instancia correspondiente:

![Docker Machine AWS Instance](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/DockerAWSInstance.png)

Luego se ha de usar el comando

```
eval $(docker-machine env "nombre de la máquina")
```

para activar la máquina creada. Por ejemplo:

![Docker Machine AWS](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/DockerMachineAWS_2.png)

## Creación del entorno de pruebas con Docker

Descargar el contenido de la carpeta "Docker" de este repositorio. Moverse en la terminal a la carpeta y usar el siguiente comando:

```
vagrant -n"número de mundos de juego" up --provider=docker
```

Por ejemplo:

```
vagrant -n2 up --provider=docker
```

Tras unos momentos, el entorno quedará creado. Con "docker ps -a" se pueden listar los contenedores creados:

![Entorno creado](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/Docker_Vagrant_Up.png)

Se puede conectar con la máquina con "docker-machine ssh "nombre de la máquina" para comprobar que todo funciona correctamente:

![Entorno creado](https://raw.githubusercontent.com/NestorsImagination/Sample-Multiplayer-Shooter/master/Provision/Screenshots/Docker_AWS_Prueba.png)
