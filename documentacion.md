-> Práctica 3 <-

El objetivo de la práctica es el de comparar distintos sitemas de virtualización y decidir la configuración más optima en términos de recursos para una máquina virtual. Para ello pondremos de ejemplo la puesta en marcha de un servidor [vanilla](http://en.wikipedia.org/wiki/Vanilla_software) del popular juego Minecraft para un máximo de 5 jugadores. En el ejemplo propuesto se comparará VMWare frente a KVM en cuestión de prestaciones y se proseguirá a realizar las pruebas oportunas para determinar una configuración óptima y ajustada para ahorrar recursos mientras se proporciona un servicio óptimo.

-----------

# Selección del sistema operativo
Para elegir el sistema operativo adecuado deberían realizarse las pruebas pertinentes, tal y como haremos con la elección de recursos pero, puesto que no es el objetivo principal de la práctica y no va a tener un factor realmente crítico en el servidor que propondremos a continuación se utilizará Ubuntu Server en su versión LTS actual (12.04.3). Ubuntu server ha demostrado en la mayoría de casos estar a la altura si no por encima y su facilidad de uso y compatibilidad han ayudado a su elección, así como la estabilidad de las versiones LTS.

-----------

# Instalación y preparación del sistema operativo
## 1. Preparación de la imagen
Antes de proceder a preparar la imagen e instalar el sistema operativo debemos descargarnos las dependencias necesarias:
> 
    # apt-get install qemu-kvm libvirt-bin
    
A continuación, toca preparar la imagen (las almacenaremos en /vms/storage):
> 
    # mkdir -p /vms/storage
    # qemu-img create -f raw /vms/storage/serverbase.img 3G
    
Nuestra imagen va a necesitar estar lo siguiente:
* Una partición de sistema y programas (2.5GB)
* Un área de intercambio (SWAP) (0.5GB)
* Espacio de datos para almacenar todo lo relacionado con el servicio ofrecido (relativo)

Podemos definir una partición de sistema con un tamaño fijo, pues la mayoría de servicios que pudiéramos necesitar cabrán perfectamente y el área de intercambio deberá estar presente, aunque con una cantidad mínima para emergencias, pues no nos interesa hacer uso de ella, sin embargo, dejaremos el espacio reservado para datos definir, con lo que la imagen nos pesará menos y será mucho más fácil de transportar.

A esta imagen la llamaremos imagen "base", la cual dispondrá de un sistema operativo instalado y un espacio necesario para instalar la mayoría de servicios que podríamos necesitar. 

## 2. Instalación del sistema operativo
Para inicializar la instalación del sistema operativo en la imagen creada:
> 
    # qemu-system-x86_64 -hda /vms/storage/serverbase.img -cdrom /vms/isos/ubuntu-12.04.3-server-amd64.iso -boot d
    
Instalaremos el sistema operativo utilizando toda la partición, asignando 1.5GB al sistema raíz y el sobrante al área de intercambio. Lo configuraremos con los siguientes datos:
* Nombre del sistema: serverbase

## 3. Configurando imagen para servicio final
La imagen con el servicio se renombrará a kvmGM, al igual que el nombre del sistema. Para no perder nuestra imagen base:
> 
    # cp serverbase.img kvmGM.img
A partir de ahora trabajaremos con esta imagen.
    
Nuestro servicio final va a necesitar espacio suficiente para almacenar el servidor de Minecraft así como algunas copias de seguridad que desee el usuario (Esto podría proveerse como otro servicio aparte). Teniendo en cuenta que este tipo de servidores pueden llegar a ocupar entre 200MB y varios GB, porporcionaremos 10GB de espacio adicional:
> 
    # qemu-img resize kvmGM.img +10G
    
e iniciamos la máquina para configurar los nuevos 10GB:
> 
    # qemu-system-x86_64 -hda /vms/storage/kvmGM.img
    
### Cambios iniciales
#### Cambio de nombres:
        [guest]# sed -i 's/serverbase/kvmGM/g' /etc/hosts /etc/hostname
#### Creación de la nueva partición de almacenamiento (10GB):
* Ampliación de la partición extendida:
        [guest]# parted
            (parted) p
            (parted) p free
            (parted) resize 2 2797MB 14.0GB
            (parted) q
Donde 2 es la partición “extended”, 2797MB el inicio de ésta y 12.9GB el fin del Espacio libre
* Creación de la nueva partición lógica:
        [guest]# fdisk /dev/vda
        (fdisk) n
        (fdisk) l
            <enter>
            <enter>
        (fdisk) w
Podemos comprobar que se ha creado correctamente mostrando la tabla de particiones (`(fdisk) p`)
* Ordenamos al sistema operativo a leer la nueva tabla de particiones:
        [guest]# partprobe /dev/vda
* Formateamos la nueva partición:
        [guest]# mkfs.ext4 /dev/vda6
* Añadimos el montaje de la partición al arranque:
        [guest]# echo -e "`blkid | grep /dev/vda6 | cut -f2 -d ' ' | sed 's/"//g'`\t/opt\text4\trw,defaults,noatime\t0\t2" >> /etc/fstab
El valor **noatime** evita que se escriba en los metadatos del fichero su última lectura (no es necesario para nuestro servicio) evitando escrituras innecesarias en momentos de lectura

Si no aparece nada al ejecutar el “**blkid**” asegurarnos de ejecutarlo con permisos de root.

Evitar utilizar la ruta del dispositivo (**/dev/vda6**) en lugar de su **UUID** en **/etc/fstab**, pues de lo contrario daría incompatibilidades de montado en sistemas de virtualización que lo nombraran de otra forma, como por ejemplo, vmware (**/dev/sda6**)

* Montamos todo "fstab":
        [guest]# mount -a
    Podemos comprobar que todo ha ido bien con: `$ df -h` o `$ mount`

---------

# Comparación de sistemas de virtualización

A continuación, y una vez instalado el sistema operativo, se hará una comparación entre VMWare y KVM, analizando los resultados obtenidos en tiempo de CPU, velocida de disco y velocidad de red.


## 1. Instalando herramientas necesarias
Para realizar el conjunto de pruebas haremos uso de la herramienta **sysbench** y **netperf**. Aunque existen otras herramientas como **phoronix-test-suite** estas dos son capaces de realizar las pruebas deseadas sin problemas.
> 
    # apt-get install sysbench netperf -y
    

## 2. Compatibilizando la imagen
Deberemos convertir la imagen de disco al formato compatible con vmware (vmdk) usaremos “qemu-img”, de esta forma usaremos el mismo sistema “guest” en las distintos sitemas de virtualización (ahorrándonos la instalación repetida del sistema operativo).
> 
    $ qemu-img convert -f raw -O vmdk /vms/storage/serverbase.img /vms/storage/serverbase.vmdk
    
Este comando nos realizará una copia convertida al nuevo formato, vmdk de la imagen.


## 3. Realizando benchmarks

Puesto que los resultados rondan siempre los mismos valores se hará un resumen mostrando una media aproximada donde sea necesario mostrar valores:

* **CPU**: Ambos dan los mismos resultados (seguramente debido a que tanto vmware como kvm utilizan la tecnología Intel VT-x de virtualización por hardware.
    
        # sysbench --test=cpu --cpu-max-prime=150000 run

* **I/O (Disco)**: Los resultados son los siguientes:
    **kvm**: 2MB/s **vmware**: 188MB/s
        [guest]# cd /opt/
        [guest]# sysbench --test=fileio --file-total-size=5G prepare
        [guest]# sysbench --test=fileio --file-total-size=5G --file-test-mode=rndrw --init-rng=on --max-time=300 --max-requests=0 run
        [guest]# sysbench --test=fileio --file-total-size=5G cleanup
    El cambio de directorio a **/opt** se debe a que en el **raíz** no hay 5GB disponibles.
    
* ** Red **: Ambos sistemas proporcionan los mismos resultados, tanto para conexiones locales como hacia internet.
        [guest]# netperf
    Velocidad con respecto a internet
        [guest]# netperf -H  <ip_anfitrión>
    Velocidad con el anfitrión
        [guest]# netperf -H  <otra_máquina_red>
    Velocidad con otros dispositivos de la red


Aunque en teoría, kvm debería ofrecer mejores resultados, sobre todo con el uso de los drivers **virtio** para red y almacenamiento, obtiene los mismos resultados e incluso, en el trato de entrada/salida de disco, muestra una enorme deficiencia, seguramente debida a algún bug que, por desgracia, no he conseguido encontrar:
* Virtio para discos está activado ($ grep -i virtio /boot/config-`uname -r`)
* Los módulos de kvm están cargados ($ kvm-ok).
* También se ha probado a usar otros drivers que no fueran Virtio para el almacenamiento pero con los mismos resultados. La imagen usada está en formato raw y al probar a usar la misma imagen que el vmware, en formato **vmdk, pasa la velocidad de 2MB/s a 700KB/s**.

Por lo tanto, se puede decir, que tras el resultado de las pruebas, **vmware es mucho mejor opción que kvm** en este caso. Aún así, a continuación se procederá a mostrar la configuración necesaria para el uso de la máquina bajo kvm. Los resultados obtenidos al final de la prácticas son igual de válidos para vmware (obviamente, cada uno se configura de forma distinta, pero el objetivo, al menos personal, de la práctica es probar algo nuevo)

----------

# Instación del servicio
El servicio del ejemplo, un servidor de Minecraft, corre bajo una máquina virtual de java por lo que deberemos instalar una y, aunque la versión libre **openjdk** proporciene todo lo necesario, su rendimiento es algo menor a la máquina ofrecida por oracle, la cual podremos instalar sin problema legal alguno para nuestro propósito, más en concreto, su versión 7.

## 1. Instalando java

### JRE de Oracle
> 
    # echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu precise main" >> /etc/apt/sources.list
    # echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu precise main" >> /etc/apt/sources.list
    # apt-get update && apt-get install oracle-java7-installer -y

### JRE libr
> 
    # apt-get install apt-get install openjdk-7-jre

## 2. Instalando servidor vanilla de Minecraft
1. Descargamos e instalamos el gestor de servidores minecraft “[**msm**](http://msmhq.com/)”:

        $ wget -q http://git.io/Sxpr9g -O /tmp/msm && bash /tmp/msm

	Lo instalamos con los valores por defecto (**/opt/msm** como directorio de instalación, **minecraft** como usuario).

	Aceptamos la instalación de todas sus dependencias.

2. Aunque este "gestor" proporciona herramientas mucho más avanzadas y organizadas para administrar diversos servidores, para este ejemplo haremos un uso muy básico del mismo:
    
    * “Creamos” un servidor:
	        # msm server create MCServer1
        Esto nos creará un directorio (**/opt/msm/servers/MCServer1**) para este servidor con sus ficheros de configuración básicos y un sistema de gestión de ficheros gestionable mediante “msm” y nos descargará la última versión de servidor estable.
	
    Aún así, puesto que queremos utilizar específicamente la versión 1.7.4:
    > 
            # msm jargroup changeurl minecraft https://s3.amazonaws.com/Minecraft.Download/versions/1.7.4/minecraft_server.1.7.4.jar
	    Y por términos de nomenclatura:
	        # msm jargroup rename minecraft vanilla174
	
	* Acutalizamos el servidor:
	        # msm MCServer1 jar vanilla174

3. Iniciamos el servidor, pero antes, añadimos la versión de servidor que vamos a usar al fichero server.properties para que así msm tenga una idea de lo que debe hacer.
	    # echo “msm-version=minecraft/1.7.4” >> /opt/msm/servers/MCServer1/server.properties
    	# msm MCServer1 start
    Esto creará todos los ficheros de configuración necesarios y actualizará alguno ya creado anteriormente por msm.

------------

# Analizando prestacion del servicio
Para hacer las pruebas, una vez iniciado el servidor, aunque sea una vez, necesitaremos conectarnos sin cuentas autenticadas oficialmente así que modificamos la opción online-mode de server.properties y la ponemos a false. Reiniciamos el servidor para aplicar los cambios:
	# msm MCServer1 restart now

Si queremos poder acceder a la consola del servidor en de juego tendremos que darle una contraseña al usuario minecraft: `# passwd minecraft`, y a continuación loguearnos con éste, el cual podrá utilizar la mayoría de herramientas msm.


## Pruebas iniciales
En un inicio le daremos al sistema un total de 1GB de memoria RAM y 1 CPU. Esta cantidad de memoria debería ser suficiente y de sobra para las pruebas a realizar, tras las cuales determinaremos el máximo de memoria RAM necesaria para que el sistema vaya perfectamente y así ajustar a los recursos necesarios. El servidor en cuestión corre bajo una máquina virtual de java así que también será necesario estipular la memoria reservada y la máxima de dicha máquina. Msm por defecto le proporciona 1GB de RAM, lo cual es suficiente.
Para monitorizar el servidor en tiempo real y así ver los recursos utilizados pueden usarse diversas herramientas, tales como top. En este caso utilizaremos htop:
    # apt-get install htop
    # htop

* **Reposo (servicio apagado):**
Podemos apreciar por medio del htop (o el `$ free -m`, teniendo en mente siempre la resta de caché) que la memoria utilizada por el sistema sin el servidor iniciado ronda siempre ~84MB de RAM. 
	* RAM (sistema): 84MB

* **Activo sin jugadores**:
Una vez iniciado el servidor el uso de memoria asciende a ~274MB, por lo cual podríamos determinar que el servidor, de base, requiere ~190MB de memoria adicionales del sistema:
	* RAM (sistema + servicio): 274MB

* **1 Jugador:**
Por la dinámica del juego, la memoria RAM máxima necesaria en un servidor vanilla de Minecraft suele ser la memoria base del juego más la usada por un usuario activo (en movimiento por el escenario del juego) multiplicada por el número de slots de usuarios máximo. Hay que tener en cuenta que los escenarios pueden ser sobrecargados con elemenos que usen algo de memoria adicional, por lo cual habrá que lidiar con ello y, o bien, poner un colchón suficientemente amplio para soportarlo, estipular una limitación en los términos de juego o bien una mezcla de ambos, proporcionando un pequeño colchón y recomendando evitar el exceso de sobrecarga.
Sabiendo esto, para realizar la prueba se conectará a un jugador activo y se mirará el incremento de memoria usada.
	* RAM (sistema + servicio + jugador):	343MB
    * RAM (por jugador): 69MB

Aunque en un inicio el jugador únicamente consume ~59MB de memoria, curiosamente ésta no deja de aumentar. Podemos pensar (ya veremos más adelante que no es la única razón) que es debido a la forma de administrar la memoria de la máquina virtual de java (sobre todo por el [colector de basura](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)), por lo cual, para medir correctamente la memoria ram utilizada necesitaremos hacer uso de alguna herramienta más avanzada y orientada a dicha finalidad, como es, por ejemplo, el **jconsole**, un monitor gráfico de máquinas virtuales java.


## Pruebas de rendimiento con jconsole (Java)
### Preparación del proceso para ser monitorizado
1. Lo primero será modificar el fichero **/etc/msm.conf** para añadir las líneas necesarias a la línea de ejecución de java para poder conectarnos remotamente con el monitor. Localizamos el parámetro de configuración **DEFAULT_INVOCATION**, dejándolo:
        [guest]# vim /etc/msm.conf
    
        ........
        DEFAULT_INVOCATION=java -Xms{RAM}M -Xmx{RAM}M -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalPacing -XX:+AggressiveOpts -Djava.rmi.server.hostname=192.168.100.1 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9003 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -jar {JAR} nogui
        .......

	Donde 192.168.100.1 es la dirección ip de la máquina virtual, por la cual nos conectaremos a ella y **9003** un puerto a nuestra elección (Sobra decir que el puerto debe estar libre).
    
    Esto debe hacerse con el servidor de juego detenido, de lo contrario msm no detectará ningún servidor iniciado al utilizar esta línea de ejecución para detectar servidores activos (`[guest]# msm MCServer1 stop now`)

2. Con esta modificación, la próxima vez que iniciemos el servidor podremos conectarnos remótamente a éste con la aplicación **jconsole**.
        [guest]# msm MCServer1 start

3. Desde una máquina remota con acceso a nuestra máquina virtual ejecutaremos:
        $ jconsole 192.168.100.1:9003
    En caso de no poder acceder por culpa de un firewall o similar, siempre podemos realizar un túnel ssh: `$ ssh -L 9003:localhost:9003 minecraft@192.168.100.1` y conectarnos mediante `$ jconsole localhost:9003`

### Pruebas
Una vez conectados proseguiremos con las pruebas, para las cuales tomaremos los valores siempre inmediatamente después de pasar el **GC (Colector de basura)**, para ello nos vamos a la pestaña de memoria y obligamos al recolector de basura de java a que haga su pasada clickando en el botón “Perform GC”

#### Monitoreo inicial
> 1. Podemos ver que sin jugadores conectados y tras el inicio, el servicio utiliza ~60MB de RAM necesariamente. Por lo tanto, puede deducirse que la máquina virtual en sí corriendo el servicio consume 130MB (274 total - 84 de sistema - 60 de servidor Minecraft)
2. Procedemos a probar conectando a un jugador:
    * Resultados (con 1GB de RAM asignados a la **mv** de java):
        * Vacío: 180MB
        * 1 jug: 215MB, 266MB, 272MB
        * Vacío: 236MB
        * 1 jug: 270MB, 292MB
        * Vacío: 251
    * Resultados (con 256MB de RAM):
        * La memoria nunca sobrepasa la cantidad de 256MB pero se mantiene en el límite.

De estos resultados podemos dictaminar que el servidor de Minecraft, en caso de necesidad libera más memoria RAM que cuando aún tiene de sobra, en cuyo caso mantiene más ** *“cosas”* ** cargadas y por lo tanto usa más memoria de forma ** *“innecesaria”* **. Por ello, las pruebas han de hacerse de forma totalmente empírica, conectando 5 jugadores reales.

#### Monitoreo empírico (a la búsqueda del crash):
* "Un servidor vanilla de Minecraft, en teoría, debería consumir:
        Cantidad base (sin jugadores) + 2.5MB (tamaño bruto por chunk) * 32 (chunks en ram por jugador) * jugador, lo cual significa 80MB/pj + cantidad base"
Con los datos obtenidos estando el servicio recién iniciado podemos observar que no se distancian mucho de la teoría, por lo tanto, para las siguientes pruebas, en lugar de asignar memoria RAM al "tun, tun", vamos a basarnos en distintas suposiciones posibles:
    1. **La cantidad por jugador de las pruebas iniciales sin esperar el incremento de ram es correcta** `180 + (69*5) = 525 MB`: Tras realizar la prueba con los 619MB y 5 jugadores y tras un buen rato "jugando" comprobamos que no hay problema alguno.
    2. **La cantidad por jugador de las pruebas con el jconsole es correcta** `180 + (35*5) = 355 MB`: Nuevamente, no hay problema alguno.
    3. **No tenemos más suposiciones posibles, reducimos la cantidad** `180 + (25 * 5) = 305MB`: Tras un ratito jugando, el sistema crashea y deja de responder.



#### Conclusión:
Tras estos resultados afirmaremos que la memoria RAM necesaria es de 355MB para la **mv** de java, por lo tanto, la RAM necesaria para el sistema al completo es de `274+(35×5)=` ~450MB.

---------

# Conclusión
Dependiendo del número de jugadores también dependerán las necesidades de CPU, las cuales podrían requerir aumentar los núcleos asignados, la potencia de dichos núcleos y/o la disponibilidad de los mismos. Sin embargo, para este ejemplo, un sólo núcleo se suficiente.

En cuestiones de almacenamiento, éste suele depender de los plugins (que se dejará a responsabilidad del usuario) utilizados en el servidor, pero puesto que no es un recurso crítico en el anfitrión se le ha asignado una cantidad para el sistema suficiente para instalar requisitos básicos y una cantidad dedicada al hosting en sí suficiente para almacenar cierto números de copias de seguridad y permitir una expansión del escenario del juego más que suficiente para el correcto entretenimiento del mismo (10GB).

Aunque las pruebas han sido realizadas sobre kvm, los resultados finales han sido probados también sobre vmware, dando unos resultados muy similares por lo que el dato decisivo para la elección del hipervisor ha sido la deficiencia encontrada en la velocidad de entrada/salida de discos de kvm en el sistema utilizado.

En resumen:

Datos del anfitrión:
* **CPU**: Intel(R) Core(TM)2 Duo CPU     E6850  @ 3.00GHz
* **RAM**: 8GB
* **HDD**: 500GB total
* **S.O**: Ubuntu 12.10

Necesidad de máquina virtual:
* **Cores**: 1
* **RAM**: 450MB
* **HDD**: 2.5GB sistema + 0.5 SWAP + 10GB servicio
* **S.O**: Ubuntu 12.04.3 LTS

**Hipervisor óptimo**: VMWare




### Aspectos adicionales
#### Copias de seguridad
msm proporciona un sistema de copias de seguridad autogestionado, de todas formas, puede modificarse en el fichero /etc/msm.conf, /etc/cron.d/msm o en cualquier otro fichero del demonio cron. Además, lo ideal sería proporcionar un servicio de copias de seguridad adicional.

#### Algunos aspectos de seguridad
Si se quiere optimizar la seguridad de acceso y limitar al usuario que gestione el servidor (lo óptimo sería el usuario minecraft) a /opt/msm se pueden utilizar diversas técnicas, desde enjaularlo en dicho directorio hasta darle permisos de usuario o grupo sobre dicho directorio (por defecto), proporcionándole una shell en bash y un directorio home.

#### Otras consideraciones
* Para reducir el tamaño de la imagen base, en caso de que el sistema no haga sparse, es posible convertir la imagen a qcow2 o vmdk:
        # qemu-img convert -f raw -O qcow2 serverbase.img serverbase.qcow2

* Lo ideal para la máquina virtual (y máquinas virtuales en general), es tenerlas bajo un sistema operativo sin entorno de escritorio ninguno y gestionarlas todas remotamente desde algún PC con acceso a todas las máquinas virtuales administradas. Para máquinas kvm, tanto **virsh** como **virt-manager** proporcionan dicha opción y vmware seguramente también lo haga. Además, puede configurarse la salida "serie" del sistema operativo por lo que se puede visualizar desde el momento de carga del grub.
* VMWare pertenece a una empresa privada, mientras que KVM se encuentra bajo licencia GPL y, en teoría, debería proporcionar un mejor rendimiento que vmware por lo que aún con los fallos obtenidos sería conveniente encontrar una solución al problema con el que se ha lidiado en esta práctica. 
