---
layout: post
title:  "Instalación de un servidor DNS"
banner: "/assets/images/banners/dns.jpg"
date:   2020-12-11 19:04:00 +0200
categories: servicios
---
Cuando se quiere acceder a una página web en Internet se necesita conocer la **dirección IP** del servidor donde está almacenada, pero, por regla general, el usuario solo conoce el nombre del dominio. La razón no es otra que la dificultad de recordar las series numéricas del tipo **51.210.109.246**. Es por este motivo por el que surgen los llamados **dominios**, permitiendo traducir un nombre más sencillo de recordar en direcciones IP.

La función anteriormente mencionada es tarea de los servidores DNS (_Domain Name Server_). Hablando un poco sobre la historia del pasado, antiguamente existía una resolución estática, es decir, todos los nombres de dominio se encontraban contenidos en un fichero, por lo que todos los ordenadores tenían que bajarse constantemente el nuevo fichero mediante el protocolo FTP cada vez que ocurriese un cambio.

Este método dejó de ser viable con el crecimiento de Internet, además de generar conflictos, pues nadie controlaba que los nombres no se repitiesen, por lo que podían llegar a coexistir varias máquinas con el mismo nombre, problema que se solucionó con la aparición del protocolo DNS, que hace uso de un sistema de base de datos distribuida.

Se trata de un sistema jerárquico, que tiene un inicio o raíz (_root_), y a partir de ahí existen varios niveles de nombres:

* **Dominio raíz (.)**: Es gestionado por la ICANN, y aunque las personas nunca lo ponemos, los navegadores hacen uso del mismo.
* **Dominios de primer nivel (TLD)**: Se trata de un conjunto de dominios ya preestablecidos, comúnmente conocidos, como pueden ser **com**, **es**, **org**...
* **Dominios de segundo nivel**: Son aquellos nombres que pueden ser alquilados por instituciones, personas, empresas... como puede ser **alvarovf.com**, **gonzalonazareno.org**...

A partir de este nivel en el árbol, se pueden empezar a nombrar las máquinas o servicios dentro de nuestra **zona DNS**, o bien, crear subdominios que se encuentren gestionados por nosotros o delegar su gestión a otra institución, dentro del cuál se podrán nombrar también máquinas y servicios (lo veremos con más detalle a continuación).

Para aclarar conceptos, vamos a hacer un pequeño ejemplo con el dominio **es.wikipedia.org.**:

* **.**: Es el dominio raíz, que como anteriormente he mencionado, las personas nunca lo ponemos, pero los navegadores hacen uso del mismo.
* **org**: Es el dominio de primer nivel o _TLD_ y se encuentra ya predefinido.
* **wikipedia**: Es el dominio de segundo nivel y ha sido alquilado por la correspondiente institución.
* **es**: Es el nombre de una máquina o servicio dentro de la zona DNS de **wikipedia.org.**.

Como he mencionado, la estructura podría continuar en caso de crear subdominios, pudiendo nombrar máquinas y servicios dentro de los mismos.

Anteriormente hemos hecho uso del concepto **zona DNS**. Bien, una zona DNS es el conjunto de nombres y servicios definidos dentro de un dominio, algo así como una base de datos. Por ejemplo, la zona **org.** está formada por todos los dominios de segundo nivel definidos dentro de dicho dominio: **gonzalonazareno**, **google**, **alvarovf**... A su vez, la zona **gonzalonazareno.org.** está formada por todos los nombres o servicios dentro de dicho dominio: **macaco**, **papion**, **babuino**... y así sucesivamente. Existen dos tipos de zonas:

* **Zona de resolución directa**: Nos permiten resolver un nombre a una dirección IP. Son las más conocidas y utilizadas.
* **Zona de resolución inversa**: Nos permiten resolver una dirección IP a un nombre (al contrario que el caso anterior). Se utiliza en determinadas ocasiones, dado que hay sistemas de seguridad que necesitan la resolución inversa para verificar la procedencia de un paquete, por ejemplo. Están pensadas para direccionamiento privado.

Cada zona DNS se encuentra guardada en un fichero que se encuentra dentro de un servidor DNS, es decir, hay un (o varios) servidor DNS que conoce una zona determinada, que se conoce como **servidor DNS con autoridad sobre la zona**. Por ejemplo, **wikipedia.org**. tendrá varios servidores DNS que conocerán todos los nombres y sus correspondientes IPs en esa zona (**es**, **nds**...), que en la mayoría de ocasiones suelen ser los servidores DNS de la empresa con la que se ha alquilado el dominio. Existirán a su vez otros servidores DNS que conocerán la zona **org.**, es decir, conocerán a los servidores que conocen la zona de **wikipedia.org.**, y así jerárquicamete.

Dicha estructura jerárquica es necesaria ya que así podemos conseguir que la resolución de nombres sea una tarea "descentralizada". Por ejemplo, si le preguntas a los servidores con autoridad sobre la zona **org.** por la dirección de **es.wikipedia.org.**, no te sabrán decir la dirección IP asociada, pero sí la dirección de aquellos servidores con autoridad sobre la zona **wikipedia.org.**, de manera que volveremos a hacer la misma pregunta a dichos servidores, que nos devolverán ahora sí, la dirección IP asociada.

De esta manera, existen actualmente un total de 13 servidores que conocen la zona del dominio raíz (**.**), llamados **_root servers_**, que son capaces de darte las direcciones de los servidores con autoridad sobre la zona de primer nivel. Actualmente, se encuentran replicados en una gran cantidad de sitios, de manera que gracias al tipo de peticiones _anycast_, le preguntaremos al más cercano geográficamente a nosotros.

El mayor inconveniente de este sistema es el hecho de que sea jerárquico, de manera que si el primer eslabón de la cadena (los 13 _root servers_) cayese, Internet dejaría de funcionar.

Vamos a hacer, en modo de resumen final, un ejemplo completo de una posible resolución que podría llevar a cabo una máquina cualquiera conectada a Internet, sobre el dominio **www.alvarovf.com**:

- El primer paso que nuestra máquina realiza es mirar en el fichero **hosts** de nuestra máquina, en la que podemos indicar una resolución estática de nombres de forma manual, que prevalece sobre cualquier resolución DNS.
- En caso de no encontrar coincidencia, preguntará por la dirección de dicho dominio al primer servidor DNS que tengamos configurado, en el caso de Linux, en el fichero **/etc/resolv.conf**, que mirará si tiene _cacheada_ la respuesta o si es uno de los dominios a los que sirve, de manera que en caso de que así sea, nos dirá la dirección IP, pero en caso contrario, el proceso continúa.
- En caso de que el servidor DNS al que hemos preguntado del tipo _forward_, dicha petición será reenviada a un servidor DNS recursivo, que llevará a cabo los siguientes pasos:
    - La primera petición irá a uno de los 13 _root server_, que al no conocer la dirección final, nos devolverá las direcciones de los servidores con autoridad sobre la zona **com.**.
    - La segunda petición ira a uno de los servidores devueltos por el _root server_ que al tampoco conocer la dirección final, nos devolverá las direcciones de los servidores con autoridad sobre la zona **alvarovf.com.**, concretamente los de la empresa con la que he alquilado el dominio (Nominalia).
    - En este caso, la tercera petición es la última, ya que esta petición se lleva a cabo sobre un servidor DNS que conoce la zona de mi dominio en su totalidad, devolviendo por tanto la dirección IP asociada a **www.alvarovf.com**.
- La respuesta llegará al servidor DNS al que nuestra máquina está configurada para preguntar, _cacheando_ dicha respuesta por un periodo finito de tiempo (**TTL**), con la finalidad de que en caso de recibir otra petición preguntando por el mismo nombre, pueda dar la respuesta directamente, evitando así tener que realizar dichas peticiones recursivas y ofreciendo unas respuestas mucho más rápidas.

Pueden llegar a darse escenarios un poco más complejos, como por ejemplo aquel en el que un mismo dominio tenga varias zonas definidas, por ejemplo, una zona DNS alcanzable desde Internet en el servidor DNS de la empresa con la que tenemos alquilado el dominio, en la que definimos aquellas máquinas y servicios que queremos que estén expuestos a Internet asociados a direcciones IP **públicas** (como un servidor web), y otra zona en un servidor DNS en la red local, en la que definimos aquellas máquinas y servicios que no queremos que estén expuestos a Internet a direcciones IP **privadas** (como un servidor de bases de datos), o incluso que resuelvan a las mismas direcciones pero queremos obtener unas respuestas mucho más rápidas.

Además, teniendo un servidor DNS local podemos evitar el problema de tener que configurar el fichero **hosts** en todas las máquinas clientes, así como tener que modificar dichas resoluciones en el fichero en caso de que la dirección IP de una máquina o servicio cambiase, pues de esta manera, tenemos la resolución centralizada en un único punto.

Antes de dar paso al primer servidor DNS que trataremos, es importante conocer los tipos de registros o informaciones que se pueden almacenar en una zona DNS, siendo los más comunes:

* **NS (Name Server)**: Indica los servidores con autoridad sobre una zona, es decir, indica a quién hay que preguntar por este dominio. Por ejemplo:

_alvarovf.com	NS	dns1.nominalia.com._

* **A (Address)**: Indica la dirección IPv4 a la que se traduce un nombre de dominio en la zona de resolución directa. Las empresas con mucho tráfico suelen poner varios registros A, consiguiendo así un balanceador de carga. Se suele utilizar para nombrar máquinas. Por ejemplo:

_alvarovf.com	A	51.210.109.246_

* **AAAA (Address)**: Indica la dirección IPv6 a la que se traduce un nombre de dominio en la zona de resolución directa. Por ejemplo:

_ipv6.l.google.com	AAAA	2a00:1450:400c:c06::93_

* **MX (Mail Exchange)**: Indica dónde se encuentran los servidores de correo del dominio. Se pueden definir múltiples y asignarles una prioridad, siendo esta mayor mientras menor sea el número indicado. Por ejemplo:

_MX 10 mail1.alvarovf.com_
_MX 50 mail2.alvarovf.com_

* **CNAME (Canonical Name)**: Permite definir un Alias, que generalmente es utilizado para los servicios, utilizando un CNAME que apunte a una máquina en la que se encuentra alojado dicho servicio, que tendrá asociado un registro A, consiguiendo así que en caso de necesitar cambiar la dirección IP de dicha máquina, únicamente haya que hacerlo en el registro A. Por ejemplo:

_www.alvarovf.com	CNAME	alvarovf.com_

* **TXT (Text)**: Proporciona información a fuentes externas y se utiliza por distintos fines, como por ejemplo el SPF, utilizado para luchar contra el correo SPAM y cuya función es comprobar que el origen IP de un correo está autorizado por el dominio. También puede utilizarse para comprobar la propiedad o administración de un dominio.

* **PTR (Pointer)**: Se utiliza en las zonas de resolución inversa para indicar a qué nombre corresponde una dirección IP. A la hora de definir el registro, la dirección IP se nombra invirtiendo el valor de sus octetos y añadiendo a la misma la cadena "**in-addr.arpa**". Por ejemplo, en el caso de la dirección _172.22.200.184_, el resultado sería _184.200.22.172.in-addr.arpa_.

## Servidor DNSmasq

El servicio **dnsmasq** nos ofrece varias funcionalidades, encontrándose entre las más destacadas servidor **DNS**, servidor **DHCP** (con soporte para DHCPv6), servidor **PXE** y servidor **TFTP**. Es muy apropiado para redes pequeñas donde necesitamos que nuestros clientes puedan resolver nombres, recibir automáticamente la configuración de red o crear un sistema para arrancar por red. En este artículo nos vamos a centrar en las posibilidades que nos ofrece como **servidor DNS**.

Dicho servidor actúa como servidor caché y _forward_, es decir, no es capaz de realizar las preguntas recursivas por sí mismo, sino que las reenvia a otro servidor (definido en el fichero **/etc/resolv.conf**) con dicha capacidad para que las haga en su lugar, guardando a continuación dicha resolución en caché para acelerar así las resoluciones posteriores.

La gran característica es que todos aquellos nombres definidos en el fichero **/etc/hosts** de la máquina servidora podrán ser consultados por los clientes tanto de forma directa como inversa. Esto, conlleva una serie de limitaciones como es lógico, pero se compensan con la rápida y sencilla instalación que requiere dicho servicio. De todas formas, para un uso cotidiano es más que válido, aunque a continuación trataremos otro servidor DNS un poco más complejo pero que ofrece gama de posibilidades bastante más superior.

Para este caso, he instalado previamente una máquina virtual con Debian Buster que es la que actuará como servidora (con dirección IP **172.22.200.184**) y sobre la que instalaremos el correspondiente servicio. Considero que la explicación de dicha instalación se sale del objetivo de este artículo, así que vamos a obviarla.

Tras ello, vamos a proceder con la instalación y configuración de dicho servidor DNS, de manera que lo primero que tendremos que hacer será instalar el paquete **dnsmasq**, no sin antes actualizar toda la paquetería existente en la máquina, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@dns:~# apt update && apt upgrade && apt install dnsmasq
{% endhighlight %}

Una vez instalado, es imprescindible indicar al servicio en qué interfaz o interfaces debe escuchar peticiones (por defecto, en el puerto **53/UDP**), por lo que procederemos a modificar el fichero **/etc/dnsmasq.conf**, haciendo para ello uso del comando:

{% highlight shell %}
root@dns:~# nano /etc/dnsmasq.conf
{% endhighlight %}

Dentro del mismo, tendremos que buscar la directiva **interface** y asignarle el valor de la interfaz de red de la máquina servidora conectada al exterior, en este caso, **eth0**, quedando de la siguiente forma:

{% highlight shell %}
interface=eth0
{% endhighlight %}

Hasta aquí ha llegado la configuración del servicio, que como se ha podido apreciar, ha sido realmente simple.

Es siempre recomendable que las máquinas servidoras tengan correctamente configurado el nombre (**_hostname_**), así como el nombre totalmente cualificado (**_FQDN_**), así que vamos a nombrarla dentro del dominio cuya zona DNS configuraré posteriomente, **iesgn.org**.

En la máquina virtual Debian Buster que estoy utilizando existe una pequeña peculiaridad, y es que el fichero **/etc/hosts** se genera dinámicamente durante el arranque, gracias al estándar **cloud-init**, así que para deshabilitarlo y así conseguir un fichero estático, tendremos que cambiar el valor de la directiva **manage_etc_hosts** a **false** en el fichero **/etc/cloud/cloud.cfg**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dns:~# sed -i 's/manage_etc_hosts: true/manage_etc_hosts: false/g' /etc/cloud/cloud.cfg
{% endhighlight %}

Para verificar que el valor de dicha directiva ha sido modificado, vamos a visualizar el contenido de dicho fichero, estableciendo un filtro por nombre:

{% highlight shell %}
root@dns:~# cat /etc/cloud/cloud.cfg | egrep 'manage_etc_hosts'
manage_etc_hosts: false
{% endhighlight %}

Efectivamente, su valor ha sido modificado, así que ya podemos proceder a modificar el fichero **/etc/hosts** para así asignar correctamente la resolución del _hostname_ y _FQDN_ de la máquina, así como añadir algunos registros para comprobar la efectividad de las resoluciones por parte del servidor DNS. Para modificar dicho fichero haremos uso del comando:

{% highlight shell %}
root@dns:~# nano /etc/hosts
{% endhighlight %}

El contenido que encontramos por defecto es el siguiente:

{% highlight shell %}
127.0.1.1 dns.novalocal dns
127.0.0.1 localhost

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
{% endhighlight %}

La línea que nos interesa modificar es la primera de ellas, la cuál resuelve la dirección **127.0.1.1**, siendo lo primero que se indique el **FQDN**, en este caso, **alvaro.iesgn.org** y lo segundo, el **hostname**, en este caso, **alvaro**.

Además, añadiremos nuevas líneas de la forma **[IP] [Nombre]** para unos sitios web ficticios que se podrían encontrar alojados en esta misma máquina servidora (para así comprobar el correcto funcionamiento), siendo **[IP]** la dirección IP alcanzable desde el exterior (en este caso **172.22.200.184**, ya que si ponemos la dirección **127.0.0.1**, la resolución no funcionaría correctamente para los clientes) y **[Nombre]** el nombre de dominio que vamos a resolver a dicha dirección, siendo dichos nombres en este caso **www.iesgn.org** y **departamentos.iesgn.org**, por ejemplo, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.1.1 alvaro.iesgn.org alvaro
127.0.0.1 localhost
172.22.200.184 www.iesgn.org
172.22.200.184 departamentos.iesgn.org

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
{% endhighlight %}

Tras ello, guardaremos los cambios realizados en el fichero. Todavía no hemos finalizado de configurar el nombre para la máquina, ya que lo que hemos establecido ha sido la resolución del mismo, faltando por configurar todavía el fichero **/etc/hostname**, que es donde se declara propiamente el nombre de la misma. Para modificar dicho fichero ejecutaremos el comando:

{% highlight shell %}
root@dns:~# nano /etc/hostname
{% endhighlight %}

Dentro del mismo, tendremos que indicar el _hostname_ de la máquina, que deberá ser exactamente el mismo que hemos indicado hace un momento en el fichero **/etc/hosts**, en este caso, **alvaro**.

Para que los cambios surtan efecto, tendremos que reiniciar, consiguiendo así que la máquina cargue su nuevo nombre, así como que el servicio **dnsmasq** cargue su nueva configuración (para conseguir esto último bastaría con reiniciar el servicio, pero así _matamos dos pájaros de un tiro_). Para ello, haremos uso del comando:

{% highlight shell %}
root@dns:~# reboot
{% endhighlight %}

Cuando la máquina haya completado su reinicio, podremos apreciar que el _prompt_ habrá variado, pero aun así, ejecutaremos los siguientes comandos para verificar el cambio:

{% highlight shell %}
debian@alvaro:~$ hostname
alvaro

debian@alvaro:~$ hostname -f
alvaro.iesgn.org
{% endhighlight %}

Efectivamente, el _hostname_ y el _FQDN_ de la máquina es ahora el correcto, por lo que nuestra labor en la máquina servidora ha finalizado, volviendo por tanto a la máquina anfitriona, que hará el papel de cliente, para así llevar a cabo las correspondientes pruebas de funcionamiento.

Es muy importante mencionar que es necesario que el puerto **53/UDP** de la máquina servidora se encuentre abierto, para así poder atender las solicitudes por parte de los clientes, que en este caso, lo damos por hecho.

El paso anterior a la realización de pruebas consiste en indicar el servidor DNS a utilizar por defecto por parte del cliente (en una situación real, esto sería una tarea del servidor DHCP), modificando para ello el fichero **/etc/resolv.conf**, haciendo para ello uso del comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/resolv.conf
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea **nameserver** indicando la dirección IP del servidor DNS al que queremos preguntar por defecto (de manera que lo colocaremos en primera posición), en este caso, **172.22.200.184**, quedando de la siguiente forma:

{% highlight shell %}
domain gonzalonazareno.org
search gonzalonazareno.org. 41011038.41.andared.ced.junta-andalucia.es.
nameserver 172.22.200.184
nameserver 192.168.202.2
nameserver 192.168.204.2
{% endhighlight %}

Listo, ya está todo preparado para llevar a cabo las pruebas de funcionamiento del servidor **dnsmasq**, de manera que haremos uso de la herramienta `dig` para llevar a cabo dichas consultas al servidor DNS. Al no venir instalada por defecto, tendremos que instalarla manualmente, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt install dnsutils
{% endhighlight %}

La primera resolución que vamos a tratar de llevar a cabo va a ser la correspondiente al nombre **www.iesgn.org**, por lo que haremos uso del comando:

{% highlight shell %}
alvaro@debian:~$ dig www.iesgn.org

; <<>> DiG 9.11.5-P4-5.1+deb10u2-Debian <<>> www.iesgn.org
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12601
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.iesgn.org.			IN	A

;; ANSWER SECTION:
www.iesgn.org.		0	IN	A	172.22.200.184

;; Query time: 2 msec
;; SERVER: 172.22.200.184#53(172.22.200.184)
;; WHEN: jue nov 26 09:32:58 CET 2020
;; MSG SIZE  rcvd: 58
{% endhighlight %}

Como se puede apreciar en la **ANSWER SECTION**, el nombre **www.iesgn.org.** tiene asociado un registro **A** que apunta a la dirección IPv4 **172.22.200.184**, tal y como hemos configurado en el fichero **/etc/hosts** de la máquina servidora. Tenemos además la certeza de que el servidor que nos ha respondido ha sido **172.22.200.184**, tal y como se puede apreciar en la directiva **SERVER**.

Repetiremos la misma prueba esta vez con el nombre **departamentos.iesgn.org**, que también fue definido en el fichero de resolución estática del servidor, por lo que la respuesta mostrada debería ser la misma:

{% highlight shell %}
alvaro@debian:~$ dig +short departamentos.iesgn.org
172.22.200.184
{% endhighlight %}

Donde:

* **+short**: Indicamos que muestre una respuesta más concreta, con la finalidad de no ensuciar demasiado.

Efectivamente, la respuesta ha sido correcta, al igual que la anterior, por lo que ahora vamos a ir un paso mas allá y vamos a realizar una consulta de un nombre que no sea conocido por nuestro servidor DNS, de manera que tendrá que actuar como servidor DNS _forward_ y reenviar dicha petición a uno de los servidores configurados en su fichero **/etc/resolv.conf**, como por ejemplo **www.josedomingo.org**:

{% highlight shell %}
alvaro@debian:~$ dig www.josedomingo.org
...
;; ANSWER SECTION:
www.josedomingo.org.	900	IN	CNAME	playerone.josedomingo.org.
playerone.josedomingo.org. 542	IN	A	137.74.161.90
...
;; Query time: 163 msec
;; SERVER: 172.22.200.184#53(172.22.200.184)
;; WHEN: jue nov 26 09:33:15 CET 2020
;; MSG SIZE  rcvd: 322
{% endhighlight %}

Como se puede apreciar en la salida que manualmente he recortado para no ensuciar, la resolución se ha realizado correctamente en un tiempo total de **163 ms**, informando de que el nombre por el que hemos preguntado tiene asociado un registro **CNAME** al nombre **playerone.josedomingo.org.**, teniendo este último a su vez asociado un registro **A** que apunta a la dirección IPv4 **137.74.161.90**.

Como consecuencia de ser un servidor DNS caché, la respuesta que acaba de obtener la habrá almacenado, de manera que si volvemos a preguntar por el mismo nombre, la respuesta será prácticamente inmediata:

{% highlight shell %}
alvaro@debian:~$ dig www.josedomingo.org
...
;; ANSWER SECTION:
www.josedomingo.org.	900	IN	CNAME	playerone.josedomingo.org.
playerone.josedomingo.org. 542	IN	A	137.74.161.90
...
;; Query time: 4 msec
;; SERVER: 172.22.200.184#53(172.22.200.184)
;; WHEN: jue nov 26 09:33:26 CET 2020
;; MSG SIZE  rcvd: 103
{% endhighlight %}

Como era de esperar, el tiempo de respuesta ha variado de **163 ms** a nada más que **4 ms**, pues la respuesta estaba cacheada, de manera que no ha sido necesario reenviar dicha petición a un servidor recursivo que proceda a realizar dichas preguntas de forma recursiva.

Hasta ahora hemos estado haciendo resoluciones directas, es decir, hemos preguntado por un nombre y se nos ha devuelto una dirección IP, así que vamos a hacer justo lo contrario, vamos a preguntar por una dirección IP para que nos devuelva un nombre (resolución inversa), en este caso, para la dirección IP **172.22.200.184**:

{% highlight shell %}
alvaro@debian:~$ dig -x 172.22.200.184
...
;; ANSWER SECTION:
184.200.22.172.in-addr.arpa. 0	IN	PTR	www.iesgn.org.
...
;; Query time: 1 msec
;; SERVER: 172.22.200.184#53(172.22.200.184)
;; WHEN: jue nov 26 09:38:53 CET 2020
;; MSG SIZE  rcvd: 83
{% endhighlight %}

Donde:

* **-x**: Indicamos que queremos llevar a cabo una resolución inversa, ya que por defecto se realizan resoluciones directas.

Como se puede apreciar en la respuesta, dicha dirección apunta a **www.iesgn.org** mediante un registro **PTR**. En realidad, la resolución que estamos tratando de realizar es bastante inusual, ya que la resolución inversa se suele llevar a cabo para máquinas (de manera que su dirección IP está declarada de forma única), mientras que en este caso, estamos tratando de resolver inversamente para un servicio (encontrándose dicha dirección repetida para los servicios **www** y **departamentos**), por lo que únicamente nos ha mostrado la primera coincidencia.

En mi opinión, el uso de **dnsmasq** ha quedado ya bastante claro, así que vamos a dar paso a la instalación y configuración de un servidor DNS un poco más completo a la vez que complejo.

Antes de ello, tendremos que volver a la máquina servidora para así parar o desinstalar el servicio actual, ya que ambos servicios no puede estar en ejecución a la vez, pues hacen uso del mismo puerto. Para parar y evitar que el servicio **dnsmasq** se levante automáticamente junto a la máquina, ejecutaremos los comandos:

{% highlight shell %}
root@alvaro:~# systemctl stop dnsmasq
root@alvaro:~# systemctl disable dnsmasq
{% endhighlight %}

El servicio **dnsmasq** ha sido ya apagado y deshabilitado su arranque durante el inicio de la máquina, de manera que el puerto a usar por el siguiente servidor DNS se encuentra otra vez operativo.

## Servidor bind9

De otro lado, el servicio **bind9** es el servidor de nombres de dominio más popular en Internet, que trabaja en todas las plataformas informáticas principales y se caracteriza por su flexibilidad y seguridad. En este artículo se tratará de forma bastante detallada la configuración y mantenimiento del mismo.

Dicho servidor actúa como servidor recursivo y caché, es decir, es capaz de realizar las preguntas recursivas por sí mismo, por lo que no necesita hacer uso de otro servidor para ello, además de _cachear_, al igual que ocurría con **dnsmasq**, la respuesta obtenida, para así mejorar los tiempos de las posteriores solicitudes.

Para este caso, he instalado previamente una máquina virtual con Debian Buster que es la que actuará como servidora y sobre la que instalaremos el correspondiente servicio. Considero que la explicación de dicha instalación se sale del objetivo de este artículo, así que vamos a obviarla.

Dado que previamente hemos parado el servicio **dnsmasq** también instalado en la máquina, todo está listo para llevar a cabo la instalación de **bind9**, ejecutando para ello el comando:

{% highlight shell %}
root@alvaro:~# apt install bind9
{% endhighlight %}

Una vez instalado, tendremos que proceder con la configuración inicial del servicio. La configuración de BIND consta de varios archivos que se incluyen (con directivas **include**) desde el archivo de configuración principal, de nombre **named.conf**. Estos nombres de archivos comienzan con **named** porque ese es el nombre del proceso que BIND ejecuta (abreviatura de "_domain name daemon_").

El primer fichero de configuración que modificaremos será el de opciones, de nombre **named.conf.options**, que al igual que el resto, se encuentra ubicado en **/etc/bind/**, haciendo para ello uso del comando:

{% highlight shell %}
root@alvaro:~# nano /etc/bind/named.conf.options
{% endhighlight %}

Dentro del mismo, encontraremos una directiva principal de nombre **options**, en la cuál debemos introducir una serie de líneas:

{% highlight shell %}
recursion yes; #Permitimos que el servidor actúe como servidor DNS recursivo.
allow-recursion { any; }; #Permitimos que las peticiones de todos los clientes puedan ser recursivas.
listen-on { any; }; #Permitimos la escucha del servicio en todas las interfaces de la máquina.
allow-transfer { none; }; #Deshabilitamos la transferencia de zonas por defecto. Lo trataremos con más detalle a continuación.
{% endhighlight %}

Tras ello, guardaremos los cambios en el fichero y procederemos a modificar ahora el fichero **/etc/bind/named.conf.local**, en el que declararemos las zonas sobre las que tendrá autoridad y el fichero en el que están contenidas, así como algunos parámetros opciones sobre las mismas. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@alvaro:~# nano /etc/bind/named.conf.local
{% endhighlight %}

Lo primero que haremos será descomentar la primera línea, correspondiente al **include** del fichero **/etc/bind/zones.rfc1918**, ya que se considera una buena práctica, ya que vamos a prevenir el envío de algunas consultas inversas innecesarias a Internet, ahorrándonos por tanto tiempo, ancho de banda y carga del servidor DNS.

Tras ello, tendremos que añadir un bloque por cada una de las zonas sobre las que el servidor tiene autoridad, que en este caso serán 2, una zona de resolución directa, y una zona de resolución inversa.

La primera de ellas tendrá nombre **iesgn.org**, pues es el dominio con el que vamos a trabajar. Además, la configuraremos de tipo maestro (**master**), que todavía no hemos introducido dicho concepto pero lo haremos próximamente. Por último, el fichero en el que vamos a definir la zona DNS será **db.iesgn.org**, que al no haber indicado ninguna ruta para el mismo, lo tratará de buscar por defecto en **/var/cache/bind/**.

La segunda de ellas, al tratarse de una zona inversa, tendrá nombre **200.22.172.in-addr.arpa**, resultado de invertir los octetos de la red con la que vamos a trabajar, que en este caso se trata de una **/24** y añadir la cadena **in-addr.arpa**. Al igual que la anterior zona, la configuraremos de tipo maestro (**master**), siendo el fichero en el que vamos a definir la zona DNS **db.200.22.172**, que lo tratará de buscar por defecto en **/var/cache/bind/**. Es muy importante comprobar que la red en la que vamos a resolver de forma inversa no se encuentra contenida en el fichero **/etc/bind/zones.rfc1918**, ya que de lo contrario, no nos contestará.

El resultado final del fichero sería:

{% highlight shell %}
include "/etc/bind/zones.rfc1918";

zone "iesgn.org" {
        type master;
        file "db.iesgn.org";
};

zone "200.22.172.in-addr.arpa" {
        type master;
        file "db.200.22.172";
};
{% endhighlight %}

Una vez que hayamos guardado los cambios en el mismo, habrá llegado la hora de definir las dos zonas DNS con las que vamos a trabajar. Para facilitarnos un poco la tarea de sintaxis, podemos hacer uso de una plantilla de nombre **/etc/bind/db.empty** para a partir de ahí, modificarla a nuestras necesidades. Vamos a comenzar con la zona de resolución directa, así que copiaremos dicho fichero dentro de **/var/cache/bind/** con el nombre previamente asignado, **db.iesgn.org**, haciendo para ello uso del comando:

{% highlight shell %}
root@alvaro:~# cp /etc/bind/db.empty /var/cache/bind/db.iesgn.org
{% endhighlight %}

La zona DNS ya ha sido creada, aunque actualmente está vacía, de manera que vamos a proceder a modificarla para añadir los registros correspondientes a la misma, ejecutando para ello los comandos:

{% highlight shell %}
root@alvaro:~# nano /var/cache/bind/db.iesgn.org
{% endhighlight %}

El contenido de la plantilla es el siguiente:

{% highlight shell %}
$TTL    86400
@       IN      SOA     localhost. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
{% endhighlight %}

Los parámetros sobre los que podemos llevar a cabo modificaciones son los siguientes:

- **TTL (Time To Live)**: Tiempo de vida de las respuestas que ofrezca el DNS para un servidor DNS intermedio, es decir, indicamos el tiempo en segundos que durarán _cacheadas_ las respuestas de este servidor en aquellos servidores que actúen como caché DNS. En mi caso, considero correcto el valor por defecto (24 horas).
- **SOA (Start of Authority)**: Es una de las directivas más importantes, en la que indicaremos el _FQDN_ del servidor con autoridad maestro (es decir, el servidor con el que estamos trabajando, **alvaro.iesgn.org.**), además del correo electrónico de la persona responsable de la zona DNS (por ejemplo, **admin.iesgn.org.**). Dentro de la misma, indicaremos un conjunto de tiempos:
    - **Serial**: Identificador de la zona, que deberá ser incrementado después de cada cambio. Se recomienda que tenga la forma **YYMMDDNN**, por ejemplo, la primera modificación del día _4 de Diciembre de 2020_ sería **20120401**.
    - **Refresh**: Tiempo en segundos que han de transcurrir entre una comprobación automática de transferencia de zona y otra por parte de los esclavos. Dejaremos el valor por defecto.
    - **Retry**: Tiempo en segundos que el esclavo tardará en reintentar la comprobación de transferencia de zona en caso de que el anterior intento haya fallado. Dejaremos el valor por defecto.
    - **Expire**: Tiempo en segundos que el esclavo tardará en borrar la zona si no tiene conectividad con el servidor maestro. Dejaremos el valor por defecto.
    - **Negative Cache TTL**: Muy similar al TTL pero para aquellos nombres no existentes, de manera que el servidor DNS intermedio conocerá la inexistencia del mismo, evitando realizar preguntas reiteradas por el mismo nombre, pues no existe. Dejaremos el valor por defecto.
- **NS (Name Server)**: Tendremos que indicar el FQDN del servidor con autoridad sobre la zona, que en este caso es el servidor con el que estamos trabajando, **alvaro.iesgn.org.**.

Esta sería la configuración base mínima para configurar la zona, de manera que a partir de ahora, tendremos que ir añadiendo registros para nombrar las máquinas y servicios existentes en dicha zona. A partir de aquí, la configuración se vuelve algo más "personal", en el sentido de que puede variar según las necesidades de una persona u otra.

Lo primero que haré será indicar un servidor de correo para el dominio, que se encontrará en **correo.iesgn.org.**, asignándole una prioridad de **10** (bastante alta), haciendo para ello uso de un registro **MX**:

_@       IN      MX      10      correo.iesgn.org._

Tras ello, estableceremos un registro **$ORIGIN iesgn.org.**, indicando que todos los registros definidos a partir de ahí serán autocompletados con el dominio especificado, evitándonos así tener que escribir todo el rato el nombre de dominio para escribir el _FQDN_ de las máquinas o servicios. En mi caso, los registros introducidos son los siguientes:

_alvaro  IN      A       172.22.200.184_ #Añadimos un registro A para nombrar la máquina servidora actual.
_correo  IN      A       172.22.200.200_ #Añadimos un registro A para nombrar el servidor de correo ficticio previamente definido.
_ftp     IN      A       172.22.200.201_ #Añadimos un registro A para nombrar un servidor FTP ficticio.
_www     IN      CNAME   alvaro_ #Añadimos un registro CNAME apuntando a la máquina _alvaro_, ya que el servicio _www_ se encuentra alojado en la misma.
_departamentos   IN      CNAME   alvaro_ #Añadimos un registro CNAME apuntando a la máquina _alvaro_, ya que el servicio _departamentos_ se encuentra alojado en la misma.

El resultado final del fichero sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     alvaro.iesgn.org. admin.iesgn.org. (
                       20120401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      alvaro.iesgn.org.
@       IN      MX      10      correo.iesgn.org.

$ORIGIN iesgn.org.

alvaro  IN      A       172.22.200.184
correo  IN      A       172.22.200.200
ftp     IN      A       172.22.200.201
www     IN      CNAME   alvaro        
departamentos   IN      CNAME   alvaro
{% endhighlight %}

La definición de la zona de resolución directa ha finalizado, de manera que todavía nos queda definir la zona de resolución inversa en el fichero **db.200.22.172**, de manera que para facilitar el trabajo, vamos a utilizar el fichero de la zona de resolución como plantilla base y a partir de ahí, lo adaptaremos a nuestras necesidades. Para copiar dicho fichero haremos uso del comando:

{% highlight shell %}
root@alvaro:~# cp /var/cache/bind/db.iesgn.org /var/cache/bind/db.200.22.172
{% endhighlight %}

Una vez creada la zona de resolución inversa, tendremos que modificar el correspondiente fichero para añadir los registros correctos. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@alvaro:~# nano /var/cache/bind/db.200.22.172
{% endhighlight %}

Existen una serie de particularidades que tendremos que modificar con respecto a la zona de resolución inversa, como por ejemplo, que no podremos tener un registro **MX** en una zona de resolución inversa. Como se puede suponer, tendremos que modificar el valor del registro **$ORIGIN** a **200.22.172.in-addr.arpa.**, que será utilizado para autocompletar los registros que introduzcamos.

En mi caso, los registros introducidos son los siguientes:

_184     IN      PTR     alvaro.iesgn.org._ #Añadimos un registro PTR para hacer que la dirección **172.22.200.184** apunte a la máquina servidora actual.
_200     IN      PTR     correo.iesgn.org._ #Añadimos un registro PTR para hacer que la dirección **172.22.200.200** apunte al servidor de correo ficticio.
_201     IN      PTR     ftp.iesgn.org._ #Añadimos un registro PTR para hacer que la dirección **172.22.200.201** apunte al servidor FTP ficticio.

Como se puede apreciar, los servicios **www** y **departamentos** no han sido nombrados, ya que en las zonas de resolución inversa únicamente deben nombrarse máquinas. El resultado final del fichero sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     alvaro.iesgn.org. admin.iesgn.org. (
                       20120401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      alvaro.iesgn.org.

$ORIGIN 200.22.172.in-addr.arpa.

184     IN      PTR     alvaro.iesgn.org.
200     IN      PTR     correo.iesgn.org.
201     IN      PTR     ftp.iesgn.org.
{% endhighlight %}

Una vez finalizada la definición de la zona DNS, vamos a proceder a comprobar que la sintaxis de los ficheros de configuración **named** sea correcta, antes proceder a cargar la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@alvaro:~# named-checkconf
{% endhighlight %}

La ejecución del comando no ha devuelto ninguna salida, por lo que podemos concluir que la sintaxis de los ficheros de configuración es correcta. A su vez, vamos a verificar la sintaxis de los ficheros de zona, pasándole a `named-checkzone` el nombre de una zona y el archivo en el que se encuentra contenida, haciendo para ello uso de los comandos:

{% highlight shell %}
root@alvaro:~# named-checkzone iesgn.org /var/cache/bind/db.iesgn.org
zone iesgn.org/IN: loaded serial 20120401
OK

root@alvaro:~# named-checkzone 200.22.172.in-addr.arpa /var/cache/bind/db.200.22.172 
zone 200.22.172.in-addr.arpa/IN: loaded serial 20120401
OK
{% endhighlight %}

La comprobación de sintaxis de ambas zonas DNS ha finalizado correctamente, por lo que podremos proceder a reiniciar el servicio para que así cargue la nueva configuración realizada. Es importante puntualizar que en caso de haber creado una nueva zona (y tuviésemos más zonas que hayan sido definidas previamente), no tendría sentido reiniciar el servicio al completo, ya que únicamente ha cambiado una zona de todas las definidas, pudiendo reiniciar una zona concreta con `rndc reload [zona]`. Sin embargo, en esta ocasión, la modificación se ha llevado a cabo sobre la totalidad de las zonas definidas, por lo que no existiría diferencia. El comando a ejecutar sería:

{% highlight shell %}
root@alvaro:~# systemctl restart bind9
{% endhighlight %}

Tras ello, volveremos a la máquina anfitriona que actuará una vez más como cliente, no siendo necesario volver a modificar el fichero **/etc/resolv.conf**, ya que lo hicimos con anterioridad. Una vez dentro de la misma, llevaré a cabo una serie de pruebas para verificar que todas las respuestas son correctas:

{% highlight shell %}
alvaro@debian:~$ dig +short alvaro.iesgn.org
172.22.200.184

alvaro@debian:~$ dig www.iesgn.org
...
;; ANSWER SECTION:
www.iesgn.org.		86400	IN	CNAME	alvaro.iesgn.org.
alvaro.iesgn.org.	86400	IN	A	172.22.200.184
...

alvaro@debian:~$ dig +short ftp.iesgn.org
172.22.200.201

alvaro@debian:~$ dig ns iesgn.org
...
;; ANSWER SECTION:
iesgn.org.		86400	IN	NS	alvaro.iesgn.org.

;; ADDITIONAL SECTION:
alvaro.iesgn.org.	86400	IN	A	172.22.200.184
...

alvaro@debian:~$ dig mx iesgn.org
...
;; ANSWER SECTION:
iesgn.org.		86400	IN	MX	10 correo.iesgn.org.

;; ADDITIONAL SECTION:
correo.iesgn.org.	86400	IN	A	172.22.200.200
...

alvaro@debian:~$ dig www.josedomingo.org
...
;; ANSWER SECTION:
www.josedomingo.org.	900	IN	CNAME	playerone.josedomingo.org.
playerone.josedomingo.org. 900	IN	A	137.74.161.90
...

alvaro@debian:~$ dig -x 172.22.200.200
...
;; ANSWER SECTION:
200.200.22.172.in-addr.arpa. 86400 IN	PTR	correo.iesgn.org.
...
{% endhighlight %}

Como se puede apreciar en la salida de todas las resoluciones, las respuestas han sido las correctas, por lo que podemos confirmar que tanto la recursividad como las zonas de resolución directa e inversa, están funcionando correctamente.

Para proceder con una configuración un poco más compleja en el escenario, siempre se recomienda configurar al menos un servidor DNS secundario (**esclavo**) que responda a las solicitudes si el primario (**maestro**) deja de estar disponible, consiguiendo así establecer una alta disponibilidad, pues dicho servidor DNS secundario conoce exactamente la misma información que el primario.

En realidad, en un principio puede parecer un poco confuso, ya que un mismo servidor puede ser **maestro** de una zona y **esclavo** de otra, pues es un rol que se establece a nivel de zona, no a nivel de máquina.

Para este caso, he instalado una segunda máquina virtual con Debian Buster que es la que actuará como esclava (con dirección IP **172.22.200.106**) y sobre la que instalaremos el correspondiente servicio. He llevado a cabo la configuración del **_hostname_** y **_FQDN_**, siendo **afrodita** y **afrodita.iesgn.org**, respectivamente, siguiendo para ello exactamente los mismos pasos que en la máquina servidora.

Antes de comenzar a configurar el servidor DNS secundario, debemos llevar a cabo dos modificaciones en el servidor maestro, para así permitir que las zonas puedan transferirse al esclavo, así como definir al nuevo servidor como servidor con autoridad sobre las zonas.

Para permitir las transferencias de zonas, tendremos que modificar el fichero **/etc/bind/named.conf.local**, ejecutando para ello el comando:

{% highlight shell %}
root@alvaro:~# nano /etc/bind/named.conf.local
{% endhighlight %}

Dentro del mismo, tendremos que añadir dos directivas a cada una de las zonas: **allow-transfer**, en la que indicaremos la dirección IP del servidor DNS secundario al que las zonas van a ser transferidas, para así permitirlas (ya que por defecto las hemos deshabilitado por seguridad), es este caso, **172.22.200.106**, y una directiva **notify yes**, para que así cuando se produzca una modificación en la zona declarada en el maestro, se envíe una notificación a los esclavos para que soliciten una transferencia de la misma. El resultado final del fichero sería:

{% highlight shell %}
include "/etc/bind/zones.rfc1918";

zone "iesgn.org" {
        type master;
        file "db.iesgn.org";
        allow-transfer { 172.22.200.106; };
        notify yes;
};

zone "200.22.172.in-addr.arpa" {
        type master;
        file "db.200.22.172";
        allow-transfer { 172.22.200.106; };
        notify yes;
};
{% endhighlight %}

Tras ello, tendremos que asignar al nuevo servidor como servidor con autoridad sobre las zonas existentes, así que procederemos a modificar los ficheros **/var/cache/bind/db.iesgn.org** y **/var/cache/bind/db.200.22.172** con `nano`, para posteriormente añadir un nuevo registro **NS** a los mismos, así como nombrar a la máquina **afrodita** con un registro **A** o **PTR**, dependiendo del caso. Para ello, haremos uso de los comandos:

{% highlight shell %}
root@alvaro:~# nano /var/cache/bind/db.iesgn.org

$TTL    86400
@       IN      SOA     alvaro.iesgn.org. admin.iesgn.org. (
                       20120801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      alvaro.iesgn.org.
@       IN      NS      afrodita.iesgn.org.
@       IN      MX      10      correo.iesgn.org.

$ORIGIN iesgn.org.

alvaro  IN      A       172.22.200.184
afrodita        IN      A       172.22.200.106
correo  IN      A       172.22.200.200
ftp     IN      A       172.22.200.201
www     IN      CNAME   alvaro
departamentos   IN      CNAME   alvaro

root@alvaro:~# nano /var/cache/bind/db.200.22.172

$TTL    86400
@       IN      SOA     alvaro.iesgn.org. admin.iesgn.org. (
                       20120801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      alvaro.iesgn.org.
@       IN      NS      afrodita.iesgn.org.

$ORIGIN 200.22.172.in-addr.arpa.

184     IN      PTR     alvaro.iesgn.org.
106     IN      PTR     afrodita.iesgn.org.
200     IN      PTR     correo.iesgn.org.
201     IN      PTR     ftp.iesgn.org.
{% endhighlight %}

Como se puede apreciar, el **Serial** no ha sido incrementado, pues a pesar de anteriormente haber dicho que es necesario incrementarlo tras cada modificación, en este caso no es necesario. Para entender por qué no es necesario debemos conocer que la transferencia de una zona se lleva a cabo del maestro al esclavo cuando existe una diferencia de Serial, es decir, si el Serial del maestro es **2** y el del esclavo es **1**, se produciría una transferencia.

Sin embargo, si no modificamos dicho número, no va a existir diferencia entre ellos, por lo que la transferencia no se produciría. Es por eso, que el identificador debe modificarse después de cualquier cambio. Ahora bien, ¿por qué no lo hemos aumentado ahora? La respuesta es bastante sencilla, y es que el servidor DNS secundario no ha realizado ninguna transferencia de zona por ahora, por lo que es indiferente, ya que no tiene un Serial con el que comparar, de manera que la primera transferencia se va a completar sí o sí.

Es importante saber por tanto, que a partir de la primera transferencia, es necesario **incrementar** dicho número después de cada cambio (es común que se olvide hacerlo), para así conseguir que las zonas sean exactamente iguales en el maestro y en el esclavo en todo momento, evitando inconsistencias.

Tras ello, tendremos que volver a cargar la configuración del servicio en memoria para así aplicar los nuevos cambios, ejecutando para ello el comando:

{% highlight shell %}
root@alvaro:~# systemctl restart bind9
{% endhighlight %}

La configuración en el lado del servidor maestro habrá finalizado, así que entraremos al servidor esclavo para así comenzar con su configuración.

Como es lógico, el primer paso consistirá en instalar el paquete **bind9**, no sin antes actualizar toda la paquetería existente en la máquina, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@afrodita:~# apt update && apt upgrade && apt install bind9
{% endhighlight %}

Al igual que ocurrió con el servidor maestro, el primer fichero de configuración que modificaremos será el de opciones, de nombre **named.conf.options**, ubicado en **/etc/bind/**, haciendo para ello uso del comando:

{% highlight shell %}
root@afrodita:~# nano /etc/bind/named.conf.options
{% endhighlight %}

Dentro del mismo, estableceremos en la directiva principal de nombre **options** las mismas líneas que en el servidor maestro, ya que nos interesa que la configuración sea lo más similar posible:

{% highlight shell %}
recursion yes; #Permitimos que el servidor actúe como servidor DNS recursivo.
allow-recursion { any; }; #Permitimos que las peticiones de todos los clientes puedan ser recursivas.
listen-on { any; }; #Permitimos la escucha del servicio en todas las interfaces de la máquina.
allow-transfer { none; }; #Deshabilitamos la transferencia de zonas por defecto. Lo trataremos con más detalle a continuación.
{% endhighlight %}

Tras ello, guardaremos los cambios en el fichero y procederemos a modificar ahora el fichero **/etc/bind/named.conf.local**, en el que declararemos las zonas sobre las que tendrá autoridad y el fichero en el que están contenidas. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@afrodita:~# nano /etc/bind/named.conf.local
{% endhighlight %}

De nuevo, lo primero que haremos en dicho fichero será descomentar la primera línea, correspondiente al **include** del fichero **/etc/bind/zones.rfc1918**.

Tras ello, tendremos que añadir un bloque por cada una de las zonas sobre las que el servidor tiene autoridad, que en este caso serán 2, una zona de resolución directa, y una zona de resolución inversa.

La primera de ellas tendrá nombre **iesgn.org**. Además, la configuraremos de tipo esclavo (**slave**), indicando el fichero en el que estará contenida la zona (se generará automáticamente durante la transferencia), que en este caso será **db.iesgn.org**, que al no haber indicado ninguna ruta para el mismo, lo tratará de buscar por defecto en **/var/cache/bind/**. Por último, tendremos que indicar la dirección IP de la máquina maestra en la directiva **masters**, que será a la máquina a la que solicitará las transferencias de zonas, siendo en este caso, **172.22.200.184**.

La segunda de ellas tendrá nombre **200.22.172.in-addr.arpa**. Además, la configuraremos de tipo esclavo (**slave**), indicando el fichero en el que estará contenida la zona (se generará automáticamente durante la transferencia), que en este caso será **db.200.22.172**, que lo tratará de buscar por defecto en **/var/cache/bind/**. Por último, tendremos que indicar la dirección IP de la máquina maestra en la directiva **masters**, que será a la máquina a la que solicitará las transferencias de zonas, siendo en este caso, **172.22.200.184**. Es muy importante comprobar que la red en la que vamos a resolver de forma inversa no se encuentra contenida en el fichero **/etc/bind/zones.rfc1918**, ya que de lo contrario, no nos contestará.

El resultado final del fichero sería:

{% highlight shell %}
include "/etc/bind/zones.rfc1918";

zone "iesgn.org" {
        type slave;
        file "db.iesgn.org";
        masters { 172.22.200.184; };
};

zone "200.22.172.in-addr.arpa" {
        type slave;
        file "db.200.22.172";
        masters { 172.22.200.184; };
};
{% endhighlight %}

Es muy importante mencionar que es necesario que el puerto **53/TCP** de la máquina servidora maestra se encuentre abierto, para así poder atender las solicitudes de transferencia de zona por parte de las esclavas.

Tras ello, tendremos que reiniciar el servicio para que así cargue en memoria la nueva configuración, además de conseguir que se lleve a cabo por tanto la primera transferencia de zona, haciendo para ello uso del comando:

{% highlight shell %}
root@afrodita:~# systemctl restart bind9
{% endhighlight %}

Tras ello, vamos a comprobar el estado del servicio, en el que se deberían mostrar mensajes informativos referentes al estado de la transferencia de zona, por lo que ejecutaremos el comando:

{% highlight shell %}
root@afrodita:~# systemctl status bind9
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2020-12-08 15:37:21 UTC; 1min 14s ago
     Docs: man:named(8)
  Process: 781 ExecStart=/usr/sbin/named $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 783 (named)
    Tasks: 4 (limit: 562)
   Memory: 11.9M
   CGroup: /system.slice/bind9.service
           └─783 /usr/sbin/named -u bind

Dec 08 15:37:21 afrodita named[783]: zone 200.22.172.in-addr.arpa/IN: transferred serial 20120401
Dec 08 15:37:21 afrodita named[783]: transfer of '200.22.172.in-addr.arpa/IN' from 172.22.200.184#53: Transfer status: success
Dec 08 15:37:21 afrodita named[783]: transfer of '200.22.172.in-addr.arpa/IN' from 172.22.200.184#53: Transfer completed: 1 messages, 6 records, 214 bytes, 0.001 secs (214000 bytes/sec)
Dec 08 15:37:21 afrodita named[783]: zone iesgn.org/IN: Transfer started.
Dec 08 15:37:21 afrodita named[783]: transfer of 'iesgn.org/IN' from 172.22.200.184#53: connected using 10.0.0.13#42197
Dec 08 15:37:21 afrodita named[783]: zone iesgn.org/IN: transferred serial 20120401
Dec 08 15:37:21 afrodita named[783]: transfer of 'iesgn.org/IN' from 172.22.200.184#53: Transfer status: success
Dec 08 15:37:21 afrodita named[783]: transfer of 'iesgn.org/IN' from 172.22.200.184#53: Transfer completed: 1 messages, 9 records, 247 bytes, 0.002 secs (123500 bytes/sec)
Dec 08 15:37:22 afrodita named[783]: managed-keys-zone: Key 20326 for zone . acceptance timer complete: key now trusted
Dec 08 15:37:22 afrodita named[783]: resolver priming query complete
{% endhighlight %}

Como se puede apreciar, la transferencia de las zonas con Serial **20120401** se ha completado satisfactoriamente (**Transfer status: success**), por lo que a partir de este momento, nuestros servidores DNS se encuentran replicados en alta disponibilidad, de manera que si cayese cualquiera de ellos, el otro se encontraría en disposició para responder. Vuelvo a recalcar que en caso de llevarse a cabo una modificación ahora, sería indispensable aumentar el Serial en el servidor maestro, para que la transferencia de zona pueda completarse con la nueva información.

Antes de proceder a realizar las correspondientes pruebas, vamos a listar el contenido del directorio **/var/cache/bind/** para así verificar que los correspondientes ficheros con las zonas DNS han sido generados como consecuencia de las transferencias de zonas, haciendo para ello uso del comando:

{% highlight shell %}
root@afrodita:~# ls /var/cache/bind/
db.200.22.172  db.iesgn.org  managed-keys.bind	managed-keys.bind.jnl
{% endhighlight %}

Efectivamente, los correspondientes ficheros han sido correctamente generados, por lo que nuestra labor en los servidores DNS habrá finalizado, de manera que volveremos a la máquina anfitriona que actuará una vez más como cliente, para así realizar las correspondientes pruebas, no sin antes añadir el nuevo servidor DNS al fichero **/etc/resolv.conf**, de manera que si el servidor maestro fallase, podría hacer la misma consulta al servidor secundario. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/resolv.conf
{% endhighlight %}

El contenido final de dicho fichero sería:

{% highlight shell %}
domain gonzalonazareno.org
search gonzalonazareno.org. 41011038.41.andared.ced.junta-andalucia.es.
nameserver 172.22.200.184
nameserver 172.22.200.106
nameserver 192.168.202.2
{% endhighlight %}

Una vez establecido el nuevo servidor al que el cliente puede acudir, vamos a proceder a realizar determinadas pruebas. La primera de ellas consistirá en solicitar información referente al registro SOA para ambos servidores, con la finalidad de comparar la respuesta:

{% highlight shell %}
alvaro@debian:~$ dig +norec @172.22.200.184 iesgn.org. soa
...
;; flags: qr aa ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3
...
;; ANSWER SECTION:
iesgn.org.		86400	IN	SOA	alvaro.iesgn.org. admin.iesgn.org. 20120801 604800 86400 2419200 86400
...

alvaro@debian:~$ dig +norec @172.22.200.106 iesgn.org. soa
...
;; flags: qr aa ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3
...
;; ANSWER SECTION:
iesgn.org.		86400	IN	SOA	alvaro.iesgn.org. admin.iesgn.org. 20120801 604800 86400 2419200 86400
...
{% endhighlight %}

Donde:

* **@**: Forzamos la petición a un servidor DNS en concreto, ya que si no, utilizaría siempre el primero especificado en el fichero **/etc/resolv.conf**.

Como se puede apreciar, la respuesta por parte de ambos servidores ha devuelvo una flag **aa**, indicando que el servidor tiene autoridad sobre la zona DNS del dominio en cuestión. Además, tanto el Serial como los tiempos coinciden en ambos servidores, por lo que podemos concluir que la réplica es exacta.

Por último, vamos a llevar a cabo una comprobación de seguridad, para así verificar que la transferencia de zona únicamente puede ser realizada al servidor DNS secundario, y que un cliente cualquiera, no podría obtener dicha información, ya que un atacante podría utilizarla para conocer a la perfección la estructura existente. Para ello, llevaremos a cabo la misma petición desde la máquina servidora secundaria y un cliente cualquiera, como puede ser la máquina anfitriona:

{% highlight shell %}
root@afrodita:~# dig @172.22.200.184 iesgn.org. axfr

; <<>> DiG 9.11.5-P4-5.1+deb10u2-Debian <<>> @172.22.200.184 iesgn.org. axfr
; (1 server found)
;; global options: +cmd
iesgn.org.		86400	IN	SOA	alvaro.iesgn.org. admin.iesgn.org. 20120801 604800 86400 2419200 86400
iesgn.org.		86400	IN	NS	alvaro.iesgn.org.
iesgn.org.		86400	IN	NS	afrodita.iesgn.org.
iesgn.org.		86400	IN	MX	10 correo.iesgn.org.
afrodita.iesgn.org.	86400	IN	A	172.22.200.106
alvaro.iesgn.org.	86400	IN	A	172.22.200.184
correo.iesgn.org.	86400	IN	A	172.22.200.200
departamentos.iesgn.org. 86400	IN	CNAME	alvaro.iesgn.org.
ftp.iesgn.org.		86400	IN	A	172.22.200.201
www.iesgn.org.		86400	IN	CNAME	alvaro.iesgn.org.
iesgn.org.		86400	IN	SOA	alvaro.iesgn.org. admin.iesgn.org. 20120801 604800 86400 2419200 86400
;; Query time: 3 msec
;; SERVER: 172.22.200.184#53(172.22.200.184)
;; WHEN: Tue Dec 08 16:10:45 UTC 2020
;; XFR size: 11 records (messages 1, bytes 325)

alvaro@debian:~$ dig @172.22.200.184 iesgn.org. axfr

; <<>> DiG 9.11.5-P4-5.1+deb10u2-Debian <<>> @172.22.200.184 iesgn.org. axfr
; (1 server found)
;; global options: +cmd
; Transfer failed.
{% endhighlight %}

Como era de esperar, la transferencia de zona únicamente puede ser realizada al servidor DNS secundario, y no a un cliente cualquiera, ya que hemos denegado las transferencias por defecto en el fichero de configuración.

La última configuración que vamos a llevar a cabo en nuestro escenario es la referente a los subdominios, así que vamos a tratar un poco de teoría para entender de qué trata.

Hasta ahora, hemos estado trabajando con un nombre de dominio principal de segundo nivel (**iesgn.org.**) en el que hemos estado nombrando máquinas y servicios (**www.iesgn.org.**, **correo.iesgn.org.**...), pero si recordamos, anteriormente hemos mencionado que también podemos crear subdominios gestionados por nosotros mismos, o bien, delegar su gestión a otra persona o institución. Esto, nos permitiría tener por ejemplo, un nombre **departamento.iesgn.org.** dentro del cuál nombraríamos máquinas y servicios (**www.departamento.iesgn.org.**, **ftp.departamento.iesgn.org.**...).

Para la obtención de un subdominio tenemos dos alternativas:

* **Crear un subdominio virtual**: La gestión del subdominio se llevaría a cabo por el mismo servidor DNS que gestiona el dominio principal.
* **Delegar el subdominio**: La gestión del subdominio se delega a otro servidor DNS. Es la que nosotros utilizaremos. Para que se comprenda mejor, es lo que ocurre en la estructura jerárquica, pues los 13 _root servers_ delegan la gestión de los dominios de primer nivel a otras entidades, y así sucesivamente.

En este caso, tenemos un servidor con autoridad sobre la zona **iesgn.org** (**alvaro.iesgn.org**) que va a delegar la gestión del subdominio **informatica.iesgn.org** a otro servidor, en este caso, a **afrodita**.

Para llevar a cabo dicha configuración, lo primero que tendremos que hacer es modificar el fichero de zona **/var/cache/bind/db.iesgn.org** en el servidor maestro, ejecutando para ello el comando:

{% highlight shell %}
root@alvaro:~# nano /var/cache/bind/db.iesgn.org
{% endhighlight %}

Dentro del mismo, tendremos que añadir un nuevo registro **$ORIGIN** para el subdominio **informatica.iesgn.org.**, en el que indicaremos con un registro **NS** el servidor con autoridad sobre dicha zona, además de un registro **A** para indicar su dirección IPv4. Por último, tendremos que aumentar el serial en una unidad, para que así el cambio sea notificado también al servidor esclavo. El resultado final sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     alvaro.iesgn.org. admin.iesgn.org. (
                       20120802         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      alvaro.iesgn.org.
@       IN      NS      afrodita.iesgn.org.
@       IN      MX      10      correo.iesgn.org.

$ORIGIN iesgn.org.

alvaro  IN      A       172.22.200.184
afrodita        IN      A       172.22.200.106
correo  IN      A       172.22.200.200
ftp     IN      A       172.22.200.201
www     IN      CNAME   alvaro
departamentos   IN      CNAME   alvaro

$ORIGIN informatica.iesgn.org.

@       IN      NS      afrodita
afrodita        IN      A       172.22.200.106
{% endhighlight %}

La configuración en el lado del servidor DNS principal habrá finalizado, habiéndose completado por tanto la delegación del subdominio, de manera que únicamente nos quedaría reiniciar el servicio para así cargar la nueva configuración y que la transferencia de zona se complete tal y como debería. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@alvaro:~# systemctl restart bind9
{% endhighlight %}

Tras ello, volveremos al servidor DNS esclavo y modificaremos el fichero **/etc/bind/named.conf.local**, declarando por tanto la nueva zona sobre la que dicho servidor tendrá autoridad, actuándo ahora como maestro de la misma (a esto me refería con que un mismo servidor puede ser maestro de unas zonas y esclavo de otras a su vez), haciendo para ello uso del comando:

{% highlight shell %}
root@afrodita:~# nano /etc/bind/named.conf.local

include "/etc/bind/zones.rfc1918";

zone "iesgn.org" {
        type slave;
        file "db.iesgn.org";
        masters { 172.22.200.184; };
};

zone "200.22.172.in-addr.arpa" {
        type slave;
        file "db.200.22.172";
        masters { 172.22.200.184; };
};

zone "informatica.iesgn.org" {
        type master;
        file "db.informatica.iesgn.org";
};
{% endhighlight %}

Por último, tendremos que crear el fichero **/var/cache/bind/db.informatica.iesgn.org** que contendrá los registros de la zona DNS y definirlos, siguiendo para ello la misma que estructura que hasta ahora, con la única diferencia de que no vamos a trabajar con un dominio de segundo nivel, sino de tercer nivel. El comando a ejecutar sería:

{% highlight shell %}
root@afrodita:~# nano /var/cache/bind/db.informatica.iesgn.org

$TTL    86400
@       IN      SOA     afrodita.informatica.iesgn.org. admin.informatica.iesgn.org. (
                       20120401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      afrodita.informatica.iesgn.org.
@       IN      MX      10      correo.informatica.iesgn.org.

$ORIGIN informatica.iesgn.org.

afrodita        IN      A       172.22.200.106
correo  IN      A       172.22.200.205
www     IN      A       172.22.200.206
ftp     IN      CNAME   afrodita
{% endhighlight %}

Como se puede apreciar, he insertado algunos registros de prueba para ahora realizar las correspondientes peticiones desde un cliente, para así verificar que su funcionamiento es el correcto. Antes de ello, tendremos que reiniciar el servicio para que la nueva configuración sea cargada en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@afrodita:~# systemctl restart bind9
{% endhighlight %}

Tras ello, volveremos a la máquina anfitriona que actuará una vez más como cliente, no siendo necesario volver a modificar el fichero **/etc/resolv.conf**, ya que lo hicimos con anterioridad. Una vez dentro de la misma, llevaré a cabo una serie de pruebas para verificar que todas las respuestas son correctas, preguntando siempre al servidor DNS **alvaro**, para así verificar que la delegación ha funcionado:

{% highlight shell %}
alvaro@debian:~$ dig @172.22.200.184 www.informatica.iesgn.org
...
;; ANSWER SECTION:
www.informatica.iesgn.org. 86354 IN	A	172.22.200.206
...

alvaro@debian:~$ dig @172.22.200.184 ftp.informatica.iesgn.org
...
;; ANSWER SECTION:
ftp.informatica.iesgn.org. 86400 IN	CNAME	afrodita.informatica.iesgn.org.
afrodita.informatica.iesgn.org.	86400 IN A	172.22.200.106
...

alvaro@debian:~$ dig @172.22.200.184 mx informatica.iesgn.org
...
;; ANSWER SECTION:
informatica.iesgn.org.	86256	IN	MX	10 correo.informatica.iesgn.org.
...
;; ADDITIONAL SECTION:
correo.informatica.iesgn.org. 86393 IN	A	172.22.200.205
...

alvaro@debian:~$ dig +norec @172.22.200.184 informatica.iesgn.org. soa
...
;; AUTHORITY SECTION:
informatica.iesgn.org.	86400	IN	NS	afrodita.informatica.iesgn.org.

;; ADDITIONAL SECTION:
afrodita.informatica.iesgn.org.	86180 IN A	172.22.200.106
...
{% endhighlight %}

Como se puede apreciar en la salida de todas las resoluciones, las respuestas han sido las correctas, por lo que podemos además que el servidor **afrodita** tiene autoridad sobre las zonas **iesgn.org.** e **informatica.iesgn.org.**, mientras que el servidor **alvaro** únicamente tiene autoridad sobre la zona **iesgn.org.**, pues la autoridad de **informatica.iesgn.org.** ha sido delegada a otro servidor DNS.