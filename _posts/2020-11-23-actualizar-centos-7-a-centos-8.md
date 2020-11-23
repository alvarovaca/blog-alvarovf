---
layout: post
title:  "Actualización de CentOS 7 a CentOS 8"
banner: "/assets/images/banners/centos.jpg"
date:   2020-11-23 20:21:00 +0200
categories: sistemas openstack
---
Se recomienda realizar una previa lectura del _post_ [Instalación y configuración del escenario OpenStack](http://www.alvarovf.com/hlc/openstack/2020/11/15/instalacion-escenario-openstack.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de ésta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una actualización de la máquina **Quijote**, una máquina totalmente limpia (en cuanto a servicios instalados se refiere) que tiene actualmente instalado un **_CentOS 7_**, una versión bastante antigua de la distribución, así que pasaremos a la versión más reciente, **_CentOS 8_**.

Antes de proceder, me gustaría dejar claro que la serie de instrucciones a ejecutar que mencionaré están basadas en mi experiencia personal, y no tienen por qué ser iguales para otra máquina, ya que entran en juego varios factores. Aún así, servirán para hacerse una idea del procedimiento a seguir y de los posibles problemas que pueden surgir por el camino.

Recomiendo muy encarecidamente realizar un _backup_ del servidor antes de proceder con la actualización, para que en el improbable caso de que la máquina quedase inútil, se pudiese recuperar su estado anterior. No considero necesario explicar cómo hacer un _backup_ ya que se saldría del objetivo de éste artículo.

Otra recomendación personal es que previamente se trate de llevar a cabo el proceso en una máquina lo más parecida posible que no se encuentre en producción, para así experimentar un poco antes de hacerlo en la máquina original, pudiendo anteponerse así, a posibles dificultades.

Antes de comenzar con la actualización, vamos a comprobar la _release_ de _CentOS_ actualmente instalada, para así verificar que realmente estamos haciendo uso de _CentOS 7_. Dicha información podremos encontrarla contenida en el fichero **/etc/redhat-release**, por lo que visualizaremos su contenido haciendo uso de `cat`:

{% highlight shell %}
[centos@quijote ~]$ cat /etc/redhat-release 
CentOS Linux release 7.8.2003 (Core)
{% endhighlight %}

Como se puede apreciar, actualmente estamos haciendo uso de la _release_ **7.8.2003**, por lo que podremos proceder a realizar la actualización a _CentOS 8_.

El primer paso será instalar el repositorio **EPEL** (_Extra Packages Enterprise Linux_), repositorio que nos ofrece paquetes de mucha utilidad que no están incluidos en los repositorios por defecto, que puede ser utilizado en varias distribuciones, entre las que se encuentra _CentOS_.

Para su instalación, haremos uso de la herramienta `yum`, el gestor de paquetes por defecto en _CentOS 7_, al que le pasaremos el enlace del fichero **.rpm** a instalar, o bien, el nombre de paquete **epel-release**.

En mi caso, he optado por la última opción, ya que es más cómoda, pues se encargará de descargar e instalar la última versión disponible del repositorio. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
[root@quijote ~]# yum install epel-release
{% endhighlight %}

Para verificar que el nuevo repositorio se ha añadido y habilitado correctamente, volveremos a hacer uso de dicho comando, ésta vez con la opción **repolist enabled**, que nos mostrará aquellos repositorios que se encuentran actualmente habilitados, estableciendo a su vez el correspondiente filtro por nombre:

{% highlight shell %}
[root@quijote ~]# yum repolist enabled | egrep 'epel'
epel/x86_64           Extra Packages for Enterprise Linux 7 - x86_64      13,464
{% endhighlight %}

Como se puede apreciar, el nuevo repositorio **EPEL** se encuentra correctamente añadido y habilitado.

El siguiente paso consiste en instalar dos paquetes que nos facilitarán la tarea de actualización:

* **rpmconf**: Herramienta que busca posibles conflictos en la configuración de un determinado paquete.
* **yum-utils**: Colección de utilidades que extienden las funcionalidades de _yum_.

Para llevar a cabo la correspondiente instalación, haremos uso del comando:

{% highlight shell %}
[root@quijote ~]# yum install rpmconf yum-utils
{% endhighlight %}

Tras ello, procederemos a hacer uso de la herramienta que acabamos de instalar para así comprobar que no existan conflictos en los ficheros de configuración de los paquetes, en cuyo caso tendríamos luz verde para proceder con la actualización. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# rpmconf -a
{% endhighlight %}

Donde:

* **-a**: Comprueba los ficheros de configuración de todos los paquetes instalados en el sistema.

En mi caso, no hubo ninguna salida en el comando, deduciendo por tanto que no existía ningún conflicto. En caso de haberlos, es recomendable marcar la opción **N**, manteniendo así el fichero de configuración original, aunque depende del sistema.

Posteriormente, haremos uso del comando `package-cleanup` existente en el paquete _yum-utils_ previamente instalado para así listar aquellas dependencias que ya no son necesarias (**--leaves**) y aquellos paquetes que no se encuentran disponibles en los repositorios actualmente configurados (**--orphans**), con la finalidad de desinstalarlos.

Es muy importante lo último que he mencionado, ya que he estado leyendo algunos artículos en Internet que se limitan a listar los paquetes, pero no le pasan la salida de dicho comando a `yum` para desinstalarlos, por lo que realmente, no están limpiando la máquina. Para ello, ejecutaremos los comandos:

{% highlight shell %}
[root@quijote ~]# yum remove `package-cleanup --leaves`
[root@quijote ~]# yum remove `package-cleanup --orphans`
{% endhighlight %}

Nuestra máquina ya está limpia de paquetes innecesarios, así que es importante saber que en _CentOS 8_, el gestor de paquetes predeterminado es `dnf`, en lugar de _yum_. Realmente, podríamos hacer uso de éste último sin problema alguno, ya que existe compatibilidad, pero por elección propia, he decidido cambiarlo. Para instalar el nuevo gestor de paquetes, haremos uso del comando:

{% highlight shell %}
[root@quijote ~]# yum install dnf
{% endhighlight %}

Dado que ya hemos instalado el nuevo gestor de paquetes, podremos proceder a desinstalar el anterior, ya que no nos será necesario, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# dnf remove yum yum-metadata-parser
{% endhighlight %}

A pesar de haber desinstado el paquete, podremos comprobar que todavía existe el correspondiente directorio en **/etc/**, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# ls -l /etc/yum
total 0
drwxr-xr-x. 2 root root 28 Nov 20 12:03 pluginconf.d
drwxr-xr-x. 2 root root 26 Nov 12 09:36 protected.d
drwxr-xr-x. 2 root root 37 Apr  2  2020 vars
{% endhighlight %}

Dado que ya no necesitamos dicho directorio, procederemos a eliminarlo, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# rm -rf /etc/yum
{% endhighlight %}

Donde:

* **-r**: Indica que la eliminación se lleve a cabo de forma recursiva, eliminando los subdirectorios y sus contenidos.
* **-f**: Evitamos que pida confirmación para cada uno de los ficheros contenidos, forzando por tanto su eliminación.

En un principio, el subdirectorio referente al paquete _yum_ ha sido ya eliminado del directorio **/etc/**, así que procederemos a comprobarlo una vez más, ejecutando para ello el comando utilizado con anterioridad:

{% highlight shell %}
[root@quijote ~]# ls -l /etc/yum
ls: cannot access /etc/yum: No such file or directory
{% endhighlight %}

Efectivamente, el subdirectorio ha sido eliminado. Cada vez estamos más cerca de proceder con la actualización como tal, pero antes de ello, vamos a asegurarnos de que toda la paquetería esté completamente actualizada, haciendo para ello uso del nuevo gestor de paquetes instalado:

{% highlight shell %}
[root@quijote ~]# dnf upgrade
{% endhighlight %}

Tras ello, habrá llegado el momento de instalar la última versión de los paquetes necesarios para _CentOS 8_ (_repos_, _release_ y _gpg-keys_), que podremos encontrar en el [mirror](http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/) oficial de _CentOS_. Su instalación la llevaremos a cabo ejecutando el comando:

{% highlight shell %}
[root@quijote ~]# dnf install \
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-repos-8.2-2.2004.0.2.el8.x86_64.rpm \
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-release-8.2-2.2004.0.2.el8.x86_64.rpm \
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-gpg-keys-8.2-2.2004.0.2.el8.noarch.rpm
{% endhighlight %}

Una vez que la instalación de los paquetes referentes a la versión **8.2.2004** ha finalizado, tendremos que actualizar también la versión del repositorio **EPEL**, para que dicha versión sea ahora la que usaremos en _CentOS 8_. Para ello, volveremos a hacer uso del comando:

{% highlight shell %}
[root@quijote ~]# dnf upgrade epel-release
{% endhighlight %}

Posteriormente, vamos a limpiar los archivos temporales existentes en la máquina, como consecuencia de la paquetería que se ha tenido que descargar y que habrá quedado cacheada, ocupando espacio en disco de manera innecesaria. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# dnf clean all
62 files removed
{% endhighlight %}

Como se puede apreciar, se han eliminado un total de **62** archivos que no eran necesarios.

Todavía no hemos terminado, ya que la actualización a _CentOS 8_, implica a su vez una actualización de _kernel_, así que procederemos a eliminar todos los paquetes asociados al que estamos utilizando actualmente, pues no vamos a necesitarlo más, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# rpm -e `rpm -q kernel`
{% endhighlight %}

Donde:

* **-e**: Indicamos que queremos desinstalar el paquete introducido.
* **-q**: Nos mostrará todos aquellos paquetes que contengan una determinada cadena en su nombre, en este caso, **kernel**.

En un principio, nos queda un último paquete por desinstalar, y se trata de **sysvinit-tools**, que contiene las herramientas necesarias para el arranque básico de la máquina, el cuál desinstalaremos para evitar posibles conflictos, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# rpm -e --nodeps sysvinit-tools
{% endhighlight %}

Donde:

* **--nodeps**: Indicaremos que no revise las dependencias antes de desinstalar el paquete.

Lo que hemos estado haciendo hasta ahora son los preparativos necesarios para la actualización a _CentOS 8_, así que ha llegado el momento de llevarla a cabo. Para ello, haremos uso del comando:

{% highlight shell %}
[root@quijote ~]# dnf --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
...
Error: Transaction check error:
  file /usr/lib/python3.6/site-packages/rpmconf/__pycache__/__init__.cpython-36.opt-1.pyc from install of python3-rpmconf-1.0.21-1.el8.noarch conflicts with file from package python36-rpmconf-1.0.22-1.el7.noarch
  file /usr/lib/python3.6/site-packages/rpmconf/__pycache__/__init__.cpython-36.pyc from install of python3-rpmconf-1.0.21-1.el8.noarch conflicts with file from package python36-rpmconf-1.0.22-1.el7.noarch
  file /usr/lib/python3.6/site-packages/rpmconf/__pycache__/rpmconf.cpython-36.opt-1.pyc from install of python3-rpmconf-1.0.21-1.el8.noarch conflicts with file from package python36-rpmconf-1.0.22-1.el7.noarch
  file /usr/lib/python3.6/site-packages/rpmconf/__pycache__/rpmconf.cpython-36.pyc from install of python3-rpmconf-1.0.21-1.el8.noarch conflicts with file from package python36-rpmconf-1.0.22-1.el7.noarch
  file /usr/share/man/man3/rpmconf.3.gz from install of python3-rpmconf-1.0.21-1.el8.noarch conflicts with file from package python36-rpmconf-1.0.22-1.el7.noarch
{% endhighlight %}

Donde:

* **--releasever**: Configura _dnf_ como si la _release_ fuese la introducida. Ésto puede afectar a las rutas de caché, los valores de los ficheros de configuración y los enlaces de repositorios. En éste caso, la versión es la **8**.
* **--allowerasing**: Permite que se eliminen determinados paquetes instalados para resolver dependencias en caso de ser necesario.
* **--setopt**: Establece una opción que prevalece sobre aquella especificada en el fichero de configuración. En éste caso, desactivamos la opción **deltarpm**, cuya finalidad era ahorrar ancho de banda, pues descargaba ficheros _delta_ que posteriormente tendría que reconstruir de manera local, un trabajo bastante intensivo para la CPU, que al tratarse en ésta ocasión de una gran cantidad de ficheros, no sería eficiente.

Como se puede apreciar en la salida del comando, han habido algunos conflictos con un paquete de _Python_ previamente instalado en _CentOS 7_, concretamente con **python36-rpmconf-1.0.22-1.el7.noarch**, el cuál procederemos a desinstalar, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# dnf remove python36-rpmconf-1.0.22-1.el7.noarch
{% endhighlight %}

Tras ello, volveremos a ejecutar el comando para realizar la actualización, para intentar llevarla a cabo de forma exitosa en ésta ocasión:

{% highlight shell %}
[root@quijote ~]# dnf --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
{% endhighlight %}

Tras alrededor de 10 minutos de espera, la actualización ha finalizado sin ningún tipo de error. Antes de llevar a cabo las comprobaciones oportunas, tenemos que instalar la nueva versión del _kernel_, ya que en memoria tenemos cargada la versión de _kernel_ correspondiente a _CentOS 7_, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# dnf install kernel-core
{% endhighlight %}

Por último, vamos a tratar de actualizar dos grupos de paquetes (colección de paquetes que tienen un propósito común), para así asegurar que tenemos las utilidades mínimas para el correcto funcionamiento del sistema. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# dnf groupupdate "Core" "Minimal Install"
{% endhighlight %}

La actualización a _CentOS 8_ ha finalizado, así que para verificar que actualmente estamos haciendo uso de la nueva versión, volveremos a visualizar el contenido del fichero **/etc/redhat-release**, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# cat /etc/redhat-release
CentOS Linux release 8.2.2004 (Core)
{% endhighlight %}

Como se puede apreciar, nos encontramos haciendo uso de la _release_ **8.2.2004**. Tras ello, verificaremos el _kernel_ que actualmente estamos utilizando, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# uname -r
3.10.0-1127.19.1.el7.x86_64
{% endhighlight %}

Donde:

* **-r**: Le pedimos que nos muestre la _release_ en lugar del nombre.

Como era de esperar, la versión de _kernel_ cargada en memoria es la correspondiente a la instalada en _CentOS 7_, de manera que para cargar el nuevo núcleo instalado, no tendremos más remedio que reiniciar la máquina, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# reboot
{% endhighlight %}

Tras ello, volveremos a comprobar la versión de _kernel_ cargada en memoria, ejecutando para ello el comando visto con anterioridad:

{% highlight shell %}
[centos@quijote ~]$ uname -r
4.18.0-193.28.1.el8_2.x86_64
{% endhighlight %}

Una vez completado el reinicio, la nueva versión del núcleo ha sido correctamente cargada en memoria, por lo que podemos asegurar que la actualización se ha completado exitosamente.

Podríamos dejar aquí el artículo, ya que la actualización se ha completado, pero personalmente, considero necesario comprobar que las configuraciones llevadas a cabo previamente en la máquina siguen estando activas y funcionales, así que comenzaremos por listar la información correspondiente a las interfaces de red existentes en la máquina, haciendo para ello uso del comando:

{% highlight shell %}
[centos@quijote ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:3b:bb:e2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.5/24 brd 10.0.1.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe3b:bbe2/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la máquina tiene configurada una interfaz **eth0** con direccionamiento privado **10.0.1.5**, configurado de forma estática, tal y como establecimos en el anterior artículo.

Al tratarse de una máquina ubicada en una red privada, vamos a verificar que sus tablas de enrutamiento tienen la correspondiente entrada para asegurar la conectividad con el exterior, mediante la máquina que actúa como router (**Dulcinea**), ubicada en **10.0.1.9**, ejecutando para ello el comando:

{% highlight shell %}
[centos@quijote ~]$ ip r
default via 10.0.1.9 dev eth0 proto static metric 100 
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.5 metric 100
{% endhighlight %}

Efectivamente, su configuración es la correcta, por lo que en un principio debería tener salida al exterior. Para comprobarlo, vamos a hacer `ping` a una dirección que sabemos que nos va a responder, como por ejemplo la **8.8.8.8**:

{% highlight shell %}
[centos@quijote ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=776 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=560 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=111 time=663 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 3 received, 25% packet loss, time 1002ms
rtt min/avg/max/mdev = 560.030/666.278/776.012/88.208 ms
{% endhighlight %}

Como era de esperar, la máquina sigue teniendo conectividad con el exterior.

La siguiente comprobación consistirá en verificar que la máquina tiene correctamente configurado el **_hostname_** y el **_FQDN_**, haciendo para ello uso de los comandos:

{% highlight shell %}
[centos@quijote ~]$ hostname
quijote

[centos@quijote ~]$ hostname -f
quijote.alvaro.vaca.gonzalonazareno.org
{% endhighlight %}

Una vez más, la configuración es correcta.

He considerado necesario también el hecho de verificar que la resolución estática de nombres a las máquinas **Dulcinea** y **Sancho** sigue funcionando, así que trataremos de alcanzarlas mediante un `ping`:

{% highlight shell %}
[centos@quijote ~]$ ping dulcinea
PING dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=1 ttl=64 time=0.496 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=2 ttl=64 time=0.806 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=3 ttl=64 time=0.665 ms
^C
--- dulcinea.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 82ms
rtt min/avg/max/mdev = 0.496/0.655/0.806/0.130 ms

[centos@quijote ~]$ ping sancho
PING sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=64 time=4.01 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=64 time=1.22 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=64 time=0.895 ms
^C
--- sancho.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.895/2.042/4.012/1.399 ms
{% endhighlight %}

La resolución estática de nombres ha funcionado correctamente, de manera que nos quedaría comprobar que la resolución mediante servidores DNS también lo hace, tratando de llevar a cabo para ello un `ping` a **google.es**, por ejemplo:

{% highlight shell %}
[centos@quijote ~]$ ping google.es
{% endhighlight %}

Al parecer, hay un problema con la resolución de nombres (podemos asegurar que no es de conectividad ya que anteriormente hemos sido capaces de alcanzar a **8.8.8.8**). Lo primero que haremos será revisar el fichero en el que se establecen los servidores DNS a los que preguntar, es decir, el fichero **/etc/resolv.conf**, ejecutando para ello el comando:

{% highlight shell %}
[centos@quijote ~]$ cat /etc/resolv.conf
search openstacklocal
nameserver 192.168.200.2
{% endhighlight %}

Como se puede apreciar, ha habido un problema, y es que únicamente ha configurado 1 de los 3 servidores DNS (el cuál no se encuentra operativo) previamente especificados en el fichero de configuración de la interfaz de red, muy posiblemente, debido a alguna configuración del mecanismo **cloud-init** existente en _CentOS 8_ pero no en _CentOS 7_.

Tras algo de investigación al respecto, descubrí que la solución era bastante sencilla. Lo primero que haremos será limpiar el fichero de configuración de la interfaz de red, es decir, **/etc/sysconfig/network-scripts/ifcfg-eth0**, eliminando todas las entradas referentes a los servidores DNS. Para ello, haremos uso del comando:

{% highlight shell %}
[centos@quijote ~]$ sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0
{% endhighlight %}

El contenido actual es el siguiente:

{% highlight shell %}
BOOTPROTO=none
DEVICE=eth0
HWADDR=fa:16:3e:3b:bb:e2
MTU=8950
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
IPADDR=10.0.1.5
PREFIX=24
GATEWAY=10.0.1.9
DNS1=192.168.202.2
DNS2=192.168.200.2
DNS3=8.8.8.8
{% endhighlight %}

El contenido tras eliminar las entradas referentes a los servidores DNS es el siguiente:

{% highlight shell %}
BOOTPROTO=none
DEVICE=eth0
HWADDR=fa:16:3e:3b:bb:e2
MTU=8950
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
IPADDR=10.0.1.5
PREFIX=24
GATEWAY=10.0.1.9
{% endhighlight %}

Tras ello, guardaremos los cambios y procederemos a modificar directamente el fichero **/etc/resolv.conf**, estableciendo en el mismo los servidores DNS que queremos utilizar, que serán los mismos que estaban previamente definidos en el fichero que acabamos de modificar. Para ello, ejecutaremos el comando:

{% highlight shell %}
[centos@quijote ~]$ sudo vi /etc/resolv.conf
{% endhighlight %}

El resultado final sería el siguiente:

{% highlight shell %}
search openstacklocal
nameserver 192.168.202.2
nameserver 192.168.200.2
nameserver 8.8.8.8
{% endhighlight %}

Cuando hayamos terminado la modificación del mismo, guardaremos los cambios y volveremos a tratar de hacer `ping` a **google.es**, para comprobar si la resolución se lleva ahora a cabo de forma correcta:

{% highlight shell %}
[centos@quijote ~]$ ping google.es
PING google.es (216.58.211.35) 56(84) bytes of data.
64 bytes from muc03s14-in-f3.1e100.net (216.58.211.35): icmp_seq=1 ttl=112 time=43.2 ms
64 bytes from muc03s14-in-f3.1e100.net (216.58.211.35): icmp_seq=2 ttl=112 time=43.3 ms
64 bytes from muc03s14-in-f3.1e100.net (216.58.211.35): icmp_seq=3 ttl=112 time=43.5 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 466ms
rtt min/avg/max/mdev = 43.207/43.364/43.544/0.277 ms
{% endhighlight %}

Efectivamente, la resolución del nombre **google.es** se ha realizado correctamente, además de haber conseguido que dicha configuración perdure incluso tras un reinicio.

Por último, voy a verificar que el reloj del sistema esté correctamente sincronizado, pues fue la última configuración que llevé a cabo en _CentOS 7_, haciendo para ello uso del comando:

{% highlight shell %}
[centos@quijote ~]$ timedatectl
               Local time: Sun 2020-11-22 13:50:03 CET
           Universal time: Sun 2020-11-22 12:50:03 UTC
                 RTC time: Sun 2020-11-22 12:50:03
                Time zone: Europe/Madrid (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
{% endhighlight %}

Como era de esperar, el reloj se encuentra correctamente sincronizado (**System clock synchronized**) con la zona horaria de **Madrid**, por lo que la actualización de _CentOS 7_ a _CentOS 8_ y su correspondiente comprobación de funcionamiento ha llegado hasta aquí.