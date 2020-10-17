---
layout: post
title:  "VirtualHosting con Apache"
banner: "/assets/images/banners/virtualhosting.jpg"
date:   2020-10-17 19:07:00 +0200
categories: servicios
---
Gracias a la funcionalidad "**VirtualHost**" del servidor web _apache2_ podemos hacer que una misma máquina con una única interfaz de red (es decir, con una única dirección IP) escuchando en un único puerto (generalmente el puerto **80** para HTTP y **443** para HTTPS) pueda servir diferentes páginas web que se puedan acceder de forma independiente.

Para ello, se hace uso de las cabeceras HTTP que contiene la petición que recibe el servidor, que contiene un parámetro "**Host**" que indica el nombre del servidor al que se le hace la petición (por ejemplo, **www.prueba1.com** o **www.prueba2.com**).

Al igual que se puede especificar que la separación se haga mediante los nombres de dominio, también se puede hacer basada en la dirección IP, de manera que cada sitio tenga una dirección IP diferente, pero no es algo tan "sorprendente", por lo que no lo trataremos en este post.

Es muy probable que hasta el día de hoy no supieses de la existencia de dicha funcionalidad, y pensabas que por cada sitio web que tuviese que servir _apache2_, había que tener una máquina (ya sea física o virtual) para cada uno de ellos, pero gracias a esta funcionalidad, se ahorrarán costes tanto monetarios como computacionales, al servir todos los sitios web desde la misma máquina, sin necesidad de ejecutar ninguna máquina virtual.

Pero si yo no he configurado nada, ¿cómo es posible que al acceder a **localhost:80**, me sirva automáticamente el contenido que hay en **/var/www/html**? Bien, la respuesta es muy sencilla, y es que _apache2_ se instala por defecto con un VirtualHost configurado, de nombre "**000-default**", sin especificar el ServerName desde el que es accesible, es decir, significando que podemos entrar con cualquier nombre, siendo el DocumentRoot (directorio donde se encuentran alojados los ficheros a servir por el sitio web) "**/var/www/html**". Además, en el fichero de configuración se indica que se escuchan las peticiones desde __*:80__, por lo que se puede acceder desde cualquier dirección IP, siempre y cuando sea al puerto **80**.

Podemos encontrar los ficheros de configuración de todos los VirtualHost creados en Apache2 en **/etc/apache2/sites-available/**, pero eso no significa que todos los que se encuentren ahí estén habilitados y actualmente activos. Los sitios actualmente activos los encontramos en **/etc/apache2/sites-enabled/**, que al fin y al cabo son un enlace simbólico al fichero de configuración existente en _/etc/apache2/sites-available/_.

El objetivo de esta práctica es el de construir en nuestro servidor web _apache2_ dos sitios web con nombres distintos pero compartiendo dirección IP y puerto:

* El primero de ellos tendrá nombre de dominio **www.iesgn.org**, con DocumentRoot en **/srv/www/iesgn** y cuyo contenido será una página llamada **index.html**, donde se verá una bienvenida a la página del Instuto Gonzalo Nazareno.
* El segundo de ellos tendrá nombre de dominio **www.departamentosgn.org**, con DocumentRoot en **/srv/www/departamentos** y cuyo contenido será una página llamada **index.html**, donde se verá una bienvenida a la página de los departamentos del instuto.

Para la práctica he creado un pequeño escenario en **Vagrant**, quedando de la siguiente forma:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
   config.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
   config.vm.hostname = "apache" #Establecemos el nombre (hostname) de la máquina.
   config.vm.network :private_network, ip: "192.168.50.2" #Creamos una interfaz de red host-only, que tendrá un direccionamiento estático, pues se trata del servidor y nos conviene que siempre sea la misma.
end
{% endhighlight %}

Tras ello, levantaremos el escenario Vagrant haciendo uso de la instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/apache$ vagrant up
{% endhighlight %}

A continuación comenzará a descargar el **box** (en caso de que no lo tuviésemos previamente) y a generar la máquina virtual con los parámetros que le hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la misma ejecutando la siguiente instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/apache$ vagrant ssh
Linux apache 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que la interfaz de red se ha generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@apache:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86320sec preferred_lft 86320sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:9f:1c:04 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.2/24 brd 192.168.50.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9f:1c04/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada en modo host-only al anfitrión, con dirección IP estática **192.168.50.2**.

Una vez que se ha llevado a cabo la comprobación, procedemos a configurar el servidor web _apache2_. Para ello, lo primero que haremos será instalar el paquete necesario (**apache2**), no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@apache:~# apt update && apt upgrade && apt install apache2
{% endhighlight %}

Una vez instalado el paquete, tendremos que modificar el fichero de configuración principal (**/etc/apache2/apache2.conf**), ya que se pide que los sitios web se alojen en **/srv/www**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@apache:~# nano /etc/apache2/apache2.conf
{% endhighlight %}

Dentro del mismo, tendremos que buscar la sección donde se encuentran los "**Directory**":

{% highlight shell %}
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

Como se puede apreciar, tenemos un apartado "**Directory /var/www/**", con sus determinados permisos, que se heredarán a sus hijos de forma recursiva, por lo tanto, es dentro del mismo donde debemos alojar los diferentes DocumentRoot para los sitios web, pero hay un problema, y es que no queremos que sea ahí donde alojemos los dos sitios que vamos a crear, sino en **/srv/www**, por lo que tendremos que crear un nuevo apartado "**Directory /srv/www/**" con los correspondientes permisos (pueden ser los mismos que en "**Directory /var/www/**"). Debería quedar de la siguiente forma:

{% highlight shell %}
<Directory /var/www/>
       Options Indexes FollowSymLinks
       AllowOverride None
       Require all granted
</Directory>

<Directory /srv/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

El apartado "**Directory /var/www/**" podríamos haberlo comentado ya que no lo vamos a usar, pero no voy a hacerlo ya que el sitio **000-default** se encuentra activo y utilizando dicho directorio (aunque podría haberlo desactivado de igual forma).

Tras ello, guardaremos los cambios y crearemos los nuevos DocumentRoot para ambos sitios dentro de **/srv/www**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:~# mkdir -p /srv/www/{iesgn,departamentos}
{% endhighlight %}

Gracias a la opción **-p** de `mkdir` crearemos el directorio padre en caso de no existir. Para verificar que la creación se ha llevado a cabo correctamente, listaremos el contenido de **/srv/www**, haciendo uso del comando:

{% highlight shell %}
root@apache:~# ls -l /srv/www/
total 8
drwxr-xr-x 2 root root 4096 Oct 15 07:48 departamentos
drwxr-xr-x 2 root root 4096 Oct 15 07:48 iesgn
{% endhighlight %}

Efectivamente, han sido creados. Una vez creados los directorios, tendremos que crear los correspondientes ficheros de configuración para los sitios web, que se encuentran en **/etc/apache2/sites-available**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@apache:~# cd /etc/apache2/sites-available/
{% endhighlight %}

Tras ello, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls
000-default.conf  default-ssl.conf
{% endhighlight %}

En este caso, existen dos ficheros, pero el que nos interesa es **000-default.conf**, así que para no complicarnos, vamos a copiar dicho fichero dos veces para tener una plantilla base que posteriormente modificaremos y adaptaremos a cada uno de los sitios web que vamos a crear. Para ello, ejecutaremos los comandos:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# cp 000-default.conf iesgn.conf
root@apache:/etc/apache2/sites-available# cp 000-default.conf departamentos.conf
{% endhighlight %}

Para verificar que los ficheros se han copiado correctamente, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls
000-default.conf  default-ssl.conf  departamentos.conf	iesgn.conf
{% endhighlight %}

Empezaré modificando el fichero **iesgn.conf**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano iesgn.conf
{% endhighlight %}

El contenido de dicho fichero es el siguiente (eliminando líneas comentadas para dejar una salida más limpia):

{% highlight shell %}
<VirtualHost *:80>
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Como se puede apreciar, la línea referente al **ServerName** se encuentra comentada, por lo que el sitio es accesible desde cualquier nombre de dominio, al igual que es accesible desde cualquier dirección IP existente en la máquina servidora en el puerto 80 (__*:80__). Lo primero que haremos será descomentar dicha línea e indicar el nombre de dominio desde el que será accesible, en este caso, **www.iesgn.org**. Tras ello, tendremos que modificar el **DocumentRoot** e indicar el que usará, en este caso, **/srv/www/iesgn**. Con dichas modificaciones es suficiente, aunque podríamos indicar una dirección de correo electrónico de contacto que se mostrará al cliente en caso de algún error (**ServerAdmin**) o modificar la ubicación de los ficheros de log (**ErrorLog** y **CustomLog**). El resultado es el siguiente:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/www/iesgn

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Tras ello, repetiremos el mismo proceso en el fichero **departamentos.conf**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano departamentos.conf
{% endhighlight %}

Dentro del mismo, indicaremos como **ServerName** el nombre **www.departamentosgn.org** y como **DocumentRoot** el directorio **/srv/www/departamentos**, quedando de la siguiente forma:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.departamentosgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/www/departamentos

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Una vez que los ficheros de configuración hayan sido creados y correctamente modificados, me gustaría comentar una pequeña curiosidad, y es que al haber configurado los tres sitios para que sean accesibles desde cualquier dirección IP, en caso de acceder a través de la **dirección IP** en lugar de hacerlo mediante nombres de dominio, el servidor web no sabrá a cuál de ellos referirse, por lo que ordenará alfabéticamente el nombre de los sitios y te mostrará el primero (es por eso que el VirtualHost por defecto de _apache2_ tiene como nombre **000-default**).

Tras esta pequeña curiosidad, podremos proceder a crear el fichero **index.html** que servirá cada uno de los sitios web. En este caso, he creado un fichero muy simple para cada uno de ellos con la siguiente estructura:

{% highlight html %}
<h2>
   [texto]
</h2>
{% endhighlight %}

Para crear dichos ficheros en sus respectivos directorios, con el correspondiente contenido en su interior, ejecutaremos los comandos:

{% highlight shell %}
root@apache:~# echo -e "<h2>\n\tBienvenidos a iesgn\n<h2>" > /srv/www/iesgn/index.html
root@apache:~# echo -e "<h2>\n\tBienvenidos a los departamentos de iesgn\n<h2>" > /srv/www/departamentos/index.html
{% endhighlight %}

Para verificar que dichos ficheros se han generado correctamente en sus correspondientes directorios, procedemos a listar el contenido del directorio **/srv/www** de forma recursiva, haciendo uso del comando `ls -lR`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls -lR /srv/www/
/srv/www/:
total 8
drwxr-xr-x 2 root root 4096 Oct 15 07:58 departamentos
drwxr-xr-x 2 root root 4096 Oct 15 07:58 iesgn

/srv/www/departamentos:
total 4
-rw-r--r-- 1 root root 52 Oct 15 07:58 index.html

/srv/www/iesgn:
total 4
-rw-r--r-- 1 root root 31 Oct 15 07:58 index.html
{% endhighlight %}

Efectivamente, los ficheros se han generado correctamente, pero si nos fijamos con detalle, el usuario creador y el grupo correspondiente a dichos ficheros y directorios es **root**, cosa que jamás debe ocurrir ya que el usuario propietario por defecto a través del cuál se sirven las páginas web es **www-data** (en este caso no habría inconveniente ya que el grupo "otros", al que pertenece dicho usuario tiene permisos de **lectura**, que es lo único que necesitamos para servir el fichero, pero en caso de necesitar escribir en dichos ficheros, no contaría con dichos permisos de **escritura**), por lo que procedemos a cambiar dicho propietario y grupo de forma recursiva, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# chown -R www-data:www-data /srv/www/*
{% endhighlight %}

De nuevo, volveremos a listar el contenido de dicho directorio de forma recursiva, haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls -lR /srv/www/
/srv/www/:
total 8
drwxr-xr-x 2 www-data www-data 4096 Oct 15 07:58 departamentos
drwxr-xr-x 2 www-data www-data 4096 Oct 15 07:58 iesgn

/srv/www/departamentos:
total 4
-rw-r--r-- 1 www-data www-data 52 Oct 15 07:58 index.html

/srv/www/iesgn:
total 4
-rw-r--r-- 1 www-data www-data 31 Oct 15 07:58 index.html
{% endhighlight %}

Como era de esperar, el usuario y el grupo propietario se ha visto modificado. Listo, toda la configuración necesaria ya ha sido realizada, así que únicamente queda habilitar dichos sitios y probar que realmente funcionan. Para activar los sitios, podríamos crear el enlace simbólico al fichero de configuración ubicado en **/etc/apache2/sites-available** dentro de **/etc/apache2/sites-enabled** de forma manual, pero también podemos hacer uso del comando `a2ensite` que lo hace por nosotros (para desactivarlos podemos hacer uso de `a2dissite`). Para ello, ejecutamos los comandos:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# a2ensite iesgn
Enabling site iesgn.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

{% highlight shell %}
root@apache:/etc/apache2/sites-available# a2ensite departamentos
Enabling site departamentos.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, los sitios han sido correctamente habilitados, pero para activar la nueva configuración, tendremos que recargar el servicio _apache2_, ejecutando para ello el comando:

{% highlight shell %}
root@apache:~# systemctl reload apache2
{% endhighlight %}

Una vez que el servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/apache2/sites-enabled** para verificar que los correspondientes enlaces simbólicos han sido correctamente creados. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@apache:~# ls -l /etc/apache2/sites-enabled/
total 0
lrwxrwxrwx 1 root root 35 Oct 15 07:47 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root 37 Oct 15 08:00 departamentos.conf -> ../sites-available/departamentos.conf
lrwxrwxrwx 1 root root 29 Oct 15 08:00 iesgn.conf -> ../sites-available/iesgn.conf
{% endhighlight %}

Efectivamente, los tres sitios se encuentran actualmente activos, así que es hora de realizar las correspondientes pruebas de acceso. Para ello, volveremos a la máquina anfitriona y configuraremos la resolución estática de nombres en la misma, para así poder realizar la traducción del nombre. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** por cada uno de los sitios que hayamos creado, siendo **[IP]** la dirección IP de la máquina servidora (en ambos casos **192.168.50.2**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.50.2    www.iesgn.org
192.168.50.2    www.departamentosgn.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para que se lleve a cabo la resolución estática de nombres, que tiene prioridad sobre las peticiones al servidor DNS. Al primero que intentaré acceder será a **www.iesgn.org**:

![iesgn](https://i.ibb.co/CVJMcmc/Captura-de-pantalla-de-2020-10-13-14-38-50.png "www.iesgn.org")

Como podemos apreciar, se ha podido acceder sin problema, ya que hemos realizado correctamente la configuración del **ServerName** en el fichero de configuración y a la hora de comparar la cabecera **Host** existente en la petición HTTP, ha encontrado coincidencia con dicho VirtualHost. Por último, probaré también con **www.departamentosgn.org**:

![departamentosgn](https://i.ibb.co/TBFQ7Qp/Captura-de-pantalla-de-2020-10-13-14-38-52.png "www.departamentosgn.org")

Efectivamente, también ha sido accesible, por lo que podemos concluir que hemos conseguido alojar dos sitios web en una única máquina, siendo accesibles ambos de ellos desde la misma dirección IP y puerto.