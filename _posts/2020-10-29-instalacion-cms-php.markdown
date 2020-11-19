---
layout: post
title:  "Instalación de un CMS PHP"
banner: "/assets/images/banners/cms.jpg"
date:   2020-10-29 19:45:00 +0200
categories: aplicaciones
---
Se recomienda realizar una previa lectura del _post_ [VirtualHosting con Apache](http://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Tarea 1: Instalación de un servidor LAMP.

Un servidor **LAMP** contiene una unión de diferentes tecnologías, con la que podremos definir la infraestructura de un servidor web. Las tecnologías implicadas son las siguientes:

* **L**inux, el sistema operativo (en este caso, Debian Buster).
* **A**pache, el servidor web (en este caso, _apache 2.4_).
* **M**ySQL/**M**ariaDB, el gestor de bases de datos.
* **P**HP, el lenguaje de programación.

En este caso, vamos a tener un **CMS** (aplicación web de propósito general) escrito en PHP. Para servir dicho CMS necesitamos un **servidor de aplicaciones** capaz de interpretar dicho código PHP y convertirlo en HTML, un lenguaje legible por el navegador web. Además, necesitamos un **servidor web** que devuelva el HTML a los clientes tras las correspondientes peticiones. Para ello, vamos a utilizar _apache2_ como servidor web y gracias a un módulo que instalaremos, también lograremos la funcionalidad del servidor de aplicaciones, de manera que estará unificado en un único servicio. Éstas aplicaciones CMS guardan sus datos en una base de datos, es decir, el código PHP que se ejecuta, accede a la base de datos mediante llamadas a la misma, por lo que también necesitaremos un **servidor de bases de datos**.

### Crea una instancia de Vagrant basado en un box Debian o Ubuntu.

Para ésta práctica, necesitaremos una máquina que actuará como servidora desde un primer momento y además, otra máquina a la que posteriormente moveremos la base de datos, de manera que en un principio tendremos toda la pila LAMP en la primera máquina y posteriormente, el servidor de bases de datos no será local, sino remoto.

La primera de ellas deberá tener una interfaz de red host-only (de manera que tenga conexión directa con la máquina anfitriona y podamos acceder a la misma desde el navegador) y una dirección dentro de una red interna, estando dentro de la misma red interna la segunda máquina (no es necesario que la segunda máquina tenga una interfaz host-only, pues no necesitamos acceder desde la máquina anfitriona). En este caso, el fichero de configuración **Vagrantfile** tiene la siguiente forma:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
    config.vm.define :servidor do |servidor| #Definimos la primera máquina, en este caso, el servidor.
        servidor.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
        servidor.vm.hostname = "cms" #Establecemos el nombre (hostname) de la máquina. En este caso, "cms".
        servidor.vm.network :private_network, ip: "172.22.100.2", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor y nos conviene que la IP sea siempre la misma.
                virtualbox__intnet: "lan1" #Dado que VirtualBox crea por defecto las interfaces en modo host-only, tenemos que indicar que utilice una red interna, en este caso, una de nombre "lan1".
        servidor.vm.network :private_network, ip: "192.168.50.2" #Creamos una interfaz de red host-only, que tendrá un direccionamiento estático, pues se trata del servidor y nos conviene que siempre sea la misma.
    end
    config.vm.define :backup do |backup| #Definimos la segunda máquina, en este caso, el backup.
        backup.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
        backup.vm.hostname = "backup" #Establecemos el nombre (hostname) de la máquina. En este caso, "backup".
        backup.vm.network :private_network, ip: "172.22.100.10", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor de backup y nos conviene que la IP sea siempre la misma.
                virtualbox__intnet: "lan1" #Al igual que la segunda interfaz de red del servidor, ésta también la conectaremos a la red interna de nombre "lan1".
    end
end
{% endhighlight %}

Tras ello, levantaremos el escenario Vagrant haciendo uso de la instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/cms$ vagrant up
{% endhighlight %}

A continuación comenzará a descargar el **box** (en caso de que no lo tuviésemos previamente) y a generar las máquinas virtuales con los parámetros que les hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la primera de ellas ejecutando la siguiente instrucción (pues todavía no es necesario configurar la máquina de backup):

{% highlight shell %}
alvaro@debian:~/vagrant/cms$ vagrant ssh servidor
Linux cms 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@cms:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86367sec preferred_lft 86367sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:4d:8f:52 brd ff:ff:ff:ff:ff:ff
    inet 172.22.100.2/24 brd 172.22.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe4d:8f52/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ce:c8:6a brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.2/24 brd 192.168.50.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fece:c86a/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **tres** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **172.22.100.2**.
* **eth2:** Creada por nosotros y conectada en modo host-only al anfitrión, con dirección IP estática **192.168.50.2**.

### Instala en esa máquina virtual toda la pila LAMP.

Los paquetes que tendremos que instalar para cada una de las tecnologías implicadas son:

* **L**inux - Suponemos que ya se encuentra instalado.
* **A**pache - **apache2** y **libapache2-mod-php**.
* **M**ySQL/**M**ariaDB - **mariadb-client** y **mariadb-server**.
* **P**HP - **php** y **php-mysql**.

Esos son los paquetes a instalar para tener una pila LAMP totalmente operativa, no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@cms:~# apt update && apt upgrade && apt install apache2 mariadb-client mariadb-server php php-mysql libapache2-mod-php
{% endhighlight %}

Listo, todos los paquetes se encuentran ya instalados.

## Tarea 2: Instalación de Drupal en mi servidor local.

### Configura el servidor web con un VirtualHost para que el CMS sea accesible desde la dirección www.alvaro-drupal.org.

Para configurar un nuevo VirtualHost de _apache2_, debemos modificar el fichero de configuración **/etc/apache2/apache2.conf**, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@cms:~# nano /etc/apache2/apache2.conf
{% endhighlight %}

Dentro del mismo, tendremos añadir una sección **Directory** en el que especificaremos el directorio padre dentro del cuál alojaremos los diferentes DocumentRoot, pues dichos DocumentRoot heredarán los permisos del mismo. En este caso, he decidido que dicho directorio padre sea **/srv/**, por lo que le he configurado algunas directivas estándar, quedando de la siguiente forma:

{% highlight shell %}
<Directory /srv/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

El directorio padre donde alojaremos los diferentes DocumentRoot ya se encuentra creado, pero todavía tenemos que generar dichos DocumentRoot. Para el caso de Drupal, el primer CMS que instalaremos, voy a generar un subdirectorio dentro de **/srv/** de nombre **drupal/**, que será el DocumentRoot del VirtualHost que configuraremos a continuación, ejecutando para ello el comando:

{% highlight shell %}
root@cms:~# mkdir /srv/drupal
{% endhighlight %}

Para verificar que el directorio se ha generado correctamente, haremos uso del comando `ls`:

{% highlight shell %}
root@cms:~# ls /srv/
drupal
{% endhighlight %}

Efectivamente, el directorio se ha generado correctamente. Tras ello, únicamente nos queda crear el nuevo fichero de configuración dentro de **/etc/apache2/sites-available/**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@cms:~# cd /etc/apache2/sites-available/
{% endhighlight %}

Una vez dentro del mismo, vamos a utilizar como plantilla el fichero de configuración del VirtualHost que trae _apache2_ configurado por defecto, **000-default.conf**, así que copiaremos dicho fichero a uno de nombre **drupal.conf** (por ejemplo, aunque se podría haber usado cualquier otro nombre). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# cp 000-default.conf drupal.conf
{% endhighlight %}

Para verificar que el nuevo fichero de configuración se ha generado correctamente, volveremos a ejecutar el comando `ls`:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# ls
000-default.conf  default-ssl.conf  drupal.conf
{% endhighlight %}

Como era de esperar, el fichero se encuentra generado, así que lo único que queda es realizar las modificaciones oportunas al mismo, tales como indicar **www.alvaro-drupal.com** como **ServerName** y **/srv/drupal** como **DocumentRoot**, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# nano drupal.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.alvaro-drupal.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/drupal

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Esa sería la apariencia del fichero de configuración hasta ahora, así que únicamente nos queda habilitar el VirtualHost que acabamos de crear, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# a2ensite drupal
Enabling site drupal.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que recargar la configuración del servicio _apache2_, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez que la configuración se ha vuelto a cargar, vamos a listar el contenido de **/etc/apache2/sites-enabled** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# ls -l /etc/apache2/sites-enabled/
total 0
lrwxrwxrwx 1 root root 35 Oct 22 15:35 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root 30 Oct 22 15:44 drupal.conf -> ../sites-available/drupal.conf
{% endhighlight %}

Efectivamente, el nuevo VirtualHost se encuentra activo. Tras ello, volveremos a la máquina anfitriona y configuraremos la resolución estática de nombres en la misma, para así poder realizar la traducción del nombre a la IP de la máquina y que así podamos acceder, gracias a la cabecera Host de la petición HTTP. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** para el nuevo sitio que hemos creado, siendo **[IP]** la dirección IP de la máquina servidora (en este caso **192.168.50.2**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.50.2    www.alvaro-drupal.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para que se lleve a cabo la resolución estática de nombres, que tiene prioridad sobre las peticiones al servidor DNS, verificando así que el servidor _apache2_ se encuentra operativo y con el VirtualHost funcionando:

![hosts](https://i.ibb.co/RN5xTxV/Captura-de-pantalla-de-2020-10-22-17-55-45.png "www.alvaro-drupal.com")

Como podemos apreciar, se ha podido acceder sin problema, ya que hemos realizado correctamente la configuración del **ServerName** en el fichero de configuración y a la hora de comparar la cabecera **Host** existente en la petición HTTP, ha encontrado coincidencia con dicho VirtualHost.

### Crea un usuario en la base de datos donde se van a guardar los datos del CMS.

Lo primero que debemos hacer es acceder al servidor **mysql**, ejecutando para ello el comando (cuando nos pida una contraseña, no introduciremos nada, simplemente pulsaremos ENTER):

{% highlight shell %}
root@cms:/etc/apache2/sites-available# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 37
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Donde:

* **-u**: Indica el usuario con el que nos vamos a conectar, en este caso, **root**.
* **-p**: Indicamos la contraseña, en este caso, no introducimos ninguna.

Una vez dentro del servidor de bases de datos, tendremos que crear una nueva base de datos, valga la redundancia. En este caso, para identificarla fácilmente, de nombre **drupal**. Para ello, ejecutamos el comando:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE drupal;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

La base de datos que usará Drupal ya se encuentra creada, pero de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma. En este caso, vamos a crear un usuario de nombre "**username**" que tenga permitido el acceso desde **localhost** (es decir, desde la máquina local) y cuya contraseña sea "**usuario**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso del comando:

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'username'@'localhost' IDENTIFIED BY 'usuario';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

El usuario ya ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON drupal.* to 'username'@'localhost';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

**Nota**: No es necesario hacer uso del comando `FLUSH PRIVILEGES;`, a diferencia de varios artículos que he estado leyendo en Internet, que usan dicho comando muy a menudo sin necesidad alguna. Dicho comando, lo que hace es recargar la caché de las tablas GRANT que se encuentran en memoria, recarga que se hace de forma automática al hacer uso de una sentencia **GRANT**, de manera que no es necesario hacerlo manualmente. Para aclararlo, dicho comando únicamente debe utilizarse tras modificar las tablas GRANT de manera indirecta, es decir, tras usar sentencias INSERT, UPDATE o DELETE.

### Descarga la versión que te parezca más oportuna de Drupal y realiza la instalación.

En este caso, vamos a utilizar la última versión de Drupal (9.0.7), que podremos descargar desde la [página oficial](https://www.drupal.org/), concretamente desde [aquí](https://www.drupal.org/download-latest/tar.gz). Para ello, primero nos moveremos al DocumentRoot donde instalaremos Drupal, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/etc/apache2/sites-available# cd /srv/drupal/
{% endhighlight %}

Una vez dentro del mismo, haremos uso de `wget` para descargar el paquete comprimido de Drupal desde dicha web:

{% highlight shell %}
root@cms:/srv/drupal# wget https://www.drupal.org/download-latest/tar.gz
--2020-10-22 16:06:54--  https://www.drupal.org/download-latest/tar.gz
Resolving www.drupal.org (www.drupal.org)... 151.101.134.217
Connecting to www.drupal.org (www.drupal.org)|151.101.134.217|:443... connected.
HTTP request sent, awaiting response... 302 Moved Temporarily
Location: https://ftp.drupal.org/files/projects/drupal-9.0.7.tar.gz [following]
--2020-10-22 16:06:55--  https://ftp.drupal.org/files/projects/drupal-9.0.7.tar.gz
Resolving ftp.drupal.org (ftp.drupal.org)... 151.101.134.217
Connecting to ftp.drupal.org (ftp.drupal.org)|151.101.134.217|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16863270 (16M) [application/octet-stream]
Saving to: ‘tar.gz’

tar.gz                100%[==========================>]  16.08M  11.0MB/s    in 1.5s    

2020-10-22 16:06:57 (11.0 MB/s) - ‘tar.gz’ saved [16863270/16863270]
{% endhighlight %}

Como curiosidad, si nos fijamos en los primeros mensajes, el enlace de descarga que hemos introducido realmente lo que tiene configurada es una redirección temporal (302) que apunta al paquete .tar.gz de la última versión disponible de Drupal (download-latest), de manera que cuando publiquen una nueva versión, cambian dicha redirección para que apunte al nuevo paquete.

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`:

{% highlight shell %}
root@cms:/srv/drupal# ls -l
total 16472
-rw-r--r-- 1 root root 16863270 Oct  7 19:48 tar.gz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**tar.gz**" con un peso total de **16.08 MB** (16863270 bytes).

Al estar comprimido el fichero, no podemos llevar a cabo la instalación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/drupal# tar -zxf tar.gz --strip 1 && rm -r tar.gz
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.
* **--strip 1**: Saltamos el primer directorio, ya que dentro del comprimido hay un directorio padre que no necesitamos.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
root@cms:/srv/drupal# ls -la
total 284
drwxr-xr-x  8 root     root       4096 Oct 23 07:37 .
drwxr-xr-x  3 root     root       4096 Oct 22 15:55 ..
-rw-r--r--  1 root     root        312 Oct  7 19:48 autoload.php
-rw-r--r--  1 root     root       3044 Oct  7 19:48 composer.json
-rw-r--r--  1 root     root     157644 Oct  7 19:48 composer.lock
drwxr-xr-x 12 root     root       4096 Oct  7 19:48 core
-rw-r--r--  1 root     root       1025 Oct  7 19:48 .csslintrc
-rw-r--r--  1 root     root        357 Oct  7 19:48 .editorconfig
-rw-r--r--  1 root     root        151 Oct  7 19:48 .eslintignore
-rw-r--r--  1 root     root         41 Oct  7 19:48 .eslintrc.json
-rw-r--r--  1 root     root       1507 Oct  7 19:48 example.gitignore
-rw-r--r--  1 root     root       3858 Oct  7 19:48 .gitattributes
-rw-r--r--  1 root     root       7567 Oct  7 19:48 .htaccess
-rw-r--r--  1 root     root       2314 Oct  7 19:48 .ht.router.php
-rw-r--r--  1 root     root        549 Oct  7 19:48 index.php
-rw-r--r--  1 root     root         95 Oct  7 19:48 INSTALL.txt
-rw-r--r--  1 root     root      18092 Nov 16  2016 LICENSE.txt
drwxr-xr-x  2 root     root       4096 Oct  7 19:48 modules
drwxr-xr-x  2 root     root       4096 Oct  7 19:48 profiles
-rw-r--r--  1 root     root       5971 Oct  7 19:48 README.txt
-rw-r--r--  1 root     root       1594 Oct  7 19:48 robots.txt
drwxr-xr-x  3 root     root       4096 Oct  7 19:48 sites
drwxr-xr-x  2 root     root       4096 Oct  7 19:48 themes
-rw-r--r--  1 root     root        848 Oct  7 19:48 update.php
drwxr-xr-x 19 root     root       4096 Oct  7 19:48 vendor
-rw-r--r--  1 root     root       4566 Oct  7 19:48 web.config
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos (en lugar de descomprimir un directorio de nombre **drupal-9.0.7** del que posteriormente tendríamos que mover los ficheros contenidos al directorio actual).

Dado que hemos indicado también la opción **-a**, nos ha mostrado los ficheros ocultos (es decir, en Linux son aquellos que empiezan por **.**). Si nos fijamos, existe uno de nombre **.htaccess**, que nos hará falta más adelante.

Ya tenemos todos los ficheros necesarios para llevar a cabo la instalación de Drupal, pero hay un pequeño detalle que todavía no hemos contemplado, pues el usuario creador y el grupo correspondiente a dichos ficheros y directorios es **root**, cosa que jamás debe ocurrir ya que el usuario propietario por defecto a través del cuál se sirven las páginas web es **www-data**, por lo que procedemos a cambiar dicho propietario y grupo de forma recursiva, haciendo para ello uso del comando `chown -R`, pues de otro modo no podría escribir en dichos ficheros durante la instalación:

{% highlight shell %}
root@cms:/srv/drupal# chown -R www-data:www-data /srv/*
{% endhighlight %}

Para verificar que el cambio se ha llevado a cabo correctamente, volveremos a listar el contenido de **/srv**, haciendo uso de `ls -l`:

{% highlight shell %}
root@cms:/srv/drupal# ls -l /srv/
total 4
drwxr-xr-x 8 www-data www-data 4096 Oct 22 16:17 drupal
{% endhighlight %}

Como era de esperar, el usuario y el grupo propietario del directorio **drupal/** se ha visto modificado, al igual que todos sus ficheros y directorios contenidos, al haberlo hecho de forma recursiva.

Ya está todo listo para llevar a cabo la instalación, así que volveremos a acceder al navegador de la máquina anfitriona y escribiremos el nombre de dominio previamente configurado, **www.alvaro-drupal.com**:

![drupal1](https://i.ibb.co/4sS8vVj/Captura-de-pantalla-de-2020-10-22-18-14-32.png "Elección de idioma")

Como se puede apreciar, se ha está mostrando correctamente el proceso de instalación de Drupal, pues gracias a una directiva **DirectoryIndex** configurada en el servidor _apache2_, ha sido capaz de leer el fichero **index.php** situado en el DocumentRoot. Lo primero que preguntará es el idioma, en este caso elegiré el **Español** y pulsaré en **Save and continue**:

![drupal2](https://i.ibb.co/WHtRZpN/Captura-de-pantalla-de-2020-10-22-18-19-40.png "Perfil de instalación")

Lo siguiente será seleccionar un perfil de instalación, que en este caso elegiremos el perfil **Estándar**, para que instale Drupal con las características más comunes preconfiguradas:

![drupal3](https://i.ibb.co/DgJqLsk/Captura-de-pantalla-de-2020-10-22-18-19-51.png "Problemas de requisitos")

Como se puede apreciar, nos ha devuelto 1 error y 2 advertencias. El error y la segunda advertencia son problemas de falta de extensiones y librerías para el correcto funcionamiento de PHP. Su solución es sencilla, pues únicamente tendremos que instalar los paquetes correspondientes, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# apt install php-gd php-xml php-mbstring
{% endhighlight %}

Los paquetes ya se encuentran instalados y por tanto el error y la segunda advertencia han sido solucionados. Todavía nos falta la primera advertencia, para poder hacer uso de URLs limpias, ofreciendo así una mejor experiencia al usuario. Un ejemplo de URL limpia sería:

{% highlight shell %}
http://www.example.com/node/83
{% endhighlight %}

Mientras que un ejemplo de una URL sucia sería:

{% highlight shell %}
http://www.example.com/?q=node/83
{% endhighlight %}

Como se puede apreciar, el aspecto visual es muy diferente, por lo que es un problema que se aconseja arreglar. Para ello, tendremos que hacer uso del módulo **rewrite** de _apache2_, así que antes de intentar cargarlo a la ligera, vamos a comprobar si ya se encuentra cargado, mirando dentro del directorio **/etc/apache2/mods-enabled/** y filtrando por dicho nombre:

{% highlight shell %}
root@cms:/srv/drupal# ls /etc/apache2/mods-enabled/ | egrep 'rewrite'
{% endhighlight %}

En este caso, el filtro no ha devuelto ningún resultado, por lo que podemos concluir que dicho módulo no se encuentra cargado en memoria, así que para cargarlo, haremos uso de `a2enmod`:

{% highlight shell %}
root@cms:/srv/drupal# a2enmod rewrite
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2
{% endhighlight %}

Si nos fijamos en los mensajes devueltos, el módulo ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que reiniciar el servicio _apache2_, cosa que no haremos todavía, ya que nos queda otra modificación por realizar, así que lo reiniciaremos al final y así conseguiremos _matar dos pájaros de un tiro_.

Si recordamos, anteriormente mencioné que el fichero **.htaccess** nos iba a ser de utilidad más adelante. Bien, pues ha llegado el momento. Dicho fichero ha sido generado por Drupal y contiene entre otras cosas un conjunto de instrucciones que permiten solucionar el problema de las URLs limpias. Pero si dicho fichero ya existe en el directorio, ¿por qué seguimos teniendo el problema? La respuesta es sencilla, y es que por defecto, la utilización de dicho fichero viene deshabilitada. Para activarla, tendremos que declarar un nuevo **Directory** dentro de nuestro fichero de configuración del VirtualHost en el que habilitemos la directiva **AllowOverride** (podríamos hacerlo también de forma generalizada en _/etc/apache/apache2.conf_, pero no es lo óptimo). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/drupal# nano /etc/apache2/sites-available/drupal.conf
{% endhighlight %}

En este caso, la sección Directory que debemos añadir a dicho fichero afectará únicamente al DocumentRoot en el que se encuentra situado Drupal, quedando de la forma:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.alvaro-drupal.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/drupal

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        <Directory /srv/drupal/>
                AllowOverride All
        </Directory>
</VirtualHost>
{% endhighlight %}

Tras ello, ya podremos reiniciar el servicio _apache2_ para que todos los cambios realizados hasta ahora surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# systemctl restart apache2
{% endhighlight %}

Por último, para verificar que el módulo **rewrite** ha sido correctamente cargado en memoria, volveremos a ejecutar el comando para comprobarlo:

{% highlight shell %}
root@cms:/srv/drupal# ls /etc/apache2/mods-enabled/ | egrep 'rewrite'
rewrite.load
{% endhighlight %}

Como era de esperar, el filtro ahora ha devuelto un resultado referente a dicho módulo, por lo que podemos concluir que se encuentra correctamente cargado en memoria.

Una vez realizadas todas las modificaciones y comprobaciones, volveremos al instalador y refrescaremos la página, de manera que vuelva a realizar la comprobación de los requisitos, por lo que en caso de que nos pida la información de la base de datos, podremos corroborar que hemos llegado al siguiente paso y que por lo tanto, todos los problemas se han solucionado. Este paso es importante, pues dentro de la misma instalará todas las tablas necesarias para el correcto funcionamiento del CMS. En este caso, el nombre de la base de datos es **drupal**, el usuario es **username** y la contraseña es **usuario**:

![drupal4](https://i.ibb.co/WKJr7sM/Captura-de-pantalla-de-2020-10-23-09-49-07.png "Base de datos")

En el caso de que las credenciales introducidas sean incorrectas, nos lo notificará y pedirá unas credenciales válidas, de manera que cuando sean válidas comenzará la instalación de Drupal:

![drupal5](https://i.ibb.co/BGL2J8Q/Captura-de-pantalla-de-2020-10-23-09-49-14.png "Instalación Drupal")

Por último, tendremos que configurar el sitio Drupal para así adaptarlo a nuestras necesidades. Nos pedirá un nombre para el sitio, que en este caso usaré **PruebaDrupal**, una dirección de correo electrónico válida, una cuenta para el mantenimiento del sitio (administrador), algunas opciones regionales, tales como el país y la zona horaria...

Tras llevar a cabo toda la configuración solicitada, nuestro sitio Drupal estará totalmente operativo y funcional:

![drupal6](https://i.ibb.co/0nBpnkz/Captura-de-pantalla-de-2020-10-23-09-51-38.png "Drupal operativo")

Como se puede apreciar, el sitio web es totalmente accesible desde el nombre de dominio previamente configurado (**www.alvaro-drupal.com**).

### Realiza una configuración mínima de la aplicación (cambia la plantilla, crea algún contenido...).

Para cambiar la plantilla de Drupal, tendremos que buscar el apartado **Apariencia** en el menú superior, para seguidamente pulsar en **Instalar nuevo tema**. Una vez ahí, tendremos dos opciones: subir manualmente el fichero que contiene el tema o indicar una URL para que descargue automáticamente el tema. En mi caso, voy a elegir la segunda opción, para descargar [este](https://ftp.drupal.org/files/projects/vani-8.x-1.2.tar.gz) tema. Una vez que lo tengamos todo listo, pulsamos en **Instalar**:

![drupal7](https://i.ibb.co/YyLMKZ8/Captura-de-pantalla-de-2020-10-23-16-40-53.png "Instalación tema")

Cuando el tema se haya descargado, tendremos que pulsar en **Instalar temas añadidos recientemente** y buscar dicho tema dentro de la sección **Temas desinstalados**, para posteriormente pulsar en la opción **Instalar y seleccionar de modo predeterminado**. Tras ello, podremos volver a la página principal y apreciar como el nuevo tema se ha instalado:

![drupal8](https://i.ibb.co/56xgdKJ/Captura-de-pantalla-de-2020-10-23-16-42-33.png "Instalación tema")

El procedimiento para crear un nuevo artículo/_post_ es también muy sencillo. Para ello, tendremos que buscar el apartado **Contenido** en el menú superior, para seguidamente pulsar en **Añadir contenido**. Tendremos que seleccionar que el contenido que queremos añadir es un **Artículo** y simplemente rellenar el contenido del nuevo artículo que queremos crear:

![drupal9](https://i.ibb.co/ftLnYkp/Captura-de-pantalla-de-2020-10-23-16-44-09.png "Nuevo post")

Si volvemos a la página principal, podremos apreciar que el nuevo artículo ha aparecido:

![drupal10](https://i.ibb.co/VjYTDWF/Captura-de-pantalla-de-2020-10-23-16-44-40.png "Nuevo post")

Efectivamente, el nuevo artículo ya es accesible y podemos leer su contenido.

### Instala un módulo para añadir alguna funcionalidad a Drupal.

Para instalar un nuevo módulo en Drupal, tendremos que buscar el apartado **Ampliar** en el menú superior, para seguidamente pulsar en **Instalar nuevo módulo**. Una vez ahí, tendremos dos opciones: subir manualmente el fichero que contiene el módulo o indicar una URL para que descargue automáticamente el módulo. En mi caso, voy a elegir la segunda opción, para descargar [este](https://ftp.drupal.org/files/projects/admin_toolbar-8.x-2.4.tar.gz) módulo, que permite crear un menú desplegable dentro del apartado **Configuración** de la parte superior, haciéndo así mucho más intuitiva y cómoda la gestión del sitio Drupal. Una vez que lo tengamos todo listo, pulsamos en **Instalar**.

Cuando el módulo se haya descargado, tendremos que pulsar en **Activar los módulos agregados recientemente** y buscar dicho módulo dentro de la correspondiente sección a la que pertenezca, para posteriormente pulsar en el botón **Instalar**:

![drupal11](https://i.ibb.co/rsHByNt/Captura-de-pantalla-de-2020-10-23-16-51-34.png "Instalación módulo")

Tras ello, podremos volver a la página principal y apreciar como el nuevo módulo se ha instalado y está totalmente operativo:

![drupal12](https://i.ibb.co/QpnKG0M/Captura-de-pantalla-de-2020-10-23-16-52-08.png "Instalación módulo")

Como se puede ver, el apartado **Configuración** del menú superior ahora muestra un menú desplegable con diversas opciones.

## Tarea 3: Configuración multinodo.

### Realiza una copia de seguridad de la base de datos.

Hasta ahora hemos estado trabajando con una base de datos alojada en la misma máquina que la encargada de servir la página web. Vamos a llevar a cabo un proceso de migración de la base de datos de dicha máquina a la otra que hemos creado.

Para ello, lo primero que tendremos que hacer es una copia de seguridad de la misma, que en el caso de _mysql_, se nos ofrece una funcionalidad que lleva a cabo dicha función, **mysqldump**. Gracias a la misma, se nos generará un fichero que contendrá todas las instrucciones necesarias para dejar la base de datos en cuestión tal y como está actualmente, fichero que posteriormente importaremos en la otra máquina.

En este caso, voy a generar un fichero de nombre **backup-file.sql** dentro del directorio raíz (aunque podríamos hacerlo en cualquier otro directorio, intentando evitar siempre el DocumentRoot de la página web, ya que en ese caso, se podría solicitar ese recurso y sería servido por _apache2_, algo totalmente inseguro). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/drupal# mysqldump drupal > /backup-file.sql
{% endhighlight %}

Para verificar que la copia de seguridad de la base de datos **drupal** se ha generado correctamente, vamos a listar el contenido de dicho directorio (raíz), filtrando por el nombre **backup**:

{% highlight shell %}
root@cms:/srv/drupal# ls / | egrep 'backup'
backup-file.sql
{% endhighlight %}

Efectivamente, la copia de seguridad se ha generado correctamente en dicho directorio.

### Crea otra máquina con vagrant, conectada con una red interna a la anterior y configura un servidor de bases de datos.

Dado que la máquina la habíamos definido y levantado con anterioridad, únicamente tendremos que conectarnos a la misma, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/vagrant/cms$ vagrant ssh backup
Linux backup 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@backup:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86109sec preferred_lft 86109sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b6:ab:0c brd ff:ff:ff:ff:ff:ff
    inet 172.22.100.10/24 brd 172.22.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb6:ab0c/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **172.22.100.10**.

Tras ello, tendremos que instalar los paquetes correspondientes a MySQL/MariaDB, es decir, **mariadb-client** y **mariadb-server**, no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@backup:~# apt update && apt upgrade && apt install mariadb-client mariadb-server
{% endhighlight %}

Tras ello, accederemos al servidor **mysql**, ejecutando para ello el comando visto con anterioridad:

{% highlight shell %}
root@backup:~# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 49
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Necesitaremos una nueva base de datos sobre la que restaurar la copia de seguridad, así que la crearemos, en este caso, de nombre **drupalbackup**:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE drupalbackup;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

Para verificar que la nueva base de datos se ha generado correctamente, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| drupalbackup       |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.001 sec)
{% endhighlight %}

Efectivamente, la base de datos **drupalbackup** ha sido correctamente generada.

En la anterior instalación, creamos un usuario que tenía el acceso permitido desde **localhost**, pero esta vez, no vamos a usar la base de datos de manera local, sino que queremos permitir el acceso desde el exterior. En este caso, crearemos un usuario de nombre **backupuser**, que tenga permitido el acceso desde **172.22.100.2**, es decir, desde la máquina que queremos que se pueda conectar (si quisiéramos permitir el acceso desde cualquier dirección IP, se podría poder **%**), y cuya contraseña será **usuario**.

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'backupuser'@'172.22.100.2' IDENTIFIED BY 'usuario';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Tras ello, tendremos que otorgarle los privilegios necesarios sobre la base de datos que acabamos de crear, ejecutando para ello el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON drupalbackup.* to 'backupuser'@'172.22.100.2';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

Esto no es todo, ya que por defecto, _mysql_ viene configurado para escuchar únicamente peticiones desde localhost, por lo que no escucharía aquellas peticiones que vengan del exterior. Para modificar esto, tenemos que editar el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf**, haciendo uso de `nano`:

{% highlight shell %}
root@backup:~# nano /etc/mysql/mariadb.conf.d/50-server.cnf
{% endhighlight %}

Dentro del mismo, encontraremos una línea con la siguiente forma:

{% highlight shell %}
bind-address = 127.0.0.1
{% endhighlight %}

Como se puede apreciar, está únicamente configurado para escuchar peticiones desde **127.0.0.1**, por lo que tendremos que realizar el correspondiente cambio para que las escuche desde **0.0.0.0** (es decir, desde todas las interfaces IPv4), quedando de la siguiente manera:

{% highlight shell %}
bind-address = 0.0.0.0
{% endhighlight %}

Tras ello, guardaremos los cambios, y como dice la _ley del informático_, cada vez que toquemos un fichero de configuración, tendremos que reiniciar el correspondiente servicio, ejecutando en este caso el comando:

{% highlight shell %}
root@backup:~# systemctl restart mariadb
{% endhighlight %}

Ya está todo listo para realizar la restauración de la base de datos y su correspondiente conexión desde el exterior, que en un principio debería de funcionar, siempre y cuando no exista un cortafuegos bloqueando la conexión (en este caso no hay problema, pues me he asegurado previamente).

### Restaura la copia de seguridad en el nuevo servidor de bases de datos.

Ha llegado la hora de pasar la copia de seguridad de una máquina a otra, es por ello que tendremos que asignarle una contraseña al usuario **vagrant** de la máquina a la que vamos a pasar la copia de seguridad, haciendo uso del comando `passwd`:

{% highlight shell %}
root@backup:~# passwd vagrant
New password: 
Retype new password: 
passwd: password updated successfully
{% endhighlight %}

Las máquinas vagrant tienen una pequeña "limitación", y es que únicamente aceptan recibir conexiones SSH que utilicen la clave privada como método de autentificación, es por ello que si intentamos conectarnos de una máquina a otra, obtendremos el siguiente mensaje:

{% highlight shell %}
root@cms:/srv/drupal# ssh vagrant@172.22.100.10
vagrant@172.22.100.10: Permission denied (publickey).
{% endhighlight %}

Esto nos dificulta un poco las cosas, y es que dado que necesitamos pasar la copia de seguridad de la base de datos de la máquina **cms** a la máquina **backup**, tenemos que buscar la forma de hacerlo. Una opción sería pasarla de la máquina _cms_ a la anfitriona y posteriormente a la _backup_, pero es un proceso que es innecesario hacer. La solución se encuentra en desactivar esta característica en el fichero de configuración SSH de la máquina vagrant que va a recibir la conexión (**/etc/ssh/sshd_config**), haciendo uso de `nano`:

{% highlight shell %}
root@backup:~# nano /etc/ssh/sshd_config
{% endhighlight %}

Dentro del mismo, encontraremos una línea con la siguiente forma:

{% highlight shell %}
PasswordAuthentication no
{% endhighlight %}

Tendremos que cambiar el valor de dicha directiva a **yes**, para así habilitar la autentificación con contraseña:

{% highlight shell %}
PasswordAuthentication yes
{% endhighlight %}

De nuevo, dado que hemos tocado el fichero de configuración de un servicio, en este caso _ssh_, tendremos que reiniciarlo:

{% highlight shell %}
root@backup:~# systemctl restart ssh
{% endhighlight %}

Tras ello, ya podremos realizar la conexión SSH haciendo uso de `scp` para llevar a cabo la transferencia del fichero en cuestión:

{% highlight shell %}
root@cms:/srv/drupal# scp /backup-file.sql vagrant@172.22.100.10:
vagrant@172.22.100.10's password: 
backup-file.sql                          100% 8169KB  77.8MB/s   00:00
{% endhighlight %}

Para verificar que el fichero ha sido correctamente transferido, volveremos a la máquina en la que hemos configurado la nueva base de datos y listaremos el contenido del directorio personal haciendo uso de `ls`:

{% highlight shell %}
root@backup:~# ls /home/vagrant/
backup-file.sql
{% endhighlight %}

Efectivamente, el fichero ha sido correctamente transferido, por lo que ya podremos importarlo a la base de datos creada con anterioridad. Para ello, únicamente inyectaremos el contenido del fichero a _mysql_, indicando la base de datos que deseamos recuperar, quedando de la siguiente forma:

{% highlight shell %}
root@backup:~# mysql drupalbackup < /home/vagrant/backup-file.sql
{% endhighlight %}

De nuevo, accederemos al servicio _mysql_ para verificar que la base de datos ha sido correctamente restaurada:

{% highlight shell %}
root@backup:~# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 55
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Tras ello, nos conectaremos a la base de datos en cuestión, con la instrucción `use`:

{% highlight shell %}
MariaDB [(none)]> use drupalbackup;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Como podemos apreciar en el último mensaje, nos hemos conectado a dicha base de datos, así que listaremos las tablas de la misma con la siguiente instrucción:

{% highlight shell %}
MariaDB [drupalbackup]> show tables;
+----------------------------------+
| Tables_in_drupalbackup           |
+----------------------------------+
| batch                            |
| block_content                    |
| ...                              |
| users_field_data                 |
| watchdog                         |
+----------------------------------+
76 rows in set (0.001 sec)
{% endhighlight %}

Efectivamente, la base de datos ha sido correctamente restaurada, pues contiene las 76 tablas que habían sido anteriormente generadas durante la instalación de Drupal.

### Desinstala el servidor de bases de datos en el servidor principal.

Para desinstalar el servidor de bases de datos haremos uso de `apt purge`, que a diferencia de `apt remove`, elimina además del paquete, sus correspondientes ficheros de configuración (entre los cuales se encuentra la base de datos), ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# apt purge mariadb-client-10.3 mariadb-server-10.3
{% endhighlight %}

Durante la desinstalación nos preguntará si estamos seguros de que se elimine el directorio **/var/lib/mysql** (que contiene las bases de datos). Seleccionaremos que sí:

![drupal13](https://i.ibb.co/vQv0w05/Captura-de-pantalla-de-2020-10-23-17-29-13.png "Desinstalación BBDD")

Tras ello, volveremos al navegador y trataremos de acceder al sitio web Drupal (**www.alvaro-drupal.com**):

![drupal14](https://i.ibb.co/6v0PQKj/Captura-de-pantalla-de-2020-10-23-17-29-28.png "Desinstalación BBDD")

Como era de esperar, el sitio web no se encuentra actualmente operativo ya que hemos eliminado la base de datos a la que hacía las peticiones.

### Realiza los cambios de configuración necesarios en Drupal para que la página funcione.

En este caso, la web no funciona ya que la base de datos a la que hace las peticiones no se encuentra operativa, pues la hemos migrado a otra máquina, pero esto no se lo hemos notificado al servidor, por lo que tendremos que llevar a cabo las correspondientes modificaciones en el fichero oportuno para indicarle la nueva base de datos a la que ha de realizar las peticiones. Este fichero se encuentra en **/srv/drupal/sites/default/**, con nombre **settings.php**, que modificaremos con `nano`:

{% highlight shell %}
root@cms:/srv/drupal# nano sites/default/settings.php
{% endhighlight %}

Dentro del mismo, nos desplazaremos al final del documento y encontraremos un bloque como el siguiente:

{% highlight shell %}
$databases['default']['default'] = array (
  'database' => 'drupal',
  'username' => 'username',
  'password' => 'usuario',
  'prefix' => '',
  'host' => 'localhost',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
);
{% endhighlight %}

Como se puede apreciar, ahí se encuentran las credenciales de acceso y la base de datos a la que acceder. En este caso, el nombre de la base de datos ha cambiado de **drupal** a **drupalbackup**, el usuario ha cambiado de **username** a **backupuser** y la contraseña se ha mantenido. Nos queda una última modificación por hacer, y es que la base de datos ya no se encuentra en **localhost**, sino en **172.22.100.10**. El puerto en el que escucha las peticiones no ha variado, pues sigue siendo el **3306**. La configuración final quedaría así:

{% highlight shell %}
$databases['default']['default'] = array (
  'database' => 'drupalbackup',
  'username' => 'backupuser',
  'password' => 'usuario',
  'prefix' => '',
  'host' => '172.22.100.10',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
);
{% endhighlight %}

Tras ello, guardaremos los cambios, y una vez más, trataremos de acceder al sitio Drupal, para ver si el fallo ha sido solucionado:

![drupal15](https://i.ibb.co/GH5cRmV/Captura-de-pantalla-de-2020-10-23-17-56-55.png "Recuperación BBDD")

Efectivamente, el sitio web ya ha vuelto a estar disponible ya que tiene acceso a la base de datos recuperada.

## Tarea 4: Instalación de otro CMS PHP.

### Configura otro VirtualHost y elige otro nombre de dominio.

Dado que este apartado consiste en repetir exactamente el mismo proceso anteriormente realizado, voy a hacerlo sin tanto detalle. En este caso, el nuevo VirtualHost lo voy a alojar dentro de **/srv/**, en un directorio de nombre **moodle/**, por lo que procederé a la creación de dicho directorio:

{% highlight shell %}
root@cms:/srv/drupal# mkdir /srv/moodle
{% endhighlight %}

Una vez generado, tendremos que crear el fichero de configuración para dicho VirtualHost, así que volveré a copiar la plantilla del VirtualHost por defecto de _apache2_:

{% highlight shell %}
root@cms:/srv/drupal# cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/moodle.conf
{% endhighlight %}

Tras ello, lo modificaremos haciendo uso de `nano` y estableceremos el DocumentRoot (**/srv/moodle**) y el ServerName (**www.alvaro-moodle.com**, por ejemplo):

{% highlight shell %}
root@cms:/srv/drupal# nano /etc/apache2/sites-available/moodle.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.alvaro-moodle.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/moodle

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Una vez realizada la correspondiente configuración en el fichero, tendremos que habilitar el nuevo VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# a2ensite moodle
Enabling site moodle.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Tal y como se nos ha solicitado, vamos a volver a cargar la configuración del servicio _apache2_, haciendo uso del comando:

{% highlight shell %}
root@cms:/srv/drupal# systemctl reload apache2
{% endhighlight %}

Únicamente nos falta un paso, añadir la correspondiente línea al fichero **/etc/hosts** de la máquina anfitriona para que lleve a cabo la resolución estatica de nombres. Para ello, ejecutamos el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Tras añadir la correspondiente línea, el resultado final es el siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.50.2    www.alvaro-drupal.com
192.168.50.2    www.alvaro-moodle.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

La máquina anfitriona ya es capaz de realizar la resolución del nombre **www.alvaro-moodle.com**.

### Elige otro CMS realizado en PHP y realiza la instalación en tu infraestructura.

Como se ha podido deducir, el CMS que vamos a instalar es **Moodle**, así que dado que vamos a necesitar otra base de datos, vamos a volver a la máquina _backup_ para crearla, con nombre **moodle**, por ejemplo:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE moodle;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

La nueva base de datos ya se encuentra creada, pero tenemos que darle permisos al usuario **backupuser** para que pueda acceder a la misma desde la máquina donde estará alojada la Moodle. Para ello, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON moodle.* to 'backupuser'@'172.22.100.2';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Listo, ya hemos finalizado la configuración de la base de datos, así que podremos volver a la máquina donde vamos a descargar e instalar el CMS.

En este caso, vamos a utilizar la última versión de Moodle (3.9.2), que podremos descargar desde la [página oficial](https://moodle.org/), concretamente desde [aquí](https://download.moodle.org/download.php/direct/stable39/moodle-3.9.2.tgz). Para ello, primero nos moveremos al DocumentRoot donde instalaremos Moodle, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# cd /srv/moodle/
{% endhighlight %}

Una vez dentro del mismo, haremos uso de `wget` para descargar el paquete comprimido de Moodle:

{% highlight shell %}
root@cms:/srv/moodle# wget https://download.moodle.org/download.php/direct/stable39/moodle-3.9.2.tgz
--2020-10-23 16:06:47--  https://download.moodle.org/download.php/direct/stable39/moodle-3.9.2.tgz
Resolving download.moodle.org (download.moodle.org)... 172.67.26.233, 104.22.65.81, 104.22.64.81, ...
Connecting to download.moodle.org (download.moodle.org)|172.67.26.233|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 57013075 (54M) [application/g-zip]
Saving to: ‘moodle-3.9.2.tgz’

moodle-3.9.2.tgz                100%[==========================>]  54.37M  11.8MB/s    in 4.6s    

2020-10-23 16:06:52 (11.8 MB/s) - ‘moodle-3.9.2.tgz’ saved [57013075/57013075]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`:

{% highlight shell %}
root@cms:/srv/moodle# ls -l
total 55680
-rw-r--r-- 1 root root 57013075 Sep 11 02:41 moodle-3.9.2.tgz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**moodle-3.9.2.tgz**" con un peso total de **54.37 MB** (57013075 bytes).

Al estar comprimido el fichero, no podemos llevar a cabo la instalación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/moodle# tar -zxf moodle-3.9.2.tgz --strip 1 && rm -r moodle-3.9.2.tgz
{% endhighlight %}

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
root@cms:/srv/moodle# ls -l
total 956
drwxr-xr-x 13 1005 1005   4096 Sep 11 02:41 admin
drwxr-xr-x  5 1005 1005   4096 Sep 11 02:41 analytics
drwxr-xr-x 17 1005 1005   4096 Sep 11 02:41 auth
drwxr-xr-x  6 1005 1005   4096 Sep 11 02:41 availability
-rw-r--r--  1 1005 1005   7380 Sep 11 02:41 babel-plugin-add-module-to-define.js
drwxr-xr-x  8 1005 1005   4096 Sep 11 02:41 backup
drwxr-xr-x  8 1005 1005   4096 Sep 11 02:41 badges
-rw-r--r--  1 1005 1005    302 Sep 11 02:41 behat.yml.dist
...
drwxr-xr-x  4 1005 1005   4096 Sep 11 02:41 theme
-rw-r--r--  1 1005 1005   1600 Sep 11 02:41 tokenpluginfile.php
-rw-r--r--  1 1005 1005   2191 Sep 11 02:41 TRADEMARK.txt
drwxr-xr-x  9 1005 1005   4096 Sep 11 02:41 user
drwxr-xr-x  2 1005 1005   4096 Sep 11 02:41 userpix
-rw-r--r--  1 1005 1005   1636 Sep 11 02:41 version.php
drwxr-xr-x  7 1005 1005   4096 Sep 11 02:41 webservice
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos (en lugar de descomprimir un directorio de nombre **moodle** del que posteriormente tendríamos que mover los ficheros contenidos al directorio actual).

Ya tenemos todos los ficheros necesarios para llevar a cabo la instalación de Moodle, pero hay un pequeño detalle que todavía no hemos contemplado, pues el usuario creador y el grupo correspondiente a dichos ficheros y directorios es **1005**, por lo que al igual que en el caso anterior, procedemos a cambiar dicho propietario y grupo de forma recursiva a **www-data**, haciendo para ello uso del comando `chown -R`, pues de otro modo no podría escribir en dichos ficheros durante la instalación:

{% highlight shell %}
root@cms:/srv/moodle# chown -R www-data:www-data /srv/*
{% endhighlight %}

Para verificar que el cambio se ha llevado a cabo correctamente, volveremos a listar el contenido de **/srv**, haciendo uso de `ls -l`:

{% highlight shell %}
root@cms:/srv/moodle# ls -l /srv/
total 8
drwxr-xr-x  8 www-data www-data 4096 Oct 23 14:34 drupal
drwxr-xr-x 56 www-data www-data 4096 Oct 23 16:10 moodle
{% endhighlight %}

Como era de esperar, el usuario y el grupo propietario del directorio **moodle/** se ha visto modificado, al igual que todos sus ficheros y directorios contenidos, al haberlo hecho de forma recursiva.

Ya está todo listo para llevar a cabo la instalación, así que volveremos a acceder al navegador de la máquina anfitriona y escribiremos el nombre de dominio previamente configurado, **www.alvaro-moodle.com**:

![moodle1](https://i.ibb.co/nccxyT8/Captura-de-pantalla-de-2020-10-23-18-39-42.png "Elección de idioma")

Lo primero que preguntará es el idioma, en este caso elegiré el **Español** y pulsaré en **Siguiente**:

![moodle2](https://i.ibb.co/8DmWmcS/Captura-de-pantalla-de-2020-10-23-18-39-48.png "Problemas de requisitos")

Como se puede apreciar, nos ha devuelto 2 errores de falta de extensiones. Su solución es sencilla, pues únicamente tendremos que instalar los paquetes correspondientes, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# apt install php-curl php-zip
{% endhighlight %}

Tras ello, tendremos que reiniciar el servicio _apache2_ para que los cambios realizados surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# systemctl restart apache2
{% endhighlight %}

Una vez realizada la instalación de las extensiones y el correspondiente reinicio del servicio, volveremos al instalador y refrescaremos la página.

El siguiente paso nos pedirá indicar la ruta del directorio de datos, que usará Moodle para guardar los archivos subidos (debe encontrarse fuera del DocumentRoot, para que no sea accesible). En mi caso, la crearé dentro de **/srv/**, con el nombre de **moodledata/**, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# mkdir /srv/moodledata
{% endhighlight %}

Este directorio requiere tener a **www-data** como propietario y grupo, así que haremos dicha modificación ejecutando el comando:

{% highlight shell %}
root@cms:/srv/moodle# chown www-data:www-data /srv/moodledata/
{% endhighlight %}

![moodle3](https://i.ibb.co/K72txcS/Captura-de-pantalla-de-2020-10-23-18-42-56.png "Confirmación rutas")

El siguiente paso será elegir el controlador de la base de datos, que en este caso será MariaDB:

![moodle4](https://i.ibb.co/SQrX36R/Captura-de-pantalla-de-2020-10-23-18-45-30.png "Controlador BBDD")

Hemos llegado a la configuración de la base de datos. Este paso es importante, pues dentro de la misma instalará todas las tablas necesarias para el correcto funcionamiento del CMS. En este caso, la dirección de la base de datos es **172.22.100.10**, el nombre de la base de datos es **moodle**, el usuario es **backupuser** y la contraseña es **usuario**. El prefijo de las tablas no es algo relevante en este caso, y el puerto que usaremos será **3306**:

![moodle5](https://i.ibb.co/f1WvKXW/Captura-de-pantalla-de-2020-10-23-18-46-57.png "Base de datos")

Tras ello, tendremos que aceptar los términos y condiciones y continuar.

![moodle6](https://i.ibb.co/CJLZHS2/Captura-de-pantalla-de-2020-10-23-18-47-22.png "Problemas de requisitos")

En este paso se me han mostrado nuevamente algunos problemas de extensiones, cuya solución es sencilla, pues únicamente tendremos que instalar los paquetes correspondientes, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# apt install php-intl php-xmlrpc php-soap
{% endhighlight %}

Tras ello, tendremos que reiniciar el servicio _apache2_ para que los cambios realizados surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# systemctl restart apache2
{% endhighlight %}

Por último, tendremos que configurar el sitio Moodle para así adaptarlo a nuestras necesidades. Nos pedirá un nombre para el sitio, que en este caso usaré **PruebaMoodle**, una dirección de correo electrónico válida, una cuenta para el mantenimiento del sitio (administrador), algunas opciones regionales, tales como el país y la zona horaria...

Tras llevar a cabo toda la configuración solicitada, nuestro sitio Moodle estará totalmente operativo y funcional:

![moodle7](https://i.ibb.co/yNh9DVF/Captura-de-pantalla-de-2020-10-23-18-53-42.png "Moodle operativo")

Como se puede apreciar, el sitio web es totalmente accesible desde el nombre de dominio previamente configurado (**www.alvaro-moodle.com**). Además, para probar el correcto funcionamiento del CMS, he creado un nuevo curso:

![moodle8](https://i.ibb.co/nkRnM2g/Captura-de-pantalla-de-2020-10-23-18-54-52.png "Curso creado")

Como se puede apreciar, el nuevo CMS instalado funciona correctamente.

## Tarea 5: Necesidad de otros servicios.

### Instala un servidor de correo electrónico en tu servidor. Debes configurar un servidor relay de correo.

En este caso, vamos a instalar un servidor de correo electrónico **postfix**, junto a **mailutils**, un paquete que proporciona una especie de envoltura para enviar correos de una manera mucho más intuitiva. Para llevar a cabo la instalación, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/moodle# apt install postfix mailutils
{% endhighlight %}

Durante la instalación de _postfix_, se nos preguntará la configuración de uso, que en este caso, debemos elegir **Satellite System**, ya que nuestro propósito es hacer que todos los correos sean reenviados a otra máquina, para su posterior entrega:

![postfix1](https://i.ibb.co/BG6Kkp9/Captura-de-pantalla-de-2020-10-24-09-52-17.png "Selección configuración")

El siguiente paso será elegir un **mail name**, es decir, un nombre que identifique a la máquina de forma única dentro del dominio (FQDN). En este caso, dado que únicamente vamos a usar ésta máquina para enviar correos, no es algo de lo que nos debamos preocupar, por lo que en mi caso, introduje "**cms**":

![postfix2](https://i.ibb.co/3FtK4RH/Captura-de-pantalla-de-2020-10-24-09-52-46.png "mail name")

Por último, tendremos que introducir la dirección de la máquina a la que vamos a reenviar los correos. En la tarea se pedía que la máquina fuese **babuino-smtp.gonzalonazareno.org**, pero dado que ésta máquina no es accesible desde el exterior, he decidido hacer uso de [SendGrid](https://sendgrid.com/), que nos proporciona dicha funcionalidad a través de una API. En este caso, la dirección que he tenido que introducir es:

![postfix3](https://i.ibb.co/60K4Jg9/Captura-de-pantalla-de-2020-10-24-10-10-17.png "SendGrid")

Tras ello, y debido al modo de funcionamiento de SendGrid, tuve que realizar algunas modificaciones extra en el fichero de configuración de _postfix_ (**/etc/postfix/main.cf**), ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# nano /etc/postfix/main.cf
{% endhighlight %}

Dentro del mismo, tuve que añadir las siguientes líneas:

{% highlight shell %}
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_tls_security_level = encrypt
header_size_limit = 4096000
relayhost = [smtp.sendgrid.net]:587
{% endhighlight %}

No vamos a entrar con mucho detalle a explicar dichas líneas, pero están relacionadas con la autorización (que usará una APIKey), con la encriptación del mensaje, el tamaño máximo de las cabeceras...

Como nos hemos fijado en las líneas anteriormente introducidas, se hace referencia a un fichero, **/etc/postfix/sasl_passwd**, en el que debemos introducir nuestras credenciales para poder usar el servicio de SendGrid. Lo que hacen, es proporcionarnos una APIKey que tendremos que configurar dentro de dicho fichero, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# nano /etc/postfix/sasl_passwd
{% endhighlight %}

Dentro del mismo, tendremos que introducir líneas de la siguiente forma:

{% highlight shell %}
<máquinarelay> <usuario>:<contraseña>
{% endhighlight %}

En este caso, tras leer la documentación de SendGrid, el usuario que debemos introducir es **apikey**, y la contraseña es la APIKey en cuestión (que en este caso estará censurada por razones obvias), quedando de la siguiente manera:

{% highlight shell %}
[smtp.sendgrid.net]:587 apikey:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
{% endhighlight %}

Por último, tendremos que cambiarle los permisos a dicho fichero para que únicamente el propietario tenga permisos de lectura y escritura (es decir, **600**), haciendo uso de `chmod`:

{% highlight shell %}
root@cms:/srv/moodle# chmod 600 /etc/postfix/sasl_passwd
{% endhighlight %}

Tras ello, tendremos que mapear las tablas de _postfix_ para que utilicen dicho fichero, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/moodle# postmap /etc/postfix/sasl_passwd
{% endhighlight %}

Listo, toda la configuración del servidor de correo ha sido realizada, únicamente nos queda reiniciar el servicio para que tome la nueva configuración, haciendo uso del comando:

{% highlight shell %}
root@cms:/srv/moodle# systemctl restart postfix
{% endhighlight %}

Es hora de hacer una pequeña prueba para comprobar que la API funciona correctamente. Si recordamos, hemos instalado **mailutils**, que nos va a proporcionar una envoltura que permite enviar correos electrónicos de una manera más sencilla, por ejemplo, inyectándole un flujo con `echo`:

{% highlight shell %}
root@cms:/srv/moodle# echo "Esto es una prueba de postfix." | mail -s "Mensaje Postfix" -A /home/vagrant/prueba.txt -a "From: avacaferreras@gmail.com" pruebapostfix@yopmail.com
{% endhighlight %}

Donde:

* **-s**: Indica el asunto del correo electrónico (Subject).
* **-A**: Indicamos la ruta de un fichero que queramos adjuntar al mismo.
* **-a**: Permite añadir valores al header del mensaje, como por ejemplo, el remitente (previamente ha tenido que ser registrado en SendGrid).

Una vez enviado, vamos a consultar los logs de _postfix_ para verificar que se ha enviado correctamente, pero filtrando por **yopmail**, ya que es una cadena contenida en el correo electrónico del destinatario. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/moodle# cat /var/log/mail.log | egrep 'yopmail'
Oct 24 08:48:21 cms postfix/smtp[4196]: 98BDDC2C63: to=<pruebapostfix@yopmail.com>, relay=smtp.sendgrid.net[159.122.219.43]:587, delay=0.85, delays=0.03/0.01/0.48/0.34, dsn=2.0.0, status=sent (250 Ok: queued as XXXXXXXXXXXXXXX)
{% endhighlight %}

Efectivamente, como se puede apreciar, el **status** del correo es **sent**, y ha devuelto un mensaje de **Ok**, por lo que podemos ir a la bandeja de entrada del correo electrónico destinario para verificarlo. En este caso, se trata de un correo electrónico temporal, así que este es el resultado:

![postfix4](https://i.ibb.co/bmgWrZQ/Captura-de-pantalla-de-2020-10-24-10-48-38.png "Prueba SendGrid")

Como se puede apreciar, el correo electrónico ha sido correctamente recibido, junto al fichero vacío de prueba.

### Configura alguno de los CMS para utilizar tu servidor de correo y realiza una prueba de funcionamiento.

En este caso, he decidido configurar el sitio Drupal para que utilice el servidor de correo, así que lo primero que haré será moverme al DocumentRoot de dicho VirtualHost, haciendo uso de `cd`:

{% highlight shell %}
root@cms:/srv/moodle# cd /srv/drupal/
{% endhighlight %}

Tras ello, volveremos a nuestro sitio web Drupal, ya que requiere que instalemos un módulo, por lo que seguiremos el procedimiento previamente visto, es decir, tendremos que buscar el apartado **Ampliar** en el menú superior, para seguidamente pulsar en **Instalar nuevo módulo**. Una vez ahí, tendremos que indicar una URL para que descargue automáticamente el [módulo](https://ftp.drupal.org/files/projects/smtp-8.x-1.0.tar.gz).

Cuando el módulo se haya descargado, tendremos que pulsar en **Activar los módulos agregados recientemente** y buscar dicho módulo dentro de la correspondiente sección a la que pertenezca, para posteriormente pulsar en el botón **Instalar**:

![postfix5](https://i.ibb.co/3z6z014/Captura-de-pantalla-de-2020-10-24-10-57-13.png "Instalar módulo")

El módulo ya se encuentra instalado, pero dicho módulo hace uso por debajo de un paquete llamado **phpmailer**, cuya versión en la paquetería de Debian es inferior a la requerida, por lo que tendremos que utilizar un sistema alternativo para su instalación. Para ello, instalaremos **composer**, un manejador de dependencias PHP que debemos instalar, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# apt install composer
{% endhighlight %}

Ya tenemos el manejador de dependencias instalado, pero todavía tenemos que instalar la dependencia en cuestión, haciendo uso del comando:

{% highlight shell %}
root@cms:/srv/drupal# composer require phpmailer/phpmailer
Using version ^6.1 for phpmailer/phpmailer
./composer.json has been updated
Loading composer repositories with package information
Updating dependencies (including require-dev)
Package operations: 1 install, 0 updates, 0 removals
  - Installing phpmailer/phpmailer (v6.1.8): Loading from cache
phpmailer/phpmailer suggests installing league/oauth2-google (Needed for Google XOAUTH2 authentication)
phpmailer/phpmailer suggests installing hayageek/oauth2-yahoo (Needed for Yahoo XOAUTH2 authentication)
phpmailer/phpmailer suggests installing stevenmaguire/oauth2-microsoft (Needed for Microsoft XOAUTH2 authentication)
Writing lock file
Generating autoload files
Hardening vendor directory with .htaccess and web.config files.
Cleaning vendor directory.
{% endhighlight %}

Listo, toda la paquetería necesaria está instalada, así que volveremos a Drupal y accederemos a la configuración del módulo previamente instalado (para ello, volvemos a pulsar en **Ampliar** y volvemos a buscar dicho módulo, para posteriormente pulsar en el desplegable y en **Configurar**).

Dentro de la configuración, tendremos que marcar la opción **Activado** para que use SMTP como el sistema de correo por defecto, además de configurar el servidor SMTP al que vamos a acceder, que en este caso se encuentra alojado en **localhost**, escuchando peticiones en el puerto **25**, además de desactivar la encriptación y el TLS. El resto de opciones lo podemos dejar por defecto:

![postfix6](https://i.ibb.co/KhXjZ7J/Captura-de-pantalla-de-2020-10-24-11-43-10.png "Configuración SMTP")

Por último, guardaremos los cambios y en la parte inferior de la misma página, encontraremos un apartado que nos permite llevar a cabo un envío de un correo electrónico de prueba, por lo que simplemente introduciremos el email del destinatario y continuaremos:

![postfix7](https://i.ibb.co/YNDqjX8/Captura-de-pantalla-de-2020-10-24-11-44-41.png "Email de prueba")

Una vez más, volveremos a mirar los logs del servicio _postfix_ para verificar que no ha habido ningún error durante el envío:

{% highlight shell %}
root@cms:/srv/drupal# cat /var/log/mail.log | egrep 'pruebadrupal'
Oct 24 09:44:29 cms postfix/smtp[1153]: 2F636C2D7A: to=<pruebadrupal@yopmail.com>, relay=smtp.sendgrid.net[159.122.219.55]:587, delay=0.55, delays=0.05/0/0.42/0.08, dsn=2.0.0, status=sent (250 Ok: queued as XXXXXXXXXXXXXXX)
{% endhighlight %}

Efectivamente, como se puede apreciar, el **status** del correo es **sent**, y ha devuelto un mensaje de **Ok**, por lo que podemos ir a la bandeja de entrada del correo electrónico destinario para verificarlo, así que este es el resultado:

![postfix8](https://i.ibb.co/59MccdQ/Captura-de-pantalla-de-2020-10-24-11-44-51.png "Prueba SendGrid")

Como se puede apreciar, el correo electrónico ha sido correctamente recibido por parte del CMS, así que podemos corroborar que la configuración del servidor de correo en el CMS ha funcionado.