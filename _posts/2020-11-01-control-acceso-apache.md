---
layout: post
title:  "Control de acceso, autentificación y autorización en Apache"
banner: "/assets/images/banners/controlacceso.jpg"
date:   2020-11-01 11:03:00 +0200
categories: servicios
---
Se recomienda realizar una previa lectura de los siguientes _posts_, pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad:

* [VirtualHosting con Apache](http://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html)
* [Mapear URL en Apache](http://www.alvarovf.com/servicios/2020/10/20/mapear-url-en-apache.html)

## Crea un escenario en Vagrant que tenga un servidor con una red publica y una privada, y un cliente conectada a la red privada. Crea un host virtual departamentos.iesgn.org.

La primera de ellas deberá tener una interfaz de red en modo bridge (de manera que tenga conexión directa con la máquina anfitriona y podamos acceder a la misma desde el navegador, utilizando una dirección IP del rango configurado por mi servidor DHCP doméstico, es decir, mi router) y una dirección dentro de una red interna, estando a su vez dentro de dicha red la segunda máquina (no es necesario que la segunda máquina tenga una interfaz bridge, pues no necesitamos acceder desde la máquina anfitriona). En este caso, el fichero de configuración **Vagrantfile** tiene la siguiente forma:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define :servidor do |servidor| #Definimos la primera máquina, en este caso, el servidor.
    servidor.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    servidor.vm.hostname = "servidor" #Establecemos el nombre (hostname) de la máquina. En este caso, "servidor".
    servidor.vm.network :public_network,:bridge=>"enp3s0" #Creamos una interfaz de red conectada en modo puente (bridge) a nuestra interfaz de red física "enp3s0", para que obtenga dirección por DHCP en nuestra red doméstica.
    servidor.vm.network :private_network, ip: "192.168.50.2", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor y nos conviene que la IP sea siempre la misma.
      virtualbox__intnet: "lan1" #Dado que VirtualBox crea por defecto las interfaces en modo host-only, tenemos que indicar que utilice una red interna, en este caso, una de nombre "lan1".
  end
  config.vm.define :cliente do |cliente| #Definimos la segunda máquina, en este caso, el cliente.
    cliente.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    cliente.vm.hostname = "cliente" #Establecemos el nombre (hostname) de la máquina. En este caso, "cliente".
    cliente.vm.network :private_network, ip: "192.168.50.10", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático.
      virtualbox__intnet: "lan1" #Al igual que la segunda interfaz de red del servidor, ésta también la conectaremos a la red interna de nombre "lan1".
  end
end
{% endhighlight %}

Tras ello, levantaremos el escenario vagrant haciendo uso de la instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/apache$ vagrant up
{% endhighlight %}

A continuación comenzará a descargar el **box** (en caso de que no lo tuviésemos previamente) y a generar las máquinas virtuales con los parámetros que les hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la primera de ellas ejecutando la siguiente instrucción (pues todavía no es necesario acceder a la máquina cliente):

{% highlight shell %}
alvaro@debian:~/vagrant/apache$ vagrant ssh servidor
Linux servidor 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@servidor:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86290sec preferred_lft 86290sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:d8:e3:d9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.137/24 brd 192.168.1.255 scope global dynamic eth1
       valid_lft 86296sec preferred_lft 86296sec
    inet6 fe80::a00:27ff:fed8:e3d9/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:85:b3:60 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.2/24 brd 192.168.50.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe85:b360/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **tres** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada en modo puente a nuestra interfaz física **enp3s0**, con dirección IP obtenida por DHCP **192.168.1.137**.
* **eth2:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **192.168.50.2**.

Una vez que se ha llevado a cabo la comprobación, procedemos a configurar el servidor web _apache2_. Para ello, lo primero que haremos será instalar el paquete necesario (**apache2**), no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@servidor:~# apt update && apt upgrade && apt install apache2
{% endhighlight %}

Para configurar un nuevo VirtualHost de _apache2_, debemos modificar el fichero de configuración **/etc/apache2/apache2.conf**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# nano /etc/apache2/apache2.conf
{% endhighlight %}

Dentro del mismo, tendremos añadir una sección **Directory** en el que especificaremos el directorio padre dentro del cuál alojaremos los diferentes DocumentRoot, pues dichos DocumentRoot heredarán los permisos del mismo. En este caso, he decidido que dicho directorio padre sea **/srv/**, por lo que le he configurado algunas directivas estándar, quedando de la siguiente forma:

{% highlight shell %}
<Directory /srv/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

El directorio padre donde alojaremos los diferentes DocumentRoot ya se encuentra creado, pero todavía tenemos que generar dicho DocumentRoot, pues en este caso, únicamente necesitamos uno. Para ello, voy a generar un subdirectorio dentro de **/srv/** de nombre **departamentos/**, que será el DocumentRoot del VirtualHost que configuraremos a continuación, pero la configuración no acaba aquí, pues más adelante necesitaremos 3 subdirectorios dentro de **departamentos/** (**intranet/**, **internet/** y **secreto/**), así que generaré dichos directorios a su vez para así _matar dos pájaros de un tiro_, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# mkdir -p /srv/departamentos/{intranet,internet,secreto}
{% endhighlight %}

Donde:

* **-p**: Creará también los directorios padre en caso de que no existan (en este caso, creará _departamentos/_).

Para verificar que los directorios se han generado correctamente, listaremos de forma recursiva el contenido de _/srv_, haciendo uso del comando `ls -R`:

{% highlight shell %}
root@servidor:~# ls -R /srv/
/srv/:
departamentos

/srv/departamentos:
internet  intranet  secreto

/srv/departamentos/internet:

/srv/departamentos/intranet:

/srv/departamentos/secreto:

{% endhighlight %}

Efectivamente, todos los directorios se han generado correctamente. Tras ello, únicamente nos queda crear el nuevo fichero de configuración dentro de **/etc/apache2/sites-available/**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@servidor:~# cd /etc/apache2/sites-available/
{% endhighlight %}

Una vez dentro del mismo, vamos a utilizar como plantilla el fichero de configuración del VirtualHost que trae _apache2_ configurado por defecto, **000-default.conf**, así que copiaremos dicho fichero a uno de nombre **departamentos.conf** (por ejemplo, aunque se podría haber usado cualquier otro nombre). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# cp 000-default.conf departamentos.conf
{% endhighlight %}

Para verificar que el nuevo fichero de configuración se ha generado correctamente, volveremos a ejecutar el comando `ls`:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# ls
000-default.conf  default-ssl.conf  departamentos.conf
{% endhighlight %}

Como era de esperar, el fichero se encuentra generado, así que lo único que queda es realizar las modificaciones oportunas al mismo, tales como indicar **departamentos.iesgn.org** como **ServerName** y **/srv/departamentos** como **DocumentRoot**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
        ServerName departamentos.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Esa sería la apariencia del fichero de configuración hasta ahora, así que únicamente nos queda habilitar el VirtualHost que acabamos de crear, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# a2ensite departamentos
Enabling site departamentos.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que recargar la configuración del servicio _apache2_, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez que la configuración se ha vuelto a cargar, vamos a listar el contenido de **/etc/apache2/sites-enabled/** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# ls -l /etc/apache2/sites-enabled/
total 0
lrwxrwxrwx 1 root root 35 Oct 25 15:35 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root 30 Oct 25 15:44 departamentos.conf -> ../sites-available/departamentos.conf
{% endhighlight %}

Efectivamente, el nuevo VirtualHost se encuentra activo. Tras ello, volveremos a la máquina anfitriona y configuraremos la resolución estática de nombres en la misma, para así poder realizar la traducción del nombre a la IP de la máquina y que así podamos acceder, gracias a la cabecera Host de la petición HTTP. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** para el nuevo sitio creado, siendo **[IP]** la dirección IP de la máquina servidora y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.1.137   departamentos.iesgn.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para que se lleve a cabo la resolución estática de nombres, que tiene prioridad sobre las peticiones al servidor DNS, verificando así que el servidor _apache2_ se encuentra operativo:

![hosts](https://i.ibb.co/51hxK6k/Captura-de-pantalla-de-2020-10-25-12-11-03.png "departamentos.iesgn.org")

Como podemos apreciar, se ha podido acceder sin problema, ya que hemos realizado correctamente la configuración del **ServerName** en el fichero de configuración y a la hora de comparar la cabecera **Host** existente en la petición HTTP, ha encontrado coincidencia con dicho VirtualHost. Además, dado que la directiva **Indexes** se encuentra habilitada en el DocumentRoot (como resultado de haberla heredado del directorio padre), podremos ver el listado de recursos disponibles en el servidor, es decir, los 3 directorios que hemos generado.

## Tarea 1: A la URL departamentos.iesgn.org/intranet sólo se debe tener acceso desde el cliente de la red local. A la URL departamentos.iesgn.org/internet, sin embargo, sólo se debe tener acceso desde la anfitriona por la red pública.

Para poder hacer la prueba de una manera más visual, vamos a crear ficheros **index.html** que serán servidos automáticamente al acceder a cada uno de los subdirectorios. La estructura de los mismos será muy sencilla, así que empezaremos por generar dicho fichero en el subdirectorio **intranet/**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano /srv/departamentos/intranet/index.html
{% endhighlight %}

El contenido del mismo será:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Intranet</title>
    </head>
    <body>
        <h2>Red interna.</h2>
    </body>
</html>
{% endhighlight %}

Tras ello, generaremos el fichero en el subdirectorio **internet/**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano /srv/departamentos/internet/index.html
{% endhighlight %}

El contenido del mismo será:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Internet</title>
    </head>
    <body>
        <h2>Red pública.</h2>
    </body>
</html>
{% endhighlight %}

Por último, generaremos el fichero en el subdirectorio **secreto/**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano /srv/departamentos/secreto/index.html
{% endhighlight %}

El contenido del mismo será:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Secreto</title>
    </head>
    <body>
        <h2>Esto es un secreto.</h2>
    </body>
</html>
{% endhighlight %}

Ya tenemos todos los ficheros **index.html** generados dentro de los correspondientes directorios, así que iremos accediendo desde el navegador a todos ellos para verificar que el fichero se está sirviendo y mostrando correctamente, empezando por ejemplo, por **departamentos.iesgn.org/intranet**:

![indexhtml1](https://i.ibb.co/Rhc13F8/Captura-de-pantalla-de-2020-10-25-12-15-06.png "Index en intranet")

Como se puede apreciar, el fichero ha sido correctamente servido, gracias a la directiva **DirectoryIndex** configurada en el servidor _apache2_, que establece la lista de ficheros que han de servirse por defecto a la hora de acceder a un directorio. La siguiente prueba la haremos para **departamentos.iesgn.org/internet**:

![indexhtml2](https://i.ibb.co/SVN1TsS/Captura-de-pantalla-de-2020-10-25-12-15-11.png "Index en internet")

De nuevo, el fichero ha sido correctamente servido, así que vamos a pasar al último directorio, **departamentos.iesgn.org/secreto**:

![indexhtml3](https://i.ibb.co/bzK6sYx/Captura-de-pantalla-de-2020-10-25-12-15-14.png "Index en secreto")

Genial, todos los ficheros se están sirviendo correctamente, pero todavía no hemos hecho lo que se nos pide, ya que estamos accediendo sin problema desde el navegador de la máquina anfitriona a los tres directorios, y lo que queremos es que únicamente podamos acceder a **internet/** desde la máquina anfitriona y a **intranet/** únicamente desde la máquina cliente conectada a la red local (el directorio **secreto/** lo trataremos más adelante).

Dado que queremos llevar a cabo una configuración concreta para este VirtualHost, no sería recomendable hacer los cambios en el fichero de configuración general de _apache2_ (**/etc/apache2/apache2.conf**), sino en el fichero de configuración del VirtualHost en concreto (**/etc/apache2/sites-available/departamentos.conf**). En este caso, queremos limitar el acceso por IP, es decir:

* **intranet/**: Únicamente sea accesible desde la red interna (**192.168.50.0/24**).
* **internet/**: Únicamente sea accesible desde la red externa (**192.168.1.0/24**).

Gracias a la directiva de control de acceso **Require ip** de _apache2_, podremos filtrar dichas conexiones, y permitir únicamente aquellas que provengan de una determinada red (también podríamos haber especificado la dirección de una máquina concreta en lugar de una red). Para ello, tendremos que crear una sección **Directory** para cada uno de los directorios que queramos configurar, modificando el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

El resultado del fichero sería el siguiente:

{% highlight shell %}
<VirtualHost *:80>
        ServerName departamentos.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        <Directory /srv/departamentos/intranet/>
                Require ip 192.168.50
        </Directory>

        <Directory /srv/departamentos/internet/>
                Require ip 192.168.1 
        </Directory>
</VirtualHost>
{% endhighlight %}

Como se puede apreciar, el directorio **intranet/** ha sido configurado para ser únicamente accesible desde la red **192.168.50.0/24** (es decir, **192.168.50**), y el directorio **internet/** ha sido configurado para ser únicamente accesible desde la red **192.168.1.0/24** (es decir, **192.168.1**).

Como hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar dicha configuración, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

De nuevo, vamos a realizar las correspondientes pruebas para verificar que el control de acceso se ha configurado correctamente. Lo primero que haremos será tratar de acceder desde el navegador de la máquina anfitriona al directorio **internet/** (recordemos que utiliza la red **192.168.1.0/24**):

![control1](https://i.ibb.co/R42DpFw/Captura-de-pantalla-de-2020-10-25-12-21-08.png "Acceso a internet")

Como era de esperar, el recurso ha sido correctamente devuelto, ya que dicho directorio tenía permitido el acceso desde la red **192.168.1.0/24**. Sin embargo, vamos a tratar de acceder esta vez al directorio **intranet/**:

![control2](https://i.ibb.co/mHXVGn5/Captura-de-pantalla-de-2020-10-25-12-21-13.png "Acceso a intranet")

En este caso, dado que dicho directorio únicamente tiene permitido el acceso desde la red **192.168.50.0/24**, no nos ha sido posible acceder, pues no pertenecemos a dicha red, por lo que nos ha devuelto un error Forbidden (403).

Ya hemos llevado a cabo las correspondientes pruebas en la máquina anfitriona, pero todavía nos queda probar en la máquina cliente, que se encuentra conectada a la red interna. Lo primero que haremos será conectarnos a la misma, haciendo uso de la instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/apache$ vagrant ssh cliente
Linux cliente 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@cliente:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86334sec preferred_lft 86334sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:7c:7f:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.10/24 brd 192.168.50.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe7c:7f02/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **192.168.50.10**.

Dado que la máquina vagrant es una máquina totalmente ajena a la anfitriona, la resolución estática de nombres no se ha configurado todavía, así que tendremos que volver a hacerlo, para así poder realizar la traducción del nombre a la IP de la máquina y que así podamos acceder, gracias a la cabecera Host de la petición HTTP. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
vagrant@cliente:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir la línea anteriormente añadida al fichero **/etc/hosts** de la máquina anfitriona, pero ésta vez, en lugar de apuntar a la dirección IP **192.168.1.137**, tendremos que apuntar a la dirección IP de la interfaz de red de la máquina servidora perteneciente a la red interna, es decir, la **192.168.50.2**, ya que de lo contrario no sería accesible, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       cliente cliente
192.168.50.2    departamentos.iesgn.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Listo. La resolución estática de nombres ha sido correctamente configurada, pero dado que la máquina cliente no tiene interfaz gráfica, tendremos que hacer uso de un paquete que simula un navegador en la terminal, para así hacerlo más similar a la realidad. En este caso, haremos uso de **elinks**, paquete que debemos instalar, no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@cliente:~# apt update && apt upgrade && apt install elinks
{% endhighlight %}

Tras ello, únicamente tendremos que ejecutar el comando `elinks` seguido de la dirección del sitio al que nos queremos conectar, en este caso, empezaremos por acceder al directorio **intranet/**:

{% highlight shell %}
root@cliente:~# elinks departamentos.iesgn.org/intranet
{% endhighlight %}

![control3](https://i.ibb.co/YXDLLWg/Captura-de-pantalla-de-2020-10-25-12-29-57.png "Acceso a intranet")

Como era de esperar, el recurso ha sido correctamente devuelto, ya que dicho directorio tenía permitido el acceso desde la red **192.168.50.0/24**. Sin embargo, vamos a tratar de acceder esta vez al directorio **internet/**, haciendo uso de la sintaxis anteriormente mencionada:

{% highlight shell %}
root@cliente:~# elinks departamentos.iesgn.org/internet
{% endhighlight %}

![control4](https://i.ibb.co/YjWKd0h/Captura-de-pantalla-de-2020-10-25-12-30-14.png "Acceso a internet")

En este caso, dado que dicho directorio únicamente tiene permitido el acceso desde la red **192.168.1.0/24**, no nos ha sido posible acceder, pues no pertenecemos a dicha red, por lo que nos ha devuelto un error Forbidden (403).

## Tarea 2: Limita el acceso a la URL departamentos.iesgn.org/secreto. Comprueba las cabeceras de los mensajes HTTP que se intercambian entre el servidor y el cliente. ¿Cómo se manda la contraseña entre el cliente y el servidor?

Para esta tarea, tendremos que crear una sección **Directory** para el nuevo directorio que vamos a configurar, modificando el fichero de configuración del VirtualHost. En este caso, vamos a hacer uso de cuatro directivas para la **autentificación básica** del mismo:

* **AuthUserFile**: Indicaremos la ruta del fichero que almacenará las credenciales de acceso de los usuarios (en caso de querer permitir el acceso mediante grupos en lugar de usuarios, tendríamos que hacer uso de **AuthGroupFile**). Debe estar fuera del DocumentRoot, para que no sea accesible.
* **AuthName**: Es el texto informativo que se mostrará en la ventana emergente a la hora de solicitar las credenciales de acceso al usuario que trata de acceder.
* **AuthType**: Es el tipo de autentificación que vamos a usar. En este caso, **Basic**.
* **Require**: Indicamos el método de verificación de acceso, que en este caso, será **valid-user**, de manera que dejará acceder a cualquier usuario existente en el fichero previamente mencionado que introduzca de forma correcta su contraseña.

Para llevar a cabo la configuración, tendremos que crear una sección Directory para el directorio que queramos proteger, modificando el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

El resultado del fichero sería el siguiente:

{% highlight shell %}
<VirtualHost *:80>
        ServerName departamentos.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        <Directory /srv/departamentos/intranet/>
                Require ip 192.168.50
        </Directory>

        <Directory /srv/departamentos/internet/>
                Require ip 192.168.1 
        </Directory>

        <Directory /srv/departamentos/secreto/>
                AuthUserFile "/etc/apache2/claves/passwd.txt"
                AuthName "Acceso de administrador"
                AuthType Basic
                Require valid-user
        </Directory>
</VirtualHost>
{% endhighlight %}

Como se puede apreciar, el fichero que almacenará las credenciales de acceso se encuentra en **/etc/apache2/claves/passwd.txt** y el texto que se mostrará en la ventana emergente será "**Acceso de administrador**".

En realidad, el fichero que contendrá las credenciales de acceso todavía no se encuentra generado, ni tampoco el directorio que contiene dicho fichero, así que empezaremos por crear el directorio **claves/** dentro de **/etc/apache2/**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# mkdir /etc/apache2/claves
{% endhighlight %}

El fichero de credenciales lo generaremos gracias a la utilidad `htpasswd`, indicando la ruta en cuestión y el usuario que queremos añadir. En este caso, crearé un usuario "**prueba**", cuya contraseña será "**password123**", ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# htpasswd -c /etc/apache2/claves/passwd.txt prueba
New password: 
Re-type new password: 
Adding password for user prueba
{% endhighlight %}

Donde:

* **-c**: Indica que se lleve a cabo la creación del fichero de credenciales. Únicamente debemos hacer uso de ésta opción la primera vez, pues de lo contrario, sobreescribirá el fichero existente.

Por curiosidad, vamos a comprobar el contenido de dicho fichero generado, haciendo uso de `cat`:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# cat /etc/apache2/claves/passwd.txt 
prueba:$apr1$shqJIbxC$FVdbC/Cs/xdF9JcXYmkZO0
{% endhighlight %}

Como se puede apreciar, existe un único usuario **prueba** cuya contraseña se encuentra encriptada como resultado de la aplicación de una función de numeración posicional. En caso de querer eliminar un usuario, es decir, hacer que deje de tener acceso, bastaría con eliminar la línea correspondiente al mismo.

Como anteriormente hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar dicha configuración, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Tras ello, trataremos de acceder al directorio **secreto/** desde el navegador de la máquina anfitriona (por ejemplo, aunque desde la máquina cliente funcionaría exactamente de la misma forma):

![basic1](https://i.ibb.co/PCCQrbP/Captura-de-pantalla-de-2020-10-25-12-42-45.png "Solicita credenciales")

Tal y como era de esperar, el navegador ha mostrado una ventana emergente solicitando las credenciales de acceso. Además, nos ha mostrado el texto previamente configurado, "**Acceso de administrador**". Tras introducir el usuario **prueba** y la contraseña **password123**, pulsaremos en **Aceptar** y veremos que hemos podido acceder correctamente.

![basic2](https://i.ibb.co/qkdcft0/Captura-de-pantalla-de-2020-10-25-16-42-09.png "Acceso exitoso")

Realmente, la parte importante que quería mostrar no era cómo acceder, sino lo que ocurre durante dicho acceso. Para ello, he dejado **Wireshark** capturando paquetes con un filtro **HTTP** en segundo plano, para posteriormente analizar con detalle qué es lo que ha ocurrido. En Internet se puede encontrar documentación sobre cómo descargar e instalar Wireshark y capturar paquetes usando un determinado filtro, pues dicha explicación se saldría del objetivo de este _post_. Estos son los paquetes capturados:

![basic3](https://i.ibb.co/Z2mM4GV/wireshark1.jpg "Captura Wireshark")

Explicar con detalle todo el proceso de negociación entre ambas máquinas sería algo extenso, así que voy a tratar de resumirlo de la mejor manera posible:

* **Paquete 1**: La máquina anfitriona hace una petición GET a la máquina servidora solicitando el recurso **/secreto**.
* **Paquete 2**: La máquina servidora le devuelve una respuesta Unauthorized (401) ya que el recurso está protegido, por lo que le pide las credenciales de acceso.
* **Paquete 3**: La máquina anfitriona envía las credenciales introducidas en la ventana emergente a la máquina servidora, volviendo a solicitar el mismo recurso.
* **Paquete 4**: La máquina servidora verifica que las credenciales sean correctas, y tras ello, devuelve una redirección permanente (301) desde **/secreto** a **/secreto/**, que es donde realmente se encuentra el directorio.
* **Paquete 5**: La máquina anfitriona vuelve a hacer otra petición GET, solicitando el nuevo recurso, es decir, **/secreto/**, indicando de nuevo las credenciales de acceso.
* **Paquete 6**: La máquina servidora devuelve el recurso solicitado (en este caso, el **index.html**), devolviendo un código de estado 200.
* **Paquete 7**: La máquina anfitriona desea mostrar todo el contenido posible, así que solicita también el favicon, indicando de nuevo las credenciales de acceso.
* **Paquete 8**: La máquina servidora devuelve un código de estado 404 ya que no ha encontrado el recurso solicitado.

Esa sería la explicación superficial de todo lo que ha ocurrido durante este proceso, pero realmente, lo importante se encuentra en el **Paquete 3**, cuando se envían las credenciales al servidor. Si nos volvemos a fijar en la imagen anteriormente mostrada, en la cabecera de la petición existe un apartado **Authorization**. Dentro del mismo, se encuentra el **usuario** y la **contraseña** en "texto plano". Lo pongo entre comillas porque no va en texto plano como tal, sino usando un sistema de numeración posicional **base64**, pero que realmente no nos impide nada, ya que lo podemos descifrar haciendo uso de cualquier [utilidad](https://www.base64decode.org/) (aunque Wireshark ya lo ha hecho automáticamente, en el apartado **Credentials**):

![basic4](https://i.ibb.co/rbs4vRm/decode.jpg "Decode BASE64")

Como podemos apreciar, este método de autentificación es poco seguro. Es por ello, que tenemos (al menos) dos alternativas. La primera de ellas sería utilizar Apache con SSL (_Secure Sockets Layer_), de manera que permite cifrar el tráfico de datos entre el navegador web y el sitio web (o entre dos servidores web), protegiendo así la conexión. De otro lado, podríamos usar un método de autentificación más seguro, como **digest**.

## Tarea 3: Modifica la autentificación para que sea del tipo digest, y sólo sea accesible a los usuarios pertenecientes al grupo directivos. Comprueba las cabeceras de los mensajes HTTP que se intercambian entre el servidor y el cliente. ¿Cómo funciona esta autentificación?

En este caso, queremos cambiar la autentificación básica a una **autentificación digest**, que es más segura que la anterior, pues soluciona el problema de la transferencia de contraseñas en claro sin necesidad de usar SSL.

El procedimiento para habilitar este tipo de autentificación es muy similar al anterior, pero con una peculiaridad, y es que el módulo que utiliza dicho tipo de autentificación (**auth_digest**) viene instalado pero no habilitado por defecto. De igual forma, antes de tratar de habilitarlo, vamos a comprobar que no se encuentra ya habilitado, listando para ello el contenido del directorio **/etc/apache2/mods-enabled/** con `ls` y filtrando por el nombre del módulo:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# ls /etc/apache2/mods-enabled/ | egrep 'auth_digest'
{% endhighlight %}

Como se puede apreciar, el filtro no ha devuelto ningún resultado, por lo que podemos estar seguros de que el módulo no se encuentra cargado en memoria, por lo que procederemos a cargarlo, haciendo uso de `a2enmod` (aunque también podríamos crear un enlace simbólico al módulo en el directorio **/etc/apache2/mods-available/** manualmente):

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# a2enmod auth_digest 
Considering dependency authn_core for auth_digest:
Module authn_core already enabled
Enabling module auth_digest.
To activate the new configuration, you need to run:
  systemctl restart apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, el módulo ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que reiniciar el servicio _apache2_, cosa que todavía no haremos, pues nos quedan más configuraciones por hacer, evitando así tener que llevar a cabo varios reinicios.

Para esta tarea, tendremos que modificar la sección **Directory** anteriormente declarada en el fichero de configuración del VirtualHost. En este caso, vamos a hacer uso de cuatro directivas para la **autentificación digest** del mismo:

* **AuthUserFile**: Indicaremos la ruta del fichero que almacenará las credenciales de acceso de los usuarios. Debe estar fuera del DocumentRoot, para que no sea accesible.
* **AuthName**: Indicamos el nombre de dominio (grupo o realm) al que ha de pertenecer el usuario que trate de acceder. Además, se mostrará en la ventana emergente a la hora de solicitar las credenciales de acceso al usuario que trata de acceder.
* **AuthType**: Es el tipo de autentificación que vamos a usar. En este caso, **Digest**.
* **Require**: Indicamos el método de verificación de acceso, que en este caso, será **valid-user**, de manera que dejará acceder a cualquier usuario existente en el fichero previamente mencionado que introduzca de forma correcta su contraseña y pertenezca al realm indicado.

Para llevar a cabo la configuración, tendremos que modificar el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

El resultado del fichero sería el siguiente:

{% highlight shell %}
<VirtualHost *:80>
        ServerName departamentos.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        <Directory /srv/departamentos/intranet/>
                Require ip 192.168.50
        </Directory>

        <Directory /srv/departamentos/internet/>
                Require ip 192.168.1 
        </Directory>

        <Directory /srv/departamentos/secreto/>
                AuthUserFile "/etc/apache2/claves/digest.txt"
                AuthName "directivos"
                AuthType Digest
                Require valid-user
        </Directory>
</VirtualHost>
{% endhighlight %}

Como se puede apreciar, el fichero que almacenará las credenciales de acceso se encuentra en **/etc/apache2/claves/digest.txt** y el realm (nombre de dominio) al que ha de pertenecer el usuario que trate de acceder, será **directivos**.

En realidad, el fichero que contendrá las credenciales de acceso todavía no se encuentra generado, así que lo generaremos gracias a la utilidad `htdigest`, indicando la ruta en cuestión, seguido del realm y el usuario que queremos añadir. En este caso, crearé un usuario "**prueba1**", que pertenecerá al realm "**directivos**" y cuya contraseña será "**password123**", ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# htdigest -c /etc/apache2/claves/digest.txt directivos prueba1
Adding password for prueba1 in realm directivos.
New password: 
Re-type new password: 
{% endhighlight %}

Donde:

* **-c**: Indica que se lleve a cabo la creación del fichero de credenciales. Únicamente debemos hacer uso de ésta opción la primera vez, pues de lo contrario, sobreescribirá el fichero existente.

Para poder hacer bien la prueba y posteriormente verificar que un usuario que no pertenezca al realm **directivos** no puede acceder, vamos a añadir un segundo usuario "**prueba2**", que pertenecerá al realm "**grupoerroneo**" y cuya contraseña será "**password123**", ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# htdigest /etc/apache2/claves/digest.txt grupoerroneo prueba2
Adding user prueba2 in realm grupoerroneo
New password: 
Re-type new password:
{% endhighlight %}

Si nos fijamos, en el comando anterior no hemos usado la opción **-c**, ya que el fichero de credenciales ya se encuentra generado. Por curiosidad, vamos a comprobar el contenido de dicho fichero generado, haciendo uso de `cat`:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# cat /etc/apache2/claves/digest.txt 
prueba1:directivos:63886ff5b2de1d219b37adc034f5ebdc
prueba2:grupoerroneo:45b8eb647af13944f4aec116234159f1
{% endhighlight %}

Como se puede apreciar, existen dos usuarios, **prueba1** con realm **directivos** y **prueba2** con realm **grupoerroneo**, cuyas contraseñas se encuentran encriptadas.

Como anteriormente hemos habilitado un módulo del servicio y hemos modificado un fichero de configuración, tendremos que reiniciarlo, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# systemctl restart apache2
{% endhighlight %}

Antes de realizar pruebas, vamos a verificar que efectivamente el módulo se encuentra cargado en memoria, ejecutando el comando visto con anterioridad:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# ls /etc/apache2/mods-enabled/ | egrep 'auth_digest'
auth_digest.load
{% endhighlight %}

Efectivamente, el módulo ya se encuentra cargado en memoria y está listo para ser usado. Tras ello, trataremos de acceder al directorio **secreto/** desde el navegador de la máquina anfitriona (por ejemplo, aunque desde la máquina cliente funcionaría exactamente de la misma forma):

![digest1](https://i.ibb.co/zh0QSMH/Captura-de-pantalla-de-2020-10-25-16-41-47.png "Solicita credenciales")

Tal y como era de esperar, el navegador ha mostrado una ventana emergente solicitando las credenciales de acceso. Además, nos ha mostrado el texto previamente configurado, "**directivos**". Tras introducir el usuario **prueba2** y la contraseña **password123**, pulsaremos en **Aceptar** y veremos que no hemos podido acceder dado que dicho usuario no pertenece al realm **directivos**.

Realmente, la parte importante que quería mostrar no era cómo acceder, sino lo que ocurre durante dicho acceso. Para ello, he dejado **Wireshark** capturando paquetes con un filtro **HTTP** en segundo plano, para posteriormente analizar con detalle qué es lo que ha ocurrido. Estos son los paquetes capturados:

![digest2](https://i.ibb.co/5xDNBKk/wireshark2.jpg "Captura Wireshark")

En este caso, no voy a explicar de nuevo el proceso de negociación entre las máquinas, ya que la filosofía es la misma. Sin embargo, donde sí quiero hacer un apunte es en la cabecera **Authentication** de la petición. Si recordamos, anteriormente existía un único parámetro **Credentials**, pero ahora hay unos cuantos más, principalmente debido a la mayor seguridad que proporciona este tipo de autentificación. El proceso simplificado de autorización es el siguiente:

* El servidor le proporciona al cliente una cadena (**nonce**) de uso único, que combina con el **usuario**, el dominio (**realm**), la **contraseña** y la **URI** (recurso solicitado).
* El cliente hace pasar todos esos campos por un método hash **MD5**, de manera que genera una **clave hash**. Tras ello, se la envía al servidor junto al nombre de usuario y el realm para tratar de autentificarse.
* El servidor lleva a cabo el mismo proceso para generar la clave hash y posteriormente, la compara con la recibida por parte del cliente, por lo que en caso de ser iguales, permite el acceso y en caso contrario, lo deniega.

Este método es más seguro, incluso ante ataques de replicación, ya que el **nonce** es de uso único, de manera que en caso de tratar de generar una clave hash usando un nonce distinto, supondría que la clave hash resultante fuese totalmente distinta.

En el anterior intento, no nos ha dejado entrar, pero si volvemos a intentarlo introduciendo ahora unas credenciales correctas de un usuario que pertenezca al realm **directivos**, tal y como es **prueba1**, podríamos acceder sin problema alguno.

![digest3](https://i.ibb.co/qkdcft0/Captura-de-pantalla-de-2020-10-25-16-42-09.png "Acceso exitoso")

Como se puede apreciar, la autentificación usando **Digest** se ha llevado a cabo de forma exitosa.

## Tarea 4: Vamos a combinar el control de acceso y la autentificación y vamos a configurar el VirtualHost para que se comporte de manera que el acceso a la URL departamentos.iesgn.org/secreto se hace forma directa desde la intranet y desde la red pública te pida autentificarte.

En este caso, tendremos que hacer uso de las políticas de acceso existentes en _apache2_, concretamente de **RequireAll** y **RequireAny**. La primera de ellas, indica que todas las condiciones existentes dentro del bloque se deben cumplir para obtener el acceso, mientras que la segunda, indica que debe cumplirse al menos una de las condiciones existentes dentro del bloque para obtener el acceso. Usaremos dos **RequireAll** para declarar los dos métodos de acceso (uno mediante la red **192.168.1.0/24**, que requerirá autentificación, y otro mediante la red **192.168.50.0/24**, que no requerirá autentificación). Por encima de ellos, existirá un **RequireAny**, de manera que en caso de que se cumpla cualquiera de los dos **RequireAll**, permitirá el acceso. Explicado con palabras puede resultar un poco lioso, así que vamos a ponernos manos a la obra para que así se vea más claro.

Para llevar a cabo la configuración, tendremos que modificar el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

El resultado del fichero sería el siguiente:

{% highlight shell %}
<VirtualHost *:80>
        ServerName departamentos.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        <Directory /srv/departamentos/intranet/>
                Require ip 192.168.50
        </Directory>

        <Directory /srv/departamentos/internet/>
                Require ip 192.168.1 
        </Directory>

        <Directory /srv/departamentos/secreto/>
                <RequireAny>
                        <RequireAll>
                                Require ip 192.168.50
                        </RequireAll>
                        <RequireAll>
                                Require ip 192.168.1
                                AuthUserFile "/etc/apache2/claves/digest.txt"
                                AuthName "directivos"
                                AuthType Digest
                                Require valid-user
                        </RequireAll>
                </RequireAny>
        </Directory>
</VirtualHost>
{% endhighlight %}

Como hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar dicha configuración, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Tras ello, trataremos de acceder al directorio **secreto/** desde el navegador de la máquina anfitriona (conexión que se llevaría a cabo desde la red **192.168.1.0/24**, por lo que debería solicitar las credenciales de acceso):

![mix1](https://i.ibb.co/zh0QSMH/Captura-de-pantalla-de-2020-10-25-16-41-47.png "Solicita credenciales")

Como era de esperar, dado que el acceso se ha realizado desde la red externa, se han solicitado las credenciales, así que las introduciremos y veremos si se muestra el contenido:

![mix2](https://i.ibb.co/qkdcft0/Captura-de-pantalla-de-2020-10-25-16-42-09.png "Acceso exitoso")

Efectivamente, así ha sido. Todavía nos queda ver la otra _cara de la moneda_, así que trataremos acceder con **elinks** desde la máquina cliente (conexión que se llevaría a cabo desde la red **192.168.50.0/24**, por lo que no debería solicitar las credenciales de acceso):

{% highlight shell %}
root@cliente:~# elinks departamentos.iesgn.org/secreto
{% endhighlight %}

![mix3](https://i.ibb.co/jDSD8BK/Captura-de-pantalla-de-2020-10-25-16-56-09.png "Acceso exitoso")

Como era de esperar, dado que el acceso se ha realizado desde la red interna, no se han solicitado las credenciales, mostrándose el contenido directamente. Por lo tanto, podemos asegurar que la configuración ha funcionado correctamente.

Por último, me gustaría dejar una pequeña anotación. Por defecto, el usuario a través del cuál se sirven las páginas web en _apache2_ es **www-data** (en este caso no habría inconveniente ya que el grupo "otros", al que pertenece dicho usuario tiene permisos de **lectura**, que es lo único que necesitamos para servir el fichero, pero en caso de necesitar escribir en dichos ficheros, no contaría con dichos permisos de **escritura**), por lo que podemos proceder a cambiar dicho propietario y grupo de forma recursiva, en caso de así desearlo, ya que el usuario y grupo actual es **root**, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@servidor:/etc/apache2/sites-available# chown -R www-data:www-data /srv/*
{% endhighlight %}

Gracias al comando ejecutado con anterioridad, todos los directorios y ficheros hijos de **/srv** serían ahora propiedad de **www-data** y haciendo uso del grupo **www-data**.