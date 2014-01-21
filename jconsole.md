#### Conexión mediante jconsole


1. Lo primero será modificar el fichero **/etc/msm.conf** para añadir las líneas necesarias a la línea de ejecución de java para poder conectarnos al proceso remotamente con el monitor. Localizamos el parámetro de configuración **DEFAULT_INVOCATION**, dejándolo:

		[guest]# vim /etc/msm.conf

		........
		DEFAULT_INVOCATION=java -Xms{RAM}M -Xmx{RAM}M -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalPacing -XX:+AggressiveOpts -Djava.rmi.server.hostname=192.168.100.1 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9003 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -jar {JAR} nogui
		.......
	* Donde 192.168.100.1 es la dirección ip de la máquina virtual, por la cual nos conectaremos a ella y **9003** un puerto a nuestra elección (Sobra decir que el puerto debe estar libre).
	* Esto debe hacerse con el servidor de juego detenido, de lo contrario msm no detectará ningún servidor iniciado al utilizar esta línea de ejecución para detectar servidores activos (`[guest]# msm MCServer1 stop now`)

2. Con esta modificación, la próxima vez que iniciemos el servidor podremos conectarnos remótamente a éste con la aplicación **jconsole**.

		[guest]# msm MCServer1 start

3. Desde una máquina remota con acceso a nuestra máquina virtual ejecutaremos:

		$ jconsole 192.168.100.1:9003

	* En caso de no poder acceder por culpa de un firewall o similar, siempre podemos realizar un túnel ssh: `$ ssh -L 9003:localhost:9003 minecraft@192.168.100.1` y conectarnos mediante `$ jconsole localhost:9003`

[**-- Volver a documentación --**](https://github.com/FdezI/Practica3/blob/master/documentacion.md#pruebas-de-rendimiento-con-jconsole-java)
