# iaw-practica-01

## Instalación de la pila LAMP

### Pasos previos a la instalación:

Antes de instalar los paquetes necesarios para la pila LAMP vamos a ejecutar una serie de comandos apt para actualizar los repositorios del servidor. Vamos a tener en cuenta que estos comandos son los del script que se ejecuta con derechos de administrador, por lo que no usaremos sudo en ninguno:

- Actualizamos los repositorios:

	`apt update -y`

- Actualizamos los paquetes instalados (si fuese necesario):

	`apt upgrade -y`

### Instalación de la pila LAMP

- Instalamos apache:

	`apt install apache2 -y`

- Instalamos MySQL Server:
	- Instalamos el paquete 
		
		`apt install mysql-server -y`

	- Cambiamos la contraseña del usuario root por seguridad, ya que de base viene sin contraseña. Primero definimos una variable para poder modificarla cuando quiera:

		`DB_ROOT_PASSWD=root`

	- Luego, viendo que tenemos acceso a MySQL, la cambiamos mediante comandos de MySQL:

		`mysql -u root <<< "ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '$DB_ROOT_PASSWD';"`

		`mysql -u root <<< "FLUSH PRIVILEGES;"`

- Instalamos PHP, el cual requiere varios módulos:

	`apt install php libapache2-mod-php php-mysql -y`

### Instalación de herramientas de gestión para el servidor:

- Descargamos Adminer. Es una herramienta alternativa a phpMyAdmin que no necesita instalación:
	- Creamos el directorio en el que vamos a guardar el archivo:

		`mkdir /var/www/html/adminer`

	- En ese directorio descargamos el archivo php de Adminer:

		`cd /var/www/html/adminer`

		`wget https://github.com/vrana/adminer/releases/download/v4.7.7/adminer-4.7.7-mysql.php`

	- Cambiamos el nombre del archivo por index.php para que cuando accedamos vía navegador a la carpeta aparezca Adminer por defecto:

		`mv adminer-4.7.7-mysql.php index.php`

- Instalamos phpMyAdmin, herramienta vía web para la gestión de MySQL:
	- Primero generamos una clave aleatoria en una variable:

		``AUTOGENERATED_PASSWD=`tr -dc A-Za-z0-9 < /dev/urandom | head -c 64``

	- Configuramos la instalación con la herramienta debconf para que se haga de manera automática. Esta herramienta automatiza las ventanas de instalación de los paquetes:

		`echo "phpmyadmin phpmyadmin/reconfigure-webserver multiselect apache2"  | debconf-set-selections`

		`echo "phpmyadmin phpmyadmin/dbconfig-install boolean true"  | debconf-set-selections`

		`echo "phpmyadmin phpmyadmin/mysql/app-pass password $AUTOGENERATED_PASSWD"  |debconf-set-selections`

		`echo "phpmyadmin phpmyadmin/app-password-confirm password $AUTOGENERATED_PASSWD"  | debconf-set-selections`

	- Instalamos el paquete:

		`apt install phpmyadmin php-mbstring php-zip php-gd php-json php-curl -y`

- Instalamos GoAccess, herramienta para la administración de los logs vía web:
	- Descargamos las claves y el repositorio:

		`echo "deb http://deb.goaccess.io/ $(lsb_release -cs) main"  | tee -a /etc/apt/sources.list.d/goaccess.list`

		`wget -O - https://deb.goaccess.io/gnugpg.key | sudo apt-key add -`

		`apt-get update -y`

		`apt-get install goaccess -y`

	- Creamos el directorio para consultar las estadísticas:

		`mkdir -p /var/www/html/stats`

		`nohup goaccess /var/log/apache2/access.log -o /var/www/html/stats/index.html --log-format=COMBINED --real-time-html &`

	- Creamos el archivo de contraseñas para poder proteger el acceso a esta herramienta:

		`htpasswd -bc $HTTPASSWD_DIR/.htpasswd $HTTPASSWD_USER  $HTTPASSWD_PASSWD`

		Las variables del último comando están añadidas al principio del script:
		```
		HTTPASSWD_USER=usuario

		HTTPASSWD_PASSWD=usuario

		HTTPASSWD_DIR=/home/ubuntu
		```

	- Para finalizar tenemos que modificar el archivo de configuración de apache. Para automatizar el proceso creamos el archivo a parte para copiarlo en la ruta del archivo de configuración. El contenido del archivo es este:
		```
		<VirtualHost *:80>
				#ServerName www.example.com
				ServerAdmin webmaster@localhost
				DocumentRoot /var/www/html

				<Directory "/var/www/html/stats">
				AuthType Basic
				AuthName "Acceso restringido"
				AuthBasicProvider file
				AuthUserFile "/home/usuario/.htpasswd"
				Require valid-user
				</Directory>

				ErrorLog ${APACHE_LOG_DIR}/error.log
				CustomLog ${APACHE_LOG_DIR}/access.log combined
		</VirtualHost>
		```
	- Lo guardamos como 000-default.conf, lo sustituimos por el archivo de apache y reiniciamos el servicio:

		`cp /home/ubuntu/000-default.conf /etc/apache2/sites-available/`

		`systemctl restart apache2`

### Instalación de la aplicación web propuesta

- Descargamos los archivos del repositorio:

	`git clone https://github.com/josejuansanchez/iaw-practica-lamp.git $DIR_GIT`

	La variable $DIR_GIT será el directorio donde queramos guardar el repositorio antes de mover los archivos.

- Borramos el index.html que hay en la ruta /var/www/html para no tener conflicto con el de la aplicación web:

	`rm /var/www/html/index.html`

- Movemos el contenido de la carpeta src de la aplicación web al directorio /var/www/html

	`mv $DIR_GIT/src/* /var/www/html`

- Insertamos la base de datos en nuestro servidor:

	`mysql --user=root --password=$DB_ROOT_PASSWD < $DIR_GIT/db/database.sql`