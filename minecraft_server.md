### Instalando servidor vanilla de Minecraft
1. Descargamos e instalamos el gestor de servidores minecraft “[**msm**](http://msmhq.com/)”:

        $ wget -q http://git.io/Sxpr9g -O /tmp/msm && bash /tmp/msm
	* Lo instalamos con los valores por defecto (**/opt/msm** como directorio de instalación, **minecraft** como usuario).
	* Aceptamos la instalación de todas sus dependencias.

2. Aunque este "gestor" proporciona herramientas mucho más avanzadas y organizadas para administrar diversos servidores, para este ejemplo haremos un uso muy básico del mismo:

* “Creamos” un servidor:
	
		# msm server create MCServer1
	* Esto nos creará un directorio (**/opt/msm/servers/MCServer1**) para este servidor con sus ficheros de configuración básicos y un sistema de gestión de ficheros gestionable mediante “msm” y nos descargará la última versión de servidor estable.

	Aún así, puesto que queremos utilizar específicamente la versión 1.7.4:

		# msm jargroup changeurl minecraft https://s3.amazonaws.com/Minecraft.Download/versions/1.7.4/minecraft_server.1.7.4.jar

	Y por términos de nomenclatura:
	
		# msm jargroup rename minecraft vanilla174


* Actualizamos el servidor:

		# msm MCServer1 jar vanilla174



3. Iniciamos el servidor, pero antes, añadimos la versión de servidor que vamos a usar al fichero server.properties para que así msm tenga una idea de lo que debe hacer.

		# echo “msm-version=minecraft/1.7.4” >> /opt/msm/servers/MCServer1/server.properties
		# msm MCServer1 start
	* Esto creará todos los ficheros de configuración necesarios y actualizará alguno ya creado anteriormente por msm.
