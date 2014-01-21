## Instación del servicio
El servicio del ejemplo, un servidor de Minecraft, corre bajo una máquina virtual de java por lo que deberemos instalar una y, aunque la versión libre **openjdk** proporciene todo lo necesario, su rendimiento es algo menor a la máquina ofrecida por oracle, la cual podremos instalar sin problema legal alguno para nuestro propósito, más en concreto, su versión 7.

### 1. Instalando java

#### JRE de Oracle

    # echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu precise main" >> /etc/apt/sources.list
    # echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu precise main" >> /etc/apt/sources.list
    # apt-get update && apt-get install oracle-java7-installer -y

#### JRE libre

    # apt-get install apt-get install openjdk-7-jre

### 2. Instalando servidor vanilla de Minecraft

* **Descargamos e instalamos** el gestor de servidores minecraft “[**msm**](http://msmhq.com/)”:

        $ wget -q http://git.io/Sxpr9g -O /tmp/msm && bash /tmp/msm
	* Lo instalamos con los valores por defecto (**/opt/msm** como directorio de instalación, **minecraft** como usuario).
	* Aceptamos la instalación de todas sus dependencias.

* Aunque este "gestor" proporciona herramientas mucho más avanzadas y organizadas para administrar diversos servidores, para este ejemplo haremos un **uso muy básico** del mismo:

	1. *“Creamos” un servidor*:

			# msm server create MCServer1
		* Esto nos creará un directorio (**/opt/msm/servers/MCServer1**) para este servidor con sus ficheros de configuración básicos y un sistema de gestión de ficheros gestionable mediante “msm” y nos descargará la última versión de servidor estable.

		Aún así, puesto que queremos utilizar específicamente la versión 1.7.4:

			# msm jargroup changeurl minecraft https://s3.amazonaws.com/Minecraft.Download/versions/1.7.4/minecraft_server.1.7.4.jar

		Y por términos de nomenclatura:

	    	# msm jargroup rename minecraft vanilla174

	2. *Actualizamos el servidor*:

			# msm MCServer1 jar vanilla174

* **Iniciamos el servidor**, pero antes, añadimos la versión de servidor que vamos a usar al fichero server.properties para que así msm tenga una idea de lo que debe hacer.

		# echo “msm-version=minecraft/1.7.4” >> /opt/msm/servers/MCServer1/server.properties
		# msm MCServer1 start
			
	* De esta forma se crearán todos los ficheros de configuración necesarios y actualizará alguno ya creado anteriormente por msm.

[-- Volver a documentación --](https://github.com/FdezI/Practica3/edit/master/documentacion.md#instalaci%C3%B3n-del-servicio)
