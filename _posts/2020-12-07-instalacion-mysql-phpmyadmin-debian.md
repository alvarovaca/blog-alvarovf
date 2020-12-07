---
layout: post
title:  "Instalación de MySQL y phpMyAdmin en Debian 10"
banner: "/assets/images/banners/mysql.jpg"
date:   2020-12-07 14:03:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una instalación básica de **MariaDB (MySQL)**, un gestor de bases de datos relacionales (_SQL_) sobre una máquina **Debian Buster**, así como la configuración necesaria para admitir peticiones desde máquinas remotas. Además, instalaremos una herramienta de administración web de nombre **phpMyAdmin**, la cuál configuraremos para proporcionar la seguridad que su uso requiere, así como aprovechar todas las funcionalidades que nos ofrece.

Para esta ocasión, he traído los deberes hechos y he instalado previamente una máquina virtual con **Debian Buster**, pero no he llevado a cabo ninguna configuración, para así partir desde un punto totalmente limpio. No considero necesario explicar el proceso de instalación de dicha distribución, ya que es bastante sencillo, además de salirse del objetivo del artículo.

El paquete del gestor de bases de datos que instalaremos será el proporcionado por los repositorios oficiales de Debian, una versión mantenida por usuarios de la distribución, de manera que no es necesario importar ningún repositorio externo.

Dado que la instalación de MariaDB no requiere ninguna configuración previa, podremos dar paso a la misma, no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando (con privilegios de administrador, haciendo uso de `su -`):

{% highlight shell %}
root@mysql:~# apt update && apt upgrade && apt install mariadb-server
{% endhighlight %}

El paquete **mariadb-server** ha sido instalado en la máquina, pero antes de continuar, me gustaría mencionar que por defecto, se incluye un _script_ cuya finalidad es ser ejecutado en aquellas máquinas que actúen como servidoras de bases de datos en entornos de producción, llevando a cabo una configuración inicial pensada para ello:

* Estableceremos una contraseña para el administrador de MariaDB.
* Eliminaremos el usuario anónimo que viene creado por defecto.
* Desactivaremos la conexión remota a _root_, es decir, únicamente se podrá llevar a cabo desde _localhost_.
* Eliminaremos la base de datos de pruebas que viene creada por defecto.
* Volveremos a cargar las tablas de privilegios, para que los cambios surtan efecto.

En esta ocasión, el servidor no lo vamos a poner en producción, pero aún así voy a proceder a ejecutar dicho _script_ para mostrar los pasos que se llevan a cabo durante el mismo, aumentando así la seguridad sobre el servidor.

Lo primero que preguntará el _script_ será la contraseña actual del usuario **root** de MariaDB. Lo dejaremos vacío y pulsaremos **ENTER**, ya que por ahora no tiene ninguna contraseña configurada. Tras ello, tendremos que ir introduciendo "**Y**" a las opciones que nos vayan saliendo, para así llevar a cabo toda la configuración anteriormente mencionada. Para ejecutar dicho _script_, haremos uso del comando:

{% highlight shell %}
root@mysql:~# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Set root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!

Remove anonymous users? [Y/n] Y
 ... Success!

Disallow root login remotely? [Y/n] Y
 ... Success!

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
{% endhighlight %}

Como curiosidad, me gustaría mencionar que la contraseña que hemos especificado para el usuario _root_ de MariaDB no sirve "para nada", lo pongo entre comillas ya que por defecto, la autentificación para dicho usuario se lleva a cabo mediante un _Socket UNIX_, es decir, mientras que te encuentres con sesión iniciada en _root_ o bien antepongas `sudo` al comando, podrás hacer uso de MariaDB sin conocer la contraseña que acabas de indicar.

Esto, puede ser bueno o malo, según se vea. En caso de querer autentificarte mediante **Socket UNIX** (opción por defecto), tendrás que asegurarte de proteger el acceso al usuario _root_ del sistema, mientras que si deseas autentificarte con **credenciales**, tendrás que asegurarte de proteger el acceso al usuario _root_ de MariaDB. En caso de considerar más oportuna la última opción, podrás encontrar [aquí](https://www.itzgeek.com/how-tos/linux/debian/how-to-install-mariadb-on-debian-10.html) un artículo sobre cómo cambiar el método de autentificación.

Como consecuencia del proceso que está actualmente ejecutándose, se habrá abierto un _socket TCP/IP_ en el puerto por defecto de MariaDB (**3306**) que estará escuchando peticiones provenientes de **_localhost_**, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@mysql:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.

Efectivamente, el proceso está escuchando peticiones tal y como debería, de manera que procederemos a abrir una _shell_ de MySQL para así poder gestionar el motor, ejecutando para ello el comando:

{% highlight shell %}
root@mysql:~# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 57
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Donde:

* **-u**: Indica el usuario con el que nos vamos a conectar, en este caso, **root**.
* **-p**: Indicamos la contraseña, en este caso, no introducimos ninguna.

Como se puede apreciar, la _MySQL shell_ se ha abierto correctamente y está lista para su uso.

Para hacer un artículo un poco más completo, vamos a proceder a simular una situación real, en la que se tuviese que definir una nueva base de datos con un usuario con permisos para administrarla. En este caso, voy a crear una base de datos de nombre **bd_viajes** en la que va a existir un usuario **alvaro** que será el administrador de la misma, pero totalmente aislado del resto de bases de datos que se pudiesen crear en un futuro.

Lo primero que haremos estando dentro del servidor de bases de datos, será crear una nueva base de datos, valga la redundancia. En este caso, de nombre **bd_viajes**. Para ello, haremos uso de la instrucción:

{% highlight sql %}
MariaDB [(none)]> CREATE DATABASE bd_viajes;
Query OK, 1 row affected (0.000 sec)
{% endhighlight %}

La base de datos en cuestión habrá sido generada, pero para verificarlo, vamos a listar todas las bases de datos existentes, ejecutando para ello la instrucción:

{% highlight sql %}
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| bd_viajes          |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.000 sec)
{% endhighlight %}

Efectivamente, la base de datos **bd_viajes** ha sido correctamente generada, sin embargo, de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma. En este caso, vamos a crear un usuario de nombre "**alvaro**" que tenga permitido el acceso desde **%** (es decir, desde cualquier dirección IP) y cuya contraseña sea también "**alvaro**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso de la instrucción:

{% highlight sql %}
MariaDB [(none)]> CREATE USER 'alvaro'@'%' IDENTIFIED BY 'alvaro';
Query OK, 0 rows affected (0.000 sec)
{% endhighlight %}

El usuario ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos la instrucción:

{% highlight sql %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_viajes.* TO 'alvaro'@'%';
Query OK, 0 rows affected (0.000 sec)
{% endhighlight %}

Una vez realizadas las oportunas modificaciones, podremos salir de _mysql_, con la finalidad de conectarnos al nuevo usuario para así verificar que todo funciona correctamente, haciendo para ello uso del comando:

{% highlight sql %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

**Nota**: No es necesario hacer uso del comando `FLUSH PRIVILEGES;`, a diferencia de varios artículos que he estado leyendo en Internet, que usan dicho comando muy a menudo sin necesidad alguna. Dicho comando, lo que hace es recargar la caché de las tablas GRANT que se encuentran en memoria, recarga que se hace de forma automática al hacer uso de una sentencia **GRANT**, de manera que no es necesario hacerlo manualmente. Para aclararlo, dicho comando únicamente debe utilizarse tras modificar las tablas GRANT de manera indirecta, es decir, tras usar sentencias INSERT, UPDATE o DELETE.

Para comprobar que todo ha funcionado correctamente, vamos a realizar un intento de conexión utilizando para ello el nuevo usuario **alvaro** que acabamos de generar, haciendo para ello uso del comando:

{% highlight shell %}
root@mysql:~# mysql -u alvaro -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 60
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Como se puede apreciar, la autenticación ha funcionado correctamente y estamos actualmente conectados a una _MySQL shell_ haciendo uso del usuario que acabamos de definir, el cuál cuenta con privilegios de administrador sobre la base de datos que acabamos de crear. El gestor de bases de datos se encuentra ya totalmente funcional.

Al no haber especificado explícitamente a qué base de datos nos queremos conectar, no nos ha conectado a ninguna, por lo que haremos uso de la instrucción `USE` para cambiarnos a la base de datos **bd_viajes**:

{% highlight sql %}
MariaDB [(none)]> USE bd_viajes;
Database changed
{% endhighlight %}

Cuando estemos haciendo uso del usuario y la base de datos correcta, procederé a crear algunas tablas e insertar una serie de registros. Para no ensuciar el artículo con la inserción de tablas y registros, se podrá encontrar [aquí](https://pastebin.com/SBYz1jmi) las instrucciones ejecutadas para ello.

Una vez insertadas las correspondientes tablas y registros, saldremos del cliente ejecutando la instrucción `quit` para así continuar con la parte referente al uso de dicha base de datos de forma remota.

Lo primero que haremos será visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
root@mysql:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:6a:8b:b5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.152/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86275sec preferred_lft 86275sec
    inet6 fe80::a00:27ff:fe6a:8bb5/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
{% endhighlight %}

De las dos interfaces mostradas, la única que nos interesa es aquella de nombre **enp0s3**, que tiene un direccionamiento **192.168.1.152/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_). Nos será necesario conocer dicha información para la correspondiente conexión desde el cliente.

Como anteriormente hemos mostrado, el servicio **mariadb** únicamente está escuchando peticiones provenientes de **_localhost_** en el puerto **3306**, de manera que no aceptará conexiones remotas hasta que no hagamos que escuche también en la interfaz **enp0s3**. Para llevar a cabo dicha adaptación, tendremos que modificar el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf**, haciendo para ello uso del comando:

{% highlight shell %}
root@mysql:~# nano /etc/mysql/mariadb.conf.d/50-server.cnf
{% endhighlight %}

Dentro del mismo, buscaremos la directiva **bind-address**, que tendrá la siguiente forma:

{% highlight shell %}
bind-address            = 127.0.0.1
{% endhighlight %}

En nuestro caso, nos interesa modificar el valor para que en lugar de escuchar en la dirección **127.0.0.1**, escuche en **0.0.0.0**, significando por tanto que aceptará conexiones en todas las interfaces de red existentes en la máquina. A pesar de ello, podríamos indicar que únicamente escuche en **192.168.1.152**, pero no tendría sentido, ya que en ese caso, bloquearía las conexiones entrantes desde _localhost_, cosa que no nos interesa. El resultado final sería:

{% highlight shell %}
bind-address            = 0.0.0.0
{% endhighlight %}

Personalmente, no tengo inconveniente en que el puerto que dicho servicio utilice para las conexiones entrantes sea el **3306**, pero en caso contrario, se podría modificar el valor de la directiva **port** en ese mismo fichero, estableciendo así el que más convenga.

Tras ello, guardaremos los cambios realizados y dado que hemos modificado un fichero de configuración, tendremos que reiniciar el servicio para que así cargue la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@mysql:~# systemctl restart mariadb
{% endhighlight %}

Una vez reiniciado el servicio, el _socket TCP/IP_ que anteriormente estaba únicamente configurado para escuchar peticiones provenientes de _localhost_ (**127.0.0.1**), debe estar ahora escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), por lo que procederemos a verificarlo haciendo uso una vez más de `netstat`:

{% highlight shell %}
root@mysql:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN
{% endhighlight %}

Efectivamente, el puerto 3306 está ahora abierto para escuchar peticiones desde cualquier interfaz.

Resumiendo, tenemos un servicio que está actualmente escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**) en el puerto **3306**. Además, hemos configurado un usuario **alvaro** sin ninguna restricción de autenticación a nivel de usuario, por lo que se podría usar cualquier dirección IP para conectarse al mismo (**%**), aunque podríamos haberle establecido una limitación en cuanto a ello en caso de haber sido necesario.

De otro lado, he creado otra máquina virtual conectada a su vez en modo puente (_bridge_) a mi red doméstica, de manera que cuenta con direccionamiento dentro de la misma red que la máquina servidora, encontrándose actualmente totalmente limpia y sin ninguna configuración realizada.

Tras ello, procederemos a instalar el paquete necesario para llevar a cabo la conexión remota, de nombre **mariadb-client**, no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando (con privilegios de administrador, haciendo uso de `su -`):

{% highlight shell %}
root@cliente:~# apt update && apt upgrade && apt install mariadb-client
{% endhighlight %}

Una vez instalado el paquete necesario, podremos llevar a cabo la conexión remota, haciendo para ello uso de las opciones **-u** y **-p** anteriormente utilizadas para indicar el usuario y contraseña a utilizar, con la única diferencia que tendremos que indicar además la dirección IP del servidor al que nos vamos a tratar de conectar, utilizando la opción **-h**, en este caso, **192.168.1.151**. La instrucción a ejecutar sería:

{% highlight shell %}
root@cliente:~# mysql -u alvaro -p -h 192.168.1.152
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 36
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Al parecer, la conexión se ha realizado sin ningún problema, pues tal y como he mencionado con anterioridad, la máquina cliente cuenta con un direccionamiento dentro de la red local, siendo la máquina servidora, por tanto, totalmente alcanzable, además de estar correctamente configurada para aceptar dichas conexiones.

De nada nos serviría tener conexión si no podemos mostrar la información contenida, así que antes de nada, vamos a hacer uso de la base de datos **bd_viajes**, pues es sobre la que el usuario actual tiene permisos, utilizando para ello la instrucción:

{% highlight sql %}
MariaDB [(none)]> USE bd_viajes;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Una vez conectados a la misma, vamos a proceder a listar las tablas existentes, para así asegurarnos de que tenemos acceso a la información, ejecutando para ello la instrucción:

{% highlight sql %}
MariaDB [bd_viajes]> SHOW TABLES;
+---------------------+
| Tables_in_bd_viajes |
+---------------------+
| Empleados           |
| Viajes              |
| ViajesPorEmpleado   |
+---------------------+
3 rows in set (0.001 sec)
{% endhighlight %}

Como era de esperar, las 3 tablas que previamente he creado son visibles desde la máquina cliente, así que vamos a ir un paso mas allá y vamos a tratar de mostrar los registros insertados en la tabla **Empleados**, por ejemplo, haciendo para ello uso la instrucción:

{% highlight sql %}
MariaDB [bd_viajes]> SELECT * FROM Empleados;
+-----------+---------------------------+------------------------+-----------+-----------------+
| DNI       | Nombre                    | Direccion              | Telefono  | FechaNacimiento |
+-----------+---------------------------+------------------------+-----------+-----------------+
| 12777631G | Merlino Rosado Cordero    | C/ Henan Cortes, 58    | 609841755 | 1993-11-27      |
| 18232747A | Aristarco Caban Meraz     | Puerta Nueva, 67       | 691204722 | 1994-05-07      |
| 52315160G | Heinz Collado Caraballo   | Escuadro, 60           | 600173822 | 1984-02-13      |
| 56228957Y | Dinorah Viera Tello       | Ctra. Villena, 22      | 642852778 | 1987-05-28      |
| 61242562W | Manases Castillo Camacho  | Ctra. Hornos, 91       | 607853354 | 1991-10-29      |
| 67227129S | Ian Esquivel Laboy        | C/ Inglaterra, 64      | 728005136 | 1992-07-21      |
| 68219319P | Tabare Chapa Alcantar     | C/ Arana, 12           | 682227206 | 1992-03-09      |
| 85145590G | Anabel Lerma Dominguez    | Crta. Cadiz, 1         | 675014823 | 1981-01-07      |
| 90389058R | Joaquin Marrero Covas     | C/ Hijuela de Lojo, 22 | 618385118 | 1997-01-28      |
| 94106513N | Marian Fonseca Betancourt | C/ Manuel Iradier, 37  | 638415823 | 1979-07-17      |
+-----------+---------------------------+------------------------+-----------+-----------------+
10 rows in set (0.001 sec)
{% endhighlight %}

Como se puede apreciar en la salida, se han mostrado todos los registros pertenecientes a la tabla indicada, por lo que podemos corroborar que la conexión remota a la base de datos está totalmente operativa y funcional.

La primera parte del artículo, referente a la instalación y configuración de MySQL para aceptar conexiones remotas ha finalizado, de manera que vamos a dar paso a la instalación de **phpMyAdmin**, una herramienta de administración web que permite interactuar con MariaDB pero sin requerir los conocimientos para hacerlo con una _shell_, al contar con las facilidades que dicha aplicación nos aporta.

Para ello, tendremos que instalar una pila **LAMP**, es decir, una unión de diferentes tecnologías, con la que podremos definir la infraestructura de un servidor web. Las tecnologías implicadas son las siguientes:

* **L**inux, el sistema operativo (en este caso, Debian Buster).
* **A**pache, el servidor web (en este caso, _apache 2.4_).
* **M**ySQL/**M**ariaDB, el gestor de bases de datos.
* **P**HP, el lenguaje de programación.

Dado que la aplicación **phpMyAdmin** se comunica de forma directa con la instalación de MariaDB, teniendo permisos para realizar acciones sobre la misma al contar con las credenciales para ello, es un blanco común de los atacantes, de manera que se recomienda que en caso de instalarla en una máquina ajena a la servidora de bases de datos, la conexión entre ambas sea totalmente cifrada. En esta ocasión, vamos a tener todos los servicios unificados en la misma máquina.

Para hacer uso de dicha aplicación, necesitamos un **servidor de aplicaciones** capaz de interpretar dicho código PHP y convertirlo en HTML, un lenguaje legible por el navegador web. Además, necesitamos un **servidor web** que devuelva el HTML a los clientes tras las correspondientes peticiones. Para ello, vamos a utilizar _apache2_ como servidor web y gracias a un módulo que instalaremos, también lograremos la funcionalidad del servidor de aplicaciones, de manera que estará unificado en un único servicio. Esta aplicación, como es lógico, accede a la base de datos mediante llamadas a la misma, por lo que también necesitaremos el **servidor de bases de datos** que ya tenemos instalado.

Los paquetes que tendremos que instalar para cada una de las tecnologías implicadas son:

* **L**inux - Suponemos que ya se encuentra instalado.
* **A**pache - **apache2** y **libapache2-mod-php**.
* **M**ySQL/**M**ariaDB - Suponemos que ya se encuentra instalado.
* **P**HP - **php** y **php-mysql**.

* **Extra**: Instalaremos además cuatro librerías necesarias (dependencias) para el correcto funcionamiento de la aplicación web (**php-zip**, **php-bz2**, **php-mbstring** y **php-gd**), así como una herramienta que nos permitirá generar contraseñas bastante complejas, que nos será necesaria más adelante (**pwgen**).

Esos son los paquetes a instalar para tener una pila LAMP totalmente operativa, de manera que ejecutaremos el comando:

{% highlight shell %}
root@mysql:~# apt install apache2 php php-mysql libapache2-mod-php php-zip php-bz2 php-mbstring php-gd pwgen
{% endhighlight %}

Todos los paquetes necesarios han sido instalados, de manera que los servicios se encontrarán ya en ejecución y listos para su uso.

Para verificar que _apache2_ está funcionando correctamente, vamos a acceder con el navegador web de la máquina anfitriona a la dirección IP de la máquina servidora (**192.168.1.152**), para así comprobar que esté sirviendo la página por defecto:

![apache](https://i.ibb.co/4F3rW5p/apache.jpg "Apache2.4 operativo")

Efectivamente, el servidor web se encuentra sirviendo la página por defecto. Internamente, lo que ha ocurrido, es que _apache2_ viene con un **VirtualHost** configurado, cuyo **DocumentRoot** es **/var/www/html/**, que contiene a su vez un fichero **index.html** que es el que se está sirviendo. Explicar con detalle cómo funcionan los VirtualHost en _apache2_ puede ser bastante extenso, además de salirse del objetivo de este articulo, de manera que [aquí](https://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html) podrás encontrar otro _post_ en el que se explica con mayor detalle.

A lo quiero llegar con esta breve explicación, es que vamos a reutilizar dicho VirtualHost para la configuración de **phpMyAdmin**. Realmente, podríamos crear un VirtualHost nuevo, en otra ruta diferente, pero ya que _apache2_ nos ofrece dicha facilidad, vamos a aprovecharla. Para ello, vamos a proceder a modificar el fichero de configuración del VirtualHost por defecto ubicado en **/etc/apache2/sites-available/**, de nombre **000-default**, haciendo para ello uso del comando:

{% highlight shell %}
root@mysql:~# nano /etc/apache2/sites-available/000-default.conf
{% endhighlight %}

Dentro del mismo, encontraremos la siguiente configuración por defecto (tras eliminar líneas comentadas para así dejar una salida más limpia):

{% highlight shell %}
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
{% endhighlight %}

La configuración por defecto la vamos a mantener, de manera que lo único que vamos a hacer es añadir nuevas directivas, entre las que se encuentran la definición de la ruta en la que se encontrarán los ficheros de **phpMyAdmin**, así como un Alias para que al acceder a la ruta virtual "**/phpmyadmin**", se muestre dicho contenido (lo veremos más adelante). Por último, prohibiremos el acceso a aquellos ficheros sensibles. El resultado final del VirtualHost sería:

{% highlight shell %}
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        Alias /phpmyadmin /usr/share/phpmyadmin

        <Directory /usr/share/phpmyadmin>
            Options SymLinksIfOwnerMatch
            DirectoryIndex index.php

            <IfModule mod_php5.c>
                <IfModule mod_mime.c>
                    AddType application/x-httpd-php .php
                </IfModule>
                <FilesMatch ".+\.php$">
                    SetHandler application/x-httpd-php
                </FilesMatch>

                php_value include_path .
                php_admin_value upload_tmp_dir /var/lib/phpmyadmin/tmp
                php_admin_value open_basedir /usr/share/phpmyadmin/:/etc/phpmyadmin/:/var/lib/phpmyadmin/:/usr/share/php/php-gettext/:/usr/share/php/php-php-gettext/:/usr/share/javascript/:/usr/share/php/tcpdf/:/usr/share/doc/phpmyadmin/:/usr/share/php/phpseclib/
                php_admin_value mbstring.func_overload 0
            </IfModule>
            <IfModule mod_php.c>
                <IfModule mod_mime.c>
                    AddType application/x-httpd-php .php
                </IfModule>
                <FilesMatch ".+\.php$">
                    SetHandler application/x-httpd-php
                </FilesMatch>

                php_value include_path .
                php_admin_value upload_tmp_dir /var/lib/phpmyadmin/tmp
                php_admin_value open_basedir /usr/share/phpmyadmin/:/etc/phpmyadmin/:/var/lib/phpmyadmin/:/usr/share/php/php-gettext/:/usr/share/php/php-php-gettext/:/usr/share/javascript/:/usr/share/php/tcpdf/:/usr/share/doc/phpmyadmin/:/usr/share/php/phpseclib/
                php_admin_value mbstring.func_overload 0
            </IfModule>
        </Directory>
        
        <Directory /usr/share/phpmyadmin/templates>
            Require all denied
        </Directory>
        <Directory /usr/share/phpmyadmin/libraries>
            Require all denied
        </Directory>
        <Directory /usr/share/phpmyadmin/setup/lib>
            Require all denied
        </Directory>
</VirtualHost>
{% endhighlight %}

Como se puede apreciar, el fichero de configuración final es bastante complejo, aunque para la tranquilidad de los lectores, no es para nada necesario comprender qué hacen las directivas contenidas para hacer uso de la aplicación web. Cuando hayamos finalizado, simplemente guardaremos los cambios en el fichero.

Por defecto, **phpMyAdmin** espera encontrar todos sus ficheros de configuración dentro del directorio **/usr/share/**, de manera que vamos a ubicar todos los ficheros correspondientes a la aplicación web en un subdirectorio dentro de dicha ruta. Lo primero que haremos será movernos dentro de dicho directorio, haciendo para ello uso de `cd`:

{% highlight shell %}
root@mysql:~# cd /usr/share/
{% endhighlight %}

En este caso, vamos a utilizar la última versión de **phpMyAdmin** disponible (5.0.4), que podremos descargar desde la [página oficial](https://www.phpmyadmin.net/), concretamente desde [aquí](https://files.phpmyadmin.net/phpMyAdmin/5.0.4/phpMyAdmin-5.0.4-all-languages.tar.gz). Para ello, haremos uso de `wget` para descargar el paquete comprimido de **phpMyAdmin** desde dicha web:

{% highlight shell %}
root@mysql:/usr/share# wget https://files.phpmyadmin.net/phpMyAdmin/5.0.4/phpMyAdmin-5.0.4-all-languages.tar.gz
--2020-12-05 06:58:06--  https://files.phpmyadmin.net/phpMyAdmin/5.0.4/phpMyAdmin-5.0.4-all-languages.tar.gz
Resolving files.phpmyadmin.net (files.phpmyadmin.net)... 195.181.167.19
Connecting to files.phpmyadmin.net (files.phpmyadmin.net)|195.181.167.19|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12898226 (12M) [application/octet-stream]
Saving to: ‘phpMyAdmin-5.0.4-all-languages.tar.gz’

phpMyAdmin-5.0.4-all-languages.tar.gz                100%[==========================>]  12.30M  11.3MB/s    in 1.1s    

2020-12-05 06:58:12 (11.3 MB/s) - ‘phpMyAdmin-5.0.4-all-languages.tar.gz’ saved [12898226/12898226]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, listaremos el contenido del directorio actual haciendo para ello uso del comando `ls -l`, estableciendo a su vez un filtro por nombre:

{% highlight shell %}
root@mysql:/usr/share# ls -l | egrep 'phpMyAdmin'
-rw-r--r--    1 root root 12898226 Oct 15 14:55 phpMyAdmin-5.0.4-all-languages.tar.gz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**phpMyAdmin-5.0.4-all-languages.tar.gz**" con un peso total de **12.30 MB** (12898226 _bytes_).

Al estar comprimido el fichero, no podremos hacer uso de la aplicación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@mysql:/usr/share# tar -zxf phpMyAdmin-5.0.4-all-languages.tar.gz && rm phpMyAdmin-5.0.4-all-languages.tar.gz
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.

Como resultado de dicha extracción, se habrá generado un subdirectorio de nombre **phpMyAdmin-5.0.4-all-languages/** con todos los ficheros contenidos, cuyo nombre modificaremos a **phpmyadmin/**, para así conseguir que sea más sencillo trabajar con el mismo, además de ser el nombre configurado en el VirtualHost de _apache2_, ya que de lo contrario, no sería capaz de encontrar los ficheros cuando el cliente quiera hacer uso de la aplicación web. Para llevar a cabo la modificación del nombre, haremos uso del comando:

{% highlight shell %}
root@mysql:/usr/share# mv phpMyAdmin-5.0.4-all-languages/ phpmyadmin/
{% endhighlight %}

Para verificar que el cambio se ha realizado correctamente, volveremos a listar el contenido del directorio actual, estableciendo una vez más un filtro por nombre:

{% highlight shell %}
root@mysql:/usr/share# ls -l | egrep 'phpmyadmin'
drwxr-xr-x   12 root root  4096 Oct 15 14:07 phpmyadmin
{% endhighlight %}

Como era de esperar, el nombre del directorio que contiene todos los ficheros necesarios para el funcionamiento de la aplicación web es ahora **phpmyadmin/**, por lo que nos moveremos dentro del mismo para así seguir trabajando, ejecutando para ello el comando:

{% highlight shell %}
root@mysql:/usr/share# cd phpmyadmin/
{% endhighlight %}

Una vez dentro del correspondiente subdirectorio, vamos a proceder a listar el contenido del mismo, para así asegurarnos de que todos los ficheros han sido correctamente descomprimidos, haciendo uso del comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# ls -l
total 944
-rw-r--r--  1 root root   2011 Oct 15 14:07 ajax.php
-rw-r--r--  1 root root   1815 Oct 15 14:07 browse_foreigners.php
-rw-r--r--  1 root root  33015 Oct 15 14:07 ChangeLog
-rw-r--r--  1 root root   3114 Oct 15 14:07 changelog.php
...
-rw-r--r--  1 root root   1197 Oct 15 14:07 version_check.php
-rw-r--r--  1 root root   7186 Oct 15 14:07 view_create.php
-rw-r--r--  1 root root   3529 Oct 15 14:07 view_operations.php
-rw-r--r--  1 root root 111929 Oct 15 14:07 yarn.lock
{% endhighlight %}

Como se puede apreciar en la salida que manualmente he recortado para no ensuciar, existen una gran cantidad de ficheros en el directorio actual, necesarios para el correcto funcionamiento de la aplicación web, por lo que podemos asegurar que se han descomprimido correctamente.

Sin embargo, como consecuencia de haber instalado la aplicación web desde su página oficial de descargas en lugar de utilizar un gestor de paquetes (principalmente porque no está disponible en los repositorios Debian), no contamos con determinada configuración que lleva a cabo el gestor de paquetes de forma automática en el momento de la instalación, de manera que tendremos que llevar a cabo dicha configuración de forma manual.

Lo primero que tendremos que hacer será generar un nuevo directorio que utilizará **phpMyAdmin** para almacenar sus ficheros temporales, ubicado en **/var/lib/phpmyadmin/tmp/**. Para llevar a cabo su creación, ejecutaremos el comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# mkdir -p /var/lib/phpmyadmin/tmp
{% endhighlight %}

Donde:

* **-p**: Indica que se genere además el directorio padre en caso de no existir. En este caso, generará el subdirectorio **phpmyadmin/** dentro de **/var/lib/**, ya que no existe.

Dado que el usuario propietario por defecto a través del cuál se sirven las páginas web en _apache2_ es **www-data**, procedemos a cambiar el usuario y grupo propietario de forma recursiva para dicho subdirectorio, haciendo para ello uso del comando `chown -R`, pues de otro modo no podría escribir en dichos directorios en caso de necesitarlo:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# chown -R www-data:www-data /var/lib/phpmyadmin
{% endhighlight %}

La aplicación web ya es capaz de escribir en dichos directorios, al haber modificado su usuario y grupo propietario. De otro lado, uno de los ficheros resultantes de la previa extracción del comprimido es un fichero de configuración de ejemplo (**config.sample.inc.php**) que utilizaremos como fichero de configuración base, para así adaptarlo a nuestras necesidades, de manera que lo copiaremos a uno de nombre **config.inc.php**, ejecutando para ello el comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# cp config.sample.inc.php config.inc.php
{% endhighlight %}

Antes de comenzar con la configuración de dicho fichero, vamos a modificar también el usuario y grupo propietario para los ficheros del directorio actual, de manera que **www-data** tenga permisos sobre los mismos, haciendo para ello uso del comando visto con anterioridad:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# chown -R www-data:www-data .
{% endhighlight %}

Para verificar que la modificación se ha llevado a cabo correctamente, vamos a listar una vez más el contenido del directorio actual:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# ls -l
total 944
-rw-r--r--  1 www-data www-data   2011 Oct 15 14:07 ajax.php
-rw-r--r--  1 www-data www-data   1815 Oct 15 14:07 browse_foreigners.php
-rw-r--r--  1 www-data www-data  33015 Oct 15 14:07 ChangeLog
-rw-r--r--  1 www-data www-data   3114 Oct 15 14:07 changelog.php
...
-rw-r--r--  1 www-data www-data   1197 Oct 15 14:07 version_check.php
-rw-r--r--  1 www-data www-data   7186 Oct 15 14:07 view_create.php
-rw-r--r--  1 www-data www-data   3529 Oct 15 14:07 view_operations.php
-rw-r--r--  1 www-data www-data 111929 Oct 15 14:07 yarn.lock
{% endhighlight %}

Como se puede apreciar en la salida que manualmente he recortado para no ensuciar, el usuario y grupo propietario de dichos ficheros ha pasado de ser **root** a ser **www-data**, de manera que _apache2_ no tendrá ya ningún problema para hacer uso de los mismos.

Si recordamos, anteriormente hemos instalado una herramienta de nombre **pwgen**. Bien, ha llegado el momento de darle uso, ya que necesitamos generar una contraseña de 32 caracteres aleatorios, de manera que ejecutaremos el comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# pwgen -s 32 1
ym2IVM03F6yyENnWdyn1k7lYVtNSJeon
{% endhighlight %}

Donde:

* **-s**: Indicamos que genere contraseñas totalmente aleatorias, difíciles de memorizar por personas humanas.
* **32**: Indicamos el numero de caracteres que tendrá la contraseña a generar.
* **1**: Indicamos el número de contraseñas a generar haciendo uso de las opciones anteriormente introducidas.

La contraseña que ha generado de forma aleatoria ha sido **ym2IVM03F6yyENnWdyn1k7lYVtNSJeon**, que lógicamente no me importa mostrar ya que este servidor no se va a poner en producción, pero en caso contrario, tendríais que guardarla bajo llave. Recomiendo almacenarla en el portapapeles ya que la tendremos que introducir en el fichero de configuración en unos instantes, el cuál modificaremos haciendo uso del comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# nano config.inc.php
{% endhighlight %}

Dentro del mismo, tendremos que buscar una línea que comienza con **$cfg['blowfish_secret']** y darle el valor de la contraseña que acabamos de generar, quedando en mi caso de la siguiente forma:

{% highlight shell %}
$cfg['blowfish_secret'] = 'ym2IVM03F6yyENnWdyn1k7lYVtNSJeon'; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
{% endhighlight %}

Internamente, **phpMyAdmin** utiliza por defecto un método de autenticación de **_cookie_**, consistente en almacenar la contraseña del usuario que pretende autenticarse en una _cookie_ temporal cifrada mediante el algoritmo AES, que hará uso de la cadena que acabamos de introducir para llevar a cabo dicho cifrado. Lo menciono como curiosidad, ya que no es necesario conocer el funcionamiento para hacer uso de la aplicación web.

Tras ello, bajaremos hasta la sección **/* User used to manipulate with storage */**, en la que encontraremos el siguiente contenido:

{% highlight shell %}
// $cfg['Servers'][$i]['controlhost'] = '';
// $cfg['Servers'][$i]['controlport'] = '';
// $cfg['Servers'][$i]['controluser'] = 'pma';
// $cfg['Servers'][$i]['controlpass'] = 'password';
{% endhighlight %}

En dicha sección encontramos algunas directivas referentes al uso de una base de datos de nombre **pma**, que lleva a cabo determinadas tareas administrativas dentro de **phpMyAdmin**. Según la documentación oficial, el uso de dicha base de datos únicamente es necesaria cuando vayan a realizarse varias conexiones simultáneas a la aplicación web por parte de varios usuarios (en nuestro caso, va a haber una única conexión y no habría ningún inconveniente, pero aun así, voy a proceder con su configuración para así mostrar cómo sería).

De las cuatro directivas existentes, únicamente nos será necesario descomentar las dos últimas, actualizando además la contraseña, para así no hacer uso de una genérica y que usuarios remotos pudiesen acceder a nuestra aplicación muy fácilmente. En mi caso, he decidido establecer la contraseña **pmapass** (aunque siempre es recomendable utilizar una más segura), quedando de la siguiente forma:

{% highlight shell %}
// $cfg['Servers'][$i]['controlhost'] = '';
// $cfg['Servers'][$i]['controlport'] = '';
$cfg['Servers'][$i]['controluser'] = 'pma';
$cfg['Servers'][$i]['controlpass'] = 'pmapass';
{% endhighlight %}

Ya hemos finalizado con la configuración de dicha sección, así que pasaremos a la siguiente, aquella de nombre **/* Storage database and tables */**, que incluye una serie de directivas que definen la configuración del almacenamiento de **phpMyAdmin**, una base de datos y una serie de tablas (que crearemos pronto) utilizadas por el usuario **pma** previamente definido, que servirán para habilitar determinadas funcionalidades, como generación de ficheros PDF, marcadores, comentarios... de las que nosotros no haremos uso en este caso, pero estarán disponibles.

Lo único que tendremos que hacer será descomentar todas las directivas pertenecientes a dicha sección, quedando finalmente de la siguiente forma:

{% highlight shell %}
$cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
$cfg['Servers'][$i]['bookmarktable'] = 'pma__bookmark';
$cfg['Servers'][$i]['relation'] = 'pma__relation';
$cfg['Servers'][$i]['table_info'] = 'pma__table_info';
$cfg['Servers'][$i]['table_coords'] = 'pma__table_coords';
$cfg['Servers'][$i]['pdf_pages'] = 'pma__pdf_pages';
$cfg['Servers'][$i]['column_info'] = 'pma__column_info';
$cfg['Servers'][$i]['history'] = 'pma__history';
$cfg['Servers'][$i]['table_uiprefs'] = 'pma__table_uiprefs';
$cfg['Servers'][$i]['tracking'] = 'pma__tracking';
$cfg['Servers'][$i]['userconfig'] = 'pma__userconfig';
$cfg['Servers'][$i]['recent'] = 'pma__recent';
$cfg['Servers'][$i]['favorite'] = 'pma__favorite';
$cfg['Servers'][$i]['users'] = 'pma__users';
$cfg['Servers'][$i]['usergroups'] = 'pma__usergroups';
$cfg['Servers'][$i]['navigationhiding'] = 'pma__navigationhiding';
$cfg['Servers'][$i]['savedsearches'] = 'pma__savedsearches';
$cfg['Servers'][$i]['central_columns'] = 'pma__central_columns';
$cfg['Servers'][$i]['designer_settings'] = 'pma__designer_settings';
$cfg['Servers'][$i]['export_templates'] = 'pma__export_templates';
{% endhighlight %}

Por último, bajaremos al final del documento y añadiremos una directiva en la que definiremos el directorio previamente creado para almacenar ficheros temporales por parte de la aplicación web, permitiendo así una carga mucho más rápida de las páginas:

{% highlight shell %}
$cfg['TempDir'] = '/var/lib/phpmyadmin/tmp';
{% endhighlight %}

La configuración del fichero ha finalizado, por lo que saldremos guardando los cambios en el mismo.

El siguiente paso será generar la base de datos con las correspondientes tablas de las que hará uso el usuario **pma**. Para ello, tenemos a nuestra disposición un fichero de nombre **create_tables.sql** dentro del directorio **/usr/share/phpmyadmin/sql/**, que contendrá las instrucciones necesarias para llevar a cabo dicha creación, por lo que inyectaremos el contenido de dicho fichero a `mariadb`, ejecutando para ello el comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# mariadb < sql/create_tables.sql
{% endhighlight %}

Para verificar que la creación se ha llevado a cabo correctamente, accederemos a la _MySQL shell_ como usuario **root**, haciendo para ello uso del comando ejecutado con anterioridad:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 64
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Una vez conectados a la _shell_, listaremos las bases de datos existentes, para así comprobar que se ha generado aquella de nombre **phpmyadmin**, que será la que utilizará el usuario **pma**, ejecutando para ello la instrucción:

{% highlight shell %}
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| bd_viajes          |
| information_schema |
| mysql              |
| performance_schema |
| phpmyadmin         |
+--------------------+
5 rows in set (0.090 sec)
{% endhighlight %}

Efectivamente, la base de datos ha sido correctamente generada, de manera que nos queda un último paso antes de poder empezar a hacer uso de la aplicación web, consistente en crear el usuario **pma**, así como otorgarle permisos sobre la base de datos **phpmyadmin**. La contraseña que le asignaremos al mismo será aquella indicada en el fichero de configuración, en este caso, **pmapass**:

{% highlight shell %}
MariaDB [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE ON phpmyadmin.* TO 'pma'@'localhost' IDENTIFIED BY 'pmapass';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez creado el correspondiente usuario y otorgados los privilegios necesarios, saldremos del cliente ejecutando la instrucción `quit` para así proceder a comprobar que toda la configuración que hemos llevado a cabo ha funcionado, no sin antes volver a cargar en memoria la configuración del servicio _apache2_ para que así surtan efecto los cambios realizados con anterioridad. Para ello, haremos uso del comando:

{% highlight shell %}
root@mysql:/usr/share/phpmyadmin# systemctl reload apache2
{% endhighlight %}

Tras ello, accederemos desde el navegador web de la máquina anfitriona a la dirección IP de la máquina servidora (**192.168.1.152**), solicitando concretamente el recurso **/phpmyadmin**, que es una ruta virtual que mostrará la aplicación web **phpMyAdmin**:

![phpmyadmin1](https://i.ibb.co/fpRrqNx/phpmyadmin2.jpg "phpMyAdmin")

Como se puede apreciar, se ha mostrado correctamente el formulario de _login_, en el cuál indicaremos en este caso, el usuario **alvaro** y la contraseña **alvaro**, correspondientes al usuario con privilegios sobre la base de datos **bd_viajes**, que es sobre la que hemos estado trabajando a lo largo del artículo. Pulsaremos en **Continuar**:

![phpmyadmin2](https://i.ibb.co/kcgCGKT/phpmyadmin3.jpg "phpMyAdmin")

Efectivamente, hemos podido acceder sin problemas, y de un vistazo rápido podemos apreciar que en la parte izquierda se muestran las bases de datos visibles para el usuario actual, además de determinada información sobre los servidores en la parte derecha y una serie de utilidades en el menú superior. Para hacer una pequeña prueba, pulsaremos sobre la base de datos **bd_viajes** para ver si la información de la misma es accesible:

![phpmyadmin3](https://i.ibb.co/ryTZzXy/phpmyadmin4.jpg "phpMyAdmin")

Como era de esperar, los nombres de las tablas existentes en dicha base de datos han sido mostrados sin problemas, así que para finalizar, vamos a pulsar en la tabla **Empleados**, para comprobar si podemos acceder a los registros de la misma:

![phpmyadmin4](https://i.ibb.co/mGYs2B4/phpmyadmin5.jpg "phpMyAdmin")

Los registros existentes en la tabla **Empleados** han sido mostrados a través de la aplicación web **phpMyAdmin**, ejecutando internamente una instrucción `SELECT * FROM Empleados;`, pero que el cliente que haga uso de la aplicación web, no tiene por qué conocer. Por tanto, podemos asegurar que la configuración realizada ha funcionado correctamente y que por tanto, la aplicación se encuentra totalmente operativa y securizada.

Por último, me gustaría mencionar que **phpMyAdmin** tiene una gran cantidad de utilidades que no voy a explicar en este artículo ya que se saldría del objetivo del mismo (quizás en un futuro me animo a hacer uno al respecto). Sin embargo, navegando por Internet podréis encontrar bastante información sobre ello.