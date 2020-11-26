---
layout: post
title:  "Configuración de cliente OpenVPN con certificados X.509"
banner: "/assets/images/banners/vpn.jpg"
date:   2020-11-02 19:14:00 +0200
categories: hlc
---
El objetivo de esta tarea es el de establecer una conexión de acceso remoto entre nuestro equipo de casa y _sputnik_, una máquina virtual contratada en OVH, a través de la cuál se va a crear una doble conexión **VPN** para poder acceder desde el exterior a las máquinas del departamento de informática del IES Gonzalo Nazareno, haciendo uso del puerto **1194/TCP**.

Para llevar a cabo dicha tarea, necesitaremos disponer de un certificado **X.509** para nuestro equipo, que haya sido previamente firmado por la autoridad certificadora (CA) del IES Gonzalo Nazareno, por lo tanto, lo primero que tendremos que hacer será crear una solicitud de firma de certificado (CSR o _Certificate Signing Request_). En este caso, vamos a hacerlo con **openssl**, pero se podría hacer con otras múltiples opciones de software:

El primer paso será generar una clave privada RSA de 4096 bits, que será almacenada en **/etc/ssl/private/** (con permisos de administrador, ejecutando el comando `su -`):

{% highlight shell %}
openssl genrsa 4096 > /etc/ssl/private/[nombremaquina].key
{% endhighlight %}

En mi caso, mi máquina tiene nombre **debian**, por lo que el nombre del fichero final será **debian.key**:

{% highlight shell %}
root@debian:~# openssl genrsa 4096 > /etc/ssl/private/debian.key
Generating RSA private key, 4096 bit long modulus (2 primes)
............++++
......++++
e is 65537 (0x010001)
{% endhighlight %}

Una vez generado, cambiaremos los permisos de la clave privada que acabamos de generar a **400**, de manera que únicamente el propietario pueda leer el contenido, pues se trata de una clave privada. Este paso no es obligatorio pero sí recomendable por seguridad. Para ello, haremos uso de `chmod`:

{% highlight shell %}
root@debian:~# chmod 400 /etc/ssl/private/debian.key
{% endhighlight %}

Para verificar que los permisos han sido correctamente modificados, listaremos el contenido de dicho directorio haciendo uso de `ls -l`:

{% highlight shell %}
root@debian:~# ls -l /etc/ssl/private/
total 4
-r-------- 1 root root 3243 nov  2 10:36 debian.key
{% endhighlight %}

Efectivamente, los permisos han sido correctamente modificados a sólo lectura por parte del propietario.

Tras ello, crearemos un fichero **.csr** de solicitud de firma de certificado para que sea firmado por la autoridad certificadora (CA) del IES Gonzalo Nazareno, que estará asociado a la clave privada que acabamos de generar. Dicho fichero deberá ser almacenado en una ruta accesible por el usuario común, ya que posteriormente habrá que subirlo para su correspondiente proceso de firmado:

{% highlight shell %}
openssl req -new -key /etc/ssl/private/[nombremaquina].key -out [ruta]/[nombremaquina].csr
{% endhighlight %}

En mi caso, mi máquina tiene el nombre **debian**, por lo que el nombre del fichero final será **debian.csr**, que se encontrará almacenado en una ruta accesible por el usuario común, como por ejemplo, el directorio personal (**/home/alvaro/**):

{% highlight shell %}
root@debian:~# openssl req -new -key /etc/ssl/private/debian.key -out /home/alvaro/debian.csr
{% endhighlight %}

Durante la ejecución, nos pedirá una serie de valores para identificar al certificado, que tendremos que rellenar de la siguiente manera:

{% highlight shell %}
Country Name (2 letter code) [AU]:ES
State or Province Name (full name) [Some-State]:Sevilla
Locality Name (eg, city) []:Dos Hermanas
Organization Name (eg, company) [Internet Widgits Pty Ltd]:IES Gonzalo Nazareno
Organizational Unit Name (eg, section) []:Informatica
Common Name (e.g. server FQDN or YOUR name) []:debian-alvaro.vaca
Email Address []:avacaferreras@gmail.com
{% endhighlight %}

Todos los valores son genéricos excepto los dos últimos:

* **Common Name (CN)**: Indica el nombre del equipo, que ha de ser único. Para evitar problemas, vamos a poner el nombre de la máquina (**debian**), seguido del nombre de usuario del IES Gonzalo Nazareno (**alvaro.vaca**) mediante un guión, quedando finalmente **debian-alvaro.vaca**.
* **Email Address**: La dirección de correo electrónico personal. En este caso, **avacaferreras@gmail.com**.

Tras ello, se pedirán una serie de valores cuya introducción es opcional. En mi caso, no los rellené:

{% highlight shell %}
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endhighlight %}

Para verificar que el fichero de solicitud de firma ha sido correctamente generado, listaremos el contenido del directorio donde ha sido almacenado y usaremos un filtro de nombre:

{% highlight shell %}
root@debian:~# ls /home/alvaro/ | egrep 'debian'
debian.csr
{% endhighlight %}

Como se puede apreciar, el fichero se encuentra generado en dicho directorio, con nombre **debian.csr**.

El siguiente paso será subir el fichero de solicitud de firma mediante la aplicación [Gestiona](https://dit.gonzalonazareno.org/gestiona/cert/), para que sea firmado y así obtengamos el certificado que necesitamos, con extensión **.crt** o **.pem**. El fichero debe subirse en el apartado "**Certificados de Equipos**":

![cert1](https://i.ibb.co/PZd3P8t/Captura-de-pantalla-de-2020-11-02-10-31-53.png "Subida del fichero")

Una vez subido, tendremos que esperar unas horas (pues la firma del mismo se lleva a cabo manualmente) y tras ello, nos aparecerá de la siguiente forma:

![cert2](https://i.ibb.co/KVG6Pxw/Captura-de-pantalla-de-2020-11-02-10-31-53-copia.png "Certificado firmado")

Como se puede apreciar, nos aparece para **descargar** o **revocar** el certificado firmado, con extensión **.crt**. En este caso, lo descargaremos.

Tras ello, tendremos que descargar además el certificado de la propia autoridad certificadora del IES Gonzalo Nazareno dentro de **/etc/ssl/certs/**, de manera que se pueda llevar a cabo la comprobación de la firma y así poder verificar que quién nos lo ha firmado es quien realmente dice ser. Para ello, nos moveremos dentro de dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# cd /etc/ssl/certs/
{% endhighlight %}

Una vez dentro del mismo, haremos uso de la utilidad `wget` para descargar el fichero pasándole una [URL de descarga](https://dit.gonzalonazareno.org/gestiona/info/documentacion/doc/gonzalonazareno.crt):

{% highlight shell %}
root@debian:/etc/ssl/certs# wget https://dit.gonzalonazareno.org/gestiona/info/documentacion/doc/gonzalonazareno.crt
--2020-11-02 10:38:26--  https://dit.gonzalonazareno.org/gestiona/info/documentacion/doc/gonzalonazareno.crt
Resolviendo dit.gonzalonazareno.org (dit.gonzalonazareno.org)... 80.59.1.152
Conectando con dit.gonzalonazareno.org (dit.gonzalonazareno.org)[80.59.1.152]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 3634 (3,5K) [application/x-x509-ca-cert]
Grabando a: “gonzalonazareno.crt”

gonzalonazareno.crt                100%[==========================>]   3,55K  --.-KB/s    en 0s      

2020-11-02 10:38:26 (58,4 MB/s) - “gonzalonazareno.crt” guardado [3634/3634]
{% endhighlight %}

Para verificar que el certificado de la CA Gonzalo Nazareno ha sido correctamente descargado, vamos a listar el contenido del directorio actual, estableciendo de nuevo, un filtro por nombre:

{% highlight shell %}
root@debian:/etc/ssl/certs# ls | egrep 'gonzalonazareno'
gonzalonazareno.crt
{% endhighlight %}

Como se puede apreciar, el certificado se encuentra descargado en el directorio actual, con nombre **gonzalonazareno.crt**.

A continuación, como es lógico, necesitaremos un cliente que nos permita llevar a cabo dicha conexión de acceso remoto, así que recurriremos a **OpenVPN**. Dicho paquete no viene instalado por defecto, así que tendremos que hacerlo manualmente, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/etc/ssl/certs# apt install openvpn
{% endhighlight %}

Como consecuencia de la instalación, se habrá generado un nuevo directorio dentro de **/etc/**, de nombre **openvpn/**, en el que tendremos que alojar la configuración del mismo. Para ello, nos moveremos a dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/etc/ssl/certs# cd /etc/openvpn/
{% endhighlight %}

Lo primero que haremos será mover nuestro certificado firmado desde el directorio **Descargas/** al directorio actual (realmente, se podría dejar allí, pero siempre es mejor mantener una organización), haciendo uso del comando `mv`:

{% highlight shell %}
root@debian:/etc/openvpn# mv /home/alvaro/Descargas/debian.crt ./
{% endhighlight %}

Tener el certificado firmado no es el único requisito para que **OpenVPN** funcione. Además, necesitaremos un fichero de configuración con extensión **.conf** (exigencia de _OpenVPN_) en el que se encuentren indicadas todas las directivas necesarias para su funcionamiento. En mi caso, he decidido que el nombre de dicho fichero sea **vpn.conf**, aunque se podría haber usado cualquier otro. Para crear y modificar el contenido de dicho fichero, ejecutaremos el comando:

{% highlight shell %}
root@debian:/etc/openvpn# nano vpn.conf
{% endhighlight %}

El contenido del mismo será:

{% highlight shell %}
dev tun
remote sputnik.gonzalonazareno.org
ifconfig 172.23.0.0 255.255.255.0
pull
proto tcp-client
tls-client
remote-cert-tls server
ca [rutacertificadoCA]
cert [rutacertificadoFirmado]
key [rutaclaveprivada]
comp-lzo
keepalive 10 60
log /var/log/openvpn-sputnik.log
verb 1
{% endhighlight %}

En este caso, los parámetros que debo usar son:

* **rutacertificadoCA**: _/etc/ssl/certs/gonzalonazareno.crt_
* **rutacertificadoFirmado**: _/etc/openvpn/debian.crt_
* **rutaclaveprivada**: _/etc/ssl/private/debian.key_

El resultado final del fichero sería el siguiente:

{% highlight shell %}
dev tun
remote sputnik.gonzalonazareno.org
ifconfig 172.23.0.0 255.255.255.0
pull
proto tcp-client
tls-client
remote-cert-tls server
ca /etc/ssl/certs/gonzalonazareno.crt
cert /etc/openvpn/debian.crt
key /etc/ssl/private/debian.key
comp-lzo
keepalive 10 60
log /var/log/openvpn-sputnik.log
verb 1
{% endhighlight %}

Tras ello, guardaremos los cambios ejecutando la combinación de teclas **CTRL + X**, seguido de **Y** y de **ENTER**.

Para verificar que el certificado firmado fue movido de directorio con éxito y que el fichero de configuración **vpn.conf** ha sido correctamente generado, listaremos el contenido del directorio actual, haciendo uso del comando `ls -l`:

{% highlight shell %}
root@debian:/etc/openvpn# ls -l
total 28
drwxr-xr-x 2 root   root    4096 feb 20  2019 client
-rw-r--r-- 1 alvaro alvaro 10075 nov  2 10:40 debian.crt
drwxr-xr-x 2 root   root    4096 feb 20  2019 server
-rwxr-xr-x 1 root   root    1468 feb 20  2019 update-resolv-conf
-rw-r--r-- 1 root   root     297 nov  2 10:41 vpn.conf
{% endhighlight %}

Como se puede apreciar, el certificado firmado (**debian.crt**) se encuentra correctamente alojado en el directorio actual, al igual que el fichero de configuración de _OpenVPN_ (**vpn.conf**).

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que reiniciarlo para que haga uso de la nueva configuración existente. Para ello, haremos uso del comando:

{% highlight shell %}
root@debian:/etc/openvpn# systemctl restart openvpn.service
{% endhighlight %}

Llegados a este punto se pueden plantear dos posibilidades distintas:

* **Que queramos usar el túnel VPN desde el arranque de la máquina:**

Como consecuencia del reinicio del servicio, el servicio se habrá iniciado (_active_) y por defecto, viene habilitado para que se inicie durante el arranque de la propia máquina (_enabled_).

En _OpenVPN_ existe un servicio principal o maestro (_openvpn.service_) que es el que se encuentra actualmente activo y habilitado para su inicio durante el arranque y varios "subservicios", uno por cada uno de los túneles VPN que gestionemos (_openvpn@[nombre].service_).

Por defecto, los subservicios vienen desactivados (_inactive_), de manera que en cuanto lo arranquemos por primera vez (_start_), se levantará y tendremos conectividad en todo momento, incluso tras un reinicio, de manera automatizada, ya que el servicio principal o maestro se encuentra _enabled_. Para arrancar el subservicio, tendremos que introducir el nombre del fichero de configuración como **nombre** (sin la extensión), quedando de la siguiente manera:

{% highlight shell %}
root@debian:/etc/openvpn# systemctl start openvpn@vpn.service
{% endhighlight %}

El túnel VPN ya se habrá levantado, de manera que si listamos nuestras reglas de enrutamiento veremos que han sido añadidas:

{% highlight shell %}
root@debian:/etc/openvpn# ip r
default via 192.168.1.1 dev br0 
169.254.0.0/16 dev br0 scope link metric 1000 
172.22.0.0/16 via 172.23.0.57 dev tun0 
172.23.0.1 via 172.23.0.57 dev tun0 
172.23.0.57 dev tun0 proto kernel scope link src 172.23.0.58 
192.168.1.0/24 dev br0 proto kernel scope link src 192.168.1.136 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown
{% endhighlight %}

Efectivamente, las correspondientes reglas de encaminamiento han sido añadidas, de manera que aunque se efectúe un reinicio, volverán a añadirse automáticamente, como consecuencia de tener el servicio principal (_openvpn.service_) habilitado durante el arranque, y haber arrancado manualmente por primera vez dicho túnel VPN (_openvpn@vpn.service_).

* **Que queramos levantar el túnel VPN manualmente (es mi caso):**

Tal y como he mencionado en la posibilidad anterior, como consecuencia del reinicio del servicio maestro, el servicio se habrá iniciado (_active_) y por defecto, viene habilitado para que se inicie durante el arranque de la propia máquina (_enabled_), pero esto no es lo que buscamos, pues nosotros queremos que únicamente esté activo cuando nos sea necesario, y que por tanto, tampoco se inicie durante el arranque de la máquina.

Es por ello, que vamos a apagar el servicio (_stop_) y lo deshabilitaremos para que no inicie durante el arranque (_disable_):

{% highlight shell %}
root@debian:/etc/openvpn# systemctl stop openvpn.service 
{% endhighlight %}

{% highlight shell %}
root@debian:/etc/openvpn# systemctl disable openvpn.service
Synchronizing state of openvpn.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable openvpn
{% endhighlight %}

El servicio maestro se encuentra apagado y deshabilitado durante el arranque, de manera que cuando queramos hacer uso del túnel VPN, iniciaremos el subservicio (_start_) y cuando no queramos seguir usándolo, lo apagaremos (_stop_), de la siguiente manera:

{% highlight shell %}
root@debian:/etc/openvpn# systemctl start openvpn@vpn.service
{% endhighlight %}

El túnel VPN ya se habrá levantado, de manera que si listamos nuestras reglas de enrutamiento veremos que han sido añadidas:

{% highlight shell %}
root@debian:/etc/openvpn# ip r
default via 192.168.1.1 dev br0 
169.254.0.0/16 dev br0 scope link metric 1000 
172.22.0.0/16 via 172.23.0.57 dev tun0 
172.23.0.1 via 172.23.0.57 dev tun0 
172.23.0.57 dev tun0 proto kernel scope link src 172.23.0.58 
192.168.1.0/24 dev br0 proto kernel scope link src 192.168.1.136 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown
{% endhighlight %}

Como era de esperar, las reglas correspondientes se han añadido, de manera que ya tenemos conectividad con las máquinas del departamento de informática del IES Gonzalo Nazareno. De igual forma, vamos a verificar el contenido del fichero de log ubicado en **/var/log/openvpn-sputnik.log**, para así asegurarnos de que se ha establecido la conexión, haciendo uso de `cat`:

{% highlight shell %}
root@debian:/etc/openvpn# cat /var/log/openvpn-sputnik.log 
Mon Nov  2 13:31:47 2020 OpenVPN 2.4.7 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Feb 20 2019
Mon Nov  2 13:31:47 2020 library versions: OpenSSL 1.1.1d  10 Sep 2019, LZO 2.10
Mon Nov  2 13:31:47 2020 WARNING: using --pull/--client and --ifconfig together is probably not what you want
Mon Nov  2 13:31:47 2020 TCP/UDP: Preserving recently used remote address: [AF_INET]92.222.86.77:1194
Mon Nov  2 13:31:47 2020 Attempting to establish TCP connection with [AF_INET]92.222.86.77:1194 [nonblock]
Mon Nov  2 13:31:48 2020 TCP connection established with [AF_INET]92.222.86.77:1194
Mon Nov  2 13:31:48 2020 TCP_CLIENT link local: (not bound)
Mon Nov  2 13:31:48 2020 TCP_CLIENT link remote: [AF_INET]92.222.86.77:1194
Mon Nov  2 13:31:49 2020 [sputnik.gonzalonazareno.org] Peer Connection Initiated with [AF_INET]92.222.86.77:1194
Mon Nov  2 13:31:50 2020 TUN/TAP device tun0 opened
Mon Nov  2 13:31:50 2020 /sbin/ip link set dev tun0 up mtu 1500
Mon Nov  2 13:31:50 2020 /sbin/ip addr add dev tun0 local 172.23.0.58 peer 172.23.0.57
Mon Nov  2 13:31:50 2020 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Mon Nov  2 13:31:50 2020 Initialization Sequence Completed
{% endhighlight %}

Como se puede apreciar en la salida del fichero, se ha establecido correctamente una conexión TCP con _sputnik_ (**92.222.86.77**) en el puerto **1194**. Además, el túnel VPN se ha levantado haciendo uso de la interfaz **tun0**, así que vamos a ver la información correspondiente a dicha interfaz, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/etc/openvpn# ip a show dev tun0
9: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none 
    inet 172.23.0.58 peer 172.23.0.57/32 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::df50:a561:396e:852b/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, la interfaz anteriormente mencionada se encuentra correctamente configurada, pero de nada nos sirve si todavía no hemos verificado si realmente funciona.

Antes de llevar a cabo las pruebas, voy a modificar mi fichero **/etc/hosts** para llevar a cabo la resolución estática de nombres de las máquinas más utilizadas en el centro (pues no existe ningún servidor DNS en Internet que almacene las direcciones IP correspondientes, al ser direcciones privadas), quedando de la siguiente manera:

{% highlight shell %}
root@debian:/etc/openvpn# nano /etc/hosts
{% endhighlight %}

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
172.22.222.1    jupiter
172.22.0.1      macaco
{% endhighlight %}

Por lo tanto, si ahora mismo hiciésemos `ping` a la dirección **172.22.0.1** (o lo que es lo mismo, a **macaco**), éste debería responder:

{% highlight shell %}
root@debian:/etc/openvpn# ping macaco
PING macaco (172.22.0.1) 56(84) bytes of data.
64 bytes from macaco (172.22.0.1): icmp_seq=1 ttl=63 time=82.9 ms
64 bytes from macaco (172.22.0.1): icmp_seq=2 ttl=63 time=83.9 ms
64 bytes from macaco (172.22.0.1): icmp_seq=3 ttl=63 time=83.6 ms
^C
--- macaco ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 82.901/83.471/83.909/0.421 ms
{% endhighlight %}

Efectivamente, la resolución estática de nombres se ha llevado a cabo correctamente. Además, hemos recibido respuesta a los paquetes ICMP enviados, por lo que podemos concluir que el túnel VPN funciona correctamente.

Supongamos que ya no necesitamos seguir haciendo uso de dicho túnel, así que para ello, sería tan sencillo como parar el subservicio correspondiente, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/etc/openvpn# systemctl stop openvpn@vpn.service
{% endhighlight %}

Si volvemos a listar de nuevo las reglas de encaminamiento existentes en nuestra máquina, veremos que han sido eliminadas:

{% highlight shell %}
root@debian:/etc/openvpn# ip r
default via 192.168.1.1 dev br0 
169.254.0.0/16 dev br0 scope link metric 1000 
192.168.1.0/24 dev br0 proto kernel scope link src 192.168.1.136 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown
{% endhighlight %}

Como era de esperar, las reglas correspondientes se han eliminado y por tanto, podemos concluir que el túnel VPN se ha apagado y que ya no tenemos conectividad con las máquinas del departamento de informática.