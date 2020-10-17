---
layout: post
title:  "Servidor DHCP con NAT"
banner: "/assets/images/banners/dhcp.jpg"
date:   2020-10-10 13:42:00 +0200
categories: dhcp nat servicios
---
## Tarea 1: Lee el documento [Teoría: Servidor DHCP](https://fp.josedomingo.org/serviciosgs/u02/dhcp.html) y explica el funcionamiento del servidor DHCP resumido en este gráfico.

![dhcp](https://fp.josedomingo.org/serviciosgs/u02/img/dhcp.png "DHCP")

El proceso de configuración de los clientes DHCP se encuentra claramente diferenciado por varias fases o estados internos:

Cuando se inicializa el cliente DHCP, se encuentra en estado **INIT**, es decir, desconoce sus parámetros de red y los de los servidores DHCP de dicha red, es por ello que envía un mensaje **DHCPDISCOVER** al puerto **67** de la dirección **broadcast** de la red (**255.255.255.255**), utilizando la IP origen **0.0.0.0**, tras haber esperado un tiempo aleatorio entre **1** y **10** segundos, con la finalidad de evitar una posible colisión con otros clientes DHCP de la misma red, como por ejemplo, cuando se les da energía a todos a la vez tras una interrupción. En caso de que en dicha red no exista ningún servidor DHCP, el router deberá tener un agente **DHCP relay**, para hacer llegar dicho mensaje DHCPDISCOVER a otras redes.

Tras enviar el DHCPDISCOVER, el cliente pasa al estado **SELECTING**, donde recibirá las ofertas por parte del servidor (o servidores) DHCP de la red (**DHCPOFFER**). Tras ello, seleccionará una de ellas en caso de haber varias, comunicándoselo al servidor DHCP a través de un paquete **DHCPREQUEST** y quedando por tanto informados de la decisión el resto de servidores, el cuál le responderá con un **DHCPACK** que contendrá la configuración de red para el cliente. Opcionalmente, el cliente envía una petición **ARP** a la dirección broadcast con la dirección IP que el servidor DHCP le ha ofrecido para asegurarse que nadie le responde y por tanto, concluir que es única en la red y no va a causar una colisión. Llegados a este punto se plantean **dos** escenarios:

* Que la IP ya se encuentre en uso por otra máquina, por lo que se informaría al servidor a través de un **DHCPDECLINE**, rechazando por tanto la configuración ofrecida en el DHCPACK y volviendo al estado **INIT**, por lo que el proceso se repetiría de nuevo.
* Que la IP se encuentre libre, por lo que el cliente acepta el DHCPACK y realiza la autoconfiguración con los parámetros especificados en dicho paquete, pasando por lo tanto al estado **BOUND**.

En el estado **BOUND**, el cliente recibe 3 valores de temporización, que empiezan a contar desde que se generó el paquete DHCPACK:

* **T1:** Temporizador de renovación de alquiler. El valor puede ser especificado en el servidor, de lo contrario, se calcula como **T3 x 0,5**.
* **T2:** Temporizador de reenganche. El valor puede ser especificado en el servidor, de lo contrario, se calcula como **T3 x 0,875**.
* **T3:** Temporizador de alquiler. El valor ha de ser especificado en el servidor obligatoriamente.

El cliente ha estado utilizando la configuración de red asignada hasta que expira **T1**. Pasado este tiempo, el cliente pasa al estado **RENEWING** y ha de contactar por primera vez con el servidor (pues hasta este momento únicamente ha contactado para hacer la petición de configuración inicial), preguntando si puede seguir con la configuración anteriormente asignada o de lo contrario, es necesario cambiarla. En este caso se plantean **tres** posibles escenarios:

* Que el servidor acepte en renovar la misma concesión, de manera que el cliente no tiene que cambiar la configuración de red. Los temporizadores se vuelven a configurar gracias a un paquete DHCPACK que el servidor ha enviado y el cliente vuelve al estado BOUND.
* Que el servidor no acepte la renovación de la concesión, de manera que servidor enviará un paquete DHCPNACK y el cliente deberá liberar la configuración de red que tenía asignada y moverse al estado INIT, para volver a repetir el proceso y obtener una nueva configuración de red. Puede ocurrir cuando se cambia alguna configuración del servidor, como por ejemplo el rango de direcciones IP que puede conceder.
* Que el servidor no responda, de manera que el cliente seguirá tratando de contactar con el mismo. En caso de que no obtenga ninguna respuesta, pasará al estado **REBINDING** cuando haya expirado el temporizador **T2**. Puede ocurrir cuando el servidor ha tenido algún tipo de problema con el servicio encargado o incluso por un fallo físico.

En el estado **REBINDING**, el cliente pasa al _plan B_, de manera que deja de intentar contactar con el servidor que le dio la configuración inicial, ya que da por hecho que el servidor original ha caído e intenta contactar con cualquier servidor DHCP existente en la red a través de un paquete DHCPREQUEST. Podemos plantear de nuevo **tres** posibles escenarios:

* Si cualquier servidor DHCP de los existentes en la red le responde con un paquete DHCPACK, el cliente seguirá utilizando la configuración de red que tenía anteriormente asignada, volviendo a configurar los temporizadores con los valores incluidos en dicho paquete y volviendo por tanto, al estado BOUND.
* Si cualquier servidor DHCP de los existentes en la red le responde con un paquete DHCPNACK, el cliente deberá liberar la configuración de red que tenía asignada y moverse al estado INIT, para volver a repetir el proceso y obtener una nueva configuración de red.
* Si tras intentar de contactar con cualquiera de los servidores DHCP sigue sin obtener respuesta y expira el temporarizador **T3**, el cliente debe liberar toda la configuración de red que tenía configurada.

En caso de que el cliente que tuviese que liberar la configuración de red fuese un cliente **Linux**, la configuración queda totalmente eliminada, y por tanto, la tarjeta de red queda desconfigurada, mientras que en el caso de un cliente **Windows**, la configuración se elimina pero se establece de forma automática una dirección **APIPA** (Automatic Private Internet Protocol Addressing), dentro del rango **169.254.0.1** a **169.254.255.254** con máscara **255.255.0.0** (/16), con la finalidad de permitir que el equipo funcione en un esquema de red local, pero no proporcionará salida fuera de la misma a otras redes, pues no tiene ninguna configuración más allá de la dirección IP (como por ejemplo los servidores DNS o la puerta de enlace predeterminada).

El cliente puede liberar dicha configuración de red de forma voluntaria sin tener que esperar a que expiren los temporizadores, cancelando por tanto el alquiler y volviendo a quedar disponible para otro usuario dicha dirección IP. Es lo que se conoce como **DHCPRELEASE**. En Linux debemos ejecutar el comando `dhclient -r` y en Windows, `ipconfig /release`.

Si los clientes DHCP cuentan con disco duro, la dirección IP asignada queda almacenada en un fichero y cuando la máquina vuelva a arrancar, puede hacer una nueva petición utilizando dichos valores (estado **REBOOTING**).

## Tarea 2: Entrega el fichero Vagrantfile que define el escenario.

En este caso, vamos a montar un escenario con **Vagrant** que contará con las siguientes máquinas:

* **servidor:** Cuenta con dos tarjetas de red, una pública y una privada, estando esta última conectada a una red interna.
* **nodo_lan1:** Cuenta con una tarjeta de red conectada a la red interna.

Tendremos que instalar un servidor DHCP dentro de la máquina **servidor** que de servicio a los ordenadores de la red interna, teniendo en cuenta que el tiempo de concesión sea de **12 horas** y utilice un direccionamiento **192.168.100.0/24**. En este caso, el fichero de configuración **Vagrantfile** tiene la siguiente forma:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define :servidor do |servidor| #Definimos la primera máquina, en este caso, el servidor.
    servidor.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    servidor.vm.hostname = "servidor" #Establecemos el nombre (hostname) de la máquina. En este caso, "servidor", tal y como se ha pedido.
    servidor.vm.network :public_network,:bridge=>"enp3s0", #Creamos una interfaz de red conectada en modo puente (bridge) a nuestra interfaz de red física "enp3s0", para que obtenga dirección por DHCP en nuestra red doméstica.
      use_dhcp_assigned_default_route: true #Especificamos que utilice los valores obtenidos por DHCP a través de esta interfaz como valores predeterminados, de manera que no hará uso de los valores DHCP obtenidos por la interfaz "eth0" que crea Vagrant automáticamente.
    servidor.vm.network :private_network, ip: "192.168.100.1", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor y nos conviene que la IP sea siempre la misma, en este caso, la primera dirección válida dentro de dicha red.
      virtualbox__intnet: "lan1" #Dado que VirtualBox crea por defecto las interfaces en modo host-only, tenemos que indicar que utilice una red interna, en este caso, una de nombre "lan1".
  end
  config.vm.define :nodo_lan1 do |nodo_lan1| #Definimos la segunda máquina, en este caso, el cliente.
    nodo_lan1.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    nodo_lan1.vm.hostname = "nodolan1" #Establecemos el nombre (hostname) de la máquina. En este caso, "nodolan1", tal y como se ha pedido.
    nodo_lan1.vm.network :private_network, #Creamos una interfaz de red conectada a una red privada, en este caso sin especificar un direccionamiento estático ni DHCP.
      auto_config: false, #En este caso, la configuración la vamos a llevar a cabo manualmente en el fichero /etc/network/interfaces, por el motivo que se explicará a continuación.
      virtualbox__intnet: "lan1" #Al igual que la segunda interfaz de red del servidor, ésta también la conectaremos a la red interna de nombre "lan1".
    config.vm.provision "shell", #Especificamos que vamos a ejecutar un script durante el arranque de la máquina.
      inline: "sudo ip r del default via 10.0.2.2", #Se especifica el comando a ejecutar, en este caso, eliminamos la puerta de enlace por defecto que obtiene la máquina cliente a través del DHCP por la interfaz "eth0".
      run: "always" #Indicamos que el script se ejecute cada vez que se arranca la máquina, ya que por defecto lo hace la primera vez.
  end
end
{% endhighlight %}

Considero que es necesario explicar el por qué de haber elegido configurar la red directamente en el **/etc/network/interfaces** en lugar de hacerlo en el fichero **Vagrantfile**, indicando que la configuración sea **type: "dhcp"**. Bien, partimos del punto en que Vagrant permite utilizar la instrucción **use_dhcp_assigned_default_route: true**. Gracias a la misma, indicamos a Vagrant que utilice por defecto la configuración DHCP obtenida a través de dicha interfaz, en lugar de utilizar aquella que se asigna a través de la interfaz "**eth0**" que crea Vagrant de forma automática, pero esta instrucción tiene un inconveniente, y es que únicamente funciona con interfaces de red conectadas a redes públicas, por lo que en la máquina "**servidor**" funciona perfectamente, ya que la configuración la obtenemos desde la interfaz que se encuentra conectada en modo puente a nuestra red doméstica, sin embargo, en la máquina "**nodo_lan1**" no funciona, ya que la configuración la obtenemos por DHCP a través de una red privada.

Dejando dicha instrucción de lado, si ponemos **type: "dhcp"** en la configuración de dicha interfaz, Vagrant va a configurar todos los parámetros con la configuración obtenida por DHCP a través de dicha interfaz excepto la puerta de enlace predeterminada, por lo que deja la del router virtual de VirtualBox (**10.0.2.2**). Es algo realmente extraño. Desconozco si es un bug, pero en la versión **2.2.3** de Vagrant, ese es el funcionamiento.

La otra opción sería ejecutar un script que eliminase la puerta de enlace predeterminada del router virtual de VirtualBox, para que así Vagrant pueda configurar automáticamente la nueva puerta de enlace obtenida a través de la interfaz que se encuentra en la red interna, pero tampoco es posible, ya que el script es lo último que se ejecuta durante el arranque, de manera que cuando trate de eliminar dicha puerta de enlace, la configuración a través de DHCP ya se habrá llevado a cabo, y lo único que conseguiremos es quedarnos sin puerta de enlace predeterminada.

Otra de las opciones posibles era ejecutar dicho script que eliminase la puerta de enlace predeterminada y a continuación, indicarle que añada la que deseemos, en este caso, la **192.168.100.1**, pero me parece una solución un poco cutre, ya que la puerta de enlace predeterminada podría cambiar, por lo que sería necesario realizar el cambio aquí también.

Finalmente, me decanté por eliminar en el script la puerta de enlace predeterminada del router de VirtualBox y realizar la configuración de red en el fichero **/etc/network/interfaces** (ya que dicha configuración se lleva a cabo _a posteriori_, de manera que cuando se ejecute, no habrá puerta de enlace predeterminada y no tendrá más remedio que usar aquella obtenida a través de DHCP por la red interna).

## Tarea 3: Muestra el fichero de configuración del servidor, la lista de concesiones, la modificación en la configuración que has hecho en el cliente para que tome la configuración de forma automática y muestra la salida del comando `ip address`.

De primeras vamos a iniciar únicamente la máquina **servidor**, ya que no tiene sentido iniciar la máquina **nodo_lan1**, pues todavía no tenemos ningún servidor DHCP configurado. Para ello, ejecutamos el comando:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant up servidor
{% endhighlight %}

A continuación comenzará a descargar el **box** (en caso de que no lo tuviésemos previamente) y a generar la máquina virtual con los parámetros que le hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la misma ejecutando la siguiente instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant ssh servidor
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
       valid_lft 86351sec preferred_lft 86351sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:48:cb:42 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.140/24 brd 192.168.1.255 scope global dynamic eth1
       valid_lft 86357sec preferred_lft 86357sec
    inet6 fe80::a00:27ff:fe48:cb42/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:5f:43:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5f:43cf/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **tres** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada en modo puente a nuestra interfaz física **enp3s0**, con dirección IP obtenida por DHCP **192.168.1.140**.
* **eth2:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **192.168.100.1**.

Para verificar también que la puerta de enlace predeterminada se ha configurado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@servidor:~$ ip r
default via 192.168.1.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.140 
192.168.100.0/24 dev eth2 proto kernel scope link src 192.168.100.1
{% endhighlight %}

En este caso, la puerta de enlace predeterminada es mi router doméstico, es decir, la **192.168.1.1**, gracias a haber habilitado la instrucción **use_dhcp_assigned_default_route**.

Una vez que se han llevado a cabo todas las comprobaciones oportunas, procedemos a configurar el servidor DHCP que dará servicio a las máquinas de la red interna. Para ello, lo primero que haremos será instalar el paquete necesario para configurar el servidor DHCP (**isc-dhcp-server**), no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@servidor:~# apt update && apt upgrade && apt install isc-dhcp-server
Reading package lists... Done
Building dependency tree       
Reading state information... Done
...
Setting up isc-dhcp-server (4.4.1-2) ...
Generating /etc/default/isc-dhcp-server...
Job for isc-dhcp-server.service failed because the control process exited with error code.
See "systemctl status isc-dhcp-server.service" and "journalctl -xe" for details.
invoke-rc.d: initscript isc-dhcp-server, action "start" failed.
● isc-dhcp-server.service - LSB: DHCP server
   Loaded: loaded (/etc/init.d/isc-dhcp-server; generated)
   Active: failed (Result: exit-code) since Wed 2020-10-07 14:57:19 GMT; 14ms ago
     Docs: man:systemd-sysv-generator(8)
  Process: 12851 ExecStart=/etc/init.d/isc-dhcp-server start (code=exited, status=1/FAILURE)

Oct 07 14:57:17 servidor dhcpd[12863]: bugs on either our web page at www.isc.org or in the README file
Oct 07 14:57:17 servidor dhcpd[12863]: before submitting a bug.  These pages explain the proper
Oct 07 14:57:17 servidor dhcpd[12863]: process and the information we find helpful for debugging.
Oct 07 14:57:17 servidor dhcpd[12863]: 
Oct 07 14:57:17 servidor dhcpd[12863]: exiting.
Oct 07 14:57:19 servidor isc-dhcp-server[12851]: Starting ISC DHCPv4 server: dhcpdcheck syslog for diagnostics. ... failed!
Oct 07 14:57:19 servidor isc-dhcp-server[12851]:  failed!
Oct 07 14:57:19 servidor systemd[1]: isc-dhcp-server.service: Control process exited, code=exited, status=1/FAILURE
Oct 07 14:57:19 servidor systemd[1]: isc-dhcp-server.service: Failed with result 'exit-code'.
Oct 07 14:57:19 servidor systemd[1]: Failed to start LSB: DHCP server.
Processing triggers for man-db (2.8.5-2) ...
Processing triggers for libc-bin (2.28-10) ...
Processing triggers for systemd (241-7~deb10u4) ...
{% endhighlight %}

Como se puede apreciar en la salida por pantalla, la instalación nos ha devuelto un error. No hay nada de qué preocuparse, es un fallo totalmente normal, pues el servidor ha detectado que no existe ninguna configuración existente y por ello, devuelve dicho error.

El siguiente paso será indicar la interfaz de red a través de la cuál va a servir el servidor DHCP, en este caso **eth2**, pues es la que se encuentra dentro de la red interna **192.168.100.0/24**. Para ello, vamos a modificar el fichero **/etc/default/isc-dhcp-server**, ejecutando el comando:

{% highlight shell %}
root@servidor:~# nano /etc/default/isc-dhcp-server
{% endhighlight %}

{% highlight shell %}
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#       Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#       Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4=""
INTERFACESv6=""
{% endhighlight %}

En este caso, en la línea que nos debemos fijar es aquella con nombre **INTERFACESv4**. Ahí debemos indicar la interfaz de red a través de la cuál va a servir direcciones. En caso de querer configurar un servidor **DHCPv6**, tendríamos que especificarlo en **INTERFACESv6**, pero no es el caso. La línea debe quedar de la siguiente forma:

{% highlight shell %}
INTERFACESv4="eth2"
{% endhighlight %}

Tras ello, guardaremos los cambios, pero la configuración todavía no ha terminado, pues queda lo más importante, el fichero principal de configuración **/etc/dhcp/dhcpd.conf**. Comenzaremos a editarlo haciendo uso del comando:

{% highlight shell %}
root@servidor:~# nano /etc/dhcp/dhcpd.conf
{% endhighlight %}

{% highlight shell %}
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#

# option definitions common to all supported networks...
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;
...
{% endhighlight %}

Este fichero de configuración se encuentra dividido en **dos** partes:

* **Parte principal:** Especifica los parámetros generales que afectarán a todas las secciones, a no ser que dentro de las mismas se indique lo contrario.
* **Secciones:** Pueden ser **subnets**, que especifican los rangos de direcciones IP que serán cedidas a los clientes que lo soliciten o **hosts**, que especifican una configuración concreta para un determinado equipo.

En la **parte principal** podemos configurar los siguientes parámetros, que posteriormente podremos especificar en las secciones en caso de que queramos una configuración más concreta para dicha sección, que tendrá prioridad sobre aquella especificada en la parte principal:

* `max-lease-time:` Es el tiempo máximo en segundos de concesión que un cliente puede solicitar. Si por ejemplo, un cliente solicita una concesión de 900 segundos pero el tiempo máximo es de 600 segundos, la concesión tendrá una duración de 600 segundos. No tiene por qué ser **T3** o **temporizador de alquiler**.
* `min-lease-time:` Es el tiempo mínimo en segundos de concesión que un cliente puede solicitar. Si por ejemplo, un cliente solicita una concesión de 900 segundos pero el tiempo mínimo es de 1200 segundos, la concesión tendrá una duración de 1200 segundos.
* `default-lease-time:` Es el tiempo por defecto en segundos de concesión que se le asignará a un cliente en caso de que éste no haya solicitado ningún periodo en concreto. No confundir con **T1** o **temporizador de renovación de alquiler**.
* `option dhcp-renewal-time:` Es el tiempo en segundos que ha de transcurrir hasta que el cliente pase al estado RENEWAL. También conocido como **T1** o **temporizador de renovación de alquiler**. No confundir con **default-lease-time**.
* `option dhcp-rebinding-time:` Es el tiempo en segundos que ha de transcurrir hasta que el cliente pase al estado REBINDING. También conocido como **T2** o **temporizador de reenganche**.
* `option routers:` Se especifica la dirección del router a través de la cuál saldrá al exterior (puerta de enlace predeterminada).
* `option domain-name-servers:` Se especifican las direcciones de los servidores DNS que va a utilizar el cliente, separados por comas.
* `option domain­-name:` Nombre del dominio que se envía al cliente.
* `option subnet­mask:` Máscara de red que vamos a utilizar, aunque se puede especificar en la parte de la subnet.
* `option broadcast-­address:` Dirección de difusión (broadcast) de la red.

En las **secciones** debemos especificar obligatoriamente los siguientes parámetros (es compatible con aquellos indicados en la parte principal):

* `subnet:` Indica la red sobre la que se van a realizar las concesiones.
* `netmask:` Indica la máscara de la red sobre la que se van a realizar las concesiones.
* `range:` Indica el rango de direcciones que el servidor DHCP puede asignar.

En este caso, el fichero trae una configuración de ejemplo por defecto, la cuál vamos a comentar para no confundir y dejar el fichero mucho más limpio, especificando todos los parámetros de forma concreta dentro de la sección, ya que no tiene sentido especificar el tiempo de concesión en la parte principal dado que uno de los siguientes ejercicios pide crear otra sección con un tiempo de concesión distinto. Para esta tarea se ha pedido que el tiempo de concesión de **12 horas** y que la red sea la **192.168.100.0/24**, por lo que el fichero debe quedar de la siguiente manera:

{% highlight shell %}
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.10 192.168.100.200;
  default-lease-time 43200;
  max-lease-time 43200;
  option routers 192.168.100.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
{% endhighlight %}

La red que hemos especificado es la **192.168.100.0**, con una máscara de red **255.255.255.0** (/24). El rango de direcciones dentro de dicha que se van a asignar va desde la **192.168.100.10** hasta la **192.168.100.200**, aunque se podría haber utilizado uno totalmente distinto. El tiempo de concesión por defecto y máximo lo hemos establecido en **43200** segundos (12 horas), vuelvo a hacer hincapié en que no ha de confundirse con T1 y T3, respectivamente. La puerta de enlace predeterminada es la IP de la interfaz de red del servidor perteneciente a la red interna, es decir, la **192.168.100.1** y por último, usaremos el DNS de Google (**8.8.8.8**) y el **1.1.1.1**.

Siempre que hagamos modificaciones en un fichero de configuración de un servicio, debemos reiniciarlo para que así cargue la nueva configuración. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# systemctl restart isc-dhcp-server
{% endhighlight %}

Tras ello, verificaremos que no hayamos cometido ningún error en la configuración de dicho servicio y que por lo tanto, se haya reiniciado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# systemctl status isc-dhcp-server
● isc-dhcp-server.service - LSB: DHCP server
   Loaded: loaded (/etc/init.d/isc-dhcp-server; generated)
   Active: active (running) since Wed 2020-10-07 14:59:28 GMT; 5s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 13156 ExecStart=/etc/init.d/isc-dhcp-server start (code=exited, status=0/SUCCESS)
    Tasks: 1 (limit: 544)
   Memory: 6.3M
   CGroup: /system.slice/isc-dhcp-server.service
           └─13168 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf eth2

Oct 07 14:59:26 servidor systemd[1]: Starting LSB: DHCP server...
Oct 07 14:59:26 servidor isc-dhcp-server[13156]: Launching IPv4 server only.
Oct 07 14:59:26 servidor dhcpd[13168]: Wrote 0 leases to leases file.
Oct 07 14:59:26 servidor dhcpd[13168]: Server starting service.
Oct 07 14:59:28 servidor isc-dhcp-server[13156]: Starting ISC DHCPv4 server: dhcpd.
Oct 07 14:59:28 servidor systemd[1]: Started LSB: DHCP server.
{% endhighlight %}

Efectivamente, el servicio se ha reiniciado correctamente y ya está totalmente operativo (**active**). Ya estamos preparados para arrancar ahora sí la máquina cliente **nodo_lan1**. Para ello, abriremos una nueva terminal y así tendremos cada una de las máquinas en una terminal para no tener que estar saliendo y entrando todo el rato. Para levantar dicha máquina cliente, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant up nodo_lan1
{% endhighlight %}

Una vez que haya finalizado el proceso, nos podremos conectar a la misma ejecutando la siguiente instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant ssh nodo_lan1
Linux nodolan1 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86323sec preferred_lft 86323sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada a una red interna **lan1**, sin ninguna configuración ya que hemos indicado que no la configure.

Para verificar también que la puerta de enlace predeterminada se ha eliminado correctamente gracias a la ejecución del script, ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip r
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15
{% endhighlight %}

En este momento, la máquina cliente **nodo_lan1** no tiene ninguna puerta de enlace predeterminada. Es momento de configurar el fichero **/etc/network/interfaces** para establecer la configuración de dicha interfaz de red. Para ello, ejecutamos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ sudo nano /etc/network/interfaces
{% endhighlight %}

{% highlight shell %}
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
{% endhighlight %}

En este caso, debemos añadir las siguientes líneas para configurar la interfaz **eth1** de manera automática por DHCP:

{% highlight shell %}
auto eth1
iface eth1 inet dhcp
{% endhighlight %}

De esta forma, la interfaz se levantará automáticamente durante el arranque (**auto**) y obtendrá la configuración de red por DHCP (**inet dhcp**). Para no tener que reiniciar la máquina, podemos hacer la petición DHCP de forma manual para la interfaz **eth1**, ejecutando para ello el comando:

{% highlight shell %}
vagrant@nodolan1:~$ sudo dhclient eth1
{% endhighlight %}

Tras ello, ejecutaremos el comando `ip a show dev eth1` para que nos muestre información referente únicamente a la interfaz de red de nombre **eth1**, pues es la que nos interesa:

{% highlight shell %}
vagrant@nodolan1:~$ ip a show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.10/24 brd 192.168.100.255 scope global dynamic eth1
       valid_lft 43189sec preferred_lft 43189sec
    inet6 fe80::a00:27ff:fe2c:2572/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la interfaz de red ya se encuentra completamente configurada y con la primera dirección dentro del rango especificado, la **192.168.100.10**. De igual manera, vamos a ejecutar el comando `ip r` para verificar que ha añadido dicha puerta de enlace predeterminada:

{% highlight shell %}
vagrant@nodolan1:~$ ip r
default via 192.168.100.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.100.0/24 dev eth1 proto kernel scope link src 192.168.100.10
{% endhighlight %}

Como era de esperar, la puerta de enlace predeterminada que ha asignado ha sido la que nosotros hemos especificado, es decir, la dirección IP de la interfaz de red del router perteneciente a la red interna, en este caso, la **192.168.100.1**. Otra prueba muy importante es verificar que también ha asignado los dos servidores DNS que hemos establecido. Para ello, tendremos que ver el contenido del fichero **/etc/resolv.conf**, ejecutando para ello el comando:

{% highlight shell %}
vagrant@nodolan1:~$ cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
{% endhighlight %}

Efectivamente, se encuentran configurados ambos servidores DNS. Por último vamos a hacer un `ping` para comprobar la conectividad al exterior, en este caso a una máquina que sabemos que nos va a contestar seguro, como por ejemplo:

{% highlight shell %}
vagrant@nodolan1:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
11 packets transmitted, 0 received, 100% packet loss, time 246ms
{% endhighlight %}

En este caso, no ha sido posible tener conectividad con el exterior, ya que nos falta un router que ejecute el protocolo NAT para realizar la traducción de direcciones y permitir así que los ordenadores de la red interna puedan salir al exterior haciendo uso de la IP "pública" del router, en este caso, la que ha sido concedida por el DHCP de mi router doméstico, la **192.168.1.140**.

Antes de comenzar a configurar NAT, vamos a pasar al servidor para comprobar el fichero del servidor para ver la lista de concesiones actualmente existentes (que en un principio, únicamente debería existir la concesión de la máquina **nodo_lan1**). Este fichero es **/var/lib/dhcp/dhcpd.leases**. Para visualizarlo, ejecutamos el comando:

{% highlight shell %}
vagrant@servidor:~$ cat /var/lib/dhcp/dhcpd.leases
# The format of this file is documented in the dhcpd.leases(5) manual page.
# This lease file was written by isc-dhcp-4.4.1

# authoring-byte-order entry is generated, DO NOT DELETE
authoring-byte-order little-endian;

server-duid "\000\001\000\001'\020\224N\010\000'_C\317";

lease 192.168.100.10 {
  starts 3 2020/10/07 15:42:11;
  ends 4 2020/10/08 03:42:11;
  cltt 3 2020/10/07 15:42:11;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 08:00:27:2c:25:72;
  client-hostname "nodolan1";
}
{% endhighlight %}

En este caso, existe una única concesión (lease) de IP **192.168.100.10**, perteneciente a la máquina (client-hostname) de nombre **nodolan1**, con MAC (hardware ethernet) **08:00:27:2c:25:72** que comenzó (starts) el día **2020/10/07** a las **15:42:11** y finaliza (ends) el día **2020/10/08** a las **03:42:11**, es decir, justamente 12 horas después.

## Tarea 4: Configura el servidor para que funcione como router y NAT, de forma que los clientes tengan Internet.

Es hora de hacer que las máquinas pertenecientes a la red interna sean capaces de salir al exterior, para ello, vamos a configurar NAT en la máquina servidora. Lo primero que tendremos que hacer será activar el **bit de forward**, permitiendo así que los paquetes puedan pasar de una interfaz a otra en el servidor, ya que por defecto, en Debian viene deshabilitado por razones de seguridad. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# echo 1 > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}

Esta modificación no sería persistente, ya que se encuentra en memoria, por lo que si quisiésemos hacer que perdure en el tiempo, tendríamos que modificar el fichero **/etc/sysctl.conf** y asignar el valor `1` a la instrucción `net.ipv4.ip_forward`.

Tras ello, tendremos que instalar el paquete que nos va a permitir hacer NAT. En este caso se trata de **nftables**, un cortafuegos que consta con dicha funcionalidad. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# apt install nftables
{% endhighlight %}

Cuando el paquete haya terminado de instalarse, tendremos que arrancar y habilitar el demonio, de manera que se inicie cada vez que se arranque la máquina y lea la configuración almacenada en el fichero (lo veremos a continuación). Para ello, ejecutamos los comandos:

{% highlight shell %}
root@servidor:~# systemctl start nftables.service
root@servidor:~# systemctl enable nftables.service
{% endhighlight %}

Una vez habilitado y arrancado el demonio, tendremos que crear una nueva tabla en nftables de nombre **nat**. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# nft add table nat
{% endhighlight %}

Para verificar que la creación de dicha tabla se ha realizado correctamente, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# nft list tables
table inet filter
table ip nat
{% endhighlight %}

Efectivamente, así ha sido. Dentro de dicha tabla tendremos que crear la cadena **POSTROUTING**, es decir, aquella que permite modificar paquetes justo antes de que salgan del equipo, permitiendo por tanto hacer **Source NAT** (SNAT), para posteriormente alojar dentro de la misma, la regla que necesitamos. Para crear dicha cadena ejecutaremos el comando:

{% highlight shell %}
root@servidor:~# nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
{% endhighlight %}

En este caso, hemos especificado que esta cadena sea poco prioritaria (a mayor sea el número, menor es la prioridad), de manera que si en un futuro quisiéramos añadir una cadena **PREROUTING**, es decir, aquella que permite modificar paquetes entrantes antes de que se tome una decisión de enrutamiento, permitiendo por tanto hacer **Destination NAT** (DNAT), bastaría con indicarle a esta última una prioridad más alta, de manera que las reglas albergadas en el interior de la cadena **PREROUTING** se ejecuten antes que las de la cadena **POSTROUTING**.

De nuevo, vamos a verificar que dicha cadena se ha generado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# nft list chains
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

Nos queda un último paso, añadir la regla necesaria a la cadena POSTROUTING, de manera que nos permita hacer SNAT dinámico (**masquerade**), pues la dirección IP "pública" del servidor no siempre va a ser la misma, al haber sido asignada por DHCP a través de mi router doméstico. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@servidor:~# nft add rule ip nat postrouting oifname "eth1" ip saddr 192.168.100.0/24 counter masquerade
{% endhighlight %}

En este caso, hemos especificado que se trata de una regla para la cadena **postrouting** (pues debe aplicarse justo antes de salir de la máquina), además, la interfaz (oifname) que debemos introducir será aquella por la que van a salir los paquetes, es decir, la que está conectada a Internet, en este caso, **eth1**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que provengan de la red (ip saddr) **192.168.100.0/24**. Además, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a enmascararlos, pues la IP "pública" del router es dinámica (**masquerade**).

Por último, vamos a volver a verificar que dicha regla se ha añadido correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# nft list ruleset
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
		oifname "eth1" ip saddr 192.168.100.0/24 counter packets 2 bytes 143 masquerade
	}
}
{% endhighlight %}

Efectivamente, así ha sido. Al igual que pasaba con el **bit de forward**, esta configuración se encuentra cargada en memoria, por lo que para que conseguir que perdure en el tiempo, vamos a ejecutar el comando:

{% highlight shell %}
root@servidor:~# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Gracias a ello, habremos guardado la configuración en un fichero en **/etc/** de nombre **nftables.conf** que se importará de forma automática cuando reiniciemos la máquina gracias al daemon que se encuentra habilitado. En caso de que no cargase de nuevo la configuración, podríamos hacerlo manualmente con el comando `nft -f /etc/nftables.conf`.

Es hora de comprobar que esta configuración funciona, así que volveremos a la máquina cliente y trataremos de hacer `ping` a un nombre, para así comprobar además que la resolución DNS se lleva a cabo, por ejemplo a **google.es**. Para ello, ejecutamos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ping google.es
PING google.es (216.58.201.163) 56(84) bytes of data.
64 bytes from arn02s06-in-f163.1e100.net (216.58.201.163): icmp_seq=1 ttl=118 time=11.2 ms
64 bytes from arn02s06-in-f163.1e100.net (216.58.201.163): icmp_seq=2 ttl=118 time=11.5 ms
64 bytes from arn02s06-in-f163.1e100.net (216.58.201.163): icmp_seq=3 ttl=118 time=11.8 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 11.159/11.507/11.819/0.270 ms
{% endhighlight %}

La máquina tiene conectividad con el exterior y es capaz de realizar la resolución DNS.

## Tarea 5: Realizar una captura desde el servidor usando tcpdump de los cuatro paquetes que corresponden a una concesión: DISCOVER, OFFER, REQUEST y ACK.

Lo primero que tendremos que hacer es liberar la concesión de la dirección actualmente asignada al cliente, para posteriormente hacer una nueva petición y capturarla, así que volveremos a la terminal del cliente y ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ sudo dhclient -r eth1
Killed old client process
{% endhighlight %}

Para verificar que la dirección IP se ha liberado correctamente, volvemos a ejecutar el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip a show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe2c:2572/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, **eth1** ya no cuenta con dirección **IPv4** asignada, así que es hora de volver al servidor para configurar la captura de paquetes. Lo primero que tendremos que hacer será instalar el paquete necesario para ello (**tcpdump**), ejecutando el comando:

{% highlight shell %}
vagrant@servidor:~# sudo apt install tcpdump
{% endhighlight %}

Una vez instalado, tendremos que tener claro qué es lo que queremos capturar, para poder aplicar un filtro y que salgan únicamente los paquetes deseados. En este caso necesitamos capturar el tráfico en el servidor únicamente a través de **eth2** (opción **-i**), pues es la interfaz conectada a la red interna. Además, para quitar posible tráfico indeseado que viajase por esa interfaz, vamos a filtrar únicamente el tráfico de los puertos **67** y **68** (opción **port**), pues son los puertos que usa DHCP. Además, vamos a capturar el modo _verbose_ (opción **-vv**) para que muestre la máxima información posible. El comando final a ejecutar sería:

{% highlight shell %}
vagrant@servidor:~$ sudo tcpdump -vv -i eth2 port 67 and 68
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
{% endhighlight %}

El servidor ya se encuentra capturando paquetes, así que vamos a volver al cliente y vamos a hacer la petición DHCP para capturarla. Para ello, ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ sudo dhclient eth1
{% endhighlight %}

Para verificar que el cliente ha obtenido correctamente una dirección IP por DHCP en dicha interfaz, ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip a show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.10/24 brd 192.168.100.255 scope global dynamic eth1
       valid_lft 43186sec preferred_lft 43186sec
    inet6 fe80::a00:27ff:fe2c:2572/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, ha obtenido la dirección **192.168.100.10** por DHCP, así que el tráfico ha debido quedar correctamente capturado en el servidor. Vamos a volver al mismo para comprobarlo.

{% highlight shell %}
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
07:06:33.032842 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    0.0.0.0.bootpc > 255.255.255.255.bootps: [udp sum ok] BOOTP/DHCP, Request from 08:00:27:2c:25:72 (oui Unknown), length 300, xid 0x96dc2e7d, Flags [none] (0x0000)
	  Client-Ethernet-Address 08:00:27:2c:25:72 (oui Unknown)
	  Vendor-rfc1048 Extensions
	    Magic Cookie 0x63825363
	    DHCP-Message Option 53, length 1: Discover
	    Hostname Option 12, length 8: "nodolan1"
	    Parameter-Request Option 55, length 13: 
	      Subnet-Mask, BR, Time-Zone, Default-Gateway
	      Domain-Name, Domain-Name-Server, Option 119, Hostname
	      Netbios-Name-Server, Netbios-Scope, MTU, Classless-Static-Route
	      NTP
07:06:34.053330 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    192.168.100.1.bootps > 192.168.100.10.bootpc: [udp sum ok] BOOTP/DHCP, Reply, length 300, xid 0x96dc2e7d, Flags [none] (0x0000)
	  Your-IP 192.168.100.10
	  Client-Ethernet-Address 08:00:27:2c:25:72 (oui Unknown)
	  Vendor-rfc1048 Extensions
	    Magic Cookie 0x63825363
	    DHCP-Message Option 53, length 1: Offer
	    Server-ID Option 54, length 4: 192.168.100.1
	    Lease-Time Option 51, length 4: 43200
	    Subnet-Mask Option 1, length 4: 255.255.255.0
	    Default-Gateway Option 3, length 4: 192.168.100.1
	    Domain-Name-Server Option 6, length 8: dns.google,one.one.one.one
07:06:34.054493 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    0.0.0.0.bootpc > 255.255.255.255.bootps: [udp sum ok] BOOTP/DHCP, Request from 08:00:27:2c:25:72 (oui Unknown), length 300, xid 0x96dc2e7d, Flags [none] (0x0000)
	  Client-Ethernet-Address 08:00:27:2c:25:72 (oui Unknown)
	  Vendor-rfc1048 Extensions
	    Magic Cookie 0x63825363
	    DHCP-Message Option 53, length 1: Request
	    Server-ID Option 54, length 4: 192.168.100.1
	    Requested-IP Option 50, length 4: 192.168.100.10
	    Hostname Option 12, length 8: "nodolan1"
	    Parameter-Request Option 55, length 13: 
	      Subnet-Mask, BR, Time-Zone, Default-Gateway
	      Domain-Name, Domain-Name-Server, Option 119, Hostname
	      Netbios-Name-Server, Netbios-Scope, MTU, Classless-Static-Route
	      NTP
07:06:34.059148 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    192.168.100.1.bootps > 192.168.100.10.bootpc: [udp sum ok] BOOTP/DHCP, Reply, length 300, xid 0x96dc2e7d, Flags [none] (0x0000)
	  Your-IP 192.168.100.10
	  Client-Ethernet-Address 08:00:27:2c:25:72 (oui Unknown)
	  Vendor-rfc1048 Extensions
	    Magic Cookie 0x63825363
	    DHCP-Message Option 53, length 1: ACK
	    Server-ID Option 54, length 4: 192.168.100.1
	    Lease-Time Option 51, length 4: 43200
	    Subnet-Mask Option 1, length 4: 255.255.255.0
	    Default-Gateway Option 3, length 4: 192.168.100.1
	    Domain-Name-Server Option 6, length 8: dns.google,one.one.one.one
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
{% endhighlight %}

Efectivamente, así ha sido. La captura se compone de **cuatro** paquetes claramente diferenciados:

* **DHCPDISCOVER:** Utiliza la dirección **0.0.0.0** para enviarse a la dirección broadcast **255.255.255.255**, contiene además la dirección MAC del cliente (**08:00:27:2c:25:72**) y el hostname de dicha máquina (**nodolan1**). Si nos fijamos, está solicitando una serie de recursos, indicados en el apartado **Parameter-Request**, así como la máscara de red, la puerta de enlace, los DNS...
* **DHCPOFFER:** En este caso, dado que existe un único servidor que sirva a través de dicha interfaz, únicamente hemos capturado un paquete DHCPOFFER, que incluye los recursos anteriormente solicitados por parte del cliente, además de la IP ofrecida (**192.168.100.10**), la IP de dicho servidor (**192.168.100.1**), el tiempo de concesión (**43200 segundos**) y la dirección MAC del cliente al que va dirigida dicha oferta (**08:00:27:2c:25:72**).
* **DHCPREQUEST:** Dado que no tiene más paquetes DHCPOFFER, se queda con el único recibido, de manera que vuelve a utilizar la dirección **0.0.0.0** para enviarse a la dirección broadcast **255.255.255.255** junto con la dirección MAC del cliente (**08:00:27:2c:25:72**), indicando en este caso la dirección IP que quiere solicitar (**192.168.100.10**) y la dirección IP del servidor al que realiza dicha petición (**192.168.100.1**). Vuelve a solicitar de nuevo los parámetros de red para confirmar que son correctos y coinciden con los anteriormente ofrecidos.
* **DHCPACK:** Es el último paso de la concesión, que incluye de nuevo dichos recursos solicitados, identificando al cliente gracias a su dirección MAC (**08:00:27:2c:25:72**) y al servidor gracias a su dirección IP (**192.168.100.1**). Una vez recibido el paquete por parte del cliente, es posible que realice una petición ARP para verificar que esa IP es única en la red, y tras ello, realizar la autoconfiguración, para ya estar totalmente operativo.

## Tarea 6: Crea una reserva para el que el cliente tome siempre la dirección 192.168.100.100. Indica las modificaciones realizadas en los ficheros de configuración y entrega una comprobación de que el cliente ha tomado esa dirección.

Para crear una reserva, al igual que para indicar las subnets, tendremos que modificar el fichero **/etc/dhcp/dhcpd.conf**, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# nano /etc/dhcp/dhcpd.conf
{% endhighlight %}

Dentro del mismo, tendremos que añadir una nueva sección de tipo **host** con un nombre que identifique de manera única la configuración para dicho equipo. En este caso, tendremos que indicar la **dirección física** (MAC) del cliente y la **dirección IP** que le queremos reservar dentro del rango (por ejemplo, la **192.168.100.100**). En mi caso, la sección queda de esta forma:

{% highlight shell %}
host nodo_lan1 {
  hardware ethernet 08:00:27:2c:25:72;
  fixed-address 192.168.100.100;
}
{% endhighlight %}

Siempre que hagamos modificaciones en un fichero de configuración de un servicio, debemos reiniciarlo para que así cargue la nueva configuración. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# systemctl restart isc-dhcp-server
{% endhighlight %}

En mi caso opté por reiniciar la máquina cliente (comando `reboot`) para así forzarla a hacer una petición DHCP al servidor, de manera que le rechazaría la concesión con un paquete DHCPNACK y volverá al estado **INIT**, realizando de nuevo el proceso desde el principio, para así obtener una dirección IP dentro del rango (que en este caso será fija gracias a la reserva).

Para verificar que el cliente ha obtenido dicha dirección IP ejecutaremos el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip a show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global dynamic eth1
       valid_lft 43075sec preferred_lft 43075sec
    inet6 fe80::a00:27ff:fe2c:2572/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, ahora tiene asignada la dirección IP **192.168.100.100** gracias a la reserva creada. Esto puede ser muy útil en caso de que necesitemos que un determinado equipo tenga siempre la misma IP dentro del rango DHCP, aunque otra opción sería rediseñar el rango y poner dicho equipo con una IP estática fuera de dicho rango.

Vamos a verificar de la misma forma que también ha obtenido la puerta de enlace predeterminada, ejecutando el comando:

{% highlight shell %}
vagrant@nodolan1:~$ ip r
default via 192.168.100.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.100.0/24 dev eth1 proto kernel scope link src 192.168.100.100
{% endhighlight %}

En este caso, ha obtenido todos los parámetros correctamente, así que vamos a hacer _la prueba de fuego_, haciendo un `ping` al exterior, por ejemplo a **google.es**:

{% highlight shell %}
vagrant@nodolan1:~$ ping google.es
PING google.es (216.58.211.227) 56(84) bytes of data.
64 bytes from mad01s24-in-f227.1e100.net (216.58.211.227): icmp_seq=1 ttl=118 time=11.3 ms
64 bytes from mad01s24-in-f227.1e100.net (216.58.211.227): icmp_seq=2 ttl=118 time=11.7 ms
^C
--- google.es ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 11.323/11.531/11.740/0.234 ms
{% endhighlight %}

La máquina ha resuelto y alcanzado a **google.es** sin problemas.

## Tarea 7: Entrega el nuevo fichero Vagrantfile que define el escenario.

En este caso, vamos a modificar el escenario (apagando previamente las máquinas con `vagrant halt`) para añadir las siguientes características:

* **servidor:** Una nueva tarjeta de red conectada a una nueva red interna.
* **nodo_lan2:** Un nuevo cliente que cuenta con una tarjeta de red conectada a esta última la red interna.

Hay que configurar el servidor DHCP para que de servicio a los ordenadores de la nueva red interna, teniendo en cuenta que el tiempo de concesión sea de **24 horas** y utilice un direccionamiento **192.168.200.0/24**. En este caso, el fichero de configuración **Vagrantfile** tiene la siguiente forma:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define :servidor do |servidor|
    servidor.vm.box = "debian/buster64"
    servidor.vm.hostname = "servidor"
    servidor.vm.network :public_network,:bridge=>"enp3s0",
      use_dhcp_assigned_default_route: true
    servidor.vm.network :private_network, ip: "192.168.100.1",
      virtualbox__intnet: "lan1"
    servidor.vm.network :private_network, ip: "192.168.200.1", #Creamos una interfaz de red privada, que tendrá un direccionamiento estático, ya que se trata del servidor y nos conviene que la IP sea siempre la misma, en este caso, la primera dirección válida dentro de dicha red.
      virtualbox__intnet: "lan2" #Dado que VirtualBox crea por defecto las interfaces en modo host-only, tenemos que indicar que utilice una red interna, en este caso, una de nombre "lan2".
  end
  config.vm.define :nodo_lan1 do |nodo_lan1|
    nodo_lan1.vm.box = "debian/buster64"
    nodo_lan1.vm.hostname = "nodolan1"
    nodo_lan1.vm.network :private_network,
      auto_config: false,
      virtualbox__intnet: "lan1"
    config.vm.provision "shell",
      inline: "sudo ip r del default via 10.0.2.2",
      run: "always"
  end
  config.vm.define :nodo_lan2 do |nodo_lan2| #Definimos la tercera máquina, en este caso, un nuevo cliente.
    nodo_lan2.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    nodo_lan2.vm.hostname = "nodolan2" #Establecemos el nombre (hostname) de la máquina. En este caso, "nodolan2", tal y como se ha pedido.
    nodo_lan2.vm.network :private_network, #Creamos una interfaz de red conectada a una red privada, en este caso sin especificar un direccionamiento estático ni DHCP.
      auto_config: false, #En este caso, la configuración la vamos a llevar a cabo manualmente en el fichero /etc/network/interfaces, por el motivo anteriormente mencionado.
      virtualbox__intnet: "lan2" #Al igual que la tercera interfaz de red del servidor, ésta también la conectaremos a la red interna de nombre "lan2".
    config.vm.provision "shell", #Especificamos que vamos a ejecutar un script durante el arranque de la máquina.
      inline: "sudo ip r del default via 10.0.2.2", #Se especifica el comando a ejecutar, en este caso, eliminamos la puerta de enlace por defecto que obtiene la máquina cliente a través del DHCP por la interfaz "eth0".
      run: "always" #Indicamos que el script se ejecute cada vez que se arranca la máquina, ya que por defecto lo hace la primera vez.
  end
end
{% endhighlight %}

## Tarea 8: Explica las modificaciones que has hecho en los distintos ficheros de configuración. Entrega las comprobaciones necesarias de que los dos ámbitos están funcionando.

De primeras vamos a iniciar únicamente la máquina **servidor**, para configurar lo necesario para servir a la segunda máquina cliente y tras ello, arrancar ambos clientes a la vez. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant up servidor
{% endhighlight %}

Tras ello nos conectaremos mediante SSH haciendo uso de la instrucción `vagrant ssh servidor` y verificaremos que todas las interfaces se encuentran correctamente creadas, ejecutando para ello el comando:

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
       valid_lft 86338sec preferred_lft 86338sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:48:cb:42 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.140/24 brd 192.168.1.255 scope global dynamic eth1
       valid_lft 86355sec preferred_lft 86355sec
    inet6 fe80::a00:27ff:fe48:cb42/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:5f:43:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5f:43cf/64 scope link 
       valid_lft forever preferred_lft forever
5: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8a:ce:98 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.1/24 brd 192.168.200.255 scope global eth3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe8a:ce98/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **cuatro** interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada en modo puente a nuestra interfaz física **enp3s0**, con dirección IP obtenida por DHCP **192.168.1.140**.
* **eth2:** Creada por nosotros y conectada a una red interna **lan1**, con dirección IP estática **192.168.100.1**.
* **eth3:** Creada por nosotros y conectada a una red interna **lan2**, con dirección IP estática **192.168.200.1**.

Para verificar también que la puerta de enlace predeterminada se ha configurado correctamente y se ha añadido la regla de encaminamiento, ejecutaremos el comando:

{% highlight shell %}
vagrant@servidor:~$ ip r
default via 192.168.1.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.140 
192.168.100.0/24 dev eth2 proto kernel scope link src 192.168.100.1 
192.168.200.0/24 dev eth3 proto kernel scope link src 192.168.200.1
{% endhighlight %}

El siguiente paso será indicar la nueva interfaz de red a través de la cuál va a servir el servidor DHCP, en este caso **eth3**, pues es la que se encuentra dentro de la red interna **192.168.200.0/24**. Para ello, vamos a modificar el fichero **/etc/default/isc-dhcp-server**, ejecutando el comando:

{% highlight shell %}
root@servidor:~# nano /etc/default/isc-dhcp-server
{% endhighlight %}

Modificaremos la línea `INTERFACESv4="eth2"` por `INTERFACESv4="eth2 eth3"` y guardaremos los cambios, pero la configuración todavía no ha terminado, pues queda lo más importante, el fichero principal de configuración **/etc/dhcp/dhcpd.conf**. Lo editaremos haciendo uso del comando:

{% highlight shell %}
root@servidor:~# nano /etc/dhcp/dhcpd.conf
{% endhighlight %}

Para esta tarea se ha pedido que el tiempo de concesión de **24 horas** y que la red sea la **192.168.200.0/24**, por lo que debemos añadir lo siguiente al fichero:

{% highlight shell %}
subnet 192.168.200.0 netmask 255.255.255.0 {
  range 192.168.200.10 192.168.200.200;
  default-lease-time 86400;
  max-lease-time 86400;
  option routers 192.168.200.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
}
{% endhighlight %}

La red que hemos especificado es la **192.168.200.0**, con una máscara de red **255.255.255.0** (/24). El rango de direcciones dentro de dicha que se van a asignar va desde la **192.168.200.10** hasta la **192.168.200.200**, aunque se podría haber utilizado uno totalmente distinto. El tiempo de concesión por defecto y máximo lo hemos establecido en **86400** segundos (24 horas), vuelvo a hacer hincapié en que no ha de confundirse con T1 y T3, respectivamente. La puerta de enlace predeterminada es la IP de la interfaz de red del servidor perteneciente a la red interna, es decir, la **192.168.200.1** y por último, usaremos el DNS de Google (**8.8.8.8**) y el **1.1.1.1**.

Siempre que hagamos modificaciones en un fichero de configuración de un servicio, debemos reiniciarlo para que así cargue la nueva configuración. Para ello, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# systemctl restart isc-dhcp-server
{% endhighlight %}

Ya estamos preparados para arrancar ahora sí las máquinas cliente **nodo_lan1** y **nodo_lan2**. Para ello, abriremos dos nuevas terminales y así tendremos cada una de las máquinas en una terminal para no tener que estar saliendo y entrando todo el rato. Para levantar dichas máquinas, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/vagrant/dhcp$ vagrant up nodo_lan1 nodo_lan2
{% endhighlight %}

Una vez que haya finalizado el proceso, nos podremos conectar a las máquinas ejecutando las instrucciones `vagrant ssh nodo_lan1` y `vagrant ssh nodo_lan2`, respectivamente.

Al igual que hicimos con la primera máquina cliente, es momento de configurar el fichero **/etc/network/interfaces** para establecer la configuración de dicha interfaz de red. Para ello, ejecutamos el comando:

{% highlight shell %}
vagrant@nodolan2:~$ sudo nano /etc/network/interfaces
{% endhighlight %}

Añadiremos las siguientes líneas:

{% highlight shell %}
auto eth1
iface eth1 inet dhcp
{% endhighlight %}

De esta forma, la interfaz se levantará automáticamente durante el arranque (**auto**) y obtendrá la configuración de red por DHCP (**inet dhcp**). Para no tener que reiniciar la máquina, podemos hacer la petición DHCP de forma manual para la interfaz **eth1**, ejecutando para ello el comando:

{% highlight shell %}
vagrant@nodolan2:~$ sudo dhclient eth1
{% endhighlight %}

En un principio, ambas máquinas se encuentran ya correctamente configuradas, así que para verificarlo, ejecutaremos los comandos `ip a` e `ip r` en ambas.

{% highlight shell %}
vagrant@nodolan1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86033sec preferred_lft 86033sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:25:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global dynamic eth1
       valid_lft 42835sec preferred_lft 42835sec
    inet6 fe80::a00:27ff:fe2c:2572/64 scope link 
       valid_lft forever preferred_lft forever

vagrant@nodolan1:~$ ip r
default via 192.168.100.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.100.0/24 dev eth1 proto kernel scope link src 192.168.100.100
{% endhighlight %}

{% highlight shell %}
vagrant@nodolan2:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86200sec preferred_lft 86200sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ff:cc:b1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.10/24 brd 192.168.200.255 scope global dynamic eth1
       valid_lft 86399sec preferred_lft 86399sec
    inet6 fe80::a00:27ff:feff:ccb1/64 scope link 
       valid_lft forever preferred_lft forever

vagrant@nodolan2:~$ ip r
default via 192.168.200.1 dev eth1 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
192.168.200.0/24 dev eth1 proto kernel scope link src 192.168.200.10
{% endhighlight %}

Como se puede apreciar, en ambas máquinas se ha configurado correctamente la puerta de enlace predeterminada y la regla de encaminamiento, así como las siguientes interfaces:

* **eth0:** Generada automáticamente por VirtualBox y conectada a un router virtual NAT, con dirección IP **10.0.2.15**.
* **eth1:** Creada por nosotros y conectada a una red interna **lan1** en caso de la máquina **nodo_lan1** con IP **192.168.100.100** gracias a la reserva, y conectada a una red interna **lan2** en caso de la máquina **nodo_lan2** con IP **192.168.200.10** gracias a la asignación por DHCP.

## Tarea 9: Realiza las modificaciones necesarias para que los cliente de la segunda red local tengan acceso a Internet. Entrega las comprobaciones necesarias.

Para verificar que tras el reinicio se ha restaurado correctamente la configuración de _nftables_ y por tanto el cliente **nodo_lan1** sigue teniendo conectividad al exterior, vamos a hacer `ping` desde ambos clientes a **8.8.8.8**:

{% highlight shell %}
vagrant@nodolan1:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=41.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=41.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=113 time=57.7 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 41.550/46.952/57.675/7.584 ms
{% endhighlight %}

{% highlight shell %}
vagrant@nodolan2:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
6 packets transmitted, 0 received, 100% packet loss, time 202ms
{% endhighlight %}

Actualmente, la máquina cliente **nodo_lan1** tiene conectividad al exterior, pero la máquina cliente **nodo_lan2** no, ya que no hemos configurado la correspondiente regla para la cadena **POSTROUTING**. Para añadir dicha regla, ejecutamos el comando:

{% highlight shell %}
root@servidor:~# nft add rule ip nat postrouting oifname "eth1" ip saddr 192.168.200.0/24 counter masquerade
{% endhighlight %}

En este caso, hemos especificado que se trata de una regla para la cadena **postrouting** (pues debe aplicarse justo antes de salir de la máquina), además, la interfaz (oifname) que debemos introducir será aquella por la que van a salir los paquetes, es decir, la que está conectada a Internet, en este caso, **eth1**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que provengan de la red (ip saddr) **192.168.200.0/24**. Además, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a enmascararlos, pues la IP "pública" del router es dinámica (**masquerade**).

Por último, vamos a volver a verificar que dicha regla se ha añadido correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@servidor:~# nft list ruleset
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
		oifname "eth1" ip saddr 192.168.100.0/24 counter packets 8 bytes 573 masquerade
		oifname "eth1" ip saddr 192.168.200.0/24 counter packets 0 bytes 0 masquerade
	}
}
{% endhighlight %}

Efectivamente, así ha sido. Una vez más, vamos a guardar dicha configuración para que perdure tras los reinicios:

{% highlight shell %}
root@servidor:~# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Nos falta hacer la última prueba, para verificar que el último cliente es capaz de salir a Internet. Para ello, haremos `ping` a **google.es**, obligándolo así a resolver DNS, de manera que también podremos comprobar si funciona.

{% highlight shell %}
vagrant@nodolan2:~$ ping google.es
PING google.es (216.58.211.227) 56(84) bytes of data.
64 bytes from mad01s24-in-f227.1e100.net (216.58.211.227): icmp_seq=1 ttl=118 time=10.6 ms
64 bytes from mad01s24-in-f227.1e100.net (216.58.211.227): icmp_seq=2 ttl=118 time=10.9 ms
64 bytes from mad01s24-in-f227.1e100.net (216.58.211.227): icmp_seq=3 ttl=118 time=11.5 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 169ms
rtt min/avg/max/mdev = 10.632/11.003/11.495/0.382 ms
{% endhighlight %}

Efectivamente, ambos clientes tienen ya conectividad con el exterior y son capaces de resolver por DNS.