---
layout: post
title:  "Instalación de un proxy squid"
banner: "/assets/images/banners/squid.jpg"
date:   2021-02-28 15:15:00 +0200
categories: servicios
---
Instalamos **squid**:

{% highlight shell %}
root@proxy:~# apt update && apt upgrade && apt install squid
{% endhighlight %}

Modificamos la configuración del proxy, en la que establecemos las redes y puertos que queremos permitir, así como el puerto de funcionamiento:

{% highlight shell %}
root@proxy:~# nano /etc/squid/squid.conf

acl localnet src 10.0.0.0/24
acl localnet src 192.168.200.0/24

acl SSL_ports port 443
acl Safe_ports port 80
acl Safe_ports port 443
acl Safe_ports port 21
acl CONNECT method CONNECT

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

http_access allow localnet
http_access allow localhost

http_access deny all

http_port 3128

coredump_dir /var/spool/squid

root@proxy:~# systemctl restart squid
{% endhighlight %}

Accedemos a la configuración de proxy del navegador y lo configuramos de manera que haga uso de la máquina servidora, para así poder controlar el tráfico desde la misma:

![squid1](https://i.ibb.co/zNkCpxf/Captura-de-pantalla-de-2021-02-26-11-38-32.png "Configuración navegador")

Tratamos de acceder a cualquier página para verificar que el proxy está funcionando y es accesible desde la máquina anfitriona:

![squid2](https://i.ibb.co/RHZQQcF/Captura-de-pantalla-de-2021-02-26-11-48-06.png "Prueba 1")

Comprobamos los logs en la máquina squid para así verificar que dicha conexión ha dejado "rastro":

{% highlight shell %}
root@proxy:~# cat /var/log/squid/access.log
1614336540.269  57635 192.168.200.1 TCP_TUNNEL/200 102191 CONNECT translate.googleapis.com:443 - HIER_DIRECT/216.58.215.138 -
1614336540.269  58196 192.168.200.1 TCP_TUNNEL/200 4433 CONNECT cdn.jsdelivr.net:443 - HIER_DIRECT/104.16.86.20 -
1614336540.269  58189 192.168.200.1 TCP_TUNNEL/200 7537 CONNECT translate.google.com:443 - HIER_DIRECT/216.58.209.78 -
1614336540.269  58023 192.168.200.1 TCP_TUNNEL/200 5801 CONNECT www.countryflags.io:443 - HIER_DIRECT/172.67.135.78 -
1614336540.271  58201 192.168.200.1 TCP_TUNNEL/200 3407 CONNECT cdnjs.cloudflare.com:443 - HIER_DIRECT/104.16.19.94 -
1614336540.271  60721 192.168.200.1 TCP_TUNNEL/200 705030 CONNECT www.alvarovf.com:443 - HIER_DIRECT/151.101.133.0 -
1614336540.922    128 192.168.200.1 TCP_MISS/200 818 POST http://ocsp.pki.goog/gts1o1core - HIER_DIRECT/216.58.211.227 application/ocsp-response
1614336540.929    130 192.168.200.1 TCP_MISS/200 818 POST http://ocsp.pki.goog/gts1o1core - HIER_DIRECT/216.58.211.227 application/ocsp-response
1614336540.946    194 192.168.200.1 TCP_TUNNEL/200 4692 CONNECT www.gstatic.com:443 - HIER_DIRECT/142.250.185.3 -
1614336540.957    134 192.168.200.1 TCP_MISS/200 818 POST http://ocsp.pki.goog/gts1o1core - HIER_DIRECT/216.58.211.227 application/ocsp-response
1614336540.975    200 192.168.200.1 TCP_TUNNEL/200 5630 CONNECT www.gstatic.com:443 - HIER_DIRECT/142.250.185.3 -
{% endhighlight %}

Tras ello, revertimos los cambios en el navegador para que vuelva a hacer uso del proxy según la configuración del sistema, para ahora proceder a configurar dicho proxy de forma general en el sistema, quedando de la siguiente forma:

![squid3](https://i.ibb.co/s1cmYvW/Captura-de-pantalla-de-2021-02-26-11-51-08.png "Configuración sistema")

Volvemos a acceder a otra página para verificar que la configuración del proxy en el sistema funciona:

![squid4](https://i.ibb.co/tX7ZrFW/Captura-de-pantalla-de-2021-02-26-11-51-24.png "Configuración sistema")

Comprobamos los logs en la máquina squid para así verificar que dicha conexión ha dejado "rastro":

{% highlight shell %}
root@proxy:~# cat /var/log/squid/access.log
1614336680.067    134 192.168.200.1 TCP_MISS/301 643 GET http://fp.josedomingo.org/ - HIER_DIRECT/37.187.119.60 text/html
1614336680.228     42 192.168.200.1 TCP_MISS/200 981 POST http://r3.o.lencr.org/ - HIER_DIRECT/212.230.153.18 application/ocsp-response
1614336685.463      0 192.168.200.1 NONE/000 0 NONE error:transaction-end-before-headers - HIER_NONE/- -
1614336686.465   6104 192.168.200.1 TCP_TUNNEL/200 1787 CONNECT fp.josedomingo.org:443 - HIER_DIRECT/37.187.119.60 -
1614336686.465   6104 192.168.200.1 TCP_TUNNEL/200 41333 CONNECT fp.josedomingo.org:443 - HIER_DIRECT/37.187.119.60 -
1614336686.465   6105 192.168.200.1 TCP_TUNNEL/200 9339 CONNECT fp.josedomingo.org:443 - HIER_DIRECT/37.187.119.60 -
1614336686.465   6106 192.168.200.1 TCP_TUNNEL/200 43692 CONNECT fp.josedomingo.org:443 - HIER_DIRECT/37.187.119.60 -
1614336686.466   6392 192.168.200.1 TCP_TUNNEL/200 26863 CONNECT fp.josedomingo.org:443 - HIER_DIRECT/37.187.119.60 -
{% endhighlight %}

Desde una máquina conectada a una red interna (sin salida al exterior mediante NAT), vamos a realizar determinadas pruebas, no sin antes establecer la correspondiente variable de entorno para que la máquina utilice el servidor proxy en cuestión:

{% highlight shell %}
root@buster:~# export http_proxy='http://10.0.0.10:3128'
{% endhighlight %}

Verificamos que tenemos salida al exterior a través de dicho proxy descargando el index.html de google:

{% highlight shell %}
root@buster:~# wget www.google.es
--2021-02-26 11:08:45--  http://www.google.es/
Connecting to 10.0.0.10:3128... connected.
Proxy request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘index.html’

index.html                                               [ <=>                                                                                                                  ]  14.33K  --.-KB/s    in 0s      

2021-02-26 11:08:45 (63.8 MB/s) - ‘index.html’ saved [14675]
{% endhighlight %}

Pero sin embargo, no podemos hacerle ping:

{% highlight shell %}
root@buster:~# ping www.google.es
PING www.google.es (216.58.211.227) 56(84) bytes of data.
^C
--- www.google.es ping statistics ---
7 packets transmitted, 0 received, 100% packet loss, time 253ms
{% endhighlight %}

Vamos a añadir una lista negra a squid para evitar el tráfico hacia dichas páginas, para ello modificamos la configuración del servicio, quedando de la siguiente forma:

{% highlight shell %}
root@proxy:~# nano /etc/squid/squid.conf

acl localnet src 10.0.0.0/24
acl localnet src 192.168.200.0/24

acl SSL_ports port 443
acl Safe_ports port 80
acl Safe_ports port 443
acl Safe_ports port 21
acl CONNECT method CONNECT

acl domain_blacklist dstdomain "/etc/squid/domain_blacklist.txt"

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

http_access deny domain_blacklist

http_access allow localnet
http_access allow localhost

http_access deny all

http_port 3128

coredump_dir /var/spool/squid
{% endhighlight %}

Dentro de dicha blacklist ponemos un sitio de ejemplo, como twitter.com y todos sus subdominios:

{% highlight shell %}
root@proxy:~# nano /etc/squid/domain_blacklist.txt

.twitter.com

root@proxy:~# systemctl restart squid
{% endhighlight %}

Si tratamos de acceder ahora a twitter.com ocurrirá lo siguiente:

![squid4](https://i.ibb.co/g38ZzTT/Captura-de-pantalla-de-2021-02-26-12-16-30.png "Prueba 2")

En los logs, podremos apreciar lo siguiente:

{% highlight shell %}
root@proxy:~# cat /var/log/squid/access.log

1614338187.184      0 192.168.200.1 TCP_DENIED/403 3953 CONNECT twitter.com:443 - HIER_NONE/- text/html
{% endhighlight %}

Ahora, en lugar de configurar una blacklist, vamos a configurar una whitelist (solo se permiten las páginas indicadas explícitamente), quedando el fichero de configuración de la siguiente forma:

{% highlight shell %}
root@proxy:~# nano /etc/squid/squid.conf

acl localnet src 10.0.0.0/24
acl localnet src 192.168.200.0/24

acl SSL_ports port 443
acl Safe_ports port 80
acl Safe_ports port 443
acl Safe_ports port 21
acl CONNECT method CONNECT

acl domain_whitelist dstdomain "/etc/squid/domain_whitelist.txt"

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

http_access deny !domain_whitelist

http_access allow localnet
http_access allow localhost

http_access deny all

http_port 3128

coredump_dir /var/spool/squid
{% endhighlight %}

Dentro de dicha whitelist ponemos un sitio de ejemplo, como alvarovf.com y todos sus subdominios:

{% highlight shell %}
root@proxy:~# nano /etc/squid/domain_whitelist.txt

.alvarovf.com

root@proxy:~# systemctl restart squid
{% endhighlight %}

Si tratamos de acceder ahora a google.com ocurrirá lo siguiente:

![squid5](https://i.ibb.co/dPBP242/Captura-de-pantalla-de-2021-02-26-12-23-26.png "Prueba 3")

Sin embargo, en alvarovf.com:

![squid6](https://i.ibb.co/DtQXqTr/Captura-de-pantalla-de-2021-02-26-12-24-46.png "Prueba 4")

En los logs, podremos apreciar lo siguiente:

{% highlight shell %}
root@proxy:~# cat /var/log/squid/access.log 
1614338729.105      0 192.168.200.1 TCP_DENIED/403 3962 CONNECT www.google.com:443 - HIER_NONE/- text/html
1614338754.926  56656 192.168.200.1 TCP_TUNNEL/200 702338 CONNECT www.alvarovf.com:443 - HIER_DIRECT/151.101.133.0 -
1614338755.060      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT cdnjs.cloudflare.com:443 - HIER_NONE/- text/html
1614338755.061      0 192.168.200.1 TCP_DENIED/403 3968 CONNECT cdn.jsdelivr.net:443 - HIER_NONE/- text/html
1614338755.061      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT cdnjs.cloudflare.com:443 - HIER_NONE/- text/html
1614338755.061      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT cdnjs.cloudflare.com:443 - HIER_NONE/- text/html
1614338755.061      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT cdnjs.cloudflare.com:443 - HIER_NONE/- text/html
1614338755.062      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.063      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.064      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.064      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.065      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.066      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT translate.google.com:443 - HIER_NONE/- text/html
1614338755.069      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.069      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.086      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.087      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.088      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.089      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.090      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.090      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.090      0 192.168.200.1 TCP_DENIED/403 3977 CONNECT www.countryflags.io:443 - HIER_NONE/- text/html
1614338755.091      0 192.168.200.1 TCP_DENIED/403 3980 CONNECT translate.google.com:443 - HIER_NONE/- text/html
{% endhighlight %}