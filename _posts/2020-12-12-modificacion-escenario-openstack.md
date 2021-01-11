---
layout: post
title:  "Modificación del escenario OpenStack"
banner: "/assets/images/banners/openstack.png"
date:   2020-12-12 22:00:00 +0200
categories: hlc openstack
---
Se recomienda realizar una previa lectura del _post_ [Instalación y configuración del escenario OpenStack](http://www.alvarovf.com/hlc/openstack/2020/11/15/instalacion-escenario-openstack.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una modificación para añadir una nueva máquina de nombre **Freston** que estará ubicada dentro de la red interna previamente creada (**10.0.1.0/24**). No ocurrirá lo mismo con **Quijote**, que sufrirá una modificación y se encontrará a partir de ahora en una nueva **red DMZ** de direccionamiento **10.0.2.0/24** que generaremos. La infraestructura a la que se pretende llegar es la siguiente:

![escenario1](https://i.ibb.co/ZNTJWD5/escenario2.png "Escenario OpenStack")

- **Dulcinea**:
    - _Debian Buster_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Acessible a través de la red externa _10.0.0.0/24_ con una IP flotante.
    - Conectada a la red interna _10.0.1.0/24_.
    - Conectada a la red DMZ _10.0.2.0/24_.
- **Sancho**:
    - _Ubuntu 20.04_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Conectada a la red interna _10.0.1.0/24_.
- **Quijote**:
    - _CentOS 8_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Conectada a la red DMZ _10.0.2.0/24_.
- **Freston**:
    - _Debian Buster_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Conectada a la red interna _10.0.1.0/24_.

Como se puede apreciar, **Dulcinea** también sufre de manera indirecta una pequeña modificación, pues es necesario conectarla a la nueva **red DMZ** para que pueda así **Quijote** salir al exterior, al estar actuando dicha máquina como _router_.

## Creación de la red DMZ

Para ello, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, pulsaremos en el botón **Crear red** y estableceremos los siguientes parámetros:

- **Red**:
    - **Nombre de la red**: DMZ de alvaro.vaca
    - **Estado de administración**: ARRIBA
    - **Compartido**: _No_
    - **Crear subred**: _Sí_
- **Subred**:
    - **Nombre de subred**: _Vacío_
    - **Direcciones de red**: 10.0.2.0/24
    - **Versión de IP**: IPv4
    - **Deshabilitar puerta de enlace**: _Sí_
- **Detalles de Subred**:
    - **Habilitar DHCP**: _Sí_
    - **Pools de asignación**: 10.0.2.2,10.0.2.254
    - **Servidores DNS**: 192.168.200.2

Gracias a la configuración mencionada, habremos habilitado un servidor DHCP para que sirva direcciones IP a las máquinas pertenecientes a dicha red, aunque posteriormente configuraremos un direccionamiento estático.

De otro lado, hemos evitado que dicho servidor DHCP sirva una puerta de enlace predeterminada a las máquinas (pues todavía no conocemos cuál va a ser, ya que dependemos de la dirección que se asigne a **Dulcinea**). El resultado final sería:

![redes1](https://i.ibb.co/dD2B6rv/Captura-de-pantalla-de-2020-12-09-09-17-28.png "Creación de red")

Genial, ya hemos creado la **red DMZ** que vamos a necesitar, por lo que podemos concluir que las tres redes (**externa**, **interna** y **DMZ**) se encuentran ahora configuradas y operativas, consiguiendo que por tanto, la infraestructura del escenario sea más similar a la de una situación real, pues en posteriores artículos configuraremos un cortafuegos para controlar el tráfico que circula.

## Modificación de la subred de la red interna, habilitando el servidor DHCP

Si recordamos, en el primer artículo referente a la instalación y configuración del escenario OpenStack deshabilitamos el servidor DHCP en la subred de la red interna, pero ahora nos será necesario tenerlo de nuevo activo, para que así se asignen automáticamente direcciones dentro del rango a las máquinas conectadas, concretamente a **Freston**, la nueva máquina incorporada.

Para habilitarlo, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, buscaremos la red interna (**red interna de alvaro.vaca**) y pulsaremos en el nombre de la misma. Cuando hayamos accedido a la red, tendremos varios subapartados, así que accederemos a **Subredes** y nos aparecerá la única que hemos creado, por lo que procederemos a editarla pulsando en **Editar subred**.

Se nos habrá abierto un pequeño menú, así que iremos a **Detalles de subred**, pues es donde se encuentra la configuración referente al servidor DHCP de la subred. Simplemente marcaremos la casilla **Habilitar DHCP** y guardaremos los cambios, pulsando para ello en **Guardar**. El servidor DHCP ya habrá sido habilitado, tal y como se puede apreciar:

![dhcp1](https://i.ibb.co/D47TZtb/dhcpenabled.jpg "Habilitar DHCP")

## Adición de Dulcinea a la red DMZ

Como anteriormente hemos mencionado, es necesario que **Dulcinea**, el router, tenga una interfaz conectada a la nueva red que hemos creado, ya que de lo contrario, las máquinas que se encuentren en dicha red no podrán ser alcanzadas ni podrán salir al exterior.

Para ello, mostraremos la lista de nuestras instancias creadas accediendo al apartado **COMPUTE** en el menú izquierdo, dentro del cuál nos ubicaremos en **Instancias**. Tendremos que pulsar en el símbolo **▾** en Dulcinea, para así abrir el desplegable. Tras ello, pulsaremos en la opción **Conectar interfaz** y seleccionamos la Red de nombre "**DMZ de alvaro.vaca**". El resultado final sería:

![instancias1](https://i.ibb.co/Fb1WP3g/interfaz.jpg "Adición Dulcinea a DMZ")

Como se puede apreciar, las direcciones IP asignadas a dicha máquina han sido las siguientes:

- **Dulcinea**:
    - 10.0.0.7 - 172.22.200.134
    - 10.0.1.9
    - 10.0.2.12

La configuración todavía no ha terminado, ya que tendremos que acceder a la máquina para verificar que se le ha asignado correctamente dicha dirección IP, además de configurarle un direccionamiento estático a la misma, al igual que ocurre con el resto de sus interfaces.

Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
debian@dulcinea:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:e2:f6:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.7/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee2:f6e7/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:b5:5d:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.9/24 brd 10.0.1.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:feb5:5de7/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fa:16:3e:70:ef:a2 brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

Efectivamente, contamos con **tres** interfaces:

* **eth0**: Conectada a la red externa, con dirección IP **10.0.0.7**.
* **eth1**: Conectada a la red interna, con dirección IP **10.0.1.9**.
* **eth2**: Conectada a la red DMZ, sin dirección IP ya que no la hemos configurado.

El motivo por el que la interfaz **eth2** se encuentra actualmente apagada y sin ninguna dirección IP asignada es debido a que previamente hemos definido una configuración de direccionamiento estático para las interfaces, entre las que todavía no ha sido declarada dicha interfaz y por tanto, no sabe cuál es su configuración.

El fichero en el que debemos establecer la configuración de la nueva interfaz de red en una máquina Debian Buster es **/etc/network/interfaces**, así que lo modificaremos ejecutando el comando (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
root@dulcinea:~# nano /etc/network/interfaces
{% endhighlight %}

El contenido previamente establecido en el interior será:

{% highlight shell %}
allow-hotplug eth0
iface eth0 inet static
        address 10.0.0.7
        netmask 255.255.255.0
        gateway 10.0.0.1

allow-hotplug eth1
iface eth1 inet static
        address 10.0.1.9
        netmask 255.255.255.0
{% endhighlight %}

Como se puede apreciar, en el interior del mismo estaban declaradas las interfaces **eth0** y **eth1**, pero no **eth2**, por lo que añadiremos las siguientes líneas:

{% highlight shell %}
allow-hotplug eth2
iface eth2 inet static
        address 10.0.2.12
        netmask 255.255.255.0
{% endhighlight %}

Como se puede apreciar, hemos declarado una nueva interfaz de red perteneciente a la red DMZ, cuya dirección IP estática es la misma que se le concedió en su momento por DHCP, **10.0.2.12**. Es importante mencionar que la puerta de enlace únicamente debe ser configurada para la interfaz **eth0**, que es la que tiene salida a Internet, concretamente a través del router situado en **10.0.0.1**.

Genial, todos las modificaciones referentes a las interfaces de red han finalizado, por lo que es hora de aplicar los nuevos cambios reiniciando para ello la máquina al completo (comando `reboot`), para así verificar además que los cambios perduran tras un reinicio.

Tras unos segundos de espera mientras la máquina se reiniciaba, ya podremos apreciar los cambios que hemos realizado, ejecutando para ello el comando `ip a` para verificar que las direcciones IP estáticas se han asignado correctamente:

{% highlight shell %}
root@dulcinea:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:e2:f6:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.7/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee2:f6e7/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:b5:5d:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.9/24 brd 10.0.1.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:feb5:5de7/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:70:ef:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.12/24 brd 10.0.2.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe70:efa2/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, las direcciones han sido correctamente configuradas estáticamente (podemos deducir que son estáticas ya que su **valid_lft** es **forever**).

## Creación y configuración de Freston

El primer paso será crear el volumen que utilizará la correspondiente instancia. Para ello, accederemos en el menú izquierdo al apartado **COMPUTE**, dentro del cuál pulsaremos en **Volúmenes**. Una vez ahí, pulsaremos en el botón **Crear volumen** y estableceremos los siguientes parámetros:

- **Freston**:
    - **Nombre del volumen**: Freston
    - **Descripción**: _Vacío_
    - **Origen del volumen**: Imagen
    - **Utilizar una imagen como origen**: Debian Buster 10.6 (516,5 MB)
    - **Tipo**: Sin tipo de volumen
    - **Tamaño (GiB)**: 10
    - **Zona de Disponibilidad**: nova

El resultado final tras haber generado el volumen sería:

![volumenes1](https://i.ibb.co/3022yJy/Captura-de-pantalla-de-2020-12-09-09-35-49.png "Creación del volumen")

El volumen que va a ser utilizado por la instancia ya se encuentra generado, así que no queda nada más por hacer antes de proceder a crear la correspondiente instancia, por lo que vamos a ponernos manos a la obra.

Para ello, accederemos en el menú izquierdo al apartado **COMPUTE**, dentro del cuál pulsaremos en **Instancias**. Una vez ahí, pulsaremos en el botón **Lanzar instancia** y estableceremos los siguientes parámetros:

- **Freston**:
    - **Details**:
        - **Instance Name**: Freston
        - **Zona de Disponibilidad**: nova
        - **Count**: 1
    - **Source**:
        - **Seleccionar Origen de arranque**: Volumen
        - **Delete Volume on Instance Delete**: No
        - **Allocated**: Freston
    - **Sabor**:
        - **Allocated**: m1.mini
    - **Redes**:
        - **Allocated**: red interna de alvaro.vaca
    - **Par de claves**:
        - **Allocated**: Linux

La instancia ya ha sido generada, de manera que el resultado final sería:

![instancias2](https://i.ibb.co/G0G0TGf/Captura-de-pantalla-de-2020-12-09-10-00-06.png "Creación de Freston")

Finalmente, la dirección IP asignada a dicha máquina ha sido la siguiente:

- **Freston**:
    - 10.0.1.7

Una vez creada la nueva máquina, es hora de llevar a cabo la _prueba de fuego_, consistente en comprobar la conectividad con la misma, de manera que haremos uso del comando `ping` desde la máquina _Dulcinea_ para probar la conectividad con **10.0.1.7** (_Freston_):

{% highlight shell %}
root@dulcinea:~# ping 10.0.1.7
PING 10.0.1.7 (10.0.1.7) 56(84) bytes of data.
64 bytes from 10.0.1.7: icmp_seq=1 ttl=64 time=5.59 ms
64 bytes from 10.0.1.7: icmp_seq=2 ttl=64 time=1.01 ms
64 bytes from 10.0.1.7: icmp_seq=3 ttl=64 time=1.02 ms
^C
--- 10.0.1.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.007/2.539/5.587/2.155 ms
{% endhighlight %}

Efectivamente, existe conectividad entre las máquinas, así que para empezar a trabajar en la nueva instancia, vamos a verificar que también podemos llevar a cabo una conexión `ssh` a la misma, tratando de acceder a **10.0.1.7** (_Freston_):

{% highlight shell %}
root@dulcinea:~# ssh debian@10.0.1.7
Linux freston 4.19.0-11-cloud-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Como era de esperar, la conexión **ssh** también se ha llevado a cabo correctamente, gracias a haber exportado nuestro agente de claves de la máquina anfitriona, de manera que **Dulcinea** puede hacer uso de las mismas.

Dado que la instancia no viene con una contraseña configurada, tendremos que establecerla manualmente para así evitar perder el acceso a la misma en caso que hubiese algún tipo de problema, pues en ese caso, podríamos acceder desde **Horizon**. El comando a ejecutar para establecer una contraseña al usuario **debian** sería:

{% highlight shell %}
root@freston:~# passwd debian
New password:
Retype new password:
passwd: password updated successfully
{% endhighlight %}

La contraseña al usuario **debian** ha sido correctamente asignada, tal y como se puede apreciar en la salida del comando.

De otro lado, la dirección IP asignada a la interfaz de red de la máquina ha sido concedida por DHCP, pero como todos sabemos, no es buena idea dejar direcciones dinámicas en los servidores, por lo que vamos a configurar direccionamiento estático en la misma (concretamente haciendo uso de la misma dirección que se nos ha concedido por DHCP).

El fichero de configuración en el que debemos establecer la configuración de las interfaces de red de una máquina Debian Buster es **/etc/network/interfaces**, así que lo modificaremos ejecutando el comando:

{% highlight shell %}
root@freston:~# nano /etc/network/interfaces
{% endhighlight %}

Dentro del mismo debemos introducir las siguientes líneas:

{% highlight shell %}
allow-hotplug eth0
iface eth0 inet static
        address 10.0.1.7
        netmask 255.255.255.0
        gateway 10.0.1.9
{% endhighlight %}

Como se puede apreciar, hemos declarado una interfaz de red perteneciente a la red interna, **eth0**, cuya dirección IP estática es la misma que se le concedió en su momento por DHCP, **10.0.1.7**. Es importante mencionar que la puerta de enlace ha de ser la dirección de la máquina **Dulcinea** perteneciente a la red interna (concretamente, **10.0.1.9**), de manera que podrá hacer _SNAT_ con los paquetes que le lleguen.

En cuanto a la resolución de nombres, las instancias Debian Buster de OpenStack vienen con un fichero **/etc/resolv.conf** dinámico, lo que quiere decir que se genera dinámicamente en cada arranque, conteniendo el mismo un único servidor DNS existente en el instituto, situado en la dirección **192.168.200.2**, pero personalmente, he tomado la decisión de añadir un segundo y tercer servidor DNS, para aumentar así la disponibilidad.

Al ser dinámico dicho fichero, no podemos modificarlo a mano, sino que tendremos que modificar el fichero **/etc/resolvconf/resolv.conf.d/base** e indicarlos ahí, de manera que lo tendrá en cuenta para la siguiente ocasión, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano /etc/resolvconf/resolv.conf.d/base
{% endhighlight %}

En este caso, los servidores DNS que he decidido añadir son **192.168.202.2** y **8.8.8.8**, por lo que la apariencia de dicho fichero sería:

{% highlight shell %}
nameserver 192.168.202.2
nameserver 8.8.8.8
{% endhighlight %}

Genial, todos los cambios referentes a la red se han llevado a cabo, pero para que surtan efecto podemos reiniciar la máquina (comando `reboot`) o bien reiniciar el servicio correspondiente, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# systemctl restart networking
{% endhighlight %}

Tras unos segundos de espera mientras el servicio se reiniciaba, ya podremos apreciar los cambios que hemos realizado, ejecutando el comando `ip a` para verificar que la dirección IP estática se ha asignado correctamente:

{% highlight shell %}
root@freston:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:99:2b:18 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.7/24 brd 10.0.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe99:2b18/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la dirección ha sido correctamente configurada estáticamente (podemos deducir que es estática ya que su **valid_lft** es **forever**). Además, vamos a comprobar que la puerta de enlace predeterminada también se ha configurado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ip r
default via 10.0.1.9 dev eth0 onlink
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.7
169.254.169.254 via 10.0.1.3 dev eth0
{% endhighlight %}

Efectivamente, así ha sido. Por último, vamos a comprobar los servidores DNS que se encuentran indicados en el fichero **/etc/resolv.conf**, haciendo uso del comando:

{% highlight shell %}
root@freston:~# cat /etc/resolv.conf
nameserver 192.168.200.2
nameserver 192.168.202.2
nameserver 8.8.8.8
search openstacklocal
{% endhighlight %}

Como era de esperar, los servidores DNS se han configurado como deberían, habiéndose añadido los servidores **192.168.202.2** y **8.8.8.8**, tal y como lo hemos establecido.

Para verificar que toda la configuración ha funcionado, vamos a tratar de hacer `ping` a **google.es**, para así verificar también que la resolución de nombres se realiza correctamente:

{% highlight shell %}
root@freston:~# ping google.es
PING google.es (216.58.211.35) 56(84) bytes of data.
64 bytes from mad08s05-in-f3.1e100.net (216.58.211.35): icmp_seq=1 ttl=112 time=44.6 ms
64 bytes from mad08s05-in-f3.1e100.net (216.58.211.35): icmp_seq=2 ttl=112 time=97.3 ms
64 bytes from mad08s05-in-f3.1e100.net (216.58.211.35): icmp_seq=3 ttl=112 time=45.3 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 61ms
rtt min/avg/max/mdev = 44.597/62.390/97.253/24.654 ms
{% endhighlight %}

Como era de esperar, ya tenemos salida a Internet a través de **Dulcinea**, así que aprovecharemos y llevaremos a cabo una actualización de la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# apt update && apt upgrade
{% endhighlight %}

Tras unos segundos, la paquetería se habrá actualizado y ya estaremos utilizando las últimas versiones de los correspondientes paquetes.

En este caso, vamos a generar además un nuevo usuario **profesor** en la instancia, en el que añadiremos las claves públicas de los profesores para que puedan acceder. Además, llevaremos a cabo la correspondiente configuración para que dicho usuario pueda ejecutar `sudo` sin contraseña. El comando a ejecutar para crear un usuario **profesor** sería:

{% highlight shell %}
root@freston:~# useradd profesor -m -s /bin/bash
{% endhighlight %}

Donde:

* **-m**: Especificamos que cree un directorio personal para dicho usuario en caso de que no exista.
* **-s**: Especificamos el nombre de la shell que utilizará para el login.

El usuario ya se encuentra creado, así que nos cambiaremos de usuario para hacer uso del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# su - profesor
{% endhighlight %}

Como bien sabemos, las claves públicas autorizadas a acceder a nuestra máquina se encuentran en **~/.ssh/**, dentro de un fichero de nombre **authorized_keys**, que actualmente no se encuentra creado, así que vamos a proceder a crear dicho directorio y dicho fichero e insertar las correspondientes claves públicas en su interior. Los comandos a ejecutar serían:

{% highlight shell %}
profesor@freston:~$ mkdir .ssh

profesor@freston:~$ nano .ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfk9mRtOHM3T1KpmGi0KiN2uAM6CDXM3WFcm1wkzKXx7RaLtf9pX+KCuVqHdy/N/9d9wtH7iSmLFX/4gQKQVG00jHiGf3ABufWeIpjmHtT1WaI0+vV47fofEIjDDfSZPlI3p5/c7tefHsIAK6GbQn31yepAcFYy9ZfqAh8H/Y5eLpf3egPZn9Czsvx+lm0I8Q+e/HSayRaiAPUukF57N2nnw7yhPZCHSZJqFbXyK3fVQ/UQVBeNS2ayp0my8X9sIBZnNkcYHFLIWBqJYdnu1ZFhnbu3yy94jmJdmELy3+54hqiwFEfjZAjUYSl8eGPixOfdTgc8ObbHbkHyIrQ91Kz rafa@eco
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmjoVIoZCx4QFXvljqozXGqxxlSvO7V2aizqyPgMfGqnyl0J9YXo6zrcWYwyWMnMdRdwYZgHqfiiFCUn2QDm6ZuzC4Lcx0K3ZwO2lgL4XaATykVLneHR1ib6RNroFcClN69cxWsdwQW6dpjpiBDXf8m6/qxVP3EHwUTsP8XaOV7WkcCAqfYAMvpWLISqYme6e+6ZGJUIPkDTxavu5JTagDLwY+py1WB53eoDWsG99gmvyit2O1Eo+jRWN+mgRHIxJTrFtLS6o4iWeshPZ6LvCZ/Pum12Oj4B4bjGSHzrKjHZgTwhVJ/LDq3v71/PP4zaI3gVB9ZalemSxqomgbTlnT jose@debian
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3AUDWjyPANntK+qwHmlJihKQZ1H+AGN02k06dzRHmkvWiNgou/VcCgowhMTGR+0I6nWVwgRSWKJEUEaMu1r9rEeL63GRtUSepCWpClHJG1CuySuJKVGtRdUq+/szDntpJnJW207a78hTeQLjQsyPvbOqkbulQG7xTRCycdT3bH2UO4JI2d+341gkOlxSG/stPQ52Dsbfb274oMRom5r5f2apD3wbfxE9A6qwm4m70G9NYS7T3uKgCiXegO/3GTJD4UbK0ylGUamG5obdS5yD8Ib12vRCCXWav23SAj/4f9MzAnXX8U4ATM/du2FHZBiIzWVH12LYvIEZpUIVYKPSf alberto@roma
{% endhighlight %}

El directorio **.ssh** y el fichero **authorized_keys** se encuentran ya creados, conteniendo este último, además, las claves públicas de los profesores.

Los ficheros y directorios se crean por defecto con unos determinados permisos UNIX que no son los que nosotros queremos, pues dicha información ha de ser confidencial y únicamente accesible por el usuario **profesor**, de manera que cambiaremos los permisos del **directorio** a **700** y los permisos del **fichero** a **600**, haciendo para ello uso de los comandos:

{% highlight shell %}
profesor@freston:~$ chmod 700 .ssh/

profesor@freston:~$ chmod 600 .ssh/authorized_keys
{% endhighlight %}

Listo, los permisos ya habrán sido modificados, así que nos queda un último paso, permitir que dicho usuario haga uso del comando `sudo` sin que se le solicite una contraseña (pues actualmente no tiene, de manera que el único método de acceso al mismo es mediante par de claves SSH). Para ello debemos salir del usuario **profesor** para así volver a **root**, de manera que modificaremos el fichero **/etc/sudoers** y añadiremos una línea de la siguiente manera:

{% highlight shell %}
root@freston:~# echo "profesor ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
{% endhighlight %}

El usuario **profesor** ya está en posición de ejecutar el comando `sudo` sin que sea necesario establecer ninguna contraseña.

La última configuración necesaria a llevar a cabo para tener completamente operativa la máquina **Freston** consiste en sincronizar el reloj de la misma, haciendo uso del protocolo _Network Time Protocol_ (**NTP**), un protocolo de Internet para sincronizar los relojes de los sistemas informáticos muy utilizado hoy en día, principalmente en empresas, pues es altamente recomendable que todas las máquinas existentes tengan la hora sincronizada entre sí.

Para comprobar si nuestro servidor tiene actualmente sincronizado el reloj haciendo uso de un servidor NTP, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# timedatectl
               Local time: Wed 2020-12-09 10:28:21 UTC
           Universal time: Wed 2020-12-09 10:28:21 UTC
                 RTC time: Wed 2020-12-09 10:28:20
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: no
{% endhighlight %}

Como se puede apreciar, nos ha mostrado la información referente a la hora de nuestra máquina. Sin embargo, podemos apreciar que el estado del **NTP service** es **inactive**, y que por tanto el reloj del sistema no se encuentra sincronizado (**System clock synchronized**).

Para tratar de solucionarlo, vamos a ver el estado del servicio **systemd-timesyncd.service**, uno de los _daemon_ que se puede encargar de gestionar la hora en las máquinas, para aquellas que estén corriendo **systemd** (pues no es necesario hacer uso del _daemon_ **ntpd**). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: inactive (dead)
     Docs: man:systemd-timesyncd.service(8)
{% endhighlight %}

Como se puede apreciar, el servicio se encuentra actualmente **inactivo**, por lo que trataremos de reiniciarlo para que así arranque, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# systemctl restart systemd-timesyncd.service
{% endhighlight %}

Una vez más, mostraremos el estado del servicio para verificar si ha podido arrancar:

{% highlight shell %}
root@freston:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: inactive (dead)
Condition: start condition failed at Wed 2020-12-09 10:28:37 UTC; 5s ago
           └─ ConditionFileIsExecutable=!/usr/sbin/ntpd was not met
     Docs: man:systemd-timesyncd.service(8)

Dec 09 10:28:37 freston systemd[1]: Condition check resulted in Network Time Synchronization being skipped.
{% endhighlight %}

Como se puede apreciar, en esta ocasión tampoco ha podido arrancar, ya que aparentemente hay un conflicto entre **ntpd** y **systemd-timesyncd**, por lo que vamos a eliminar el primero de ellos, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# apt purge ntp
{% endhighlight %}

Antes de tratar de arrancar de nuevo el servicio, vamos a modificar el fichero de configuración del mismo (**/etc/systemd/timesyncd.conf**) para así indicarle el servidor que tendrá que usar para la sincronización. Lo recomendable es que sea lo más cercano a nosotros geográficamente hablando. En este caso, he elegido **es.pool.ntp.org**, así que para modificar dicho fichero ejecutaremos el comando:

{% highlight shell %}
root@freston:~# nano /etc/systemd/timesyncd.conf

[Time]
NTP=es.pool.ntp.org
{% endhighlight %}

Listo, el servidor que ha de usar ya ha sido especificado, así que volveremos a reiniciar el servicio y a comprobar su estado:

{% highlight shell %}
root@freston:~# systemctl restart systemd-timesyncd.service

root@freston:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: active (running) since Wed 2020-12-09 10:29:51 UTC; 5s ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 1408 (systemd-timesyn)
   Status: "Synchronized to time server for the first time 147.156.7.50:123 (es.pool.ntp.org)."
    Tasks: 2 (limit: 562)
   Memory: 1.3M
   CGroup: /system.slice/systemd-timesyncd.service
           └─1408 /lib/systemd/systemd-timesyncd

Dec 09 10:29:51 freston systemd[1]: Starting Network Time Synchronization...
Dec 09 10:29:51 freston systemd[1]: Started Network Time Synchronization.
Dec 09 10:29:56 freston systemd-timesyncd[1408]: Synchronized to time server for the first time 147.156.7.50:123 (es.pool.ntp.org).
{% endhighlight %}

En esta ocasión, sí ha sido capaz de arrancar (**active**) y de sincronizarse con dicho servidor, tal y como podemos apreciar en el último mensaje mostrado.

Todavía no hemos terminado, ya que cuando ejecuté el comando `timedatectl`, me mostraba una hora menos de la hora local, es decir, había un problema con la zona horaria, por lo que debemos modificarla manualmente. Para ver las zonas horarias disponibles, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# timedatectl list-timezones
{% endhighlight %}

En mi caso, me quedaré con la hora de España (**Europe/Madrid**), así que para modificar la zona horaria, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# timedatectl set-timezone Europe/Madrid
{% endhighlight %}

En un principio, la hora ya debe estar completamente sincronizada y adaptada a nuestra hora local, así que volveremos a ejecutar `timedatectl` para verificarlo:

{% highlight shell %}
root@freston:~# timedatectl
               Local time: Wed 2020-12-09 11:30:47 CET
           Universal time: Wed 2020-12-09 10:30:47 UTC
                 RTC time: Wed 2020-12-09 10:30:48
                Time zone: Europe/Madrid (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
{% endhighlight %}

Efectivamente, el servicio NTP ya se encuentra activo y el reloj del sistema sincronizado, por lo que ya muestra la hora local real (**11:30:47**).

## Modificación de la red de Quijote

Como anteriormente hemos mencionado, es necesario que **Quijote**, la máquina _CentOS 8_, pase de estar conectada a la red interna a estarlo a la nueva **red DMZ** que hemos creado.

Para ello, volveremos a la lista de nuestras instancias creadas y pulsaremos en el símbolo **▾** en Quijote, para así abrir el desplegable. Tras ello, pulsaremos en la opción **Desconectar interfaz** y seleccionamos el único puerto existente en la misma, "**10.0.1.5**".

Cuando su única interfaz haya sido desconectada, volveremos a abrir una vez más el desplegable para pulsar en esta ocasión en **Conectar interfaz** y seleccionar la Red de nombre "**DMZ de alvaro.vaca**". El resultado final de las máquinas sería:

![instancias3](https://i.ibb.co/bLCphZd/1.jpg "Cambio de red de Quijote")

Como se puede apreciar, las direcciones IP finalmente asignadas a las máquinas han sido las siguientes:

- **Dulcinea**:
    - 10.0.0.7 - 172.22.200.134
    - 10.0.1.9
    - 10.0.2.12
- **Sancho**:
    - 10.0.1.4
- **Quijote**:
    - 10.0.2.6
- **Freston**:
    - 10.0.1.7

La configuración todavía no ha terminado, ya que tendremos que acceder a la máquina para verificar que se le ha asignado correctamente dicha dirección IP, además de configurarle un direccionamiento estático a la misma, sustituyendo la anterior configuración asignada.

Para verificar que la interfaz de red ha variado de una red a otra, ejecutaremos el comando:

{% highlight shell %}
[centos@quijote ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:8b:8a:0e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.6/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86330sec preferred_lft 86330sec
    inet6 fe80::3862:8ccc:d0c3:610b/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la interfaz **eth0** se encuentra ahora conectada a la nueva **red DMZ**, con un direccionamiento **10.0.2.6** asignado automáticamente por DHCP, de manera que tendremos que llevar a cabo la correspondiente configuración para hacer que dicho direccionamiento sea estático.

El fichero de configuración en el que debemos establecer la configuración de las interfaces de red de una máquina CentOS 8 es **/etc/sysconfig/network-scripts/ifcfg-eth0**, así que lo modificaremos ejecutando el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth0
{% endhighlight %}

El contenido previamente establecido en el interior será:

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

Dentro del mismo, tendremos que llevar a cabo tres modificaciones obligatorias:

* **HWADDR**: La dirección MAC de la interfaz de red ha cambiado de **fa:16:3e:3b:bb:e2** a **fa:16:3e:8b:8a:0e**.
* **IPADDR**: La dirección IP de la interfaz de red ha cambiado de **10.0.1.5** a **10.0.2.6**.
* **GATEWAY**: La puerta de enlace predeterminada ha cambiado de **10.0.1.9** a la dirección IP del router (Dulcinea) alcanzable desde la nueva red, **10.0.2.12**.

El resultado final tras llevar a cabo las correspondientes modificaciones sería:

{% highlight shell %}
BOOTPROTO=none
DEVICE=eth0
HWADDR=fa:16:3e:8b:8a:0e
MTU=8950
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
IPADDR=10.0.2.6
PREFIX=24
GATEWAY=10.0.2.12
{% endhighlight %}

Genial, todos los cambios referentes a la red se han llevado a cabo, pero para que surtan efecto podemos reiniciar la máquina (comando `reboot`) o bien apagar y encender la interfaz de red correspondiente, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales.

En realidad, si nos encontrásemos haciendo uso de CentOS 7, podríamos reiniciar el servicio **networking** pero actualmente nos encontramos en CentOS 8, que hace uso de **NetworkManager**, por lo que debemos apagar y encender la interfaz para que así tome la nueva configuración. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# ifdown eth0 && ifup eth0
Connection 'System eth0' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/1)
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
{% endhighlight %}

Tras unos segundos de espera mientras la interfaz de red se reiniciaba, ya podremos apreciar los cambios que hemos realizado, ejecutando el comando `ip a` para verificar que la dirección IP estática se ha asignado correctamente:

{% highlight shell %}
[root@quijote ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:8b:8a:0e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.6/24 brd 10.0.2.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe8b:8a0e/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la dirección ha sido correctamente configurada estáticamente (podemos deducir que es estática ya que su **valid_lft** es **forever**). Además, vamos a comprobar que la puerta de enlace predeterminada también se ha configurado correctamente, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# ip r
default via 10.0.2.12 dev eth0 proto static metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.6 metric 100
{% endhighlight %}

Efectivamente, así ha sido. Sin embargo, la máquina **Quijote** conectada a la nueva **red DMZ** no tiene todavía salida a Internet a través de **Dulcinea**, por lo que vamos a proceder a solucionarlo.

## Configuración de NAT en Dulcinea

Si recordamos del anterior artículo, la máquina Dulcinea viene configurada con un grupo de seguridad que se asigna por defecto durante su creación, es decir, cuenta con unas reglas de cortafuegos que no necesitamos, pues nosotros queremos configurar nuestras propias reglas para hacer _SNAT_. La deshabilitación de dicho grupo de seguridad no se podía llevar a cabo desde **Horizon**, sino que hicimos uso del cliente desde la terminal, de manera que deshabilitamos el grupo de seguridad y la seguridad en los dos puertos de los que disponía en ese momento.

Sin embargo, hemos añadido un nuevo puerto a la máquina, correspondiente a la red DMZ, por lo que tendremos que deshabilitar su seguridad para así para poder operar con el mismo con nuestras propias reglas de cortafuegos. Para ello, reutilizaremos el entorno virtual creado con anterioridad y todavía existente en la máquina anfitriona, haciendo uso a su vez del fichero RC que configuramos para así acceder a nuestro proyecto.

Una vez dentro del mismo, podremos proceder a listar las instancias existentes en nuestro proyecto para así verificar que funciona correctamente. El comando a ejecutar sería:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server list
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
| ID                                   | Name       | Status  | Networks                                                                                                       | Image                    | Flavor  |
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
| 02978cea-fee1-4050-93cf-5213833652b2 | Freston    | ACTIVE  | red interna de alvaro.vaca=10.0.1.7                                                                            | N/A (booted from volume) | m1.mini |
| 24163226-9ada-445c-9b6c-18ca6ec73b62 | DNSCliente | SHUTOFF | red de alvaro.vaca=10.0.0.13, 172.22.200.106                                                                   | Debian Buster 10.6       | m1.mini |
| c997cff3-4f83-43a9-a797-a779425d7da0 | maquina    | SHUTOFF | red de alvaro.vaca=10.0.0.4, 172.22.201.19                                                                     | Debian Buster 10.6       | m1.mini |
| dc914d4c-8a26-4ac8-ae9a-d0ae2eeafb1d | Django     | SHUTOFF | red de alvaro.vaca=10.0.0.16, 172.22.200.186                                                                   | Debian Buster 10.6       | m1.mini |
| ee6e1fde-da68-44e4-89b2-536d0758f2d2 | DNS        | SHUTOFF | red de alvaro.vaca=10.0.0.15, 172.22.200.184                                                                   | Debian Buster 10.6       | m1.mini |
| d5474d1a-51af-4dc5-935f-4756848bd88b | Quijote    | ACTIVE  | DMZ de alvaro.vaca=10.0.2.6                                                                                    | N/A (booted from volume) | m1.mini |
| 93673790-a461-49ee-94d6-f0aeff3b8bdb | Sancho     | ACTIVE  | red interna de alvaro.vaca=10.0.1.4                                                                            | N/A (booted from volume) | m1.mini |
| 2e1b3676-ae28-44a4-8748-9f59609c7ef7 | Dulcinea   | ACTIVE  | DMZ de alvaro.vaca=10.0.2.12; red de alvaro.vaca=10.0.0.7, 172.22.200.134; red interna de alvaro.vaca=10.0.1.9 | N/A (booted from volume) | m1.mini |
| d73cbed2-1c32-46cf-98e1-e715a950688b | backup     | SHUTOFF | red de alvaro.vaca=10.0.0.5                                                                                    | Debian Buster 10.6       | m1.mini |
| 976b4ef3-55b2-468f-8243-ffb8db1fd521 | cms        | SHUTOFF | red de alvaro.vaca=10.0.0.8, 172.22.200.103                                                                    | Debian Buster 10.6       | m1.mini |
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
{% endhighlight %}

Efectivamente, el cliente de OpenStack se encuentra actualmente operativo y nos ha mostrado la información referente a las instancias creadas en mi proyecto.

Tras ello, procederemos a eliminar el grupo de seguridad por defecto (**default**) en todas las máquinas, ya que en el anterior artículo únicamente lo deshabilitamos en Dulcinea, pero para evitar posibles conflictos, es mejor asegurarnos de que se ha eliminado en todas ellas. Para ello, ejecutaremos los comandos:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server remove security group Dulcinea default
(openstackclient) alvaro@debian:~/virtualenv$ openstack server remove security group Sancho default
(openstackclient) alvaro@debian:~/virtualenv$ openstack server remove security group Quijote default
(openstackclient) alvaro@debian:~/virtualenv$ openstack server remove security group Freston default
{% endhighlight %}

En teoría el grupo de seguridad ya ha sido eliminado en todas las máquinas, pero para verificarlo, vamos a listar la información detallada sobre las mismas, estableciendo a su vez un filtro para mostrar únicamente aquella referente a los grupos de seguridad, haciendo uso de los comandos:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server show Dulcinea | egrep 'security_groups'
(openstackclient) alvaro@debian:~/virtualenv$ openstack server show Sancho | egrep 'security_groups'
(openstackclient) alvaro@debian:~/virtualenv$ openstack server show Quijote | egrep 'security_groups'
(openstackclient) alvaro@debian:~/virtualenv$ openstack server show Freston | egrep 'security_groups'
{% endhighlight %}

Efectivamente, podemos concluir que el grupo de seguridad ha sido correctamente eliminado de las instancias, ya que no aparece ninguna información sobre el apartado de nombre **security_groups**.

Esto no es todo, ya que también tenemos que deshabilitar la seguridad en los correspondientes puertos de las máquinas, ya que cuando una máquina no tiene ningún grupo de seguridad asignado, el _modus operandi_ por defecto es el de bloquear la conexión en todos los puertos asociados a la misma. Para ello, vamos a listar primero los puertos existentes en nuestro proyecto, ejecutando el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack port list
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                       | Status |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
| 010ab068-26f6-4df0-ab55-b07761c52db6 |      | fa:16:3e:41:0e:16 | ip_address='10.0.0.13', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299' | ACTIVE |
| 0aab3e9c-c2c9-46c2-b6ed-e33447107554 |      | fa:16:3e:b5:5d:e7 | ip_address='10.0.1.9', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| 20decc39-41db-4e9a-9294-ce5fefa138de |      | fa:16:3e:47:a8:ea | ip_address='10.0.2.2', subnet_id='c4968117-c9c8-49e6-b200-770ad51177e7'  | ACTIVE |
| 488dc43f-3a9b-4981-8cf7-bc6cbcf82203 |      | fa:16:3e:e2:f6:e7 | ip_address='10.0.0.7', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 5929e26d-ebf9-42b3-81be-0ab52eac06bc |      | fa:16:3e:15:08:1b | ip_address='10.0.0.1', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 66ef0647-c0b1-4a2c-8bfe-dd228c2fc83e |      | fa:16:3e:56:ee:36 | ip_address='10.0.0.2', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 72bca483-bd6d-4a0c-a1ff-e744337917db |      | fa:16:3e:70:ef:a2 | ip_address='10.0.2.12', subnet_id='c4968117-c9c8-49e6-b200-770ad51177e7' | ACTIVE |
| 7b6bd3ce-179e-449e-95d0-18d05bad45db |      | fa:16:3e:d8:07:f4 | ip_address='10.0.0.4', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 930d1e9d-a5d0-4a99-83ab-531a3484eea6 |      | fa:16:3e:63:2e:e4 | ip_address='10.0.0.16', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299' | ACTIVE |
| 9aba5d88-1d49-4cf3-87bd-426adb7a036b |      | fa:16:3e:a4:1f:11 | ip_address='10.0.0.15', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299' | ACTIVE |
| 9bb65414-27ad-427d-a6bb-08c1c4cb0d6a |      | fa:16:3e:c8:fd:80 | ip_address='10.0.0.5', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 9f1f5eea-a302-4126-ad9b-7f2872e2ae91 |      | fa:16:3e:99:2b:18 | ip_address='10.0.1.7', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| a42caf41-26f3-4532-9299-dae5c86a66a6 |      | fa:16:3e:8b:8a:0e | ip_address='10.0.2.6', subnet_id='c4968117-c9c8-49e6-b200-770ad51177e7'  | ACTIVE |
| c7c9209b-4df1-46c8-be51-226323baa9e8 |      | fa:16:3e:10:d8:23 | ip_address='10.0.1.4', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| cc4f146a-3685-46cb-b930-b2dc4ed5da8c |      | fa:16:3e:fb:25:57 | ip_address='10.0.1.3', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| eba43a24-dc3b-40dc-a700-3c3080e62b22 |      | fa:16:3e:83:7c:6e | ip_address='10.0.0.8', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
{% endhighlight %}

Aquí podemos ver todos los puertos existentes en nuestro proyecto, por lo que tendremos que buscar los identificadores (**ID**) de aquellos puertos asociados a las cuatro máquinas:

- **Dulcinea**:
    - **10.0.0.7**: 488dc43f-3a9b-4981-8cf7-bc6cbcf82203
    - **10.0.1.9**: 0aab3e9c-c2c9-46c2-b6ed-e33447107554
    - **10.0.2.12**: 72bca483-bd6d-4a0c-a1ff-e744337917db
- **Sancho**:
    - **10.0.1.4**: c7c9209b-4df1-46c8-be51-226323baa9e8
- **Quijote**:
    - **10.0.2.6**: a42caf41-26f3-4532-9299-dae5c86a66a6
- **Freston**:
    - **10.0.1.7**: 9f1f5eea-a302-4126-ad9b-7f2872e2ae91

Tras ello, tendremos que deshabilitar la seguridad en dichos puertos, indicando para ello el identificador en cuestión, haciendo uso de los comandos:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 488dc43f-3a9b-4981-8cf7-bc6cbcf82203
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 0aab3e9c-c2c9-46c2-b6ed-e33447107554
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 72bca483-bd6d-4a0c-a1ff-e744337917db
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security c7c9209b-4df1-46c8-be51-226323baa9e8
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security a42caf41-26f3-4532-9299-dae5c86a66a6
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 9f1f5eea-a302-4126-ad9b-7f2872e2ae91
{% endhighlight %}

Listo, ya hemos deshabilitado la seguridad en los mismos, de manera que ya tenemos accesible todo el rango de puertos. Es importante mencionar que deshabilitar el cortafuegos es una acción un tanto precipitada, por lo que únicamente debe llevarse a cabo en un entorno controlado.

En el primer artículo habíamos configurado en **Dulcinea** una tabla de nombre **nat** que cuenta con una cadena **POSTROUTING**, es decir, aquella que permite modificar paquetes justo antes de que salgan del equipo, permitiendo por tanto hacer **Source NAT** (_SNAT_).

Dentro de la misma, se añadió una regla que permitía a las máquinas existentes en la red interna (**10.0.1.0/24**) salir al exterior, haciendo _SNAT_ a la dirección de la interfaz de red perteneciente a la red externa, en este caso, la **10.0.0.7**. Para listar las reglas existentes ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset
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
                oifname "eth0" ip saddr 10.0.1.0/24 counter packets 30 bytes 1977 snat to 10.0.0.7
        }
}
{% endhighlight %}

Sin embargo, dicha regla no sirve para la red DMZ (**10.0.2.0/24**), por lo que las máquinas que se encuentren en dicha red no tienen todavía salida al exterior, de manera que tendremos que añadir una nueva regla, haciendo para ello uso del comando:

{% highlight shell %}
root@dulcinea:~# nft add rule ip nat postrouting oifname "eth0" ip saddr 10.0.2.0/24 counter snat to 10.0.0.7
{% endhighlight %}

En este caso, hemos especificado que se trata de una regla para la cadena **postrouting** (pues debe aplicarse justo antes de salir de la máquina), además, la interfaz (_oifname_) que debemos introducir será aquella por la que van a salir los paquetes, es decir, la que está conectada a Internet (aunque sea haciendo doble o triple NAT, no es algo relevante en estas circunstancias), en este caso, **eth0**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que provengan de la red (_ip saddr_) **10.0.2.0/24**. Por último, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a hacer _SNAT_ a la dirección IP perteneciente a la red exterior (_snat to_) **10.0.0.7**.

Por último, vamos a verificar que dicha regla se ha añadido correctamente, ejecutando de nuevo el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset
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
            oifname "eth0" ip saddr 10.0.1.0/24 counter packets 30 bytes 1977 snat to 10.0.0.7
            oifname "eth0" ip saddr 10.0.2.0/24 counter packets 0 bytes 0 snat to 10.0.0.7
    }
}
{% endhighlight %}

Efectivamente, así ha sido, de manera que Quijote ya tiene conectividad con el exterior. Sin embargo, esta configuración se encuentra cargada en memoria, por lo que para conseguir que perdure en el tiempo, vamos a ejecutar el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Gracias a ello, habremos guardado la configuración en un fichero en **/etc/** de nombre **nftables.conf** que se importará de forma automática cuando reiniciemos la máquina gracias al _daemon_ que se encuentra habilitado. En caso de que no cargase de nuevo la configuración, podríamos hacerlo manualmente con el comando `nft -f /etc/nftables.conf`.

## Modificación de la subred de las redes interna y DMZ, deshabilitando el servidor DHCP

Ya hemos configurado el direccionamiento estático en todas las máquinas, por lo que no será necesario tener activo el servidor DHCP en ninguna de las redes, pues no le daremos ningún uso. Para deshabilitarlo, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, buscaremos la red interna (**red interna de alvaro.vaca**) y pulsaremos en el nombre de la misma. Cuando hayamos accedido a la red, tendremos varios subapartados, así que accederemos a **Subredes** y nos aparecerá la única que hemos creado, por lo que procederemos a editarla pulsando en **Editar subred**.

Se nos habrá abierto un pequeño menú, así que iremos a **Detalles de subred**, pues es donde se encuentra la configuración referente al servidor DHCP de la subred. Simplemente desmarcaremos la casilla **Habilitar DHCP** y guardaremos los cambios, pulsando para ello en **Guardar**. El servidor DHCP ya habrá sido deshabilitado, tal y como se puede apreciar:

![dhcp2](https://i.ibb.co/F0tFgBX/dhcp2.jpg "Deshabilitar DHCP")

Tras ello, repetiremos exactamente el mismo procedimiento para la red **DMZ de alvaro.vaca**, resultando de la siguiente manera:

![dhcp3](https://i.ibb.co/k2kzcKF/dhcp1.jpg "Deshabilitar DHCP")

## Configuración de resolución estática en las cuatro instancias

Dado que todavía no hemos configurado un servidor DNS, tendremos que configurar la resolución estática de nombres en las máquinas, de manera que se conozcan entre ellas, llevando a cabo las modificaciones necesarias para conozcan a la nueva, así como a Quijote tras su modificación de red.

Dicha información está almacenada en el fichero **/etc/hosts**, así que tendremos que incluir en el mismo líneas con la siguiente sintaxis:

{% highlight shell %}
<IP> <FQDN> <hostname>
{% endhighlight %}

Donde:

* **IP**: La IP de la máquina que queremos resolver.
* **FQDN**: El _Fully Qualified Domain Name_, que se encuentra compuesto por **[hostname].alvaro.gonzalonazareno.org**.
* **hostname**: El nombre corto de la máquina.

En cada máquina debemos tener un total de tres líneas (de manera que conozca las otras tres máquinas restantes, además de a sí misma, modificando o añadiendo la línea **127.0.1.1**), es decir:

* **Dulcinea**: Debe conocer a Sancho, a Quijote, a Freston y a sí misma.
* **Sancho**: Debe conocer a Dulcinea, a Quijote, a Freston y a sí misma.
* **Quijote**: Debe conocer a Dulcinea, a Sancho, a Freston y a sí misma.
* **Freston**: Debe conocer a Dulcinea, a Sancho, a Quijote y a sí misma.

Para llevar a cabo dicha modificación, haremos uso del comando (dicho comando es genérico, excepto para **CentOS 8**, que tendremos que utilizar `vi` en lugar de `nano`):

{% highlight shell %}
nano /etc/hosts
{% endhighlight %}

Tras ello, ya podremos hacer `ping`, por ejemplo, y las máquinas deben responder en caso de hacer uso tanto del **FQDN** como del **hostname**.

### Dulcinea

{% highlight shell %}
root@dulcinea:~# nano /etc/hosts
127.0.1.1 dulcinea.alvaro.gonzalonazareno.org dulcinea.novalocal dulcinea
127.0.0.1 localhost
10.0.1.4 sancho.alvaro.gonzalonazareno.org sancho
10.0.2.6 quijote.alvaro.gonzalonazareno.org quijote
10.0.1.7 freston.alvaro.gonzalonazareno.org freston

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@dulcinea:~# ping sancho
PING sancho.alvaro.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=64 time=2.41 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=64 time=1.08 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=64 time=1.01 ms
^C
--- sancho.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.012/1.502/2.414/0.646 ms

root@dulcinea:~# ping quijote.alvaro.gonzalonazareno.org
PING quijote.alvaro.gonzalonazareno.org (10.0.2.6) 56(84) bytes of data.
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=1 ttl=64 time=1.16 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=2 ttl=64 time=0.906 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=3 ttl=64 time=0.750 ms
^C
--- quijote.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.750/0.938/1.159/0.170 ms

root@dulcinea:~# ping freston
PING freston.alvaro.gonzalonazareno.org (10.0.1.7) 56(84) bytes of data.
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=1 ttl=64 time=2.12 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=2 ttl=64 time=1.39 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=3 ttl=64 time=1.09 ms
^C
--- freston.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.094/1.532/2.118/0.433 ms
{% endhighlight %}

### Sancho

{% highlight shell %}
root@sancho:~# nano /etc/hosts
127.0.1.1 sancho.alvaro.gonzalonazareno.org sancho
127.0.0.1 localhost
10.0.1.9 dulcinea.alvaro.gonzalonazareno.org dulcinea
10.0.2.6 quijote.alvaro.gonzalonazareno.org quijote
10.0.1.7 freston.alvaro.gonzalonazareno.org freston

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@sancho:~# ping dulcinea.alvaro.gonzalonazareno.org
PING dulcinea.alvaro.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=1 ttl=64 time=1.05 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=2 ttl=64 time=1.08 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=3 ttl=64 time=1.46 ms
^C
--- dulcinea.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.045/1.194/1.463/0.190 ms

root@sancho:~# ping quijote
PING quijote.alvaro.gonzalonazareno.org (10.0.2.6) 56(84) bytes of data.
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=1 ttl=63 time=2.02 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=2 ttl=63 time=1.58 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=3 ttl=63 time=1.64 ms
^C
--- quijote.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.576/1.745/2.018/0.194 ms

root@sancho:~# ping freston.alvaro.gonzalonazareno.org
PING freston.alvaro.gonzalonazareno.org (10.0.1.7) 56(84) bytes of data.
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=1 ttl=64 time=1.38 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=2 ttl=64 time=0.676 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=3 ttl=64 time=0.680 ms
^C
--- freston.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2033ms
rtt min/avg/max/mdev = 0.676/0.912/1.381/0.331 ms
{% endhighlight %}

### Quijote

{% highlight shell %}
[root@quijote ~]# vi /etc/hosts
127.0.1.1 quijote.alvaro.gonzalonazareno.org quijote
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.2.12 dulcinea.alvaro.gonzalonazareno.org dulcinea
10.0.1.4 sancho.alvaro.gonzalonazareno.org sancho
10.0.1.7 freston.alvaro.gonzalonazareno.org freston

[root@quijote ~]# ping dulcinea.alvaro.gonzalonazareno.org
PING dulcinea.alvaro.gonzalonazareno.org (10.0.2.12) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.2.12): icmp_seq=1 ttl=64 time=0.863 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.2.12): icmp_seq=2 ttl=64 time=0.809 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.2.12): icmp_seq=3 ttl=64 time=0.775 ms
^C
--- dulcinea.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 56ms
rtt min/avg/max/mdev = 0.775/0.815/0.863/0.049 ms

[root@quijote ~]# ping sancho
PING sancho.alvaro.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=63 time=2.80 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=63 time=1.82 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=63 time=1.37 ms
^C
--- sancho.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.372/1.996/2.802/0.599 ms

[root@quijote ~]# ping freston.alvaro.gonzalonazareno.org
PING freston.alvaro.gonzalonazareno.org (10.0.1.7) 56(84) bytes of data.
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=1 ttl=63 time=2.84 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=2 ttl=63 time=2.04 ms
64 bytes from freston.alvaro.gonzalonazareno.org (10.0.1.7): icmp_seq=3 ttl=63 time=1.52 ms
^C
--- freston.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.517/2.133/2.840/0.546 ms
{% endhighlight %}

### Freston

En la nueva máquina Debian Buster hay una pequeña peculiaridad, y es que el fichero **/etc/hosts** se genera dinámicamente durante el arranque, gracias al estándar **cloud-init**, así que para deshabilitarlo y así conseguir un fichero estático, tendremos que cambiar el valor de la directiva **manage_etc_hosts** a **false** en el fichero **/etc/cloud/cloud.cfg**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@freston:~# sed -i 's/manage_etc_hosts: true/manage_etc_hosts: false/g' /etc/cloud/cloud.cfg
{% endhighlight %}

Para verificar que el valor de dicha directiva ha sido modificado, vamos a visualizar el contenido del fichero, estableciendo un filtro por nombre:

{% highlight shell %}
root@freston:~# egrep 'manage_etc_hosts' /etc/cloud/cloud.cfg
manage_etc_hosts: false
{% endhighlight %}

Efectivamente, su valor ha cambiado, así que ya podemos proceder a modificar el fichero **/etc/hosts** y a realizar las correspondientes pruebas:

{% highlight shell %}
root@freston:~# nano /etc/hosts
127.0.1.1 freston.alvaro.gonzalonazareno.org freston.novalocal freston
127.0.0.1 localhost
10.0.1.9 dulcinea.alvaro.gonzalonazareno.org dulcinea
10.0.1.4 sancho.alvaro.gonzalonazareno.org sancho
10.0.2.6 quijote.alvaro.gonzalonazareno.org quijote

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@freston:~# ping dulcinea
PING dulcinea.alvaro.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=1 ttl=64 time=0.664 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=2 ttl=64 time=1.01 ms
64 bytes from dulcinea.alvaro.gonzalonazareno.org (10.0.1.9): icmp_seq=3 ttl=64 time=1.03 ms
^C
--- dulcinea.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.664/0.901/1.029/0.169 ms

root@freston:~# ping sancho.alvaro.gonzalonazareno.org
PING sancho.alvaro.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=64 time=2.90 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=64 time=0.732 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=64 time=0.552 ms
^C
--- sancho.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.552/1.395/2.901/1.067 ms

root@freston:~# ping quijote
PING quijote.alvaro.gonzalonazareno.org (10.0.2.6) 56(84) bytes of data.
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=1 ttl=63 time=1.74 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=2 ttl=63 time=1.52 ms
64 bytes from quijote.alvaro.gonzalonazareno.org (10.0.2.6): icmp_seq=3 ttl=63 time=1.76 ms
^C
--- quijote.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.524/1.672/1.758/0.110 ms
{% endhighlight %}