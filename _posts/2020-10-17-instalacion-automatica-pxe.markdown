---
layout: post
title:  "Sistema de instalación PXE/TFTP"
banner: "/assets/images/banners/pxe.jpg"
date:   2020-10-17 12:19:00 +0200
categories: sistemas
---
El objetivo de esta tarea es el de configurar un sistema de instalación con **PXE/TFTP** que realice una instalación completa de un sistema Debian Buster de forma totalmente **desatendida**, pasándole todos los parámetros necesarios por "**preseeding**".

La idea es configurar un entorno **PXE** (_Preboot eXecution Environment_) que nos permita arrancar e instalar el sistema operativo en máquinas a través de una red, pero nos hacen falta dos elementos esenciales, el primero de ellos, un protocolo que nos permita servir los ficheros necesarios para la instalación a la máquina cliente. Ahí es donde entra el juego el protocolo **TFTP** (_Trivial File Transfer Protocol_), que es un protocolo de transferencia muy simple semejante a una versión básica de FTP.

Además del protocolo TFTP, necesitaremos hacer uso de un servidor **DHCP** que permita asignar direcciones IP dentro de la red a los clientes, ya que es un requisito básico para que los clientes se puedan comunicar con el servidor. Aquí surge el primer problema, ya que en mi red doméstica tengo un servidor DHCP (el router de mi proveedor) que sirve direcciones IP a los dispositivos que existen en mi casa, por lo que configurar otro servidor DHCP dentro de la misma red puede llegar a ocasionar conflictos. Por ello, voy a hacer uso de **Vagrant** para crear un escenario aislado de mi red doméstica, de manera que la práctica se pueda llevar a cabo de forma totalmente segura. El fichero de configuración **Vagrantfile** es el siguiente:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
  config.vm.hostname = "servidor" #Establecemos el nombre (hostname) de la máquina.
  config.vm.network :private_network, ip: "192.168.100.5", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor.
    virtualbox__intnet: "lan1" #Dado que VirtualBox crea por defecto las interfaces en modo host-only, tenemos que indicar que utilice una red interna, en este caso, una de nombre "lan1".
end
{% endhighlight %}

Como se puede apreciar, el escenario de Vagrant es bastante simple, con una única máquina que hace la función de servidor, ya que la máquina cliente la crearemos manualmente para realizar la instalación por red. Tras ello, ya podremos levantar el escenario, haciendo uso de la instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/pxe$ vagrant up
{% endhighlight %}

A continuación comenzará a descargar el **box** (en caso de que no lo tuviésemos previamente) y a generar la máquina virtual con los parámetros que le hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la misma ejecutando la siguiente instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/pxe$ vagrant ssh
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
       valid_lft 85879sec preferred_lft 85879sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:04:b7:7d brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.5/24 brd 192.168.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe04:b77d/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0**: Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**. A través de ella saldremos al exterior cuando sea necesario.
* **eth1**: Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **192.168.100.5**.

Tras realizar dicha comprobación, todo está listo para configurar el protocolo **TFTP** y el protocolo **DHCP**, pero la buena noticia es que existe un paquete que nos facilita ambos servicios. Se trata de **dnsmasq**, así que vamos a proceder a su instalación, no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@servidor:~# apt update && apt upgrade && apt install dnsmasq
{% endhighlight %}

Una vez que hayamos instalado el paquete, tendremos que configurarlo para que realice las funciones deseadas. Para ello, vamos a modificar el fichero **/etc/dnsmasq.conf**, que es una plantilla de configuración totalmente comentada, ejecutando el comando:

{% highlight shell %}
root@servidor:~# nano /etc/dnsmasq.conf
{% endhighlight %}

De todas las opciones de configuración existentes, las únicas que nos interesan son las siguientes:

{% highlight shell %}
#Activamos el servidor DHCP y establecemos el rango de direcciones que puede conceder
#(desde la .50 a la .150, aunque se podría haber usado otro), la máscara de red (/24)
#y la duración de dichas concesiones, en este caso, 12 horas.

dhcp-range=192.168.100.50,192.168.100.150,255.255.255.0,12h

#Establecemos el fichero que los clientes van a usar para arrancar a través de la red
#(se trata de pxelinux.0, que lo descargaremos posterioremente).

dhcp-boot=pxelinux.0

#Habilitamos el servidor TFTP.

enable-tftp

#Especificamos el directorio que contendrá los ficheros para servir (en este caso,
#en un subdirectorio dentro de /srv, ya que dicho directorio (/srv) está pensado para
#almacenar los ficheros que se van a servir por los diferentes protocolos, aunque realmente
#se podría haber usado cualquier otro directorio).

tftp-root=/srv/tftp
{% endhighlight %}

Ya hemos finalizado de configurar **dnsmasq**, por lo que acto seguido sería conveniente reiniciar el servicio para que cargue la nueva configuración, pero en este caso, nos falta un paso por realizar. El directorio que hemos especificado para el protocolo TFTP no se encuentra creado, así primero tendremos que crearlo, pues de lo contrario, dará error a la hora de cargar la nueva configuración. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# mkdir /srv/tftp
{% endhighlight %}

Tras ello, ya podemos reiniciar el servicio para que así cargue la nueva configuración. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# systemctl restart dnsmasq
{% endhighlight %}

Para verificar que el servicio ha cargado correctamente la nueva configuración y se encuentra actualmente activo sin ningún tipo de error, ejecutaremos el comando:

{% highlight shell %}
root@servidor:~# systemctl status dnsmasq
● dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
   Loaded: loaded (/lib/systemd/system/dnsmasq.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-10-15 14:12:18 GMT; 5s ago
  Process: 12865 ExecStartPre=/usr/sbin/dnsmasq --test (code=exited, status=0/SUCCESS)
  Process: 12866 ExecStart=/etc/init.d/dnsmasq systemd-exec (code=exited, status=0/SUCCESS)
  Process: 12874 ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf (code=exited, status=0/SUCCESS)
 Main PID: 12873 (dnsmasq)
    Tasks: 1 (limit: 544)
   Memory: 1.1M
   CGroup: /system.slice/dnsmasq.service
           └─12873 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-service --trust-anchor=.,20326,8,2,e06d44b80b8f1d39a95c0b0d7c65d08458e880409bb

Oct 15 14:12:18 servidor dnsmasq[12865]: dnsmasq: syntax check OK.
Oct 15 14:12:18 servidor dnsmasq[12873]: started, version 2.80 cachesize 150
Oct 15 14:12:18 servidor dnsmasq[12873]: DNS service limited to local subnets
Oct 15 14:12:18 servidor dnsmasq[12873]: compile time options: IPv6 GNU-getopt DBus i18n IDN DHCP DHCPv6 no-Lua TFTP conntrack ipset auth DNSSEC loop-detect inotify dumpfile
Oct 15 14:12:18 servidor dnsmasq-dhcp[12873]: DHCP, IP range 192.168.100.50 -- 192.168.100.150, lease time 12h
Oct 15 14:12:18 servidor dnsmasq-tftp[12873]: TFTP root is /srv/tftp
Oct 15 14:12:18 servidor dnsmasq[12873]: reading /etc/resolv.conf
Oct 15 14:12:18 servidor dnsmasq[12873]: using nameserver 10.0.2.3#53
Oct 15 14:12:18 servidor dnsmasq[12873]: read /etc/hosts - 5 addresses
Oct 15 14:12:18 servidor systemd[1]: Started dnsmasq - A lightweight DHCP and caching DNS server.
{% endhighlight %}

Efectivamente, el servicio se ha reiniciado correctamente y se encuentra activo. Además, si nos fijamos en los mensajes que ha devuelto, podemos apreciar que ha configurado correctamente el rango _DHCP_ especificado con su correspondiente tiempo de concesión y el protocolo _TFTP_ haciendo uso del directorio _/srv/tftp_.

Una vez que el servicio _dnsmasq_ está funcionando correctamente, es hora de descargar los ficheros necesarios en dicho directorio (_/srv/tftp_). Tras una detallada búsqueda, conseguí encontrar el [comprimido](http://ftp.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/netboot.tar.gz) que contiene los ficheros necesarios para la instalación por red. En este caso, son los ficheros para Debian **Buster**, aunque si quisiésemos cualquier otra distribución, bastaría con cambiar **buster** en la URL por la distribución deseada.

Dado que los ficheros los queremos descargar dentro de _/srv/tftp_, primero tendremos que movernos a dicho directorio. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@servidor:~# cd /srv/tftp/
{% endhighlight %}

Una vez dentro del mismo, ya podremos hacer uso de `wget` para descargar el fichero en cuestión:

{% highlight shell %}
root@servidor:/srv/tftp# wget http://ftp.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/netboot.tar.gz
--2020-10-15 14:16:21--  http://ftp.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/netboot.tar.gz
Resolving ftp.debian.org (ftp.debian.org)... 130.89.148.12, 2001:67c:2564:a119::148:12
Connecting to ftp.debian.org (ftp.debian.org)|130.89.148.12|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 37381616 (36M) [application/x-gzip]
Saving to: ‘netboot.tar.gz’

netboot.tar.gz                                       100%[========================================>]  35.65M  10.9MB/s    in 3.3s

2020-10-15 14:16:25 (10.9 MB/s) - ‘netboot.tar.gz’ saved [37381616/37381616]
{% endhighlight %}

Al parecer, el fichero ya ha terminado de descargarse, así que para verificarlo, vamos a hacer uso del comando `ls`:

{% highlight shell %}
root@servidor:/srv/tftp# ls
netboot.tar.gz
{% endhighlight %}

Efectivamente, el fichero está descargado en dicho directorio, pero los ficheros no son accesibles todavía, pues se encuentran comprimidos. Para descomprimir el **.tar.gz** y posteriormente eliminar el fichero comprimido (ya que no nos hará falta), ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# tar -zxf netboot.tar.gz && rm netboot.tar.gz
{% endhighlight %}

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.

Para verificar que se ha descomprimido correctamente y que el fichero comprimido se ha eliminado de dicho directorio, haremos uso de `ls -l`:

{% highlight shell %}
root@servidor:/srv/tftp# ls -l
total 8
drwxrwxr-x 3 root root 4096 Sep 20 23:11 debian-installer
lrwxrwxrwx 1 root root   47 Sep 20 23:11 ldlinux.c32 -> debian-installer/amd64/boot-screens/ldlinux.c32
lrwxrwxrwx 1 root root   33 Sep 20 23:11 pxelinux.0 -> debian-installer/amd64/pxelinux.0
lrwxrwxrwx 1 root root   35 Sep 20 23:11 pxelinux.cfg -> debian-installer/amd64/pxelinux.cfg
-rw-rw-r-- 1 root root   63 Sep 20 23:11 version.info
{% endhighlight %}

Como se puede apreciar, el fichero se ha descomprimido correctamente y tras ello, ha sido eliminado. Además, ya existe el fichero **pxelinux.0** que los clientes van a usar para arrancar a través de la red.

Llegados a este punto, ya tenemos una configuración totalmente funcional que permitiría a la máquina arrancar a través de la red, pero todavía nos falta llevar a cabo la configuración para automatizar dicha instalación. De igual forma, vamos a proceder a crear una máquina virtual que inicie a través de la red para asegurarnos que hasta ahora, no hemos cometido ningún error.

En este caso, he creado una máquina en VirtualBox sin ningún tipo de imagen ISO anexada, especificando el siguiente orden de arranque:

![boot](https://i.ibb.co/2WYgNsX/Captura-de-pantalla-de-2020-10-15-17-52-42.png "Orden de arranque")

Gracias al mismo, primero tratará de arrancar usando el disco duro, pero al no tener nada dentro, pasará a la siguiente opción, la **red**. Si el orden fuese al revés (primero red y luego disco duro), tras terminar la instalación tendríamos que volver a invertir el orden de arranque ya que volverá a intentar iniciar desde la red.

La otra parte de la configuración muy importante es la configuración de red. Tendremos que asegurarnos que el adaptador de red de la máquina virtual pertenezca a la red interna creada con anterioridad, "**lan1**", ya que de lo contrario, no podrá arrancar desde la red gracias al servidor previamente configurado.

![red](https://i.ibb.co/6Zfh39d/Captura-de-pantalla-de-2020-10-15-17-52-46.png "Red interna")

Una vez realizadas ambas configuraciones, podremos aplicar los cambios y tratar de iniciar la máquina para ver si inicia correctamente.

![dhcp](https://i.ibb.co/M5vFxKW/Virtual-Box-Debian-Test-15-10-2020-20-04-44.png "Petición DHCP")

Como se puede apreciar en el mensaje mostrado, la interfaz de red se ha levantado correctamente y está esperando a recibir una dirección por DHCP.

![instalador](https://i.ibb.co/Zdzc8Kk/Virtual-Box-Debian-Test-17-10-2020-09-57-52.png "Instalador Debian")

Tras esperar un par de segundos, la máquina virtual ha sido capaz de cargar el instalador por red de Debian, por lo que podemos concluir que hasta ahora, todo funciona correctamente, por lo que ya hemos realizado la mitad del recorrido.

Tras ello, nos queda configurar el **preseeding** para automatizar dicha instalación. Al fin y al cabo, lo que necesitamos el almacenar las respuestas a las preguntas que nos hace el instalador de Debian para posteriormente usarlo en las máquinas clientes. En este caso, he decidido usar un servidor web (**apache2**) para almacenar un fichero con dichas respuestas, aunque también sería posible introducir dichas respuestas en la propia imagen de arranque a través del **initrd.gz**, pero me parecía más interesante hacerlo con un servidor HTTP. Para instalar _apache2_ ejecutaremos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# apt install apache2
{% endhighlight %}

Una vez instalado, es importante mencionar que el **DocumentRoot** del **VirtualHost** que trae _apache2_ configurado por defecto se encuentra en _/var/www/html_, en castellano, esto significa que el directorio en el que debemos introducir los ficheros que deseamos servir es **/var/www/html**, por lo que dentro del mismo, debemos generar dicho fichero de configuración. En mi caso, el nombre que he decidido usar es **preseed.txt**. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# touch /var/www/html/preseed.txt
{% endhighlight %}

El fichero vacío ya se encuentra generado, así que ahora debemos introducir la configuración deseada en el mismo. Para ello, Debian nos ofrece una [plantilla](https://www.debian.org/releases/buster/example-preseed.txt) que podremos adaptar a nuestro gusto. Esta configuración es bastante personal, por lo que cada uno debe adaptarla a sus necesidades. En mi caso, copié la plantilla dentro de dicho fichero para posteriormente modificarlo, haciendo uso del comando:

{% highlight shell %}
root@servidor:/srv/tftp# nano /var/www/html/preseed.txt
{% endhighlight %}

En mi caso, he dejado la plantilla tal y como viene por defecto, exceptuando las siguientes modificaciones:

{% highlight shell %}
#Cambio el idioma y mi país, ya que por defecto estaba configurado en "en_US".

d-i debian-installer/locale string es_ES

#Cambio la distribución de teclado a la de España, ya que por defecto estaba configurado en "us".

d-i keyboard-configuration/xkb-keymap select es

#Modifico el repositorio (réplica) que voy a usar para la instalación, ya que por defecto estaba
#configurado para usar "http.us.debian.org".

d-i mirror/http/hostname string deb.debian.org

#Activo la creación de la cuenta de superusuario (root), ya que por defecto estaba desactivada.

d-i passwd/root-login boolean true

#Activo la creación de la cuenta de usuario personal, ya que por defecto estaba desactivada.

d-i passwd/make-user boolean true

#Introduzco una contraseña para el usuario root, en este caso, r00tme (en claro, aunque también
#se podría introducir cifrada haciendo uso de la opción "root-password-crypted").

d-i passwd/root-password password r00tme
d-i passwd/root-password-again password r00tme

#Introduzco un nombre real y un username para la nueva cuenta de usuario personal.

d-i passwd/user-fullname string Alvaro
d-i passwd/username string alvaro

#Introduzco una contraseña para el usuario alvaro, en este caso, prueba123123 (en claro, aunque también
#se podría introducir cifrada haciendo uso de la opción "user-password-crypted").

d-i passwd/user-password password prueba123123
d-i passwd/user-password-again password prueba123123

#Modifico la zona horaria a la mía, ya que por defecto era "US/Eastern". Los valores válidos los podemos
#encontrar en /usr/share/zoneinfo/.

d-i time/zone string Europe/Madrid

#Selecciono una de las "recetas" de particionado predefinidas para usar con LVM, en este caso, para
#separar el /home en un volumen lógico aparte, ya que por defecto iba a poner todos los ficheros en el
#mismo volumen lógico.

d-i partman-auto/choose_recipe select home

#Selecciono los paquetes que se van a instalar con tasksel (se recomienda instalar siempre las
#utilidades estándar del sistema), ya que por defecto estaba configurado para instalar un entorno
#gráfico (KDE) y un servidor web y ese no es mi objetivo.

tasksel tasksel/first multiselect standard, ssh-server

#Especifico que instale determinados paquetes además de los de tasksel (tree y dnsutils, por ejemplo).

d-i pkgsel/include string tree dnsutils

#Por último, indico que instale el GRUB en la primera partición disponible (ya que el particionado
#guiado de LVM crea una primera partición ext2 destinada para el GRUB).

d-i grub-installer/bootdev  string default
{% endhighlight %}

La configuración de preseeding ya está completada, pero todavía queda un factor muy importante a tener en cuenta. La máquina cliente tiene que ser capaz de salir al exterior para descargar la paquetería durante la instalación, ya que nosotros lo único que tenemos en el servidor es el propio instalador y el fichero de preseeding, no los paquetes necesarios. Dado que tenemos que mantener en todo momento la conectividad con el servidor (pues tendrá que hacer uso del fichero de preseeding alojado en el mismo), no podemos utilizar una interfaz de red secundaria con conectividad al exterior, ya que no tendría conectividad con el servidor. Para ello, mi solución ha sido configurar NAT en el servidor, de manera que la máquina cliente (cuya puerta de enlace predeterminada es la dirección del servidor, dado que así lo ha configurado dnsmasq) sea capaz de salir al exterior a través del servidor.

Lo primero que tendremos que hacer será activar el **bit de forward**, permitiendo así que los paquetes puedan pasar de una interfaz a otra en el servidor, ya que por defecto, en Debian viene deshabilitado por razones de seguridad. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# echo 1 > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}

Esta modificación no sería persistente, ya que se encuentra en memoria, por lo que si quisiésemos hacer que perdure en el tiempo, tendríamos que modificar el fichero **/etc/sysctl.conf** y asignar el valor `1` a la instrucción `net.ipv4.ip_forward`.

Tras ello, tendremos que instalar el paquete que nos va a permitir hacer NAT. En este caso se trata de **nftables**, un cortafuegos que consta con dicha funcionalidad. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# apt install nftables
{% endhighlight %}

Cuando el paquete haya terminado de instalarse, tendremos que arrancar y habilitar el demonio, de manera que se inicie cada vez que se arranque la máquina y lea la configuración almacenada en el fichero (lo veremos a continuación). Para ello, ejecutamos los comandos:

{% highlight shell %}
root@servidor:/srv/tftp# systemctl start nftables.service
root@servidor:/srv/tftp# systemctl enable nftables.service
{% endhighlight %}

Una vez habilitado y arrancado el demonio, tendremos que crear una nueva tabla en nftables de nombre **nat**. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft add table nat
{% endhighlight %}

Para verificar que la creación de dicha tabla se ha realizado correctamente, ejecutamos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft list tables
table inet filter
table ip nat
{% endhighlight %}

Efectivamente, así ha sido. Dentro de dicha tabla tendremos que crear la cadena **POSTROUTING**, es decir, aquella que permite modificar paquetes justo antes de que salgan del equipo, permitiendo por tanto hacer **Source NAT** (SNAT), para posteriormente alojar dentro de la misma, la regla que necesitamos. Para crear dicha cadena ejecutaremos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
{% endhighlight %}

En este caso, hemos especificado que esta cadena sea poco prioritaria (a mayor sea el número, menor es la prioridad), de manera que si en un futuro quisiéramos añadir una cadena **PREROUTING**, es decir, aquella que permite modificar paquetes entrantes antes de que se tome una decisión de enrutamiento, permitiendo por tanto hacer **Destination NAT** (DNAT), bastaría con indicarle a esta última una prioridad más alta, de manera que las reglas albergadas en el interior de la cadena **PREROUTING** se ejecuten antes que las de la cadena **POSTROUTING**.

De nuevo, vamos a verificar que dicha cadena se ha generado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft list chains
table inet filter {
	chain input {
		type filter hook input priority 0; policy accept;
	}
	chain forward {
		type filter hook forward priority 0; policy accept;
	}
	chain output {
		type filter hook output priority 0; policy accept;
	}
}
table ip nat {
	chain postrouting {
		type nat hook postrouting priority 100; policy accept;
	}
}
{% endhighlight %}

Nos queda un último paso, añadir la regla necesaria a la cadena POSTROUTING, de manera que nos permita hacer SNAT dinámico (**masquerade**), pues la dirección IP "pública" del servidor no siempre va a ser la misma, al haber sido asignada por DHCP a través del router virtual NAT de VirtualBox. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft add rule ip nat postrouting oifname "eth0" ip saddr 192.168.100.0/24 counter masquerade
{% endhighlight %}

En este caso, hemos especificado que se trata de una regla para la cadena **postrouting** (pues debe aplicarse justo antes de salir de la máquina), además, la interfaz (oifname) que debemos introducir será aquella por la que van a salir los paquetes, es decir, la que está conectada a Internet (aunque sea haciendo doble NAT, no es algo relevante en estas circunstancias), en este caso, **eth0**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que provengan de la red (ip saddr) **192.168.100.0/24**. Además, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a enmascararlos, pues la IP "pública" del router es dinámica (**masquerade**).

Por último, vamos a volver a verificar que dicha regla se ha añadido correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority 0; policy accept;
	}

	chain forward {
		type filter hook forward priority 0; policy accept;
	}

	chain output {
		type filter hook output priority 0; policy accept;
	}
}
table ip nat {
	chain postrouting {
		type nat hook postrouting priority 100; policy accept;
		oifname "eth0" ip saddr 192.168.100.0/24 counter packets 0 bytes 0 masquerade
	}
}
{% endhighlight %}

Efectivamente, así ha sido. Al igual que pasaba con el **bit de forward**, esta configuración se encuentra cargada en memoria, por lo que para que conseguir que perdure en el tiempo, vamos a ejecutar el comando:

{% highlight shell %}
root@servidor:/srv/tftp# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Gracias a ello, habremos guardado la configuración en un fichero en **/etc/** de nombre **nftables.conf** que se importará de forma automática cuando reiniciemos la máquina gracias al daemon que se encuentra habilitado. En caso de que no cargase de nuevo la configuración, podríamos hacerlo manualmente con el comando `nft -f /etc/nftables.conf`.

Una vez que se ha configurado NAT en el servidor, ya está todo listo para realizar la instalación. Para ello, volveremos a arrancar la máquina cliente, pero tenemos que realizar un paso muy importante, leer el fichero de preseeding, ya que en caso de simplemente pulsar en "**Install**", va a realizar una instalación común de forma interactiva. Para ello, cuando cargue el instalador, pulsaremos en **ESC** para que nos abra una prompt. Tras ello, tendremos que escribir una instrucción de la siguiente forma:

{% highlight shell %}
auto url=<servidor>/<fichero>
{% endhighlight %}

En este caso, mi instrucción queda de la forma:

![preseeding](https://i.ibb.co/s1GK2nQ/Virtual-Box-Debian-Prueba-15-10-2020-17-15-51.png "Preseeding")

Tras cargar la instrucción, la instalación comenzará a realizarse de forma totalmente automatizada sin preguntar nada, ya que todas las instrucciones que necesita las va a leer de dicho fichero.

Después de algunos minutos, la instalación finalizará y por tanto, la máquina virtual se reiniciará. Dado que en el momento de la creación de la máquina virtual configuramos correctamente el orden de arranque, ahora arrancará desde el disco duro, de manera que nos aparecerá el login al sistema y podremos acceder con las credenciales anteriormente especificadas:


![login](https://i.ibb.co/9ZBQ943/Virtual-Box-Debian-Test-16-10-2020-09-18-02-copia.png "Login")

Como se puede apreciar, estamos usando la ventana que nos proporciona VirtualBox para ver la máquina virtual, así que vamos a ir un paso más allá y nos vamos a conectar por **SSH** a la VM, pero para ello, necesitamos saber la dirección IP que ha obtenido por DHCP, ejecutando para ello el comando `ip a`:

![interfaces](https://i.ibb.co/wwGg3qF/Virtual-Box-Debian-Test-16-10-2020-09-18-02.png "ip a")

Como se puede apreciar, la interfaz de red que está conectada a la red interna es la de nombre "**enp0s3**", con dirección IP **192.168.100.92**, así que volveremos a la shell de la máquina virtual que actúa como servidor (muy importante realizar la conexión SSH desde el servidor, ya que desde la máquina anfitriona no es posible, al estar usando un doble NAT en el que los puertos para hacer DNAT no se encuentran "abiertos"). Para ello, ejecutaremos el comando:

{% highlight shell %}
vagrant@servidor:~$ ssh alvaro@192.168.100.92
The authenticity of host '192.168.100.92 (192.168.100.92)' can't be established.
ECDSA key fingerprint is SHA256:xZ+ob0JVGLvaqSQisKMdbsJ5LvY/6yx4pihNT4PQTjM.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.100.92' (ECDSA) to the list of known hosts.
alvaro@192.168.100.92's password: 
Linux debian 4.19.0-11-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Oct 16 09:17:48 2020
{% endhighlight %}

Ya nos encontramos dentro de la máquina cliente con un Debian instalado y totalmente funcional. Para finalizar la tarea, he decidido llevar a cabo algunas pruebas para demostrar que todo lo que hemos configurado en el fichero de preseeding realmente ha funcionado, así que lo primero que me gustaría demostrar son las contraseñas. Para ello, he creado un pequeño programa en Python haciendo uso del módulo **crypt** que nos servirá para encriptar una contraseña y compararla con la existente en el fichero **/etc/shadow**, que como todos sabemos, contiene un campo con la contraseña cifrada del usuario.

Lo primero que haremos será ver el contenido de dicho fichero, para obtener la contraseña cifrada para su posterior comparación. Para ello, haremos uso del comando:

{% highlight shell %}
root@debian:~# cat /etc/shadow
root:$6$QZZIVZ6MWDp8EM8E$uSt1WXen6HcNL4ue4XorvEy9pRnYOD/atAei8qB8WcP//9pF2yoYLAisvLcCXgGUvsAKdGtMoF74eXafFU4cT/:18551:0:99999:7:::
daemon:*:18551:0:99999:7:::
...
sshd:*:18551:0:99999:7:::
alvaro:$6$/.UbbixODq/k1Ysx$vdHgnfR.Vl1vijEWlMvrFuraLsse.ptZEBzuAWP.spkYQPkpqKDnjKoTT..RJXiOFDzxrsBuNUeHG0BKuPTYM1:18551:0:99999:7:::
systemd-coredump:!!:18551::::::
{% endhighlight %}

Como se puede apreciar, aquí tenemos (entre otras cosas), las contraseñas cifradas de **alvaro** y **root**. El programa que he creado es el siguiente:

{% highlight python %}
from crypt import crypt

password = input("Introduce una contraseña: ")
cifrada = input("Introduce la línea del /etc/shadow: ")

cifrada = cifrada.split(":")[1]

sal = cifrada[:cifrada.index("$",3)]

if crypt(password,sal) == cifrada:
    print("La contraseña introducida es correcta.")
else:
    print("La contraseña introducida es incorrecta.")
{% endhighlight %}

No voy a entrar a explicar su funcionamiento de forma detallada ya que no es el objetivo de este _post_, pero en pocas palabras, lo que hace, tal y como he mencionado antes, es encriptar una contraseña que le pasemos por teclado y comparar que el resultado de dicha encriptación es igual al almacenado en dicho fichero. Si coinciden, significa que la contraseña es la misma, si no lo hace, significa que las contraseñas son diferentes.

Primero lo comprobaré con el usuario **alvaro**, introduciendole al programa la línea completa del **/etc/shadow** referente (el propio programa se encargará de separar la contraseña del resto de campos) y la **contraseña** en claro, que en este caso es **prueba123123**.

{% highlight shell %}
alvaro@debian:~$ python3 descifrar.py 
Introduce una contraseña: prueba123123
Introduce la línea del /etc/shadow: alvaro:$6$/.UbbixODq/k1Ysx$vdHgnfR.Vl1vijEWlMvrFuraLsse.ptZEBzuAWP.spkYQPkpqKDnjKoTT..RJXiOFDzxrsBuNUeHG0BKuPTYM1:18551:0:99999:7:::
La contraseña introducida es correcta.
{% endhighlight %}

Como se puede apreciar, nos ha devuelto un mensaje indicando que la contraseña es correcta. Para verificar que el programa realmente funciona, voy a volver a hacer la misma prueba pero esta vez, introduciendo una contraseña incorrecta a propósito.

{% highlight shell %}
alvaro@debian:~$ python3 descifrar.py 
Introduce una contraseña: contraincorrecta
Introduce la línea del /etc/shadow: alvaro:$6$/.UbbixODq/k1Ysx$vdHgnfR.Vl1vijEWlMvrFuraLsse.ptZEBzuAWP.spkYQPkpqKDnjKoTT..RJXiOFDzxrsBuNUeHG0BKuPTYM1:18551:0:99999:7:::
La contraseña introducida es incorrecta.
{% endhighlight %}

Como era de esperar, ahora ha devuelto un mensaje informando que no es correcta. Volvemos a repetir las mismas pruebas con el usuario **root**, cuya contraseña es **r00tme**.

{% highlight shell %}
alvaro@debian:~$ python3 descifrar.py
Introduce una contraseña: r00tme
Introduce la línea del /etc/shadow: root:$6$QZZIVZ6MWDp8EM8E$uSt1WXen6HcNL4ue4XorvEy9pRnYOD/atAei8qB8WcP//9pF2yoYLAisvLcCXgGUvsAKdGtMoF74eXafFU4cT/:18551:0:99999:7:::
La contraseña introducida es correcta.
{% endhighlight %}

Como se puede apreciar, nos ha devuelto un mensaje indicando que la contraseña es correcta. Una vez más, vamos a comprobar que no funciona si introducimos una contraseña errónea.

{% highlight shell %}
alvaro@debian:~$ python3 descifrar.py
Introduce una contraseña: passroot
Introduce la línea del /etc/shadow: root:$6$QZZIVZ6MWDp8EM8E$uSt1WXen6HcNL4ue4XorvEy9pRnYOD/atAei8qB8WcP//9pF2yoYLAisvLcCXgGUvsAKdGtMoF74eXafFU4cT/:18551:0:99999:7:::
La contraseña introducida es incorrecta.
{% endhighlight %}

Genial, ya se ha demostrado que el funcionamiento de las contraseñas es el correcto, así que vamos a pasar a lo siguiente, el almacenamiento, para ello, haremos uso de los comandos `lsblk` y `lsblk -f`:

{% highlight shell %}
alvaro@debian:~$ lsblk
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                     8:0    0   20G  0 disk 
├─sda1                  8:1    0  243M  0 part /boot
├─sda2                  8:2    0    1K  0 part 
└─sda5                  8:5    0 19,8G  0 part 
  ├─debian--vg-root   254:0    0  6,9G  0 lvm  /
  ├─debian--vg-swap_1 254:1    0 1020M  0 lvm  [SWAP]
  └─debian--vg-home   254:2    0 11,9G  0 lvm  /home
sr0                    11:0    1 1024M  0 rom  
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ lsblk -f
NAME                  FSTYPE      LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda                                                                                           
├─sda1                ext2              c0887f4c-6c90-4b4b-90d2-efaf6aaca762      175M    20% /boot
├─sda2                                                                                        
└─sda5                LVM2_member       2dxeH6-T1K6-r5to-qX0j-pcFI-mWA8-D5PWCL                
  ├─debian--vg-root   ext4              9bfae775-eec2-40b1-b7f3-190c3f855bba      5,3G    16% /
  ├─debian--vg-swap_1 swap              4d58dfb0-0a07-4502-bcc3-9f7c57ff33aa                  [SWAP]
  └─debian--vg-home   ext4              c1422ad2-342a-4a1f-a531-6a2bfbebd651       11G     0% /home
sr0
{% endhighlight %}

Como se puede apreciar en el esquema de particionado, tenemos un disco duro virtual de nombre **sda** con una capacidad de **20G**, que fue la asignada en el momento de la creación, que cuenta con 3 particiones a su vez, siendo la primera de ellas (**sda1**) una ext2 con una capacidad de **243M** pensada para alojar el **/boot**, por lo que el GRUB se ha instalado en dicha ubicación gracias a la instrucción que especificamos en el fichero de preseeding. Por último, la tercera partición (**sda3**) es un dispositivo de bloques con una capacidad de **19.8G** perteneciente a un grupo de volúmenes LVM.

A su vez, dicho grupo de volúmenes se ha usado para crear 3 volúmenes lógicos, uno para el sistema de ficheros raíz con una capacidad de **6.9G**, otro para el área de intercambio, con una capacidad de alrededor de **1G** y otro para el /home, tal y como especificamos en el fichero de preseeding, con una capacidad de **11.9G**.

Estos valores son generados automáticamente por el particionamiento guiado de LVM que incorpora Debian, aunque en caso de haberlo necesitado, se podría haber especificado con detalle el esquema de particiones deseado en el fichero de preseeding, aunque se sale del objetivo de esta tarea.

Lo siguiente que me gustaría mostrar es el **sources.list** para verificar que realmente ha utilizado el mirror especificado (**deb.debian.org**) durante la instalación y además,ha llevado a cabo las modificaciones oportunas en dicho fichero para hacerlo persistente. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ cat /etc/apt/sources.list
# deb http://deb.debian.org/debian buster main

deb http://deb.debian.org/debian buster main
deb-src http://deb.debian.org/debian buster main

deb http://security.debian.org/debian-security buster/updates main
deb-src http://security.debian.org/debian-security buster/updates main

# buster-updates, previously known as 'volatile'
deb http://deb.debian.org/debian buster-updates main
deb-src http://deb.debian.org/debian buster-updates main
{% endhighlight %}

Efectivamente, así ha sido, ha configurado además los repositorios **deb-src** para ficheros fuente, aunque no sería necesario en este caso, por lo que se podrían comentar.

Por último, me gustaría comprobar que ha instalado los dos paquetes que se especificaron (**tree** y **dnsutils**), así que para ello, haremos uso de `apt policy`:

{% highlight shell %}
alvaro@debian:~$ apt policy tree
tree:
  Instalados: 1.8.0-1
  Candidato:  1.8.0-1
  Tabla de versión:
 *** 1.8.0-1 500
        500 http://deb.debian.org/debian buster/main amd64 Packages
        100 /var/lib/dpkg/status
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ apt policy dnsutils
dnsutils:
  Instalados: 1:9.11.5.P4+dfsg-5.1+deb10u2
  Candidato:  1:9.11.5.P4+dfsg-5.1+deb10u2
  Tabla de versión:
 *** 1:9.11.5.P4+dfsg-5.1+deb10u2 500
        500 http://deb.debian.org/debian buster/main amd64 Packages
        500 http://security.debian.org/debian-security buster/updates/main amd64 Packages
        100 /var/lib/dpkg/status
{% endhighlight %}

Como se puede apreciar, ambos paquetes han sido correctamente instalados, por lo que podemos concluir que la instalación se ha llevado a cabo siguiendo estrictamente los pasos indicados en el fichero de preseeding.