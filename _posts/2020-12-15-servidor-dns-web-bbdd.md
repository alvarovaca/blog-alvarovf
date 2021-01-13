---
layout: post
title:  "Servidor DNS, Web y Base de Datos"
banner: "/assets/images/banners/mapear.jpg"
date:   2020-12-15 11:44:00 +0200
categories: servicios openstack
---
Se recomienda realizar una previa lectura del _post_ [Modificación del escenario OpenStack](https://www.alvarovf.com/hlc/openstack/2020/12/12/modificacion-escenario-openstack.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una instalación de un servidor DNS sobre la máquina **Freston** anteriormente creada, un servidor web sobre la máquina **Quijote** y un servidor de base de datos sobre la máquina **Sancho**, máquinas que se encuentran conectadas a las redes interna (**10.0.1.0/24**) en el caso de **Freston** y **Sancho** y a la red DMZ (**10.0.2.0/24**) en el caso de **Quijote**.

## Servidor DNS

Se recomienda realizar una previa lectura del _post_ [Instalación de un servidor DNS](https://www.alvarovf.com/servicios/2020/12/11/instalacion-servidor-dns.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

Hasta ahora hemos hecho uso de un mecanismo de resolución estática de nombres en las máquinas existentes en el escenario, que puede llegar a ser tedioso de gestionar conforme van aumentando las máquinas y los servicios existentes en la red, por lo que ha llegado la hora de solventarlo haciendo para ello uso de un servidor DNS **bind9** ubicado en **Freston**.

Además, dicho servidor DNS será accesible desde el exterior (desde el instituto), ya que vamos a implementar una regla DNAT en **Dulcinea** para permitirlo, contando además con un subdominio delegado dentro del dominio principal **gonzalonazareno.org**, que tendrá, en este caso, el nombre **alvaro.gonzalonazareno.org**. Lo veremos con más detalle a continuación.

Todo está listo para llevar a cabo la instalación de **bind9**, no sin antes actualizar toda la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# apt update && apt upgrade && apt install bind9
{% endhighlight %}

Una vez instalado, tendremos que proceder con la configuración inicial del servicio, que consta de varios archivos que se incluyen desde el archivo de configuración principal, de nombre **named.conf**. Los nombres de dichos archivos comienzan por **_named_** en relación al nombre del proceso que BIND ejecuta (abreviatura de "_domain name daemon_").

El primer fichero de configuración que modificaremos será **/etc/bind/named.conf.options**, en el que estableceremos, mediante directivas, las opciones del servicio, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# nano /etc/bind/named.conf.options
{% endhighlight %}

Dentro del mismo, encontraremos una directiva principal de nombre **options**, en la cuál debemos introducir una serie de líneas:

{% highlight shell %}
recursion yes; #Permitimos que el servidor actúe como servidor DNS recursivo.
allow-recursion { any; }; #Permitimos que las peticiones de todos los clientes puedan ser recursivas.
listen-on { any; }; #Permitimos la escucha del servicio en todas las interfaces de la máquina.
allow-transfer { none; }; #Deshabilitamos la transferencia de zonas por defecto. Lo trataremos con más detalle a continuación.
{% endhighlight %}

Tras ello, guardaremos los cambios en el fichero y procederemos a modificar ahora el fichero **/etc/bind/named.conf.local**, en el que declararemos las **vistas** existentes y las zonas sobre las que tendrá autoridad, así como el fichero en el que están contenidas.

Si nos fijamos, acabo de hacer uso de un término que nunca antes habíamos utilizado, las **vistas**. Existen determinadas circunstancias en las que nos puede interesar que un mismo nombre que resuelve nuestro DNS devuelva direcciones IP distintas según en que red este conectada el cliente que realiza la consulta. Esto se entiende mucho mejor con un ejemplo gráfico:

![escenario1](https://i.ibb.co/cc0NmrM/Captura.jpg "Escenario OpenStack")

Como se puede apreciar en la topología actualmente existente en el escenario de OpenStack, disponemos de un total de 3 redes que pueden consultar de forma potencial al servidor DNS ubicado en **Freston** (ya que descartamos las consultas procedentes de la red **10.0.0.0/24**, al ser totalmente ajena al escenario).

Por ese mismo motivo, existen determinadas consultas que deberían ser resueltas a diferentes direcciones dependiendo de la procedencia del cliente, pues por ejemplo, desde el exterior no son accesibles las máquinas de la red DMZ o interna, ya que la única expuesta directamente al exterior es **Dulcinea**. Las resoluciones que deberían existir desde cada una de las redes son:

- **red interna de alvaro.vaca (interna)**:
    - **Dulcinea**: 10.0.1.9
    - **Sancho**: 10.0.1.4
    - **Quijote**: 10.0.2.6
    - **Freston**: 10.0.1.7
- **DMZ de alvaro.vaca (dmz)**:
    - **Dulcinea**: 10.0.2.12
    - **Sancho**: 10.0.1.4
    - **Quijote**: 10.0.2.6
    - **Freston**: 10.0.1.7
- **ext-net (externa)**:
    - **Dulcinea**: 172.22.200.134

Además de la zona **alvaro.gonzalonazareno.org**, vamos a configurar otras dos de resolución inversa (para las redes **10.0.1.0/24** y **10.0.2.0/24**), que serán únicamente visibles desde las redes interna y DMZ, ya que no tiene sentido resolver inversamente direcciones privadas desde el exterior.

Por último, antes de empezar a configurar, es importante pensar desde qué redes de origen se va a hacer de una vista u otra, quedando de la siguiente forma:

- **interna**:
    - **10.0.1.0/24**: Aquellas máquinas existentes en la **red interna de alvaro.vaca**.
    - **localhost**: Para que Freston, que se encuentra ubicado en la red anteriormente mencionada, también pueda hacer uso del servidor DNS.
- **dmz**:
    - **10.0.2.0/24**: Aquellas máquinas existentes en la **DMZ de alvaro.vaca**.
- **externa**:
    - **172.22.0.0/15**: Aquellas máquinas existentes en el instituto (**172.22.0.0/16**) y que accedan desde la VPN (**172.23.0.0/16**).
    - **192.168.202.2**: Para que el servidor DNS existente en el instituto pueda consultar al nuestro, necesario para la delegación de subdominio.

Como anteriormente hemos mencionado, vamos a proceder a modificar ahora el fichero **/etc/bind/named.conf.local**, en el que declararemos las vistas existentes y las zonas sobre las que tendrá autoridad, así como el fichero en el que están contenidas, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano /etc/bind/named.conf.local
{% endhighlight %}

El resultado final del fichero sería:

{% highlight shell %}
view interna {
        match-clients { 10.0.1.0/24; localhost; };

        zone "alvaro.gonzalonazareno.org" {
                type master;
                file "db.alvaro.interna";
        };

        zone "1.0.10.in-addr.arpa" {
                type master;
                file "db.1.0.10";
        };

        zone "2.0.10.in-addr.arpa" {
                type master;
                file "db.2.0.10";
        };

        include "/etc/bind/zones.rfc1918";
        include "/etc/bind/named.conf.default-zones";
};

view dmz {
        match-clients { 10.0.2.0/24; };

        zone "alvaro.gonzalonazareno.org" {
                type master;
                file "db.alvaro.dmz";
        };

        zone "1.0.10.in-addr.arpa" {
                type master;
                file "db.1.0.10";
        };

        zone "2.0.10.in-addr.arpa" {
                type master;
                file "db.2.0.10";
        };

        include "/etc/bind/zones.rfc1918";
        include "/etc/bind/named.conf.default-zones";
};

view externa {
        match-clients { 172.22.0.0/15; 192.168.202.2; };

        zone "alvaro.gonzalonazareno.org" {
                type master;
                file "db.alvaro.externa";
        };

        include "/etc/bind/zones.rfc1918";
        include "/etc/bind/named.conf.default-zones";
};
{% endhighlight %}

Finalmente, y a modo de resumen, voy a aclarar dónde se encuentra contenida cada una de las tres zonas y desde dónde es accesible:

- **alvaro.gonzalonazareno.org**:
    - **db.alvaro.interna**: Vista interna (_10.0.1.0/24_ y _localhost_).
    - **db.alvaro.dmz**: Vista dmz (_10.0.2.0/24_).
    - **db.alvaro.externa**: Vista externa (_172.22.0.0/15_ y _192.168.202.2_).
- **1.0.10.in-addr.arpa**:
    - **db.1.0.10**: Vista interna (_10.0.1.0/24_ y _localhost_) y Vista dmz (_10.0.2.0/24_).
- **2.0.10.in-addr.arpa**:
    - **db.2.0.10**: Vista interna (_10.0.1.0/24_ y _localhost_) y Vista dmz (_10.0.2.0/24_).

Un requisito indispensable a la hora de utilizar vistas es que debemos definir todas las zonas alcanzables desde la misma dentro de cada una de ellas, es por ello que hemos añadido un **include** de los ficheros **/etc/bind/zones.rfc1918** y **/etc/bind/named.conf.default-zones**.

Por consecuencia, es importante comentar la línea referente al **include** del fichero **/etc/bind/named.conf.default-zones** en el fichero **/etc/bind/named.conf**, por lo que para modificarlo, haremos uso del comando:

{% highlight shell %}
root@freston:~# nano /etc/bind/named.conf
{% endhighlight %}

Dentro del mismo, comentaremos la tercera de las líneas, quedando de la siguiente forma:

{% highlight shell %}
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
//include "/etc/bind/named.conf.default-zones";
{% endhighlight %}

Cuando hayamos guardado los cambios en el mismo, habrá llegado la hora de definir el contenido de las tres zonas DNS con las que vamos a trabajar. Para una mayor facilidad, podemos hacer uso de una plantilla de nombre **/etc/bind/db.empty** para a partir de ahí, adaptarla a nuestras necesidades.

Vamos a comenzar con la configuración del fichero de la zona de resolución directa accesible desde la vista interna, así que copiaremos dicho fichero dentro de **/var/cache/bind/**, pues es donde se deben ubicar por defecto, con el nombre previamente asignado, **db.alvaro.interna**, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# cp /etc/bind/db.empty /var/cache/bind/db.alvaro.interna
{% endhighlight %}

El primer fichero de la zona DNS (vista interna) ya ha sido creado, aunque actualmente está vacío, de manera que vamos a proceder a modificarlo para añadir los registros correspondientes a la misma, ejecutando para ello los comandos:

{% highlight shell %}
root@freston:~# nano /var/cache/bind/db.alvaro.interna
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

Para el correcto funcionamiento será necesario llevar a cabo al menos tres modificaciones sobre la configuración presentada en la plantilla:

- **SOA (Start of Authority)**: Es una de las directivas más importantes, en la que indicaremos el _FQDN_ del servidor con autoridad maestro (es decir, el servidor con el que estamos trabajando, **freston.alvaro.gonzalonazareno.org.**), además del correo electrónico de la persona responsable de la zona DNS (por ejemplo, **admin.alvaro.gonzalonazareno.org.**).
- **Serial**: Identificador de la zona, que deberá ser incrementado después de cada cambio. Se recomienda que tenga la forma **YYMMDDNN**, por ejemplo, la primera modificación del día _15 de Diciembre de 2020_ sería **20121501**.
- **NS (Name Server)**: Tendremos que indicar el FQDN del servidor con autoridad sobre la zona, que en este caso es el servidor con el que estamos trabajando, **freston.alvaro.gonzalonazareno.org.**.

Esta sería la configuración base mínima para configurar la zona, de manera que a partir de ahora, tendremos que ir añadiendo registros para nombrar las máquinas y servicios existentes en dicha zona. A partir de aquí, la configuración se vuelve algo más "personal", en el sentido de que puede variar según las necesidades de una persona u otra.

Para una mayor comodidad, estableceremos un registro **$ORIGIN alvaro.gonzalonazareno.org.**, indicando que todos los registros definidos a partir de ahí serán autocompletados con el dominio especificado, evitándonos así tener que escribir todo el rato el nombre de dominio para escribir el _FQDN_ de las máquinas o servicios. En mi caso, además de las 4 máquinas, he añadido un servicio web (**www**) y un servicio de base de datos (**bd**), para así evitar tener que hacerlo posteriormente, quedando de la siguiente forma:

{% highlight shell %}
$TTL    86400
@       IN      SOA     freston.alvaro.gonzalonazareno.org. admin.alvaro.gonzalonazareno.org. (
                       20121501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      freston.alvaro.gonzalonazareno.org.

$ORIGIN alvaro.gonzalonazareno.org.

dulcinea        IN      A       10.0.1.9
sancho  IN      A       10.0.1.4
quijote IN      A       10.0.2.6
freston IN      A       10.0.1.7
www     IN      CNAME   quijote
bd      IN      CNAME   sancho
{% endhighlight %}

La definición del primer fichero de la zona de resolución directa ha finalizado, de manera que para facilitar el trabajo, vamos a utilizar dicho fichero como plantilla base y a partir de ahí, lo adaptaremos a nuestras necesidades para los otros dos. Para copiar dicho fichero haremos uso del comando:

{% highlight shell %}
root@freston:~# cp /var/cache/bind/db.alvaro.interna /var/cache/bind/db.alvaro.dmz
{% endhighlight %}

El segundo fichero de la zona DNS (vista dmz) ya ha sido creado, de manera que vamos a proceder a modificarlo para adaptarlo a nuestras necesidades, ejecutando para ello los comandos:

{% highlight shell %}
root@freston:~# nano /var/cache/bind/db.alvaro.dmz
{% endhighlight %}

La única modificación que debemos llevar a cabo es referente a la dirección IP de **Dulcinea**, que pasa de ser **10.0.1.9** a ser **10.0.2.12**, quedando de la siguiente forma:

{% highlight shell %}
$TTL    86400
@       IN      SOA     freston.alvaro.gonzalonazareno.org. admin.alvaro.gonzalonazareno.org. (
                       20121501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      freston.alvaro.gonzalonazareno.org.

$ORIGIN alvaro.gonzalonazareno.org.

dulcinea        IN      A       10.0.2.12
sancho  IN      A       10.0.1.4
quijote IN      A       10.0.2.6
freston IN      A       10.0.1.7
www     IN      CNAME   quijote
bd      IN      CNAME   sancho
{% endhighlight %}

La definición del segundo fichero de la zona de resolución directa ha finalizado, así que volveremos a utilizar el primer fichero como plantilla base y lo adaptaremos a nuestras necesidades para el último de ellos. Para copiar dicho fichero haremos uso del comando:

{% highlight shell %}
root@freston:~# cp /var/cache/bind/db.alvaro.interna /var/cache/bind/db.alvaro.externa
{% endhighlight %}

El tercer fichero de la zona DNS (vista externa) ya ha sido creado, de manera que vamos a proceder a modificarlo para adaptarlo a nuestras necesidades, ejecutando para ello los comandos:

{% highlight shell %}
root@freston:~# nano /var/cache/bind/db.alvaro.externa
{% endhighlight %}

Las modificaciones que debemos llevar a cabo consisten en:

* Eliminar todas las máquinas que no sean accesibles de forma directa desde el exterior, es decir, todas excepto **Dulcinea**.
* Cambiar la dirección IP de **Dulcinea**, que pasa de ser **10.0.1.9** a ser **172.22.200.134**.
* Eliminar el nombre **bd**, ya que el servidor de base de datos no va a ser accesible desde el exterior.
* El CNAME del servicio **www** pasa a apuntar ahora a **Dulcinea**, que tendrá una regla DNAT para alcanzar a **Quijote**, que es donde estará alojado el servidor web.

El resultado final sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     dulcinea.alvaro.gonzalonazareno.org. admin.alvaro.gonzalonazareno.org. (
                       20121501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      dulcinea.alvaro.gonzalonazareno.org.

$ORIGIN alvaro.gonzalonazareno.org.

dulcinea        IN      A       172.22.200.134
www     IN      CNAME   dulcinea
{% endhighlight %}

La definición de la zona de resolución directa para el subdominio **alvaro.gonzalonazareno.org** ha finalizado, así que volveremos a utilizar el primer fichero como plantilla base y lo adaptaremos a nuestras necesidades para las zonas de resolución inversa. Para copiar dicho fichero haremos uso del comando:

{% highlight shell %}
root@freston:~# cp /var/cache/bind/db.alvaro.interna /var/cache/bind/db.1.0.10
{% endhighlight %}

El fichero de la primera zona inversa ya ha sido creado, de manera que vamos a proceder a modificarlo para adaptarlo a nuestras necesidades, ejecutando para ello los comandos:

{% highlight shell %}
root@freston:~# nano /var/cache/bind/db.1.0.10
{% endhighlight %}

Las modificaciones que debemos llevar a cabo se resumen en modificar el valor del registro **$ORIGIN** a **1.0.10.in-addr.arpa.**, que será utilizado para autocompletar los registros que introduzcamos y en utilizar registros **PTR** para nombrar las máquinas, en lugar de registros **A**. No se nombrarán los servicios. El resultado final sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     freston.alvaro.gonzalonazareno.org. admin.alvaro.gonzalonazareno.org. (
                       20121501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      freston.alvaro.gonzalonazareno.org.

$ORIGIN 1.0.10.in-addr.arpa.

9       IN      PTR     dulcinea
4       IN      PTR     sancho
7       IN      PTR     freston
{% endhighlight %}

La definición de la zona de resolución inversa para la red **10.0.1.0/24** ha finalizado, así que vamos a utilizar dicho fichero como plantilla base y lo adaptaremos a nuestras necesidades para la segunda y última zona de resolución inversa. Para copiar dicho fichero haremos uso del comando:

{% highlight shell %}
root@freston:~# cp /var/cache/bind/db.1.0.10 /var/cache/bind/db.2.0.10
{% endhighlight %}

El fichero de la segunda zona inversa ya ha sido creado, de manera que vamos a proceder a modificarlo para adaptarlo a nuestras necesidades, ejecutando para ello los comandos:

{% highlight shell %}
root@freston:~# nano /var/cache/bind/db.2.0.10
{% endhighlight %}

La modificaciones que debemos llevar a cabo se resumen en modificar el valor del registro **$ORIGIN** a **2.0.10.in-addr.arpa.**, que será utilizado para autocompletar los registros que introduzcamos y el nombrar con registros **PTR** las máquinas existentes en dicha red. El resultado final sería:

{% highlight shell %}
$TTL    86400
@       IN      SOA     freston.alvaro.gonzalonazareno.org. admin.alvaro.gonzalonazareno.org. (
                       20121501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      freston.alvaro.gonzalonazareno.org.

$ORIGIN 2.0.10.in-addr.arpa.

12      IN      PTR     dulcinea
6       IN      PTR     quijote
{% endhighlight %}

Una vez finalizada la definición de la zona DNS, vamos a proceder a comprobar que la sintaxis de los ficheros de configuración **named** sea correcta, antes proceder a cargar la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# named-checkconf
{% endhighlight %}

La ejecución del comando no ha devuelto ninguna salida, por lo que podemos concluir que la sintaxis de los ficheros de configuración es correcta, por lo que podremos proceder a reiniciar el servicio para que así cargue la nueva configuración realizada, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# systemctl restart bind9
{% endhighlight %}

El servicio ya se encuentra en ejecución con la nueva configuración cargada, sin embargo, la vista externa todavía no funciona como debería, ya que la máquina **Freston** no es accesible desde el exterior, sino que tendremos que añadir una regla DNAT a **Dulcinea**, que va a ser la que reciba las peticiones y las reenvíe al puerto 53 UDP de dicha máquina.

Para ello, tendremos que crear la cadena **PREROUTING** dentro de la tabla **nat**, es decir, aquella que permite modificar paquetes entrantes antes de que se tome una decisión de enrutamiento, permitiendo por tanto hacer **Destination NAT** (DNAT) para posteriormente alojar dentro de la misma, la regla que necesitamos. Para crear dicha cadena ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nft add chain nat prerouting { type nat hook prerouting priority 0 \; }
{% endhighlight %}

En este caso, hemos especificado que esta cadena sea muy prioritaria (a menor sea el número, mayor es la prioridad), de manera que las reglas albergadas en el interior de la cadena **PREROUTING** se ejecutarían antes que las de la cadena **POSTROUTING**. Vamos a verificar que dicha cadena se ha generado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# nft list chains
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
    chain prerouting {
            type nat hook prerouting priority 0; policy accept;
    }
}
{% endhighlight %}

Efectivamente, así ha sido. Nos queda un último paso, añadir la regla necesaria a la cadena **PREROUTING**, de manera que nos permita hacer _DNAT_ a la dirección de la máquina que actúa como servidor DNS, en este caso, la **10.0.1.7**. Además, añadiremos reglas para los puertos 80 y 443 TCP, ya que posteriormente nos será necesario hacer DNAT a **10.0.1.4** para el servidor web. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nft add rule ip nat prerouting iifname "eth0" udp dport 53 counter dnat to 10.0.1.7
root@dulcinea:~# nft add rule ip nat prerouting iifname "eth0" tcp dport 80 counter dnat to 10.0.2.6
root@dulcinea:~# nft add rule ip nat prerouting iifname "eth0" tcp dport 443 counter dnat to 10.0.2.6
{% endhighlight %}

En este caso, hemos especificado que se trata de reglas para la cadena **prerouting** (pues deben aplicarse justo antes de que se tome una decisión de enrutamiento), además, la interfaz (_iifname_) que debemos introducir será aquella por la que van a entrar los paquetes, es decir, la que está conectada a Internet, en este caso, **eth0**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que entren por los puertos:

* 53 UDP (_udp dport_).
* 80 TCP (_tcp dport_).
* 443 TCP (_tcp dport_).

Por último, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a hacer _DNAT_ a la dirección IP de la máquina en cuestión (_dnat to_).

Posteriormente, vamos a verificar que dichas reglas se han añadido correctamente, ejecutando para ello el comando:

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
            oifname "eth0" ip saddr 10.0.1.0/24 counter packets 521 bytes 36711 snat to 10.0.0.7
            oifname "eth0" ip saddr 10.0.2.0/24 counter packets 257 bytes 17146 snat to 10.0.0.7
    }

    chain prerouting {
            type nat hook prerouting priority 0; policy accept;
            iifname "eth0" udp dport domain counter packets 0 bytes 0 dnat to 10.0.1.7
            iifname "eth0" tcp dport http counter packets 0 bytes 0 dnat to 10.0.2.6
            iifname "eth0" tcp dport https counter packets 0 bytes 0 dnat to 10.0.2.6
    }
}
{% endhighlight %}

Efectivamente, así ha sido. Esta configuración se encuentra cargada en memoria, por lo que para conseguir que perdure en el tiempo, vamos a ejecutar el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Gracias a ello, habremos guardado la configuración en un fichero en **/etc/** de nombre **nftables.conf** que se importará de forma automática cuando reiniciemos la máquina gracias al _daemon_ que se encuentra habilitado. En caso de que no cargase de nuevo la configuración, podríamos hacerlo manualmente con el comando `nft -f /etc/nftables.conf`.

En mi caso, he decidido llevar a cabo algunas consultas al servidor DNS desde las diferentes máquinas pertenecientes al escenario, para así verificar que toda la configuración llevada a cabo hasta este momento, funciona como debería:

* [Pruebas desde Dulcinea](https://pastebin.com/5jhFiazt)
* [Pruebas desde Sancho](https://pastebin.com/F9WKmPG4)
* [Pruebas desde Quijote](https://pastebin.com/PvgcEE3h)
* [Pruebas desde Freston](https://pastebin.com/mFSYbvG0)
* [Pruebas desde el exterior](https://pastebin.com/PckKWH5q)

Dado que ya hemos configurado un servidor DNS, tendremos que eliminar la resolución estática de nombres en las máquinas, de manera que se conozcan entre ellas pero siempre haciendo uso del servidor DNS. Dicha información está almacenada en el fichero **/etc/hosts**, así que procederemos a realizar dicha modificación en las cuatro máquinas, quedando de la siguiente forma::

{% highlight shell %}
root@dulcinea:~# nano /etc/hosts
127.0.1.1 dulcinea.alvaro.gonzalonazareno.org dulcinea.novalocal dulcinea
127.0.0.1 localhost

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@sancho:~# nano /etc/hosts
127.0.1.1 sancho.alvaro.gonzalonazareno.org sancho
127.0.0.1 localhost

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

[root@quijote ~]# vi /etc/hosts
127.0.1.1 quijote.alvaro.gonzalonazareno.org quijote
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

root@freston:~# nano /etc/hosts
127.0.1.1 freston.alvaro.gonzalonazareno.org freston.novalocal freston
127.0.0.1 localhost

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
{% endhighlight %}

La resolución estática ha sido eliminada de todas las máquinas, de manera que actualmente no se conocen entre ellas. Para solucionarlo, tendremos que establecer en todas las máquinas el nuevo servidor DNS ubicado en **Freston**, para que así se haga uso del mismo de manera prioritaria a la hora de llevar a cabo una resolución. El procedimiento es diferente dependiendo de la distribución.

### Dulcinea

Las instancias Debian Buster de OpenStack vienen con un fichero **/etc/resolv.conf** dinámico, lo que quiere decir que se genera dinámicamente en cada arranque. El contenido del mismo incluye los dos servidores DNS existentes en el instituto, situados en las direcciones **192.168.202.2** y **192.168.200.2**, pero en este caso, es necesario añadir el servidor DNS ubicado en **Freston** de manera prioritaria. Al ser dinámico dicho fichero, no podemos modificarlo a mano ni tampoco indicar el servidor DNS en el **/etc/network/interfaces**, sino que tendremos que modificar los ficheros de configuración de **resolvconf**, el encargado de generar dicho fichero.

Para ello, modificaremos el fichero **/etc/resolvconf/resolv.conf.d/head** y lo indicaremos ahí, de manera que lo tendrá en cuenta para la siguiente ocasión y lo posicionará el primero en la lista, consiguiendo por tanto que tenga prioridad, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# nano /etc/resolvconf/resolv.conf.d/head
{% endhighlight %}

En este caso, el servidor DNS que necesito añadir es **10.0.1.7**, por lo que la apariencia de dicho fichero sería:

{% highlight shell %}
nameserver 10.0.1.7
{% endhighlight %}

Dado que en el primer artículo configuramos también el servidor **8.8.8.8** en el fichero **/etc/resolvconf/resolv.conf.d/base**, tendremos que eliminarlo para así no generar conflictos, haciendo para ello uso del comando:

{% highlight shell %}
root@dulcinea:~# nano /etc/resolvconf/resolv.conf.d/base
{% endhighlight %}

Dentro del mismo, eliminaremos la línea **nameserver 8.8.8.8** y añadiremos una directiva **search**, que nos permitirá autocompletar las consultas DNS que realicemos. Por ejemplo, cuando hagamos `ping sancho`, autocompletará **sancho** con el nombre de dominio que introduzcamos, resultando ser **sancho.alvaro.gonzalonazareno.org**, y por tanto, evitaremos tener que introducir el nombre al completo cada vez que queramos comunicarnos con otra máquina. La apariencia final de dicho fichero sería:

{% highlight shell %}
search alvaro.gonzalonazareno.org
{% endhighlight %}

Genial, todos los cambios necesarios se han llevado a cabo, pero para que surtan efecto y comprobar que perduran tras un reinicio, ejecutareremos el comando `reboot` y cuando el reinicio finalice, visualizaremos el contenido del fichero **/etc/resolv.conf**, para así verificar que la nueva configuración se ha tenido en cuenta durante la generación dinámica de dicho fichero:

{% highlight shell %}
root@dulcinea:~# cat /etc/resolv.conf
nameserver 10.0.1.7
nameserver 192.168.202.2
nameserver 192.168.200.2
search openstacklocal alvaro.gonzalonazareno.org
{% endhighlight %}

Como era de esperar, el servidor DNS **10.0.1.7** es el primero en la lista, de manera que será el primer servidor al que se envíen las consultas. De otro lado, la directiva **search** también se ha configurado como debería, de manera que cualquier consulta DNS será autocompletada con **alvaro.gonzalonazareno.org**, en caso de ser necesario.

### Sancho

El fichero de configuración en el que debemos establecer la configuración de los servidores DNS en una máquina Ubuntu 20.04 es **/etc/netplan/50-cloud-init.yaml**, así que procederemos a modificarlo, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# nano /etc/netplan/50-cloud-init.yaml
{% endhighlight %}

Dentro del mismo, modificaremos la directiva **addresses**, estableciendo de forma prioritaria el servidor DNS que hemos configurado. Al igual que en el caso anterior, añadiremos también una directiva **search**, que nos permitirá autocompletar las consultas DNS que realicemos. La apariencia final de dicho fichero sería:

{% highlight shell %}
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: false
            match:
                macaddress: fa:16:3e:10:d8:23
            mtu: 8950
            set-name: ens3
            addresses: [10.0.1.4/24]
            gateway4: 10.0.1.9
            nameservers:
                addresses: [10.0.1.7, 192.168.202.2, 192.168.200.2]
                search: ["alvaro.gonzalonazareno.org"]
{% endhighlight %}

Genial, todos los cambios necesarios se han llevado a cabo, pero para que surtan efecto y comprobar que perduran tras un reinicio, ejecutareremos el comando `reboot` y cuando el reinicio finalice, visualizaremos el contenido del fichero **/etc/resolv.conf**, para así verificar que la nueva configuración se ha tenido en cuenta durante la generación dinámica de dicho fichero:

{% highlight shell %}
root@sancho:~# cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search alvaro.gonzalonazareno.org
{% endhighlight %}

En este caso, el nuevo servidor DNS configurado no aparece reflejadoo en dicho fichero ya que Ubuntu implementa por defecto un servidor DNS local, capaz de cachear las respuestas, consiguiendo por tanto una navegación más rápida, pero que internamente, hará uso de los servidores DNS que le hemos especificado anteriormente para dichas peticiones. De otro lado, la directiva **search** se ha configurado como debería, de manera que cualquier consulta DNS será autocompletada con **alvaro.gonzalonazareno.org**, en caso de ser necesario.

### Quijote

El fichero de configuración en el que debemos establecer la configuración de los servidores DNS en una máquina CentOS 8 es **/etc/resolv.conf**, así que procederemos a modificarlo, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/resolv.conf
{% endhighlight %}

Dentro del mismo, estableceremos de forma prioritaria en una directiva **addresses** el servidor DNS que hemos configurado. Al igual que en el caso anterior, modificaremos también la directiva **search**, que nos permitirá autocompletar las consultas DNS que realicemos. La apariencia final de dicho fichero sería:

{% highlight shell %}
search openstacklocal alvaro.gonzalonazareno.org
nameserver 10.0.1.7
nameserver 192.168.202.2
nameserver 192.168.200.2
{% endhighlight %}

Genial, todos los cambios necesarios se han llevado a cabo, pero para que surtan efecto y comprobar que perduran tras un reinicio, ejecutareremos el comando `reboot` y cuando el reinicio finalice, visualizaremos el contenido del fichero **/etc/resolv.conf**, para así verificar que la nueva configuración se ha tenido en cuenta durante la generación dinámica de dicho fichero:

{% highlight shell %}
[root@quijote ~]# cat /etc/resolv.conf
search openstacklocal alvaro.gonzalonazareno.org
nameserver 10.0.1.7
nameserver 192.168.202.2
nameserver 192.168.200.2
{% endhighlight %}

Como era de esperar, el servidor DNS **10.0.1.7** es el primero en la lista, de manera que será el primer servidor al que se envíen las consultas. De otro lado, la directiva **search** también se ha configurado como debería, de manera que cualquier consulta DNS será autocompletada con **alvaro.gonzalonazareno.org**, en caso de ser necesario.

### Freston

Como anteriormente hemos mencionado, las instancias Debian Buster de OpenStack vienen con un fichero **/etc/resolv.conf** dinámico, por lo que no podemos modificarlo a mano ni tampoco indicar el servidor DNS en el **/etc/network/interfaces**, sino que tendremos que modificar los ficheros de configuración de **resolvconf**, el encargado de generar dicho fichero.

Para ello, modificaremos el fichero **/etc/resolvconf/resolv.conf.d/head** y lo indicaremos ahí, de manera que lo tendrá en cuenta para la siguiente ocasión y lo posicionará el primero en la lista, consiguiendo por tanto que tenga prioridad, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano /etc/resolvconf/resolv.conf.d/head
{% endhighlight %}

La apariencia final de dicho fichero sería:

{% highlight shell %}
nameserver 10.0.1.7
{% endhighlight %}

Dado que en el artículo referente a la modificación del escenario configuramos también el servidor **8.8.8.8** en el fichero **/etc/resolvconf/resolv.conf.d/base**, tendremos que eliminarlo para así no generar conflictos, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# nano /etc/resolvconf/resolv.conf.d/base
{% endhighlight %}

Dentro del mismo, eliminaremos la línea **nameserver 8.8.8.8** y añadiremos una directiva **search**, que nos permitirá autocompletar las consultas DNS que realicemos. La apariencia final de dicho fichero sería:

{% highlight shell %}
nameserver 192.168.202.2
search alvaro.gonzalonazareno.org
{% endhighlight %}

Genial, todos los cambios necesarios se han llevado a cabo, pero para que surtan efecto y comprobar que perduran tras un reinicio, ejecutareremos el comando `reboot` y cuando el reinicio finalice, visualizaremos el contenido del fichero **/etc/resolv.conf**, para así verificar que la nueva configuración se ha tenido en cuenta durante la generación dinámica de dicho fichero:

{% highlight shell %}
root@dulcinea:~# cat /etc/resolv.conf
nameserver 10.0.1.7
nameserver 192.168.200.2
nameserver 192.168.202.2
search alvaro.gonzalonazareno.org
{% endhighlight %}

Como era de esperar, el servidor DNS **10.0.1.7** es el primero en la lista, de manera que será el primer servidor al que se envíen las consultas. De otro lado, la directiva **search** también se ha configurado como debería, de manera que cualquier consulta DNS será autocompletada con **alvaro.gonzalonazareno.org**, en caso de ser necesario.

## Servidor Web

Tras ello, vamos a instalar un servidor web en **Quijote** que sea capaz de ejecutar código PHP, para posteriores aplicaciones que necesitemos desplegar en el escenario. Para ello, necesitamos llevar a cabo la instalación de **httpd** (el nombre que recibe Apache2 en esta distribución) y **php-fpm**, el servidor de aplicaciones que nos permitirá ejecutar código PHP, no sin antes actualizar toda la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# dnf update && dnf install httpd php php-fpm
{% endhighlight %}

Genial, los paquetes ya han sido instalados, sin embargo, los servicios no se encuentran actualmente activos ni mucho menos, habilitados para arrancar junto a la máquina. Por ello, tendremos que hacer uso de `systemctl` para arrancarlos y habilitar su arranque durante el inicio de la máquina, ejecutando para ello los comandos:

{% highlight shell %}
[root@quijote ~]# systemctl start httpd php-fpm
[root@quijote ~]# systemctl enable httpd php-fpm
{% endhighlight %}

Es importante mencionar que en **CentOS 8** se utilizaba por defecto un _firewall_ que de primeras es bastante restrictivo, de nombre **firewalld**. En el mismo, viene permitido poco más que el acceso por SSH, de manera que el puerto 80 se encuentra cerrado, bloqueando todas las peticiones entrantes al mismo. Para verificar el estado de dicho cortafuegos, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# firewall-cmd --state
running
{% endhighlight %}

Donde:

* **--state**: Indicamos que compruebe si el _daemon_ **firewalld** se encuentra o no activo.

Efectivamente, el servicio se encuentra activo, por lo que vamos a proceder a listar todas aquellas reglas que se encuentran actualmente configuradas y habilitadas en el cortafuegos, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: dhcpv6-client ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
{% endhighlight %}

Donde:

* **--list-all**: Indicamos que muestre todas las reglas activas para la zona por defecto.

En esta ocasión, únicamente se encuentran permitidas las conexiones entrantes de 2 servicios, entre los que lógicamente, no se encuentra contemplado el puerto 80 (HTTP), por lo que procederemos a añadir dicha regla de forma manual, junto a otra para el puerto 443 (HTTPS), ya que nos será necesario posteriormente, ejecutando para ello los comandos:

{% highlight shell %}
[root@quijote ~]# firewall-cmd --permanent --add-port=80/tcp
success
[root@quijote ~]# firewall-cmd --permanent --add-port=443/tcp
success
{% endhighlight %}

Donde:

* **--permanent**: Indicamos que la regla perdure incluso tras un reinicio. El cambio no surtirá efecto inmediatamente.
* **--add-port**: Indicamos el puerto y el protocolo permitido en dicha regla.

Al parecer, las reglas se han añadido correctamente al cortafuegos de forma permanente, por lo que tal y como he mencionado, los cambios no surtirán efecto inmediatamente, de manera que tendremos que reiniciar el servicio. Para ello podemos hacer uso de `systemctl` o bien de la propia opción que trae incluida `firewall-cmd`:

{% highlight shell %}
[root@quijote ~]# firewall-cmd --reload
success
{% endhighlight %}

Donde:

* **--reload**: Indicamos que reinicie el servicio de cortafuegos, consiguiendo así que las reglas permanentes se carguen en memoria.

Genial, en un principio las nuevas reglas ya han sido añadidas y se encuentran actualmente activas, por lo que volveremos a listar todas las reglas existentes para así verificarlo, ejecutando para ello el comando utilizado con anterioridad:

{% highlight shell %}
[root@quijote ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: dhcpv6-client ssh
  ports: 80/tcp 443/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
{% endhighlight %}

Como era de esperar, las reglas se han añadido correctamente y los puertos **80/TCP** y **443/TCP** se encuentran actualmente abiertos y con posibilidad de recibir peticiones.

Por defecto, **httpd** no crea los directorios **/etc/httpd/sites-available** y **/etc/httpd/sites-enabled**, que contendrán los ficheros de configuración para los VirtualHost que creemos, de manera que tendremos que hacerlo de forma manual, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# mkdir /etc/httpd/sites-enabled /etc/httpd/sites-available
{% endhighlight %}

Cuando los nuevos directorios se encuentren generados, tendremos que modificar el fichero **/etc/httpd/conf/httpd.conf** para indicar la ruta del nuevo directorio que contendrá un enlace simbólico a los ficheros de configuración de los VirtualHost habilitados, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/httpd/conf/httpd.conf
{% endhighlight %}

Dentro del mismo, tendremos que encontrar la siguiente directiva:

{% highlight shell %}
IncludeOptional conf.d/*.conf
{% endhighlight %}

Y sustituir **conf.d** por **sites-enabled**, quedando de la siguiente forma:

{% highlight shell %}
IncludeOptional sites-enabled/*.conf
{% endhighlight %}

Todo está listo para empezar a crear un VirtualHost en nuestro servidor web, pero dado que necesitamos ejecutar código PHP, tendremos que hacer que actúe como un proxy inverso, reenviando las peticiones necesarias a **php-fpm**, nuestro servidor de aplicaciones que interpretará el código PHP y lo transformará en HTML, para que pueda ser servido por _httpd_. Tenemos dos posibilidades para comunicar ambos servicios:

* **Socket UNIX**: Permiten un intercambio de datos eficiente entre procesos que se ejecutan y comunican de manera local (no-remota). Al fin y al cabo, es un tipo de fichero en el que un proceso escribe y el otro lee, actuando como pasarela, al que se le aplican los permisos de ficheros UNIX, de manera que se pueden restringir qué procesos pueden leer y escribir en el mismo. Es más seguro, al no estar expuestos a Internet y únicamente ser accesibles de manera local. Cuentan con un mayor rendimiento.
* **Socket TCP/IP**: Identifica un servidor basado en una dirección IP y un puerto, por ejemplo, cuando alojamos un servidor web y lo hacemos en la dirección **192.168.1.2** y en el puerto **80** y a dicha dirección y puerto se conecta una máquina cliente con dirección **192.168.1.3** a través de un puerto no privilegiado, como el **3412**, entre los que existe una comunicación bidireccional, de manera que suele utilizarse para conexiones remotas. Es más inseguro, al ser accesible por cualquier persona, a no ser que se implemente un firewall o un método de autentificación. Cuentan con menor rendimiento.

En este caso, tras comparar ambas posibilidades, y dadas las circunstancias, haremos uso de un **Socket UNIX**, pues aumentaremos así la seguridad y el rendimiento. En realidad, también podríamos haber alojado un Socket TCP/IP en la dirección local (**127.0.0.1**), pero contaríamos con las desventajas del mismo.

El primer paso será configurar el método de escucha del servicio **php-fpm**, configuración que se llevará a cabo en un archivo del grupo de recursos, pues dicho servicio puede ejecutar múltiples grupos de procesos con diferentes configuraciones, pero en este caso, haremos uso del que viene por defecto, **www.conf**. Para comprobar qué configuración está usando actualmente, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# cat /etc/php-fpm.d/www.conf | egrep 'listen ='
listen = /run/php-fpm/www.sock
{% endhighlight %}

Como se puede apreciar, hemos leído el contenido de dicho fichero, filtrando por una cadena que se usa para establecer el método de escucha del servicio. En este caso, estamos usando por defecto un **Socket UNIX** alojado en **/run/php-fpm/www.sock**, por lo que no tendremos que llevar a cabo ninguna modificación a lo que respecta.

Ha llegado la hora de crear el VirtualHost en cuestión, fichero de configuración que debemos alojar en **/etc/httpd/sites-available/**, así que generaré uno de nombre **sitioweb.conf**, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/httpd/sites-available/sitioweb.conf
{% endhighlight %}

Dentro del mismo, en mi caso, estableceré el siguiente contenido:

{% highlight shell %}
<VirtualHost *:80>
    ServerName www.alvaro.gonzalonazareno.org
    DocumentRoot /var/www/alvaro

    <Proxy "unix:/run/php-fpm/www.sock|fcgi://php-fpm">
        ProxySet disablereuse=off
    </Proxy>

    <FilesMatch \.php$>
        SetHandler proxy:fcgi://php-fpm
    </FilesMatch>

    ErrorLog /var/www/alvaro/log/error.log
    CustomLog /var/www/alvaro/log/requests.log combined
</VirtualHost>
{% endhighlight %}

Dentro del mismo, hemos establecido dos directivas totalmente necesarias:

* **ServerName**: Indicamos el nombre de dominio que se va a utilizar para acceder al VirtualHost en cuestión.
* **DocumentRoot**: Indicamos el directorio donde vamos a almacenar los ficheros que queremos que sean servidos.

Además de ellas, hemos indicado algunas directivas necesarias para hacer que el servidor web actúe como un proxy inverso, reenviando al servidor de aplicaciones **php-fpm** las peticiones de ficheros **.php**. De otro lado, hemos indicado un directorio en el que se van a almacenar los _logs_, que al igual que el DocumentRoot, también tendremos que crearlo.

Para crear el DocumentRoot y el directorio donde se almacenarán los _logs_, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# mkdir -p /var/www/alvaro/log
{% endhighlight %}

Donde:

* **-p**: Indicamos que se cree el directorio padre en caso de ser necesario. En este caso, generará el DocumentRoot **/var/www/alvaro/**.

Una vez realizadas todas las configuraciones oportunas, es hora de habilitar el VirtualHost. Para ello, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/httpd/sites-available** dentro de **/etc/httpd/sites-enabled** de forma manual. Para ello, ejecutamos el comando:

{% highlight shell %}
[root@quijote ~]# ln -s /etc/httpd/sites-available/sitioweb.conf /etc/httpd/sites-enabled/
{% endhighlight %}

Al parecer, el sitio ha sido correctamente habilitado, sin embargo, en CentOS 8 viene habilitado por defecto **SELinux**, un módulo de seguridad para el kernel Linux que proporciona el mecanismo para implementar una políticas de control de acceso para las aplicaciones, los procesos y los archivos dentro de un sistema. Esto es un problema para nosotros, ya que por defecto está únicamente configurado para trabajar con los directorios que _httpd_ crea por defecto, y nosotros hemos generado directorios adicionales.

Para solucionarlo, tendremos que modificar la política del mismo y permitir por tanto, el uso de dichos directorios, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# setsebool -P httpd_unified 1
{% endhighlight %}

Para activar toda la nueva configuración llevada a cabo, tendremos que volver a cargar la configuración del servicio _httpd_, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# systemctl restart httpd
{% endhighlight %}

Una vez que la configuración del servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/nginx/sites-enabled** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# ls -l /etc/httpd/sites-enabled/
total 0
lrwxrwxrwx. 1 root root 40 Dec 15 12:57 sitioweb.conf -> /etc/httpd/sites-available/sitioweb.conf
{% endhighlight %}

Efectivamente, el sitio se encuentra actualmente activo, pero para comprobarlo, vamos a generar en el DocumentRoot un fichero PHP de nombre **info.php** que devuelva la información de PHP, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# echo "<?php phpinfo(); ?>" > /var/www/alvaro/info.php
{% endhighlight %}

Tras ello, podremos solicitar dicho recurso desde el navegador, obteniendo el siguiente resultado:

![php1](https://i.ibb.co/w4XwzW8/Captura-de-pantalla-de-2020-12-15-13-16-22.png "phpinfo")

Como se puede apreciar, el recurso ha sido correctamente generado y devuelto en HTML para su correspondiente visualización, por lo que podemos concluir que el servidor de apliciones **php-fpm** y el servidor web **httpd** tienen conectividad y están funcionando correctamente.

## Servidor BBDD

Por último, vamos a instalar un gestor de bases de datos MariaDB en **Sancho**, para posteriores aplicaciones que necesitemos desplegar en el escenario. Para ello, necesitamos llevar a cabo la instalación de **mariadb-server**, no sin antes actualizar toda la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# apt update && apt upgrade && apt install mariadb-server
{% endhighlight %}

Una vez instalado, es recomendable aumentar la seguridad de dicha base de datos ya que se va a poner en producción, haciendo para ello uso de un _script_ que proporciona MariaDB, que llevará a cabo una configuración inicial pensada para ello:

* Estableceremos una contraseña para el administrador de MariaDB.
* Eliminaremos el usuario anónimo que viene creado por defecto.
* Desactivaremos la conexión remota a _root_, es decir, únicamente se podrá llevar a cabo desde _localhost_.
* Eliminaremos la base de datos de pruebas que viene creada por defecto.
* Volveremos a cargar las tablas de privilegios, para que los cambios surtan efecto.

Lo primero que preguntará el _script_ será la contraseña actual del usuario **root** de MariaDB. Lo dejaremos vacío y pulsaremos **ENTER**, ya que por ahora no tiene ninguna contraseña configurada. Tras ello, tendremos que ir introduciendo "**Y**" a las opciones que nos vayan saliendo, para así llevar a cabo toda la configuración anteriormente mencionada. Para ejecutar dicho _script_, haremos uso del comando:

{% highlight shell %}
root@sancho:~# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Set root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!

Remove anonymous users? [Y/n] Y
 ... Success!

Disallow root login remotely? [Y/n] Y
 ... Success!

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
{% endhighlight %}

Listo, la configuración inicial de MariaDB se habrá llevado a cabo gracias al _script_, aumentando así su seguridad.

Como curiosidad, me gustaría mencionar que la contraseña que hemos especificado para el usuario _root_ de MariaDB no sirve "para nada", lo pongo entre comillas ya que por defecto, la autentificación para dicho usuario se lleva a cabo mediante un _Socket UNIX_, es decir, mientras que te encuentres con sesión iniciada en _root_ o bien antepongas `sudo` al comando, podrás hacer uso de MariaDB sin conocer la contraseña que acabas de indicar.

Esto, puede ser bueno o malo, según se vea. En caso de querer autentificarte mediante **Socket UNIX** (opción por defecto), tendrás que asegurarte de proteger el acceso al usuario _root_ del sistema, mientras que si deseas autentificarte con **credenciales**, tendrás que asegurarte de proteger el acceso al usuario _root_ de MariaDB. En caso de considerar más oportuna la última opción, podrás encontrar [aquí](https://www.itzgeek.com/how-tos/linux/debian/how-to-install-mariadb-on-debian-10.html) un artículo sobre cómo cambiar el método de autentificación.

Tras ello, accederemos al servidor **mysql**, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 57
Server version: 10.3.25-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Vamos a realizar una prueba de conexión remota desde **Quijote**, de manera que necesitaremos crear un usuario que tenga el acceso permitido desde **10.0.2.6**. En este caso, crearemos un usuario de nombre **quijote**, que tenga permitido el acceso desde la dirección anteriormente mencionada (si quisiéramos permitir el acceso desde cualquier dirección IP, se podría poder **%**), y cuya contraseña sea **quijote**.

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'quijote'@'10.0.2.6' IDENTIFIED BY 'quijote';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez llevada a cabo la creación del correspondiente usuario, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

Esto no es todo, ya que por defecto, _mysql_ viene configurado para escuchar únicamente peticiones desde localhost, por lo que no escucharía aquellas peticiones que vengan del exterior. Para modificar esto, tenemos que editar el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf**, haciendo uso de `nano`:

{% highlight shell %}
root@sancho:~# nano /etc/mysql/mariadb.conf.d/50-server.cnf
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
root@sancho:~# systemctl restart mariadb
{% endhighlight %}

Ya está todo listo para realizar la prueba de conexión desde el exterior, concretamente desde **Quijote**, que en un principio debería de funcionar, siempre y cuando no exista un cortafuegos bloqueando la conexión (en este caso no hay problema, pues me he asegurado previamente).

Tras ello, procederemos a instalar en **Quijote** el paquete necesario para llevar a cabo la conexión remota, de nombre **mariadb**, por lo que ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# dnf install mariadb
{% endhighlight %}

Una vez instalado el paquete necesario, podremos llevar a cabo la conexión remota, haciendo para ello uso de las opciones **-u** y **-p** para indicar el usuario y contraseña a utilizar, con la única diferencia que tendremos que indicar además la dirección del servidor al que nos vamos a tratar de conectar, utilizando la opción **-h**, en este caso, **bd.alvaro.gonzalonazareno.org**. La instrucción a ejecutar sería:

{% highlight shell %}
[root@quijote ~]# mysql -u quijote -p -h bd.alvaro.gonzalonazareno.org
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 39
Server version: 10.3.25-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Al parecer, la conexión se ha realizado sin ningún problema, pues ha resuelto correctamente el nombre introducido, además de estar correctamente configurada la máquina servidora para aceptar dichas conexiones.

De nada nos serviría tener conexión si no podemos mostrar la información contenida, así que vamos a realizar una pequeña prueba listando las bases de datos existentes, utilizando para ello la instrucción:

{% highlight shell %}
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
+--------------------+
1 row in set (0.002 sec)
{% endhighlight %}

Como se puede apreciar en la salida, se ha mostrado la única base de datos actualmente existente, por lo que podemos corroborar que la conexión remota a la base de datos está totalmente operativa y funcional.