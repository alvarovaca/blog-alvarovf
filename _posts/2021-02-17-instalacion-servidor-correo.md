---
layout: post
title:  "Instalación de un servidor de correo"
banner: "/assets/images/banners/correo.jpg"
date:   2021-02-17 16:11:00 +0200
categories: servicios vps
---
Este es el cuarto artículo referente a la configuración del VPS, una máquina virtual contratada en OVH que cuenta con un direccionamiento público **51.210.109.246**. Además, se ha contratado una zona DNS en la que nombraremos dicha máquina junto a los servicios que despleguemos, en el dominio **iesgn19.es**.

La intención es la de ir desarrollando a lo largo del curso una serie de _posts_ en los que se detallen las configuraciones llevadas a cabo en dicho VPS, así como el mantenimiento de las mismas. En el día de hoy, vamos a instalar un servidor de correo **Postfix**, así como llevar a cabo determinadas configuraciones para ampliar las funcionalidades del mismo.

Antes de comenzar con la instalación de _Postfix_, es recomendable conocer de una forma más o menos superficial el método de funcionamiento del correo electrónico. Como todos bien sabemos, para enviar un correo electrónico necesitamos conocer la dirección de correo de un usuario, que tendrá la siguiente forma:

{% highlight shell %}
usuario@nombre_servicio
{% endhighlight %}

De forma general, el nombre del servicio de correo es el nombre del dominio de la organización a la que pertenece el usuario. Por ejemplo, **alvarovf.com**.

Cada vez que enviamos un correo a esa dirección, tenemos que determinar de alguna forma la dirección IP del servidor de correo existente en la organización, siguiendo para ello los siguientes pasos:

* Si el nombre detrás de la **@** está asociado a un registro _A_ en un servidor DNS, ya conocemos la dirección del servidor de correo.

* En caso contrario, se consultaría en el servidor DNS por el registro _MX_ que indique el nombre del servidor de correo asociado al nombre de dominio.

Sin entrar en demasiado detalle, existen determinados conceptos que debemos conocer respecto a los servidores de correo y la infraestructura existente a su alrededor:

* **MUA (_Mail User Agent_)**: Programa que permite a un usuario, como mínimo, leer y escribir mensajes de correo electrónico. Generalmente conocido como cliente de correo. Por ejemplo, **Evolution**, **Outlook**, **Thunderbird**...

* **MTA (_Mail Transfer Agent_)**: Programa que transfiere los mensajes de correo electrónico entre máquinas que usan el protocolo _SMTP_. Un mensaje puede pasar por varios _MTA_ hasta llegar al destino final. Generalmente conocido como servidor de correo. Por ejemplo, **Postfix**, **qmail**, **exim**...

* **MDA (_Mail Delivery Agent_)**: Programas utilizados por los agentes _MTA_ para entregar el correo electrónico al buzón de un usuario concreto. Esta entrega se puede hacer localmente en el servidor o de forma remota utilizando el protocolo _POP3_ o _IMAP_.

* **SMTP (_Simple Mail Transfer Protocol_)**: Protocolo de red utilizado para el intercambio de mensajes de correo electrónico. Existe una mejora del protocolo llamada _ESMTP_ (_Enhanced Simple Mail Transfer Protocol_).

* **POP3 (_Post Office Protocol_)**: Protocolo para recuperar correos electrónicos de un _MDA_. Su principal característica es que se descargan todos los correos.

* **IMAP (_Internet Message Access Protocol_)**: Protocolo para recuperar correos electrónicos de un _MDA_. A diferencia del anterior, se sincroniza el estado de los correos entre el servidor y el cliente.

Por último, de una manera un tanto superficial, vamos a comprender el proceso que se sigue cada vez que un correo electrónico es enviado de un cliente a otro:

![esquema1](https://i.ibb.co/TmYFbL0/mta.png "Esquema correo")

* **1.** Un usuario utiliza un _MUA_ para enviar el correo electrónico a su servidor de correos (_MTA_). Este envío se hace usando el protocolo SMTP. El nombre del servidor tendrá que estar definido en un servidor DNS, y en un principio usaremos el puerto **25/TCP**, estableciendo una conexión no cifrada ni autentificada. Por ese mismo motivo, es más común utilizar el protocolo _ESMTP_, que utiliza el puerto **587/TCP** y permite la autentificación y el cifrado de la comunicación.

- **2.** El _MTA_ recibe el correo desde el _MUA_:

    - Si la dirección del destinatario del correo es la misma que la que controla el servidor receptor, el correo no se envía a ningún _MTA_ y se le da al _MDA_ para que lo guarde en el buzón del usuario destinatario.

    - Si la dirección del destinatario del correo es distinta que la que controla el servidor receptor, el correo se envía al _MTA_ correspondiente a la dirección del destinatario, pudiéndose hacer de dos formas:

        - Si por cualquier razón el _MTA_ no puede enviar el correo directamente al _MTA_ destino, se tendrá configurado un servidor _MTA_ intermediario (_relay_) que será el responsable de enviarlo al destinatario final.

        - Si el _MTA_ no tiene configurado un servidor _relay_ intermediario, tendrá que averiguar la dirección IP del servidor correspondiente al nombre de correo del destinatario, normalmente haciendo una consulta _MX_ al servidor DNS. En caso de que existan varios servidores de correo definidos, le intentará mandar el correo al más prioritario (el que tiene el número más pequeño).

* **3.** Cuando el correo llega al _MTA_ destino, se pasa el correo al _MDA_ que lo guardará en el buzón del usuario destinatario.

- **4.** El usuario destinatario utilizará una _MUA_ para conectarse al servidor _MDA_ y recuperar el correo:

    - Puede utilizar el protocolo _POP3_, por lo que se conectará al servidor _POP3_. Esta conexión está autentificada y puede estar cifrada. El protocolo _POP3_ suele hacer uso del puerto **110/TCP**. Con este protocolo lo que hacemos es descargar todos los correos desde nuestro buzón remoto a nuestro _MUA_.

    - Puede usar el protocolo _IMAP_, por lo que se conectará al servidor _IMAP_. Esta conexión está autentificada y puede estar cifrada. El protocolo _IMAP_ suele hacer uso del puerto **143/TCP**. Con este protocolo lo que hacemos es sincronizar el estado de los correos desde nuestro buzón remoto a nuestro _MUA_, de tal manera que podemos acceder desde distintos _MUA_ y obtendremos el mismo estado de los correos en todos ellos. Es imprescindible si vamos a usar un _MUA_ que sea una aplicación web.

Una vez conocido el método de funcionamiento del correo electrónico y algunos conceptos necesarios, todo estará listo para instalar un **MTA** de nombre **Postfix**, que nos permitirá transferir mensajes de correo electrónico entre máquinas que utilicen el protocolo _SMTP_.

Sin embargo, antes de ello es necesario realizar algunas modificaciones en nuestra zona DNS, pues para poder enviar y recibir correos desde nuestro VPS necesitamos cumplir los siguientes requisitos:

* En este caso, no es necesario, ya que el VPS es capaz de enviar el correo electrónico al exterior por sí mismo, pero en otras situaciones, sería necesario configurar un servidor de correo intermediario (_relay_).

* Necesitamos configurar en nuestra zona DNS un registro _MX_ apuntando a nuestra máquina, necesario para la recepción de correos, pues así el resto de máquinas pueden conocer la IP de nuestro servidor de correo.

* Necesitamos implementar varias técnicas (al menos un registro _SPF_ del que posteriormente hablaremos) para asegurar que el servidor de correo al que mandamos nuestro correo confíe en nosotros, para así evitar suplantaciones.

* Necesitamos que nuestra IP pública esté "limpia". Generalmente, las direcciones IP dinámicas suelen estar incluidas en una lista negra que comprobarán los servicios como Gmail, Hotmail o Yahoo para evitar el _spam_, ya que previamente han sido utilizadas con dicha finalidad. Podemos comprobar si nuestra dirección IP se encuentra en una lista negra haciendo uso de [esta](https://www.dnsbl.info/dnsbl-database-check.php) utilidad.

El primer paso ha consistido en la generación de un nuevo nombre dentro de la zona DNS **iesgn19.es**, concretamente nombrando un nuevo servicio **mail** mediante un registro _A_ que apunta a la dirección **51.210.109.246**, que será el nombre del servicio al que posteriormente apuntará el registro _MX_, tal y como podemos apreciar a continuación:

![dns1](https://i.ibb.co/Pc19xYb/Captura-de-pantalla-de-2021-01-21-09-48-38.png "Zona DNS")

Muy posiblemente estéis pensando en el motivo por el que he creado un registro _A_ apuntando a la dirección IP de la VPS en lugar de crear un registro _CNAME_ que apunte a un registro _A_ ya existente.

En realidad, podríamos haber creado el registro _CNAME_ que acabamos de mencionar apuntando a un registro _A_ ya existente y funcionaría a la perfección, sin embargo, no es considerado una buena práctica, pues idealmente, los registros que apunten a servidores de correo deben ser registros _A_.

Desconozco el motivo real, pues lo único que se me ocurre es que la finalidad sea acelerar la petición, ya que se haría una única petición (registro _A_) en lugar de dos (registro _CNAME_).

Tras ello, todo estará listo para crear el nuevo registro _MX_ que indicará el servidor al que los servidores de correo externos deben enviar los correos cuyo dominio destinatario sea **iesgn19.es**. Le asignaremos la prioridad máxima (**1**), aunque en este caso es irrelevante, pues únicamente tenemos un servidor, quedando de la siguiente forma:

![dns2](https://i.ibb.co/hKxYxvP/Captura-de-pantalla-de-2021-01-19-14-06-16.png "Zona DNS")

Por último, tendremos que implementar al menos una técnica para asegurar que el servidor de correo al que mandamos nuestro correo confíe en nosotros, para así evitar suplantaciones. En este caso, vamos a configurar un registro _SPF_, no sin antes explicar en qué consiste.

En principio, cualquier máquina puede enviar mensajes de correo a cualquier destino utilizando cualquier remitente, pero esta característica del correo ha sido masivamente utilizada para inundar de mensajes no deseados a los servidores de correo y hacer suplantaciones del remitente (_email spoofing_), por lo que se ha generalizado la implementación de medidas complementarias para reducir al máximo este problema y hoy en día, la mayoría de servidores de correo o rechazan o mandan a la bandeja de _spam_ los mensajes que lleguen desde servidores que no implementen algún mecanismo adicional de autenticación.

Como ya hemos mencionado, en este caso vamos a utilizar **_SPF_** (_Sender Policy Framework_), un mecanismo de autenticación que describe de forma explícita mediante un registro DNS de tipo _TXT_ las direcciones IP y nombres DNS autorizados a enviar correo desde un determinado dominio.

Se pueden especificar diversos campos en dicho registro, pero en este caso, que tenemos un solo equipo con una dirección IPv4, pondremos un registro SPF parecido al siguiente:

{% highlight shell %}
IN  TXT "v=spf1 mx ~all"
{% endhighlight %}

Dentro del mismo, podemos hacer referencia a nuestro servidor de correo de diferentes formas:

* **a**: La IP de un registro A de un nombre del DNS.

* **mx**: Registro MX del DNS del dominio.

* **ptr**: Registro PTR del servidor de correo.

* **ip4**: Direcciones IPv4.

* **ip6**: Direcciones IPv6.

En este caso, como la dirección IP a la que apunta el registro _MX_ coincide con la máquina que va a enviar el correo al exterior, he referenciado al servidor de correo mediante **mx**, aunque también podría haber puesto **ip4:51.210.109.246/32**.

Es importante mencionar la importancia del signo que aparece antes de **all**, ya que podemos indicarle a los servidores destinatarios lo que deben hacer si reciben correo desde otra máquina diferente a las referenciadas en el registro anterior:

* **-**: Descartar el mensaje.

* **~**: Clasificarlo como _spam_.

* **?**: Aceptar el mensaje.

En este caso, no vamos a _pillarnos los dedos_ y vamos a indicar que los correos electrónicos recibidos desde una máquina distinta a la VPS se clasifique como _spam_, pero evitando que se descarte, para así comprobar que el envío se está realizando correctamente.

De esta forma, el correo que enviemos desde la máquina VPS pasará los filtros SPF en el destino y la mayoría de nuestros correos llegarán al destino con poca probabilidad de que se clasifiquen como _spam_. El registro tendrá finalmente la siguiente forma:

![dns3](https://i.ibb.co/XVg4sdq/Captura-de-pantalla-de-2021-01-19-14-10-03.png "Zona DNS")

Una vez finalizada la configuración en la zona DNS de nuestro dominio, todo estará listo para proceder con la instalación y configuración inicial del servidor de correos **Postfix**.

## Instalación de Postfix

Además de llevar a cabo la instalación del paquete **postfix**, instalaremos a su vez el paquete **bsd-mailx**, que nos proporcionará las utilidades necesarias para llevar a cabo el envío de correo (_mail_), no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando:

{% highlight shell %}
root@vps:~# apt update && apt upgrade && apt install postfix bsd-mailx
{% endhighlight %}

Durante la instalación, nos aparecerá una ventana en la que debemos seleccionar una opción para que el paquete se configure en base a nuestras necesidades. En nuestro caso, dado que la máquina es capaz de enviar y recibir correo electrónico por sí misma a través del protocolo _SMTP_, seleccionaremos la opción **Internet Site**, de la siguiente manera:

![postfix1](https://i.ibb.co/9n1mZ7H/Captura-de-pantalla-de-2021-01-19-13-45-38.png "Postfix")

En el siguiente paso de la instalación se nos pedirá el nombre del sistema de correo, es decir, el nombre que se utilizará para cualificar los correos electrónicos en cuestión, o en otras palabras, lo que irá indicado después de **@**. Por ejemplo, para el correo **debian@iesgn19.es**, el valor correcto sería **iesgn19.es**, quedando de la siguiente forma:

![postfix2](https://i.ibb.co/S5Ffk9m/CapturaX.jpg "Postfix")

El paquete **postfix** ha sido instalado en la máquina, existiendo en consecuencia un proceso que está actualmente en ejecución, que habrá abierto un _socket TCP/IP_ en el puerto por defecto del protocolo _SMTP_ (**25**) que estará escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:25              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN     
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::25585                :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 :::25                   :::*                    LISTEN     
tcp6       0      0 :::443                  :::*                    LISTEN
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.

Efectivamente, el proceso está escuchando peticiones tal y como debería. A pesar de estar escuchando peticiones en todas las interfaces de la máquina, eso no supone que cualquier persona ajena a nosotros pueda utilizar nuestro servidor para enviar correos, ya que para ello existe indicada una directiva **mynetworks** en el fichero **/etc/postfix/main.cf** que se encuentra limitada al direccionamiento **127.0.0.0/8**, lo que supone que nadie podrá hacer _relay_ en nuestro servidor.

Sin embargo, dado que el protocolo _SMTP_ también utiliza el puerto **25/TCP** para la recepción de correos (además de para hacer _relay_), será necesario tenerlo abierto para todo el mundo, pues de lo contrario, imposibilitaríamos la recepción de los mismos.

## Envío de correos desde el servidor

Una vez llevada a cabo toda la configuración necesaria, todo estará listo para hacer uso de la utilidad `mail` para enviar correos al exterior. En este caso, vamos a enviar un correo de prueba a mi correo personal, ejecutando para ello el comando:

{% highlight shell %}
debian@vps:~$ mail avacaferreras@gmail.com
Subject: Correo de prueba.
Buenos días, esto es un correo de prueba.
Cc:
{% endhighlight %}

En este caso, hemos redactado un correo desde el usuario **debian** al correo destino **avacaferreras@gmail.com**, cuyo asunto será "**Correo de prueba.**" y su contenido, "**Buenos días, esto es un correo de prueba.**". Considero necesario mencionar que en caso de hacer uso de la utilidad de línea de comandos para enviar correos, finalizaremos la escritura del mismo con la combinación de teclas **CTRL + D**.

En un principio, el envío del correo electrónico se ha producido correctamente, sin embargo, vamos a proceder a verificarlo visualizando para ello los _logs_ producidos por dicho envío, en el fichero **/var/log/mail.log**, haciendo para ello uso del comando:

{% highlight shell %}
debian@vps:~$ cat /var/log/mail.log
Jan 21 08:55:12 vps postfix/pickup[15685]: 1CD67E13D1: uid=1000 from=<debian>
Jan 21 08:55:12 vps postfix/cleanup[15693]: 1CD67E13D1: message-id=<20210121075512.1CD67E13D1@vps.iesgn19.es>
Jan 21 08:55:12 vps postfix/qmgr[4212]: 1CD67E13D1: from=<debian@iesgn19.es>, size=445, nrcpt=1 (queue active)
Jan 21 08:55:12 vps postfix/smtp[15695]: connect to gmail-smtp-in.l.google.com[2a00:1450:400c:c01::1b]:25: Network is unreachable
Jan 21 08:55:12 vps postfix/smtp[15695]: 1CD67E13D1: to=<avacaferreras@gmail.com>, relay=gmail-smtp-in.l.google.com[172.253.120.26]:25, delay=0.59, delays=0.01/0/0.44/0.15, dsn=2.0.0, status=sent (250 2.0.0 OK  1611215712 y9si3791494wrs.536 - gsmtp)
Jan 21 08:55:12 vps postfix/qmgr[4212]: 1CD67E13D1: removed
{% endhighlight %}

Como se puede apreciar en la penúltima línea mostrada, el estado del correo enviado desde **debian@iesgn19.es** a **avacaferreras@gmail.com** es **sent** (_250_), por lo que podemos asegurar que el envío se ha producido correctamente desde nuestro extremo.

Todavía queda verificar que la recepción del mismo también se ha llevado a cabo tal y como debería, de manera que accederé a mi correo personal y comprobaré si ha aparecido en la bandeja de entrada:

![gmail1](https://i.ibb.co/L0VSwbZ/Captura-de-pantalla-de-2021-01-21-08-56-09.png "Gmail")

Efectivamente, la recepción del correo electrónico ha sido correcta, gracias a haber indicado en nuestra zona DNS un registro _SPF_ para que _Gmail_ pueda verificar la procedencia de dicho correo, y por tanto, pueda confiar en nosotros.

De igual forma, podremos ver el contenido original de dicho correo electrónico junto a sus cabeceras pulsando para ello en los tres puntos ubicados en la parte derecha del mismo, seleccionando tras ello la opción "**Mostrar original**", que tal y como se puede apreciar, en este caso tiene la siguiente estructura:

![gmail2](https://i.ibb.co/qkKy641/Captura-de-pantalla-de-2021-01-21-08-56-17.png "Gmail")

Si nos fijamos con detenimiento, la directiva **Received-SPF** tiene el valor **pass**, lo que significa que la validación del remitente ha sido efectiva gracias al registro _SPF_ que previamente hemos definido, en el que hemos autorizado a la dirección IP **51.210.109.246** a enviar correos electrónicos desde el dominio **iesgn19.es**.

## Recepción de correos desde el servidor

Una vez que hemos comprobado que el envío de correos desde nuestra máquina al exterior funciona como debería, vamos a realizar el procedimiento inverso, en el que vamos a recibir correos en nuestra máquina que hayan sido enviados desde máquinas ajenas.

Para ello, vamos a responder al mensaje en _Gmail_, que en este caso irá destinado al usuario **debian** existente en el dominio **iesgn19.es**, cuyo contenido será "**Prueba de respuesta.**", de manera que el resultado final sería el siguiente:

![gmail3](https://i.ibb.co/yVZcHkL/Captura-de-pantalla-de-2021-01-21-08-56-49.png "Gmail")

En un principio, el envío y la recepción del correo electrónico se ha producido correctamente, sin embargo, vamos a proceder a verificarlo visualizando para ello los _logs_ producidos por la correspondiente recepción, en el fichero **/var/log/mail.log**, haciendo para ello uso del comando:

{% highlight shell %}
debian@vps:~$ cat /var/log/mail.log
Jan 21 08:56:58 vps postfix/smtpd[15713]: connect from mail-wm1-f42.google.com[209.85.128.42]
Jan 21 08:56:58 vps postfix/smtpd[15713]: AA1CDE13D0: client=mail-wm1-f42.google.com[209.85.128.42]
Jan 21 08:56:58 vps postfix/cleanup[15720]: AA1CDE13D0: message-id=<CANR0p-363sCZ-+s7CsHmpnD-Cr-CetvtTFwWZS_MUkZpJFzZUA@mail.gmail.com>
Jan 21 08:56:58 vps postfix/qmgr[4212]: AA1CDE13D0: from=<avacaferreras@gmail.com>, size=3381, nrcpt=1 (queue active)
Jan 21 08:56:58 vps postfix/local[15721]: AA1CDE13D0: to=<debian@iesgn19.es>, relay=local, delay=0.02, delays=0.01/0.01/0/0, dsn=2.0.0, status=sent (delivered to mailbox)
Jan 21 08:56:58 vps postfix/qmgr[4212]: AA1CDE13D0: removed
Jan 21 08:56:58 vps postfix/smtpd[15713]: disconnect from mail-wm1-f42.google.com[209.85.128.42] ehlo=2 starttls=1 mail=1 rcpt=1 bdat=1 quit=1 commands=7
{% endhighlight %}

Como se puede apreciar en la antepenúltima línea mostrada, el estado del correo enviado desde **avacaferreras@gmail.com** a **debian@iesgn19.es** es **sent** (_250_), por lo que podemos asegurar que la recepción se ha producido correctamente desde nuestro extremo.

Tras ello, volveré a hacer uso de la utilidad `mail` para acceder a la bandeja de entrada (buzón) del usuario **debian**:

{% highlight shell %}
debian@vps:~$ mail
Mail version 8.1.2 01/15/2001.  Type ? for help.
"/var/mail/debian": 1 message 1 new
>N  1 avacaferreras@gma  Thu Jan 21 08:56   70/3476  Re: Correo de prueba.
{% endhighlight %}

Efectivamente, la recepción del correo electrónico ha sido correcta, gracias a haber indicado en nuestra zona DNS un registro _MX_ para que _Gmail_ pueda encontrar la dirección IP de la máquina a la que debe enviar los correos para el dominio **iesgn19.es**.

De igual forma, podremos ver el contenido de dicho correo electrónico junto a sus cabeceras eligiendo numéricamente el correo que queremos visualizar (en este caso, **1**), pudiendo apreciar lo siguiente:

{% highlight shell %}
Message 1:
From avacaferreras@gmail.com  Thu Jan 21 08:56:58 2021
X-Original-To: debian@iesgn19.es
...
From: =?UTF-8?Q?=C3=81lvaro_Vaca_Ferreras?= <avacaferreras@gmail.com>
Date: Thu, 21 Jan 2021 08:56:47 +0100
Subject: Re: Correo de prueba.
To: Debian <debian@iesgn19.es>
Content-Type: multipart/alternative; boundary="0000000000007f234505b9646a06"

--0000000000007f234505b9646a06
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: quoted-printable

Prueba de respuesta.
...
{% endhighlight %}

Como se puede apreciar, el contenido al completo de dicho correo electrónico ha sido correctamente visualizado desde nuestra utilidad de línea de comandos, por lo que podemos verificar que la recepción de correos está totalmente operativa.

## Alias y redirecciones

Una característica muy interesante es que podemos permitir a los procesos que se encuentren en ejecución en la máquina enviar correos para así informar sobre su estado, por ejemplo, podemos crear una tarea `cron` en la que el resultado de la ejecución de dicho comando se notifique mediante un correo a un usuario concreto.

En este caso, vamos a crear una nueva tarea que nos envíe cada minuto la hora del sistema (resultado de la ejecución del comando `date`), ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# crontab -e
{% endhighlight %}

Donde:

* **-e**: Permite realizar modificaciones en el _crontab_ actual y cargar de forma automática las modificaciones llevadas a cabo.

En este caso, tendremos que indicar como mínimo dos nuevas directivas, siendo la primera de ellas aquella en la que indicaremos el usuario al que queremos enviar dichos correos informando del resultado de las ejecuciones (generalmente suele ser **root**) y la segunda, el comando en cuestión, precedido de la frecuencia de ejecución.

Explicar cómo se define la frecuencia de ejecución de cualquier comando indicado en el _crontab_ se saldría del objetivo de este artículo, de manera que [aquí](https://linux.die.net/man/5/crontab) se podrá encontrar una página del manual en la que se explica de forma detallada y con ejemplos que ayudan a su comprensión. El resultado final sería:

{% highlight shell %}
MAILTO = root
* * * * * date
{% endhighlight %}

El hecho de haber indicado 5 __*__ supone la ejecución de dicho comando 1 vez por minuto, por lo que a lo largo de una hora recibiremos 60 correos informando sobre la ejecución de dicho comando.

Es importante mencionar que la directiva **MAILTO** afecta a todos los comandos indicados tras la misma, de manera que el resultado de aquellos que se hayan indicado previamente no será notificado por correo.

Tras esperar un minuto para así dar lugar a la ejecución de dicho comando, volveré a hacer uso de la utilidad `mail` para acceder a la bandeja de entrada (buzón) del usuario **root**:

{% highlight shell %}
root@vps:~# mail
Mail version 8.1.2 01/15/2001.  Type ? for help.
"/var/mail/root": 1 message 1 new
>N  1 root@iesgn19.es    Thu Jan 21 09:32   22/683   Cron <root@vps> date
{% endhighlight %}

Efectivamente, la recepción local del correo electrónico ha sido correcta, así que procederemos a visualizar el contenido de dicho correo electrónico junto a sus cabeceras eligiendo numéricamente al igual que en el caso anterior, pudiendo apreciar lo siguiente:

{% highlight shell %}
Message 1:
From root@iesgn19.es  Thu Jan 21 09:32:01 2021
X-Original-To: root
From: root@iesgn19.es (Cron Daemon)
To: root@iesgn19.es
Subject: Cron <root@vps> date
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <MAILTO=root>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=root>
Date: Thu, 21 Jan 2021 09:32:01 +0100 (CET)

Thu 21 Jan 2021 09:32:01 AM CET
{% endhighlight %}

Como se puede apreciar, el contenido al completo de dicho correo electrónico ha sido correctamente visualizado desde nuestra utilidad de línea de comandos, por lo que podemos verificar que la recepción de correos para aquellas tareas `cron` configuradas se encuentra totalmente operativa.

En este caso estamos llevando a cabo un ejemplo que carece de sentido, pues mi intención es únicamente mostrar el funcionamiento, pero en situaciones reales, puede llegar a ser bastante útil, por ejemplo, para avisar al administrador del estado de la copia de seguridad diaria.

Sin embargo, todavía podemos ir un paso más allá y hacer uso de los **alias**, que nos permitirán redirigir el correo que un determinado usuario reciba al buzón de otro usuario ubicado en la misma máquina. Por ejemplo, podríamos redirigir el correo de todos los usuarios de la máquina al buzón de **root** y posteriormente, redirigir una vez más los correos entrantes para **root** a los usuarios administradores del sistema, para así gestionarlo de forma centralizada.

Para esta ocasión, he decidido redirigir todos los correos entrantes al usuario **root** al buzón del usuario **debian**, consiguiendo por tanto que el correo con el resultado de la ejecución de la tarea `cron` acabe finalmente en el buzón de **debian**. Para ello, tendremos que modificar el fichero **/etc/aliases**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/aliases
{% endhighlight %}

El contenido por defecto de dicho fichero es el siguiente:

{% highlight shell %}
postmaster:    root
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea por cada **alias** que deseemos crear, en la que indicaremos al principio el usuario del que queremos redirigir los correos y posteriormente, el usuario que actuará como destinatario final. En este caso, el resultado final sería:

{% highlight shell %}
postmaster:    root
root:    debian
{% endhighlight %}

Dado que hemos modificado un fichero de configuración, tendremos que ejecutar el comando `newaliases` para que los cambios surtan efecto, de la siguiente forma:

{% highlight shell %}
root@vps:~# newaliases
{% endhighlight %}

El nuevo **alias** ya habrá sido generado y se encuentra actualmente en vigor, de manera que el resultado de la próxima ejecución de la tarea `cron` acabará finalmente en el buzón de **debian**, tal y como se puede apreciar:

{% highlight shell %}
debian@vps:~$ mail
Mail version 8.1.2 01/15/2001.  Type ? for help.
"/var/mail/debian": 1 message 1 new
>N  1 root@iesgn19.es    Thu Jan 21 09:36   22/683   Cron <root@vps> date
{% endhighlight %}

Una vez más, la recepción local del correo electrónico ha sido correcta, no siendo necesario visualizar de nuevo el contenido de dicho correo.

Como hemos podido apreciar, los **alias** funcionan perfectamente entre usuarios existentes en la misma máquina, ¿pero qué ocurriría si quisiésemos reenviar dichos a un correo personal como puede ser _Gmail_? Para ello, tendríamos que acudir a las **redirecciones**, que nos permitirán enviar el correo que llegue a un usuario a una cuenta de correo exterior.

Para esta ocasión, he decidido redirigir todos los correos entrantes al usuario **debian** a mi correo electrónico personal, consiguiendo por tanto que el correo con el resultado de la ejecución de la tarea `cron` acabe finalmente en _Gmail_. Para ello, tendremos que añadir una nueva línea (o tantas como deseemos) al fichero **~/.forward**, ejecutando para ello el comando:

{% highlight shell %}
debian@vps:~$ echo "avacaferreras@gmail.com" >> ~/.forward
{% endhighlight %}

La nueva **redirección** ya habrá sido generada y se encuentra actualmente en vigor, de manera que el resultado de la próxima ejecución de la tarea `cron` acabará finalmente en mi correo electrónico personal, tal y como se puede apreciar:

![gmail4](https://i.ibb.co/VSLMdJy/forward.jpg "Gmail")

Tal y como hemos definido, la **redirección** a mi correo electrónico personal se ha llevado a cabo correctamente y he recibido en el mismo el resultado de la ejecución de la tarea, que como previamente he mencionado, puede llegar a ser algo muy útil en determinadas situaciones.

Por último, antes de finalizar con los **alias** y **redirecciones** es importante revertir todos los cambios realizados, para así retomar el comportamiento normal y deseado.

## Configuración de DKIM

Previamente hemos introducido el concepto del _SPF_, un mecanismo de autenticación para que los servidores de correo destinatarios puedan verificar nuestra identidad. Como se puede suponer, existen más mecanismos, como por ejemplo **_DKIM_** o **_DMARC_**.

En este caso vamos a implementar además **_DKIM_** (_DomainKeys Identified Mail_), un método de autenticación pensado principalmente para corroborar la procedencia del correo y asegurar que el mensaje no ha sido modificado durante la transferencia del mismo, consistente en publicar mediante un registro _TXT_ en el servidor DNS la clave pública del servidor de correos.

Posteriormente, se firmarán con la correspondiente clave privada todos los mensajes emitidos, de manera que el receptor podrá verificar cada correo emitido utilizando la clave pública.

Para configurar DKIM en nuestro servidor necesitaremos instalar los paquetes **opendkim** y **opendkim-tools**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# apt install opendkim opendkim-tools
{% endhighlight %}

Una vez instalados los paquetes necesarios, tendremos que llevar a cabo una serie de modificaciones en el fichero de configuración ubicado en **/etc/opendkim.conf**, el cuál editaremos ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/opendkim.conf
{% endhighlight %}

El contenido por defecto de dicho fichero es el siguiente:

{% highlight shell %}
Syslog                  yes
UMask                   007
PidFile                 /var/run/opendkim/opendkim.pid
OversignHeaders         From
TrustAnchorFile         /usr/share/dns/root.key
UserID                  opendkim
Socket                  local:/var/spool/postfix/opendkim/opendkim.sock
{% endhighlight %}

Dentro del mismo, tendremos que realizar las siguientes modificaciones:

* **Socket**: Modificaremos el _socket UNIX_ actualmente configurado por un _socket TCP/IP_ en el puerto **8892** en _localhost_, para evitar problemas de conexión entre los servicios.
* **Domain**: Indicaremos el dominio para el que queremos configurar el mecanismo _DKIM_. En este caso, **iesgn19.es**.
* **KeyFile**: Indicamos la ruta en la que se alojará la clave privada que posteriormente generaremos. El nombre de la misma estará compuesto por **[selector].private**. En este caso, **/etc/opendkim/keys/iesgn19.es/pruebadkim.private**.
* **Selector**: Indicamos un nombre único que posteriormente debemos utilizar a la hora de subir la clave pública al servidor DNS, para que el destinatario pueda identificarla fácilmente. En este caso, **pruebadkim**.

El resultado final sería:

{% highlight shell %}
Syslog                  yes
UMask                   007
PidFile                 /var/run/opendkim/opendkim.pid
OversignHeaders         From
TrustAnchorFile         /usr/share/dns/root.key
UserID                  opendkim
Socket                  inet:8892@localhost
Domain                  iesgn19.es
KeyFile                 /etc/opendkim/keys/iesgn19.es/pruebadkim.private
Selector                pruebadkim
{% endhighlight %}

Una vez realizadas las modificaciones oportunas, tendremos que modificar también el fichero **/etc/default/opendkim** para indicar de nuevo el _socket TCP/IP_ que se utilizará para comunicar los servicios **Postfix** y **opendkim**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/default/opendkim
{% endhighlight %}

Dentro del mismo, tendremos que referenciar al _socket TCP/IP_ ubicado en el puerto **8892** de la máquina local (_localhost_), quedando de la siguiente forma:

{% highlight shell %}
SOCKET=inet:8892@localhost
{% endhighlight %}

Tras ello, guardaremos los cambios y saldremos, para así proceder a modificar finalmente el fichero referente a **Postfix**, para que así utilice dicho mecanismo para firmar los correos salientes. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/main.cf
{% endhighlight %}

En su interior tendremos que introducir las líneas necesarias para indicarle entre otras cosas, el _socket TCP/IP_ que ha de utilizar para el firmado de dichos correos, siendo las mismas las siguientes:

{% highlight shell %}
milter_default_action = accept
milter_protocol = 2
smtpd_milters = inet:localhost:8892
non_smtpd_milters = $smtpd_milters
{% endhighlight %}

Todos los ficheros de configuración necesarios han sido correctamente modificados, de manera que únicamente nos faltaría generar dicho par de claves, siendo posteriormente utilizada la clave privada para firmar los correos y la clave pública (que tendremos que exponer en un registro _TXT_ del DNS) para comprobar dicha firma.

Como previamente hemos definido, el par de claves deberá encontrarse dentro de **/etc/opendkim/keys/**, en un subdirectorio con el nombre del dominio, en este caso, **iesgn19.es/**, así que procederemos a la creación de dicho subdirectorio haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# mkdir -p /etc/opendkim/keys/iesgn19.es
{% endhighlight %}

Donde:

* **-p**: Indica que se cree el directorio padre en caso de no existir.

El nuevo directorio ya habrá sido generado, sin embargo, dado que el contenido que se va a ubicar en el mismo es de carácter sensible, tendremos que protegerlo adecuadamente para así evitar su lectura por personas no deseadas, cambiando para ello los permisos a **700** y el usuario y grupo propietario a **opendkim**, ejecutando para ello los comandos:

{% highlight shell %}
root@vps:~# chmod 700 /etc/opendkim/keys/iesgn19.es/
root@vps:~# chown opendkim:opendkim /etc/opendkim/keys/iesgn19.es/
{% endhighlight %}

Tras ello, y con la finalidad de trabajar de una forma más comoda, nos moveremos dentro de dicho directorio haciendo uso del comando `cd`, para así proceder con la creación del par de claves. La creación es bastante sencilla, pues para ello acudiremos a la utilidad `opendkim-genkey`, de la siguiente forma:

{% highlight shell %}
root@vps:/etc/opendkim/keys/iesgn19.es# opendkim-genkey -s pruebadkim -d iesgn19.es -b 1024
{% endhighlight %}

Donde:

* **-s**: Indicamos el nombre del _selector_ para el que queremos generar el par de claves. En este caso, **pruebadkim**.
* **-d**: Indicamos el dominio que va a hacer uso del par de claves para firmar los correos. En este caso, **iesgn19.es**.
* **-b**: Indicamos el tamaño de la clave, que no deberá ser demasiado grande, para evitar las limitaciones de los registros _TXT_ en el DNS. En este caso, **1024** bits.

Una vez realizada la ejecución de dicho comando, habrán aparecido dos nuevos ficheros en el directorio actual, que podremos verificar listando el contenido del mismo, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:/etc/opendkim/keys/iesgn19.es# ls -l
total 8
-rw------- 1 root root 1679 Feb 15 16:36 pruebadkim.private
-rw------- 1 root root  512 Feb 15 16:36 pruebadkim.txt
{% endhighlight %}

Donde:

* **pruebadkim.private**: Contiene la clave privada y será utilizada para firmar los correos electrónicos salientes.
* **pruebadkim.txt**: Contiene la clave pública y será utilizada por los servidores de correo para verificar la firma sobre los correos entrantes.

Únicamente nos faltaría añadir el correspondiente registro _TXT_ a nuestra zona DNS que contenga dicha clave pública para poder empezar a hacer uso de este mecanismo, así que antes de nada, tendremos que visualizarla, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/opendkim/keys/iesgn19.es# cat pruebadkim.txt
pruebadkim._domainkey   IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDhxmxXURJ3QSnnRu4SW9aK3o2Uq8CNwckIzZTdrnA7tWhi1NXrpxPfx0EHOmF1LuJC23eSLbbmy5/xyT6hEnSToE3eNHHd+ZYezmVzi2lZtyoeqxWao15q4WWYxvF99AxLNg3CnXDxuh4T5wtMXBlcysn38iMsTQI+VnGFUxu9xQIDAQAB" )  ; ----- DKIM key pruebadkim for iesgn19.es
{% endhighlight %}

Es importante mencionar que el nombre del registro _TXT_ debe estar compuesto por **[selector]._domainkey**, que en mi caso sería **pruebadkim._domainkey** y quedaría finalmente de la siguiente forma:

![dns4](https://i.ibb.co/fQnrxWg/dkim7.jpg "Zona DNS")

Es recomendable una vez realizada la modificación en la zona DNS llevar a cabo una comprobación utilizando para ello una [herramienta](https://mxtoolbox.com/dkim.aspx) externa que compruebe que el registro _TXT_ es correcto, que nos devolverá unos resultados fácilmente interpretables:

![dkim1](https://i.ibb.co/Y45TyLk/dkim6.jpg "Comprobación DKIM")

Como se puede apreciar, en mi caso, el registro _TXT_ ha sido correctamente definido, sin embargo, esto no significa que mi máquina sea capaz de enviar firmados los correos salientes, ya que la herramienta previamente mencionada únicamente lleva a cabo comprobaciones sobre la zona DNS.

Para verificar que nuestro nuevo mecanismo de autenticación está funcionando correctamente, tendremos que reiniciar los servicios **Postfix** y **opendkim** para así aplicar los cambios llevados a cabo, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/opendkim/keys/iesgn19.es# systemctl restart opendkim postfix
{% endhighlight %}

En consecuencia de los nuevos cambios aplicados, se habrá abierto un _socket TCP/IP_ en el puerto **8892** que estará escuchando peticiones en la interfaz _loopback_, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:/etc/opendkim/keys/iesgn19.es# netstat -tlnp | egrep opendkim
tcp        0      0 127.0.0.1:8892          0.0.0.0:*               LISTEN      6851/opendkim
{% endhighlight %}

Efectivamente, el proceso está escuchando peticiones tal y como debería, de manera que todo está listo para enviar un correo al exterior. En este caso, voy a enviar un correo a la dirección **check-auth@verifier.port25.com**, que me responderá con otro correo electrónico mostrando un resumen de los resultados de las pruebas llevadas a cabo sobre el mismo, entre las que se encuentran una comprobación del mecanismo _SPF_ y otra para _DKIM_.

En mi caso, el resultado obtenido es el siguiente:

{% highlight shell %}
This message is an automatic response from Port25's authentication verifier
service at verifier.port25.com.  The service allows email senders to perform
a simple check of various sender authentication mechanisms.  It is provided
free of charge, in the hope that it is useful to the email community.  While
it is not officially supported, we welcome any feedback you may have at
<verifier-feedback@port25.com>.

Thank you for using the verifier,

The Port25 Solutions, Inc. team

==========================================================
Summary of Results
==========================================================
SPF check:          pass
"iprev" check:      pass
DKIM check:         pass

==========================================================
Details:
==========================================================

HELO hostname:  vps.iesgn19.es
Source IP:      51.210.109.246
mail-from:      root@iesgn19.es

----------------------------------------------------------
{% endhighlight %}

Como se puede apreciar, las directivas **SPF check** y **DKIM check** tienen el valor **pass**, lo que significa que la validación del remitente ha sido efectiva gracias a los registros _SPF_ y _DKIM_ que previamente hemos definido.

## Comprobación de SPF

Hasta ahora, todas las comprobaciones que se han llevado a cabo sobre los mecanismos _SPF_ y _DKIM_ han sido ejecutadas por los servidores de correo externos que recibían nuestros correos.

Sin embargo, nosotros nos encontramos "expuestos", al no haber implementado la comprobación de ningún tipo de autenticación de procedencia del correo, de manera que en este caso vamos a comprobar el registro _SPF_ para aquellos correos entrantes.

Para que **Postfix** lleve a cabo la comprobación del registro SPF del dominio origen del correo tendremos que instalar el paquete **postfix-policyd-spf-python**, pues es una funcionalidad extra que vamos a añadir a nuestro servidor de correo y viene empaquetado de una forma ajena. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# apt install postfix-policyd-spf-python
{% endhighlight %}

El hecho de haber instalado el nuevo paquete no supone que la comprobación del _SPF_ ya se esté llevando a cabo, ya que para ello necesitamos modificar los ficheros de configuración de **Postfix** para añadir las directivas necesarias. El primero que modificaremos será **/etc/postfix/master.cf**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/master.cf
{% endhighlight %}

Dentro del mismo, tendremos que incluir la siguiente directiva:

{% highlight shell %}
policyd-spf  unix  -    n       n       -       0       spawn
  user=policyd-spf argv=/usr/bin/policyd-spf
{% endhighlight %}

Gracias a la misma, se ejecutará un proceso en un _socket UNIX_ que será el que se utilice para el análisis del _SPF_. De otro lado, todavía necesitamos indicarle a nuestro servidor de correo qué debe hacer con aquellos correos que pasen el filtro, realizando una modificación en el fichero **/etc/postfix/main.cf**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/main.cf
{% endhighlight %}

Dentro del mismo, tendremos que incluir la siguiente directiva:

{% highlight shell %}
policyd-spf_time_limit = 3600
smtpd_recipient_restrictions = check_policy_service unix:private/policyd-spf
{% endhighlight %}

Gracias a la misma, hemos establecido un _timeout_, así como una restricción de manera que todos los correos entrantes han de pasar el filtro _SPF_ o de lo contrario serán rechazados.

Para verificar que nuestro nuevo mecanismo de autenticación está funcionando correctamente, tendremos que reiniciar el servicio **Postfix** para así aplicar los cambios llevados a cabo, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl restart postfix
{% endhighlight %}

Una vez que el servicio ha sido correctamente reiniciado, he procedido a enviar desde mi correo personal un correo electrónico a **root**, para así verificar que la comprobación del _SPF_ se lleva a cabo tal y como debería.

No considero necesario mostrar el proceso de envío, sino los _logs_ producidos por dicha recepción, que es donde podremos encontrar información relevante al respecto. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# cat /var/log/mail.log
Feb 15 17:09:20 vps postfix/smtpd[7345]: connect from mail-wr1-f51.google.com[209.85.221.51]
Feb 15 17:09:21 vps policyd-spf[7352]: prepend Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=209.85.221.51; helo=mail-wr1-f51.google.com; envelope-from=avacaferreras@gmail.com; receiver=<UNKNOWN>
Feb 15 17:09:21 vps postfix/smtpd[7345]: 1F3E4E1322: client=mail-wr1-f51.google.com[209.85.221.51]
Feb 15 17:09:21 vps postfix/cleanup[7355]: 1F3E4E1322: message-id=<CANR0p-1iDjvsBA3adavyRhB2uojRDW+yHFF-qFcqYNki-9k4mg@mail.gmail.com>
Feb 15 17:09:21 vps opendkim[6851]: 1F3E4E1322: s=20161025 d=gmail.com SSL
Feb 15 17:09:21 vps postfix/qmgr[7293]: 1F3E4E1322: from=<avacaferreras@gmail.com>, size=2899, nrcpt=1 (queue active)
Feb 15 17:09:21 vps postfix/local[7356]: 1F3E4E1322: to=<root@iesgn19.es>, relay=local, delay=0.39, delays=0.38/0.01/0/0, dsn=2.0.0, status=sent (delivered to mailbox)
Feb 15 17:09:21 vps postfix/qmgr[7293]: 1F3E4E1322: removed
Feb 15 17:09:21 vps postfix/smtpd[7345]: disconnect from mail-wr1-f51.google.com[209.85.221.51] ehlo=2 starttls=1 mail=1 rcpt=1 bdat=1 quit=1 commands=7
{% endhighlight %}

Como se puede apreciar en la segunda línea del resultado de la ejecución de dicho comando, el resultado del filtro ha sido correcto (**Received-SPF: Pass**), de manera que podemos asegurar que nuestro servidor de correo está teniendo en cuenta los registros _SPF_ de los dominios de los correos entrantes.

## Configuración de antispam

Los mensajes de correo no deseados (_spam_) están a la orden del día, de manera que siempre es conveniente aplicar algún que otro filtro que controle la recepción de dicho tipo de correos, para así evitar inundar nuestra bandeja de entrada de correo inútil.

En este caso, vamos a hacer uso de [SpamAssassin](https://spamassassin.apache.org/), un filtro de correo que añadiremos a **Postfix**, cuya principal intención como se puede suponer, es detectar mediante una serie de _tests_ y tratar de una determinada forma el correo _spam_.

Para el correcto funcionamiento de **SpamAssassin** tendremos que instalar los paquetes **spamassassin** y **spamc**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# apt install spamassassin spamc
{% endhighlight %}

En mi caso, una vez finalizada la instalación, el servicio **spamassassin** no se encontraba activo ni habilitado para arrancar junto al sistema, de manera que para solucionarlo ejecutaremos los siguientes comandos:

{% highlight shell %}
root@vps:~# systemctl start spamassassin
root@vps:~# systemctl enable spamassassin
{% endhighlight %}

Como cualquier otro _antispam_, **spamassassin** hace uso de una base de datos para realizar los respectivos _tests_ rutinarios cuando recibe un correo, de manera que tendremos que intentar que dicha base de datos se encuentre en todo momento lo más actualizada posible.

Para ello, tendremos que llevar a cabo una pequeña modificación en el fichero **/etc/default/spamassassin**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/default/spamassassin
{% endhighlight %}

Dentro del mismo, encontraremos una directiva que tendrá la siguiente forma:

{% highlight shell %}
CRON=0
{% endhighlight %}

Como se puede suponer, tendremos que modificar su valor a **1** para que actualice dicha base de datos una vez al día, que generalmente suele ocurrir durante la noche, ya que es cuando se supone que menos actividad hay, quedando el siguiente resultado final:

{% highlight shell %}
CRON=1
{% endhighlight %}

El hecho de haber instalado el nuevo paquete no supone que la comprobación del _spam_ ya se esté llevando a cabo, ya que para ello necesitamos modificar el fichero maestro de configuración de **Postfix** para añadir las directivas necesarias, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/master.cf
{% endhighlight %}

Dentro del mismo, encontraremos dos directivas que tendrán la siguiente forma:

{% highlight shell %}
smtp      inet  n       -       y       -       -       smtpd

submission inet n       -       y       -       -       smtpd
{% endhighlight %}

Tendremos que llevar a cabo una pequeña modificación sobre dichas directivas, para así indicar que todo el correo pase por **spamassassin** para llevar a cabo, como es lógico, las comprobaciones necesarias. Además, añadiremos una nueva directiva para el correcto funcionamiento del sistema _antispam_, quedando de la siguiente forma:

{% highlight shell %}
smtp      inet  n       -       y       -       -       smtpd
  -o content_filter=spamassassin

submission inet n       -       y       -       -       smtpd
  -o content_filter=spamassassin

spamassassin unix -     n       n       -       -       pipe
  user=debian-spamd argv=/usr/bin/spamc -f -e /usr/sbin/sendmail -oi -f ${sender} ${recipient}
{% endhighlight %}

Toda la configuración referente a **Postfix** ha finalizado, sin embargo, todavía no hemos indicado la forma en la que se tratarán a los correos que se marquen como no deseados.

Para ello, tendremos que modificar el fichero **/etc/spamassassin/local.cf**, cuya configuración puede ser todo lo compleja que deseemos para así afinarla a nuestras necesidades, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/spamassassin/local.cf
{% endhighlight %}

En mi caso, me limitaré a modificar el asunto del correo en cuestión para añadirle el encabezado "__*****SPAM*****__", de manera que tendré que localizar la directiva con la siguiente forma:

{% highlight shell %}
# rewrite_header Subject *****SPAM*****
{% endhighlight %}

Tras ello, tendremos que descomentarla, quedando de la siguiente manera:

{% highlight shell %}
rewrite_header Subject *****SPAM*****
{% endhighlight %}

Para verificar que nuestro nuevo mecanismo _antispam_ está funcionando correctamente, tendremos que reiniciar los servicios **Postfix** y **spamassassin** para así aplicar los cambios llevados a cabo, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl restart postfix spamassassin
{% endhighlight %}

Una vez que los servicios han sido correctamente reiniciados, he procedido a enviar desde mi correo personal un correo electrónico a **root**, para así verificar que la comprobación del _spam_ se lleva a cabo tal y como debería, indicando para ello en el cuerpo del mensaje un texto que se utiliza para comprobar la eficacia de dichos sistemas:

{% highlight shell %}
XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X
{% endhighlight %}

No considero necesario mostrar el proceso de envío, sino los _logs_ producidos por dicha recepción, que es donde podremos encontrar información relevante al respecto. Para ello, haremos uso del comando:

{% highlight shell %}
root@vps:~# cat /var/log/mail.log
Feb 15 17:42:08 vps postfix/smtpd[8886]: connect from mail-wm1-f52.google.com[209.85.128.52]
Feb 15 17:42:08 vps policyd-spf[8894]: prepend Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=209.85.128.52; helo=mail-wm1-f52.google.com; envelope-from=avacaferreras@gmail.com; receiver=<UNKNOWN>
Feb 15 17:42:08 vps postfix/smtpd[8886]: F3106E1325: client=mail-wm1-f52.google.com[209.85.128.52]
Feb 15 17:42:08 vps postfix/cleanup[8898]: F3106E1325: message-id=<CANR0p-3rtCkGzxQfesenfvWfO-gL_2Q6duu-U2pWG=qtg+1VQg@mail.gmail.com>
Feb 15 17:42:08 vps opendkim[6851]: F3106E1325: s=20161025 d=gmail.com SSL
Feb 15 17:42:09 vps postfix/qmgr[8769]: F3106E1325: from=<avacaferreras@gmail.com>, size=2908, nrcpt=1 (queue active)
Feb 15 17:42:09 vps spamd[8772]: spamd: connection from ::1 [::1]:34624 to port 783, fd 5
Feb 15 17:42:09 vps spamd[8772]: spamd: setuid to debian-spamd succeeded
Feb 15 17:42:09 vps spamd[8772]: spamd: processing message <CANR0p-3rtCkGzxQfesenfvWfO-gL_2Q6duu-U2pWG=qtg+1VQg@mail.gmail.com> for debian-spamd:114
Feb 15 17:42:09 vps postfix/smtpd[8886]: disconnect from mail-wm1-f52.google.com[209.85.128.52] ehlo=2 starttls=1 mail=1 rcpt=1 bdat=1 quit=1 commands=7
Feb 15 17:42:09 vps spamd[8772]: spamd: identified spam (999.9/5.0) for debian-spamd:114 in 0.2 seconds, 2979 bytes.
Feb 15 17:42:09 vps spamd[8772]: spamd: result: Y 999 - DKIM_SIGNED,DKIM_VALID,DKIM_VALID_AU,FREEMAIL_FROM,GTUBE,HTML_MESSAGE,RCVD_IN_MSPIKE_H2,SPF_PASS,TVD_SPACE_RATIO scantime=0.2,size=2979,user=debian-spamd,uid=114,required_score=5.0,rhost=::1,raddr=::1,rport=34624,mid=<CANR0p-3rtCkGzxQfesenfvWfO-gL_2Q6duu-U2pWG=qtg+1VQg@mail.gmail.com>,autolearn=no autolearn_force=no
Feb 15 17:42:09 vps postfix/pickup[8768]: 4119BE13D0: uid=114 from=<avacaferreras@gmail.com>
Feb 15 17:42:09 vps postfix/cleanup[8898]: 4119BE13D0: message-id=<CANR0p-3rtCkGzxQfesenfvWfO-gL_2Q6duu-U2pWG=qtg+1VQg@mail.gmail.com>
Feb 15 17:42:09 vps postfix/pipe[8899]: F3106E1325: to=<root@iesgn19.es>, relay=spamassassin, delay=0.36, delays=0.13/0/0/0.23, dsn=2.0.0, status=sent (delivered via spamassassin service)
Feb 15 17:42:09 vps postfix/qmgr[8769]: F3106E1325: removed
Feb 15 17:42:09 vps postfix/qmgr[8769]: 4119BE13D0: from=<avacaferreras@gmail.com>, size=6217, nrcpt=1 (queue active)
Feb 15 17:42:09 vps postfix/local[8903]: 4119BE13D0: to=<root@iesgn19.es>, relay=local, delay=0.01, delays=0.01/0/0/0, dsn=2.0.0, status=sent (delivered to mailbox)
Feb 15 17:42:09 vps postfix/qmgr[8769]: 4119BE13D0: removed
Feb 15 17:42:09 vps spamd[8695]: prefork: child states: II
{% endhighlight %}

Como se puede apreciar en el resultado de la ejecución de dicho comando, primero se ha comprobado el _SPF_ del remitente y una vez que se ha validado, el mensaje ha pasado a **spamd**, el cuál ha determinado que el mensaje es _spam_ (**identified spam**), asignándole a su vez una puntuación (que en este caso es la más alta, ya que el mensaje está pensado para ello). A pesar de ello y gracias a la configuración establecida, el correo ha sido entregado en el buzón de igual forma.

Para vericarlo, volveremos a hacer uso de la utilidad de línea de comandos, recibiendo por pantalla el siguiente resultado:

{% highlight shell %}
root@vps:~# mail
Mail version 8.1.2 01/15/2001.  Type ? for help.
"/var/mail/root": 6 messages 2 new 6 unread
 U  1 root@iesgn19.es    Thu Jan 21 09:33   23/693   Cron <root@vps> date
 U  2 root@iesgn19.es    Thu Jan 21 09:34   23/693   Cron <root@vps> date
 U  3 root@iesgn19.es    Thu Jan 21 09:35   23/693   Cron <root@vps> date
 U  4 avacaferreras@gma  Mon Feb 15 17:09   61/3136  Prueba SPF
>N  5 auth-results@veri  Mon Feb 15 17:11  232/9769  Authentication Report
 N  6 avacaferreras@gma  Mon Feb 15 17:42  128/6215  *****SPAM***** Prueba SPAM
{% endhighlight %}

Efectivamente, el último mensaje recibido ha sido marcado como _spam_ por **spamassassin** y por tanto, se le ha añadido la cabecera "_*****SPAM*****_" al asunto de dicho mensaje, para identificarlo fácilmente.

Esta medida es una de las menos restrictivas, ya que lo único que hace es marcar los mensajes detectados como _spam_, aunque en caso de así necesitarlo, podríamos descartar directamente dichos correos para que no se entregasen en el buzón.

## Configuración de antivirus

Los mensajes de correo con virus están a la orden del día, de manera que siempre es conveniente aplicar algún que otro filtro que controle la recepción de dicho tipo de correos, para así evitar inundar nuestra bandeja de entrada de correo inútil.

En este caso, vamos a hacer uso de [ClamAV](https://www.clamav.net/), un filtro de correo que añadiremos a **Postfix**, cuya principal intención como se puede suponer, es detectar mediante una serie de _tests_ y tratar de una determinada forma el correo infectado.

Para el correcto funcionamiento de **ClamAV** tendremos que instalar los paquetes **clamsmtp** y **clamav-daemon**, así como una serie de paquetería que permite tratar los ficheros comprimidos, para así poder detectar también virus en los mismos, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# apt install clamsmtp clamav-daemon arc arj bzip2 cabextract lzop nomarch p7zip pax tnef unrar-free unzip
{% endhighlight %}

El paquete **clamsmtp** ha sido instalado en la máquina, existiendo en consecuencia un proceso que está actualmente en ejecución, que habrá abierto un _socket TCP/IP_ en el puerto **25** que estará escuchando peticiones en la interfaz _loopback_, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tlnp | egrep clamsmtp
tcp        0      0 127.0.0.1:10026         0.0.0.0:*               LISTEN      11870/clamsmtpd
{% endhighlight %}

Efectivamente, el proceso está escuchando peticiones tal y como debería.

En mi caso, una vez finalizada la instalación, el demonio **clamav-daemon** no se encontraba activo, de manera que para solucionarlo haremos uso del siguiente comando:

{% highlight shell %}
root@vps:~# systemctl start clamav-daemon
{% endhighlight %}

El hecho de haber instalado el nuevo paquete no supone que la comprobación de los virus ya se esté llevando a cabo, ya que para ello necesitamos modificar los ficheros de configuración de **Postfix** para añadir las directivas necesarias. El primero que modificaremos será **/etc/postfix/master.cf**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/master.cf
{% endhighlight %}

Dentro del mismo, tendremos que incluir las siguientes directivas:

{% highlight shell %}
scan unix -       -       n       -       16       smtp
  -o smtp_data_done_timeout=1200
  -o smtp_send_xforward_command=yes
  -o disable_dns_lookups=yes
127.0.0.1:10025 inet n       -       n       -       16       smtpd
  -o content_filter=
  -o local_recipient_maps=
  -o relay_recipient_maps=
  -o smtpd_restriction_classes=
  -o smtpd_client_restrictions=
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_mynetworks,reject
  -o mynetworks_style=host
  -o smtpd_authorized_xforward_hosts=127.0.0.0/8
{% endhighlight %}

Gracias a la misma, habremos indicado que todo el correo pase por **clamav** para llevar a cabo, como es lógico, las comprobaciones necesarias. De otro lado, todavía necesitamos indicarle a nuestro servidor de correo dónde debe comunicarse con el servicio encargado de llevar a cabo las comprobaciones sobre dichos correos, realizando una modificación en el fichero **/etc/postfix/main.cf**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/main.cf
{% endhighlight %}

Dentro del mismo, tendremos que incluir la siguiente directiva:

{% highlight shell %}
content_filter = scan:127.0.0.1:10026
{% endhighlight %}

Para verificar que nuestro nuevo mecanismo _antivirus_ está funcionando correctamente, tendremos que reiniciar el servicio **Postfix** para así aplicar los cambios llevados a cabo, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl restart postfix
{% endhighlight %}

Una vez que el servicio ha sido correctamente reiniciado, he procedido a enviar desde mi correo personal un correo electrónico a **root**, para así verificar que la comprobación de los virus se lleva a cabo tal y como debería, indicando para ello en el cuerpo del mensaje un texto que se utiliza para comprobar la eficacia de dichos sistemas:

{% highlight shell %}
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
{% endhighlight %}

No considero necesario mostrar el proceso de envío, sino los _logs_ producidos por dicha recepción, que es donde podremos encontrar información relevante al respecto. Para ello, haremos uso del comando:

{% highlight shell %}
Feb 15 18:15:49 vps postfix/smtpd[12662]: connect from mail-wm1-f54.google.com[209.85.128.54]
Feb 15 18:15:49 vps policyd-spf[12665]: prepend Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=209.85.128.54; helo=mail-wm1-f54.google.com; envelope-from=avacaferreras@gmail.com; receiver=<UNKNOWN>
Feb 15 18:15:49 vps postfix/smtpd[12662]: E4F3CE13D1: client=mail-wm1-f54.google.com[209.85.128.54]
Feb 15 18:15:49 vps postfix/cleanup[12669]: E4F3CE13D1: message-id=<CANR0p-1KHMr3jt8FPXziTmAwk-nOxs0Cx-CdTHEiX=5CzM3PBg@mail.gmail.com>
Feb 15 18:15:49 vps opendkim[6851]: E4F3CE13D1: s=20161025 d=gmail.com SSL
Feb 15 18:15:49 vps postfix/qmgr[12612]: E4F3CE13D1: from=<avacaferreras@gmail.com>, size=3230, nrcpt=1 (queue active)
Feb 15 18:15:49 vps spamd[8772]: spamd: connection from ::1 [::1]:34804 to port 783, fd 5
Feb 15 18:15:49 vps spamd[8772]: spamd: setuid to debian-spamd succeeded
Feb 15 18:15:49 vps spamd[8772]: spamd: processing message <CANR0p-1KHMr3jt8FPXziTmAwk-nOxs0Cx-CdTHEiX=5CzM3PBg@mail.gmail.com> for debian-spamd:114
Feb 15 18:15:49 vps postfix/smtpd[12662]: disconnect from mail-wm1-f54.google.com[209.85.128.54] ehlo=2 starttls=1 mail=1 rcpt=1 bdat=1 quit=1 commands=7
Feb 15 18:15:50 vps spamd[8772]: spamd: clean message (-0.1/5.0) for debian-spamd:114 in 0.2 seconds, 3297 bytes.
Feb 15 18:15:50 vps spamd[8772]: spamd: result: . 0 - DKIM_SIGNED,DKIM_VALID,DKIM_VALID_AU,FREEMAIL_FROM,HTML_MESSAGE,RCVD_IN_MSPIKE_H2,SPF_PASS,TVD_SPACE_RATIO scantime=0.2,size=3297,user=debian-spamd,uid=114,required_score=5.0,rhost=::1,raddr=::1,rport=34804,mid=<CANR0p-1KHMr3jt8FPXziTmAwk-nOxs0Cx-CdTHEiX=5CzM3PBg@mail.gmail.com>,autolearn=ham autolearn_force=no
Feb 15 18:15:50 vps postfix/pickup[12611]: 34FC3E14DC: uid=114 from=<avacaferreras@gmail.com>
Feb 15 18:15:50 vps postfix/cleanup[12669]: 34FC3E14DC: message-id=<CANR0p-1KHMr3jt8FPXziTmAwk-nOxs0Cx-CdTHEiX=5CzM3PBg@mail.gmail.com>
Feb 15 18:15:50 vps postfix/pipe[12670]: E4F3CE13D1: to=<root@iesgn19.es>, relay=spamassassin, delay=0.32, delays=0.08/0/0/0.24, dsn=2.0.0, status=sent (delivered via spamassassin service)
Feb 15 18:15:50 vps postfix/qmgr[12612]: E4F3CE13D1: removed
Feb 15 18:15:50 vps opendkim[6851]: 34FC3E14DC: s=20161025 d=gmail.com SSL
Feb 15 18:15:50 vps spamd[8695]: prefork: child states: II
Feb 15 18:15:50 vps postfix/qmgr[12612]: 34FC3E14DC: from=<avacaferreras@gmail.com>, size=3806, nrcpt=1 (queue active)
Feb 15 18:15:50 vps clamsmtpd: 100007: accepted connection from: 127.0.0.1
Feb 15 18:15:50 vps postfix/smtpd[12679]: connect from localhost[127.0.0.1]
Feb 15 18:15:50 vps postfix/smtpd[12679]: 4BCE6E13D1: client=localhost[127.0.0.1]
Feb 15 18:15:50 vps postfix/smtp[12677]: 34FC3E14DC: to=<root@iesgn19.es>, relay=127.0.0.1[127.0.0.1]:10026, delay=0.1, delays=0.05/0/0.04/0, dsn=2.0.0, status=sent (250 Virus Detected; Discarded Email)
Feb 15 18:15:50 vps postfix/qmgr[12612]: 34FC3E14DC: removed
Feb 15 18:15:50 vps clamsmtpd: 100007: from=avacaferreras@gmail.com, to=root@iesgn19.es, status=VIRUS:Eicar-Signature
Feb 15 18:15:50 vps postfix/smtpd[12679]: disconnect from localhost[127.0.0.1] ehlo=1 xforward=1 mail=1 rcpt=1 rset=1 quit=1 commands=6
{% endhighlight %}

Como se puede apreciar en el resultado de la ejecución de dicho comando, primero se ha comprobado el _SPF_ del remitente y una vez que se ha validado, el mensaje ha pasado a **spamd**, el cuál ha determinado que el mensaje tampoco es _spam_ (**clean message**), pasando por último a actuar **clamsmtpd**, que finalmente ha descartado el mensaje al detectar que se trata de un virus (**250 Virus Detected; Discarded Email**).

## Configuración de Maildir

Aunque no lo hayamos mencionado con profundidad hasta ahora, el tipo de buzón que por defecto utiliza **Postfix** es **mbox**, aquel que almacena todos los mensajes en un fichero y que se utiliza para el protocolo _POP3_, en el que se descargan todos los correos desde el servidor.

Sin embargo, la idea de este artículo es dejar de hacer uso del servidor de correo de forma local, y empezar a utilizar un cliente que se conecte al mismo mediante el protocolo _SMTP_ para el envío de correos y mediante el protocolo _IMAP_ para la sincronización de los mismos, dadas las ventajas que éste último nos ofrece.

La característica del protocolo _IMAP_ es que únicamente permite la sincronización con los buzones de tipo **Maildir**, aquel que almacena los mensajes en un directorio con sus correspondientes subdirectorios.

Para llevar a cabo el cambio de buzón de tipo **mbox** a uno de tipo **Maildir**, tendremos que modificar el fichero de configuración principal de **Postfix**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/main.cf
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la siguiente forma:

{% highlight shell %}
home_mailbox = Maildir/
{% endhighlight %}

Gracias a la misma, una vez que reiniciemos el servicio y los cambios surtan efecto, los nuevos mensajes de correo recibidos se almacenarán de forma automática en un directorio de nombre **Maildir/** en el directorio personal de cada uno de los usuarios existentes en la máquina, de manera que reiniciaremos el servicio haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl restart postfix
{% endhighlight %}

A partir de este momento, no podremos leer los correos recibidos a través de la herramienta de línea de comandos `mail`, ya que no soporta de forma nativa este tipo de buzón. Para solventarlo, tendremos que instalar un nuevo cliente de línea de comandos, como por ejemplo **mutt**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# apt install mutt
{% endhighlight %}

A pesar de haber instalado el nuevo cliente, este no se encuentra todavía configurado para buscar los mensajes de correo dentro de **~/Maildir/**, pues tendremos que indicarle dicho directorio de forma explícita en un fichero de nombre **~/.muttrc**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano ~/.muttrc
{% endhighlight %}

Dentro del mismo, introduciremos el siguiente contenido:

{% highlight shell %}
set mbox_type=Maildir
set folder="~/Maildir"
set mask="!^\\.[^.]"
set mbox="~/Maildir"
set record="+.Sent"
set postponed="+.Drafts"
set spoolfile="~/Maildir"
{% endhighlight %}

**Nota**: En caso de haber indicado un nombre de directorio diferente a **~/Maildir** en la configuración de **Postfix**, deberá modificarse la ruta del mismo a la hora de establecer el valor de las directivas mostradas.

Toda la configuración necesaria para hacer uso del nuevo tipo de buzón ha finalizado, de manera que he procedido a enviar desde mi correo personal un correo electrónico a **root**, para así verificar que los nuevos correos entrantes se sitúan en un directorio de nombre **~/Maildir**, tal y como deberían.

No considero necesario mostrar el proceso de envío, sino el nuevo contenido del directorio mencionado producido por dicha recepción, concretamente en el subdirectorio de nombre **new/**, que es donde podremos encontrar información relevante al respecto. Para ello, haremos uso del comando:

{% highlight shell %}
root@vps:~# ls -l Maildir/new/
total 4
-rw------- 1 root root 3548 Feb 16 08:20 1613460040.V801I42733M857089.vps
{% endhighlight %}

Como era de esperar y como resultado de la recepción de un nuevo correo electrónico, ha aparecido un nuevo fichero en el correspondiente directorio del usuario **root** que hace referencia al mismo.

Tal y como ya hemos mencionado, no podremos visualizar el contenido de dicho correo con la herramienta `mail`, sino que procederemos a usar el nuevo cliente de línea de comandos `mutt` para ello, ejecutando por lo tanto el comando:

{% highlight shell %}
root@vps:~# mutt
{% endhighlight %}

Tras ello, se nos mostrará lo siguiente por pantalla:

![mutt1](https://i.ibb.co/qW1QGt2/mutt2.jpg "Mutt")

Efectivamente, únicamente se ha mostrado un correo electrónico con asunto "**Prueba mutt**", correspondiente al único fichero existente en el directorio **~/Maildir/new/**, de manera que pulsaremos **INTRO** para visualizar su contenido y así verificar su correcto funcionamiento:

![mutt2](https://i.ibb.co/T0fS1yC/mutt3.jpg "Mutt")

Como era de esperar, el contenido del correo electrónico recibido ha sido correctamente mostrado, por lo que podemos concluir que tanto el nuevo tipo de buzón como el nuevo cliente de línea de comandos están funcionando tal y como deberían.

## Recepción de correos desde el cliente

Como previamente hemos mencionado, si utilizamos un cliente de correo (_MUA_) externo para leer el correo guardado en el servidor de correos podemos usar dos protocolos de comunicación:

* **POP3 (_Post Office Protocol_)**: Protocolo para recuperar correos electrónicos de un _MDA_. Su principal característica es que se descargan todos los correos.

* **IMAP (_Internet Message Access Protocol_)**: Protocolo para recuperar correos electrónicos de un _MDA_. A diferencia del anterior, se sincroniza el estado de los correos entre el servidor y el cliente.

En nuestro caso, vamos a trabajar con _IMAP_, que es el protocolo actualmente utilizado para poder leer nuestro correo desde distintos clientes de correos al no descargar todos los correos del buzón, como ocurre con _POP3_.

Además, por seguridad, es recomendable que la conexión a través de estos protocolos, sea cuál sea el que hayamos elegido, se establezca de forma autenticada y cifrada. Por defecto, el protocolo _IMAP_ hace uso de puerto **143/TCP** y lleva a cabo una conexión autenticada, sin embargo, no se encuentra cifrada. Para cifrar dicha comunicación, podemos acudir a las siguientes alternativas:

* **IMAP con STARTTLS**: STARTTLS transforma una conexión insegura en una segura mediante el uso de SSL/TLS, consiguiendo por tanto tener una conexión cifrada a través del puerto **143/TCP**.

* **IMAPS**: Versión segura del protocolo _IMAP_ que hace uso del puerto **993/TCP**.

Como es lógico, necesitaremos además un servicio como **dovecot** que actúe como servidor _IMAP_, por lo que procederemos a su instalación haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# apt install dovecot-imapd
{% endhighlight %}

El paquete **dovecot-imapd** ha sido instalado en la máquina, existiendo en consecuencia un proceso que está actualmente en ejecución, que habrá abierto dos _sockets TCP/IP_ en los puertos por defecto de los protocolos _IMAP_ e _IMAPS_ (**143** y **993**) que estará escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tlnp | egrep dovecot
tcp        0      0 0.0.0.0:993             0.0.0.0:*               LISTEN      19744/dovecot
tcp        0      0 0.0.0.0:143             0.0.0.0:*               LISTEN      19744/dovecot
tcp6       0      0 :::993                  :::*                    LISTEN      19744/dovecot
tcp6       0      0 :::143                  :::*                    LISTEN      19744/dovecot
{% endhighlight %}

Efectivamente, el proceso está escuchando peticiones tal y como debería. Sin embargo, como bien sabemos, si queremos hacer uso de un protocolo cifrado, necesitaremos generar un certificado firmado por una autoridad certificadora de considerable reputación, así que en mi caso, he recurrido una vez más a **Let's Encrypt**, siguiendo para ello los pasos indicados en el artículo [Configuración de HTTPS en Nginx](https://www.alvarovf.com/seguridad/vps/2020/11/30/configuracion-https-nginx.html), pues mencionar aquí el proceso se saldría completamente del objetivo del _post_.

Sin entrar en demasiado detalle, el funcionamiento del protocolo _IMAPS_ es muy similar al del protocolo _HTTPS_, ya que contaremos con una clave privada y una clave pública (certificado), siendo esta última la que se envía al cliente que trata de realizar la conexión para así ponerse de acuerdo en la clave simétrica que se va a utilizar para cifrar la conexión, pues mantener en todo momento una conexión cifrada asimétricamente sería muy costoso computacionalmente.

Cuando nuestro certificado para el nombre de dominio **mail.iesgn19.es** haya sido correctamente generado, todo estará listo para configurar **dovecot** para que haga uso de los mismos, de manera que comenzaremos por modificar el fichero **/etc/dovecot/conf.d/10-ssl.conf**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/dovecot/conf.d/10-ssl.conf
{% endhighlight %}

Dentro del mismo, tendremos que localizar las directivas que por defecto tienen la siguiente forma:

{% highlight shell %}
ssl_cert = </etc/dovecot/private/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.key
{% endhighlight %}

Donde:

* **ssl_cert**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/letsencrypt/live/mail.iesgn19.es/fullchain.pem**.
* **ssl_key**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/letsencrypt/live/mail.iesgn19.es/privkey.pem**.

El resultado final, tras llevar a cabo las correspondientes modificaciones sería:

{% highlight shell %}
ssl_cert = </etc/letsencrypt/live/mail.iesgn19.es/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.iesgn19.es/privkey.pem
{% endhighlight %}

A pesar de haber indicado ya el certificado que ha de usarse para la conexión cifrada, el servicio no se encuentra todavía configurado para buscar los mensajes de correo dentro de **~/Maildir/** para su correspondiente sincronización con el cliente, pues tendremos que indicarle dicho directorio de forma explícita en el fichero de nombre **/etc/dovecot/conf.d/10-mail.conf**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/dovecot/conf.d/10-mail.conf
{% endhighlight %}

Dentro del mismo, tendremos que localizar la directiva que por defecto tenga la siguiente forma:

{% highlight shell %}
mail_location = mbox:~/mail:INBOX=/var/mail/%u
{% endhighlight %}

En nuestro caso, al estar haciendo uso de buzones del tipo **Maildir**, tendremos que modificar dicha directiva para que finalmente tenga el siguiente valor:

{% highlight shell %}
mail_location = maildir:~/Maildir
{% endhighlight %}

Nuestra configuración del servicio **dovecot** para hacer uso del protocolo _IMAP_ ha finalizado, de manera que tendremos que reiniciar dicho servicio para que los cambios llevados a cabo surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl restart dovecot
{% endhighlight %}

En lugar de llevar a cabo una prueba de dicho protocolo, vamos a realizar también las configuraciones necesarias para enviar correos desde un cliente, de manera que posteriormente realizaremos todos los _tests_ oportunos de manera conjunta.

## Envío de correos desde el cliente

Como previamente hemos mencionado, para el envío de correos entre _MTA_ se utiliza el protocolo _SMTP_ en el puerto **25/TCP**, sin embargo, si vamos a utilizar un cliente remoto para el envío de correos, se utilizan dos opciones distintas:

* **ESMTP + STARTTLS (_Enhanced Simple Mail Transfer Protocol_)**: STARTTLS transforma una conexión insegura en una segura mediante el uso de SSL/TLS, consiguiendo por tanto tener una conexión cifrada a través del puerto **587/TCP**.

Dicho puerto es conocido como puerto de _submission_ o presentación. Al abrir este puerto, **Postfix** esta funcionando como _MSA_ (_Mail Submission Agent_) que recibe mensajes de correo electrónico desde un _MUA_ (_Mail User Agent_) y coopera con un _MTA_ (_Mail Transport Agent_) para entregar el correo.

Tenemos que conseguir que la comunicación que se establece desde el cliente al servidor sea autenticada. Para ello, utilizamos _SASL_ (_Simple Authentication and Security Layer_), un _framework_ para autenticación y autorización en protocolos de Internet. Para realizar la autenticación vamos a usar **dovecot** (que ya tiene un mecanismo de autenticación).

De otro lado, tenemos que conseguir además que la comunicación sea cifrada, para ello vamos a utilizar _STARTTLS_ que nos permite que utilizando el mismo puerto (**587/TCP**), la conexión sea cifrada.

* **SMTPS (_Simple Mail Transfer Protocol Secure_)**: Protocolo para conseguir el cifrado de la comunicación entre el cliente y el servidor. Utiliza el puerto **465/TCP**. No es una extensión de _SMTP_. Es muy parecido a _HTTPS_.

En este caso, vamos a reutilizar el certificado generado en el anterior apartado para cifrar la conexión _SMTP_ entre el cliente y el servidor, de manera que todo estará listo para configurar **postfix** para que haga uso de los mismos, de manera que comenzaremos por modificar el fichero **/etc/postfix/main.cf**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/main.cf
{% endhighlight %}

Dentro del mismo, tendremos que localizar las directivas que por defecto tienen la siguiente forma:

{% highlight shell %}
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
{% endhighlight %}

Donde:

* **smtpd_tls_cert_file**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/letsencrypt/live/mail.iesgn19.es/fullchain.pem**.
* **smtpd_tls_key_file**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/letsencrypt/live/mail.iesgn19.es/privkey.pem**.

Además de indicar las rutas de los determinados ficheros, tendremos que añadir una serie de directivas para el correcto funcionamiento del sistema de autenticación por parte de **dovecot**. El resultado final, tras llevar a cabo las correspondientes modificaciones sería:

{% highlight shell %}
smtpd_tls_cert_file=/etc/letsencrypt/live/mail.iesgn19.es/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/mail.iesgn19.es/privkey.pem

smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_authenticated_header = yes
broken_sasl_auth_clients = yes
{% endhighlight %}

A pesar de haber indicado ya el certificado que ha de usarse para la conexión cifrada, el servicio no se encuentra todavía configurado para utilizar los puertos **587/TCP** y **465/TCP**, pues tendremos que indicárselo de forma explícita en el fichero de nombre **/etc/postfix/master.cf**, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# nano /etc/postfix/master.cf
{% endhighlight %}

Dentro del mismo, tendremos que buscar las directivas **submission** y **smtps** y descomentarlas al completo, quedando de la siguiente forma:

{% highlight shell %}
submission inet n       -       y       -       -       smtpd
  -o content_filter=spamassassin
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
{% endhighlight %}

Si recordamos, a la hora de especificarle la ruta de los ficheros del certificado a **Postfix**, hemos introducido las directivas necesarias para que **dovecot** pueda realizar la autenticación, sin embargo, todavía faltaría indicarle a **dovecot** cómo tiene que realizar dicha autenticación, de manera que procederemos a modificar el fichero **/etc/dovecot/conf.d/10-master.conf** para así poder interconectar ambos servicios, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/dovecot/conf.d/10-master.conf
{% endhighlight %}

Dentro del mismo, tendremos que localizar la directiva que por defecto tenga la siguiente forma:

{% highlight shell %}
unix_listener auth-userdb {
}
{% endhighlight %}

En este caso, la ruta del _socket UNIX_ que se utilizará para la comunicación entre ambos servicios no es correcta, ni tampoco lo son los permisos asignados, de manera que tendremos que modificarla para que finalmente tenga el siguiente aspecto:

{% highlight shell %}
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
}
{% endhighlight %}

Nuestra configuración de los servicios **Postfix** y **dovecot** para hacer uso del protocolo _SMTP_ de forma segura ha finalizado, de manera que tendremos que reiniciar dichos servicios para que los cambios llevados a cabo surtan efecto, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl restart postfix dovecot
{% endhighlight %}

En consecuencia de los nuevos cambios aplicados, se habrán abierto dos _sockets TCP/IP_ en los puertos **587** y **465** que estarán escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tln | egrep '(587|465)'
tcp        0      0 0.0.0.0:587             0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:465             0.0.0.0:*               LISTEN
tcp6       0      0 :::587                  :::*                    LISTEN
tcp6       0      0 :::465                  :::*                    LISTEN
{% endhighlight %}

Efectivamente, el proceso está escuchando peticiones tal y como debería, de manera que todo está listo para enviar un correo desde un cliente externo.

## Configuración de cliente Thunderbird

Una vez configurados los dos protocolos necesarios para recibir y enviar correos desde un cliente externo, pues hasta ahora hemos estado llevando a cabo ambas acciones directamente desde el servidor, vamos a proceder a la configuración de un cliente **Thunderbird**, que nos permitirá llevar a cabo dichas acciones cotidianas de una forma mucho más "amigable" que una utilidad de línea de comandos.

El primer paso, como es lógico, consistirá en instalar el paquete **thunderbird**, aunque podríamos hacer uso de otro cliente distinto como **Evolution**, que viene instalado por defecto. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@debian:~# apt install thunderbird
{% endhighlight %}

Tras ello, podremos ejecutar la aplicación y se nos abrirá una ventana emergente en la que podremos llevar a cabo la configuración inicial con nuestro servidor de correo. En mi caso, el resultado final sería el siguiente:

![thunderbird1](https://i.ibb.co/YyQX8tF/thunderbird1.jpg "Thunderbird")

Como se puede apreciar, hemos establecido una configuración básica como el nombre a mostrar en los correos enviados, la dirección de correo electrónico, la contraseña, el servidor, los puertos a utilizar para cada uno de los protocolos...

Una vez establecida la configuración correcta, pulsaremos en "**Done**" para así añadir la cuenta de correo electronico a nuestro cliente gráfico. En mi caso, he procedido a enviar desde mi correo personal un correo electrónico a **debian**, para así verificar que la recepción del mismo se lleva a cabo tal y como debería.

No considero necesario mostrar el proceso de envío, sino el resultado final producido por dicha recepción, que sería el siguiente:

![thunderbird2](https://i.ibb.co/2M14SNS/thunderbird2.jpg "Thunderbird")

Como era de esperar, el correo electrónico ha sido correctamente recibido en nuestro cliente **Thunderbird**, por lo que podemos concluir que el protocolo _IMAP_ se encuentra funcionando tal y como debería a través del puerto **993/TCP**, de manera que se están sincronizando correctamente los mensajes de correo con el servidor.

Todavía nos falta comprobar que el protocolo _SMTP_ también está funcionando correctamente, de manera que llevaremos ahora la prueba a la inversa, enviando desde nuestro cliente un correo electrónico a mi correo personal, de la siguiente forma:

![thunderbird3](https://i.ibb.co/PFM7zJT/thunderbird3.jpg "Thunderbird")

Tras pulsar el botón "**Send**" no se me ha mostrado ningún error por pantalla, de manera que puedo decir con casi total seguridad que el envío se ha producido correctamente. Sin embargo, hasta que no lo verifiquemos en mi correo personal, no tendremos la certeza de ello:

![gmail5](https://i.ibb.co/zSKcksd/thunderbird4.jpg "Gmail")

Efectivamente, el correo ha sido correctamente recibido en _Gmail_, por lo que podemos concluir que el protocolo _SMTP_ se encuentra funcionando tal y como debería a través del puerto **465/TCP**, de manera que se están enviando correctamente los mensajes de correo a través del servidor.

## Instalación de webmail

En un ámbito empresarial, quizás puede ser más común el hecho de utilizar una aplicación centralizada que permita gestionar el correo del equipo mediante una interfaz web, ya que así no es necesario ir configurando el cliente para todos y cada uno de los equipos existentes en la empresa.

Para alcanzar este propósito, he decidido hacer uso del servidor web **nginx** alojado en la máquina VPS que se encuentra actualmente escuchando peticiones en los puertos **80** (HTTP) y **443** (HTTPS), de manera que crearemos un VirtualHost accesible desde un determinado nombre de dominio y serviremos a través del mismo una aplicación web de correo (_webmail_), de nombre **Roundcube**.

El primer paso ha consistido en la generación de un nuevo nombre dentro de la zona DNS **iesgn19.es**, concretamente nombrando un nuevo servicio **webmail** mediante un registro _CNAME_ al registro _A_ que apunta a la dirección **51.210.109.246**, que será a través del cuál accederemos al VirtualHost en cuestión, tal y como podemos apreciar a continuación:

![dns5](https://i.ibb.co/56RMN06/webmail9.jpg "Zona DNS")

Como bien sabemos, si queremos hacer uso del protocolo HTTPS para dicho nombre de dominio, necesitaremos generar un nuevo certificado firmado por una autoridad certificadora de considerable reputación, de la misma forma que previamente lo hemos hecho para el nombre **mail.iesgn19.es**.

Cuando nuestro certificado haya sido correctamente generado, todo estará listo para configurar el nuevo VirtualHost dentro del directorio **/etc/nginx/sites-available/**, cuyo nombre he decidido que en este caso sea **roundcube**, de manera que ejecutaremos el comando:

{% highlight shell %}
root@vps:~# nano /etc/nginx/sites-available/roundcube
{% endhighlight %}

Dentro del mismo, tendremos que crear dos directivas **server**, una para el VirtualHost accesible en el puerto 80 HTTP, dentro del cuál únicamente asignaremos el ServerName correspondiente y una redirección permanente para así forzar HTTPS. En la otra, para el VirtualHost accesible en el puerto 443 HTTPS, tendremos que configurar las siguientes directivas:

* **server_name**: Indicaremos el nombre de dominio a través del cuál accederemos al servidor.
* **ssl**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **ssl_certificate**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/letsencrypt/live/webmail.iesgn19.es/fullchain.pem**.
* **ssl_certificate_key**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/letsencrypt/live/webmail.iesgn19.es/privkey.pem**.

El resultado final sería:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        server_name webmail.iesgn19.es;

        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        ssl    on;
        ssl_certificate    /etc/letsencrypt/live/webmail.iesgn19.es/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/webmail.iesgn19.es/privkey.pem;

        server_name webmail.iesgn19.es;

        root /srv/roundcube;

        index index.php index.html index.htm index.nginx-debian.html;

        location / {
            try_files $uri $uri/ /index.php;
        }

        location ~ \.php {
            try_files $uri =404;
            fastcgi_pass unix:/run/php/php7.3-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }

        location ~ /.well-known/acme-challenge {
            allow all;
        }

        location ~ ^/(README|INSTALL|LICENSE|CHANGELOG|UPGRADING)$ {
            deny all;
        }

        location ~ ^/(bin|SQL)/ {
            deny all;
        }

        location ~* \.(jpg|jpeg|gif|png|webp|svg|woff|woff2|ttf|css|js|ico|xml)$ {
            access_log        off;
            log_not_found     off;
            expires           360d;
        }
}
{% endhighlight %}

Como se puede apreciar, el fichero de configuración final es bastante complejo, aunque para la tranquilidad de los lectores, no es para nada necesario comprender qué hacen las directivas contenidas para hacer uso de la aplicación web. Cuando hayamos finalizado, simplemente guardaremos los cambios en el fichero.

Si nos fijamos, el directorio en el que debemos alojar todos los ficheros necesarios para el correcto funcionamiento de la aplicación es **/srv/roundcube**, directorio que todavía no ha sido generado, de manera que procederemos a ello haciendo uso del comando:

{% highlight shell %}
root@vps:~# mkdir /srv/roundcube
{% endhighlight %}

El directorio que contendrá todos los ficheros ya ha sido generado, de manera que para trabajar de una forma mucho más cómoda, nos moveremos dentro del mismo ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# cd /srv/roundcube/
{% endhighlight %}

En este caso, vamos a utilizar la última versión de **Roundcube** disponible (1.4.11), que podremos descargar desde el [repositorio oficial](https://github.com/roundcube/roundcubemail/), concretamente desde [aquí](https://github.com/roundcube/roundcubemail/releases/download/1.4.11/roundcubemail-1.4.11-complete.tar.gz). Para ello, haremos uso de `wget` para descargar el paquete comprimido de **Roundcube** desde dicha web:

{% highlight shell %}
root@vps:/srv/roundcube# wget https://github.com/roundcube/roundcubemail/releases/download/1.4.11/roundcubemail-1.4.11-complete.tar.gz
--2021-02-16 09:17:35--  https://github.com/roundcube/roundcubemail/releases/download/1.4.11/roundcubemail-1.4.11-complete.tar.gz
Resolving github.com (github.com)... 140.82.121.4
Connecting to github.com (github.com)|140.82.121.4|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://github-releases.githubusercontent.com/4224042/fe83ab00-6a4d-11eb-8ff3-e714480567a4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20210216%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210216T081557Z&X-Amz-Expires=300&X-Amz-Signature=08ea29d4650c0bea51417b17b9578fe657c5c6c87a05209f8991957d88123159&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=4224042&response-content-disposition=attachment%3B%20filename%3Droundcubemail-1.4.11-complete.tar.gz&response-content-type=application%2Foctet-stream [following]
--2021-02-16 09:17:35--  https://github-releases.githubusercontent.com/4224042/fe83ab00-6a4d-11eb-8ff3-e714480567a4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20210216%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210216T081557Z&X-Amz-Expires=300&X-Amz-Signature=08ea29d4650c0bea51417b17b9578fe657c5c6c87a05209f8991957d88123159&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=4224042&response-content-disposition=attachment%3B%20filename%3Droundcubemail-1.4.11-complete.tar.gz&response-content-type=application%2Foctet-stream
Resolving github-releases.githubusercontent.com (github-releases.githubusercontent.com)... 185.199.108.154, 185.199.109.154, 185.199.110.154, ...
Connecting to github-releases.githubusercontent.com (github-releases.githubusercontent.com)|185.199.108.154|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7048262 (6.7M) [application/octet-stream]
Saving to: ‘roundcubemail-1.4.11-complete.tar.gz’

roundcubemail-1.4.11-complete.tar.gz                100%[==========================>]  6.72M  6.75MB/s    in 1.0s

2021-02-16 09:17:37 (6.75 MB/s) - ‘roundcubemail-1.4.11-complete.tar.gz’ saved [7048262/7048262]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, listaremos el contenido del directorio actual haciendo para ello uso del comando `ls -l`:

{% highlight shell %}
root@vps:/srv/roundcube# ls -l
total 6884
-rw-r--r-- 1 root root 7048262 Feb  8 20:41 roundcubemail-1.4.11-complete.tar.gz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**roundcubemail-1.4.11-complete.tar.gz**" con un peso total de **6.72 MB** (7048262 _bytes_).

Al estar comprimido el fichero, no podremos hacer uso de la aplicación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/roundcube# tar -zxf roundcubemail-1.4.11-complete.tar.gz --strip 1 && rm roundcubemail-1.4.11-complete.tar.gz
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.
* **-–strip 1**: Saltamos el primer directorio, ya que dentro del comprimido hay un directorio padre que no necesitamos.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
root@vps:/srv/roundcube# ls -l
total 404
drwxr-xr-x  2 501    80   4096 Feb 16 09:18 bin
-rw-r--r--  1 501    80 186666 Feb  8 20:29 CHANGELOG
-rw-r--r--  1 501 staff    911 Feb  8 20:29 composer.json
-rw-r--r--  1 501    80    943 Feb  8 20:29 composer.json-dist
-rw-r--r--  1 501    80  89041 Feb  8 20:29 composer.lock
drwxr-xr-x  2 501    80   4096 Feb 16 09:18 config
-rw-r--r--  1 501    80  12843 Feb  8 20:29 index.php
-rw-r--r--  1 501    80  12864 Feb  8 20:29 INSTALL
drwxr-xr-x  3 501    80   4096 Feb 16 09:18 installer
-rw-r--r--  1 501    80  35147 Feb  8 20:29 LICENSE
drwxr-xr-x  2 501    80   4096 Feb 16 09:18 logs
drwxr-xr-x 35 501    80   4096 Feb 16 09:18 plugins
drwxr-xr-x  8 501    80   4096 Feb 16 09:18 program
drwxr-xr-x  3 501    80   4096 Feb 16 09:18 public_html
-rw-r--r--  1 501    80   3810 Feb  8 20:29 README.md
drwxr-xr-x  5 501    80   4096 Feb 16 09:18 skins
drwxr-xr-x  7 501    80   4096 Feb  8 20:29 SQL
drwxr-xr-x  2 501    80   4096 Feb 16 09:18 temp
-rw-r--r--  1 501    80   4148 Feb  8 20:29 UPGRADING
drwxr-xr-x  9 501    80   4096 Feb 16 09:18 vendor
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos (en lugar de descomprimir un directorio de nombre **roundcubemail-1.4.11** del que posteriormente tendríamos que mover los ficheros contenidos al directorio actual).

Ya tenemos todos los ficheros necesarios para llevar a cabo la instalación de _Roundcube_, pero hay un pequeño detalle que todavía no hemos contemplado, pues el usuario y el grupo propietario correspondiente a dichos ficheros y directorios es **root**, por lo que procedemos a cambiar dicho propietario y grupo de forma recursiva a **www-data**, haciendo para ello uso del comando `chown -R`, pues de otro modo no podría escribir en dichos ficheros durante la instalación:

{% highlight shell %}
root@vps:/srv/roundcube# chown -R www-data:www-data /srv/roundcube/
{% endhighlight %}

Listo, toda la configuración necesaria ya ha sido realizada, así que únicamente queda habilitar dicho sitio. Para activar el sitio, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/nginx/sites-available/** dentro de **/etc/nginx/sites-enabled/** de forma manual. Para ello, haremos uso del comando:

{% highlight shell %}
root@vps:/srv/roundcube# ln -s /etc/nginx/sites-available/roundcube /etc/nginx/sites-enabled/
{% endhighlight %}

Al parecer, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/roundcube# systemctl reload nginx
{% endhighlight %}

Nos falta un único paso para proceder con la instalación, y es que todavía no hemos creado la base de datos que utilizará, de manera que para acceder al motor de _mysql_, tendremos que hacer uso del comando:

{% highlight shell %}
root@vps:/srv/roundcube# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 116
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Cuando nos encontremos haciendo uso del motor _mysql_, procederemos a la creación de dicha base de datos, con nombre **bd_roundcube**, por ejemplo:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE bd_roundcube;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

La base de datos que usará _Roundcube_ ya se encuentra creada, pero de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma. En este caso, vamos a crear un usuario de nombre "**user_roundcube**" que tenga permitido el acceso desde **localhost** (es decir, desde la máquina local) y cuya contraseña sea "**pass_roundcube**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso del comando:

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'user_roundcube'@'localhost' IDENTIFIED BY 'pass_roundcube';
Query OK, 0 rows affected (0.004 sec)
{% endhighlight %}

El usuario ya ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_roundcube.* TO 'user_roundcube'@'localhost';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

**Nota**: No es necesario hacer uso del comando `FLUSH PRIVILEGES;`, a diferencia de varios artículos que he estado leyendo en Internet, que usan dicho comando muy a menudo sin necesidad alguna. Dicho comando, lo que hace es recargar la caché de las tablas GRANT que se encuentran en memoria, recarga que se hace de forma automática al hacer uso de una sentencia **GRANT**, de manera que no es necesario hacerlo manualmente. Para aclararlo, dicho comando únicamente debe utilizarse tras modificar las tablas GRANT de manera indirecta, es decir, tras usar sentencias INSERT, UPDATE o DELETE.

Antes de proceder con la correspondiente prueba de funcionamiento, tendremos que instalar dos librerías necesarias para el correcto funcionamiento de la aplicación web, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# apt install php-intl php-imagick
{% endhighlight %}

En un principio, todas las configuraciones necesarias han sido llevadas a cabo, así que es hora de realizar la correspondiente prueba de acceso. Para ello, abriremos un navegador web y trataremos de acceder a [https://webmail.iesgn19.es/installer](https://webmail.iesgn19.es/installer) para proceder con la instalación del mismo.

En caso de tener dudas sobre el proceso llevado a cabo, se recomienda llevar a cabo una lectura del artículo [Instalación de un servidor LEMP](https://www.alvarovf.com/servicios/vps/2020/11/08/instalacion-servidor-lemp.html) en el que se tratan con mayor profundidad los pasos llevados a cabo durante la instalación.

Dentro del instalador, como se puede suponer, tendremos que indicar el valor de algunas directivas necesarias para la instalación, como por ejemplo, la base de datos a utilizar por la aplicación web y las credenciales del usuario previamente generado:

![roundcube1](https://i.ibb.co/4p14QNB/webmail3.jpg "Roundcube")

Tras ello, llegamos a la parte de configuración de los protocolos para la sincronización y envío de correos. Es importante comprender llegado este punto que la conexión entre el cliente y el servidor se está llevando a cabo actualmente de forma cifrada, mediante el uso del protocolo _HTTPS_ en el puerto **443/TCP**, ya que estamos haciendo uso de una aplicación web.

Gracias al uso de dicho protocolo, la conexión entre el cliente y el servidor se encuentra cifrada, y teniendo también en cuenta que las peticiones al servidor de correos las estamos llevando a cabo de forma indirecta, ya que nosotros hacemos la petición a la aplicación web (cifrada) y es dicha aplicación la que se comunica con el servidor de correos de forma **local**, no será necesario utilizar las alternativas cifradas de los protocolos _IMAP_ y _SMTP_, ya que como hemos mencionado, el uso de dichos protocolos se ejecuta de forma local, de manera que la conexión no sale de la máquina servidora y no es necesario cifrarla, evitando por tanto un consumo de recursos innecesario.

En conclusión, para el protocolo _IMAP_ haremos uso del puerto **143/TCP**, tal y como podemos apreciar:

![roundcube2](https://i.ibb.co/N6df7jq/webmail4.jpg "Roundcube")

Lo mismo ocurre con el protocolo _SMTP_, de manera que utilizaremos el puerto **25/TCP**, de la siguiente forma:

![roundcube3](https://i.ibb.co/HDHrfXt/webmail4-1.jpg "Roundcube")

Por último, tendremos que indicar el idioma por defecto para la página. En mi caso, he optado por elegir el inglés (**en_US**), aunque podría haber elegido cualquier otro formateado de la forma indicada en el **RFC1766**:

![roundcube4](https://i.ibb.co/tHL4ZRp/webmail4-2.jpg "Roundcube")

Tras ello, podremos continuar y en el último apartado de la instalación se nos mostrará lo siguiente:

![roundcube5](https://i.ibb.co/mR3rKHk/webmail5.jpg "Roundcube")

Como se puede apreciar, la base de datos no ha sido inicializada, en el sentido de que es necesario crear una serie de tablas dentro de la misma para que la aplicación web haga uso de las mismas, de manera que pulsaremos en "**Initialize database**", obteniendo el siguiente resultado:

![roundcube6](https://i.ibb.co/6WRj2H7/webmail6.jpg "Roundcube")

Tras unos segundos, las tablas ya habrán sido generadas en la base de datos y todo estará listo para comenzar a utilizar la aplicación web, no sin antes eliminar el directorio **installer/** actualmente existente dentro del DocumentRoot, para así evitar posibles vulnerabilidades, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:/srv/roundcube# rm -r installer/
{% endhighlight %}

Una vez eliminado el directorio, todo estará listo para acceder, ahora sí, a la página principal de la aplicación web, accediendo por tanto a [https://webmail.iesgn19.es](https://webmail.iesgn19.es), pudiendo apreciar lo siguiente:

![roundcube7](https://i.ibb.co/LnTTfQk/webmail7.jpg "Roundcube")

Nos aparecerá un formulario para iniciar sesión, en el que debemos introducir el nombre de usuario generado en la máquina servidora junto a su contraseña, en este caso, **debian**:

![roundcube8](https://i.ibb.co/tMvygWf/webmail10.jpg "Roundcube")

Efectivamente, el correo electrónico ha sido correctamente sincronizado con nuestra aplicación **Roundcube**, por lo que podemos concluir que el protocolo _IMAP_ se encuentra funcionando tal y como debería a través del puerto **143/TCP**, de manera que se están sincronizando correctamente los mensajes de correo con el servidor.

Sin embargo, hay un pequeño fallo de configuración, y es que cuando enviamos un correo al exterior, se hace con el dominio **@localhost** como remitente, de manera que tendremos que modificarlo accediendo para ello al apartado **Settings** en el menú de la izquierda, seguido de **Identities** y pulsando en nuestra cuenta en cuestión. Una vez ahí, podremos establecer correctamente nuestro correo electrónico:

![roundcube9](https://i.ibb.co/p2gd7xZ/roundcubex.png "Roundcube")

Todavía nos falta comprobar que el protocolo _SMTP_ también está funcionando correctamente, de manera que llevaremos ahora la prueba a la inversa, enviando desde nuestro cliente web un correo electrónico a mi correo personal, de la siguiente forma:

![roundcube10](https://i.ibb.co/vwVX3bq/smtp1.png "Roundcube")

Tras pulsar el botón "**Send**" no se me ha mostrado ningún error por pantalla, de manera que puedo decir con casi total seguridad que el envío se ha producido correctamente. Sin embargo, hasta que no lo verifiquemos en mi correo personal, no tendremos la certeza de ello:

![gmail6](https://i.ibb.co/d5DGWjv/smtp2.png "Gmail")

Efectivamente, el correo ha sido correctamente recibido en _Gmail_, por lo que podemos concluir que el protocolo _SMTP_ se encuentra funcionando tal y como debería a través del puerto **25/TCP**, de manera que se están enviando correctamente los mensajes de correo a través del servidor.

## Test final

Por último, y a modo de comprobación, vamos a hacer uso de una [herramienta](https://www.mail-tester.com/) para verificar que todo el trabajo llevado a cabo hasta ahora ha servido y los servidores de correo que reciban mensajes procedentes de nosotros, los tendrán en cuenta de una forma muy posiblemente favorable.

Su funcionamiento es bastante sencillo a la vez que completo, consistente en enviar un correo electrónico a la dirección que se nos facilita por pantalla, tal y como se puede apreciar a continuación:

![tester1](https://i.ibb.co/6rgPjyr/test1.png "mail-tester")

En mi caso, he procedido a enviar desde mi cliente **Thunderbird** un correo electrónico a la dirección mostrada por pantalla, para así obtener una calificación resultante de llevar a cabo una serie de pruebas sobre el correo enviado.

No considero necesario mostrar el proceso de envío, sino el resultado final producido por dicha recepción, que tras pulsar en "**A continuación comprueba tu puntuación**" sería el siguiente:

![tester2](https://i.ibb.co/Qksq0m0/test3.png "mail-tester")

Finalmente, la puntuación obtenida ha sido de **10/10**, por lo que podemos concluir que hemos realizado un buen trabajo y los servidores de correo van a tratar de forma favorable los mensajes procedentes del nuestro.

A pesar de ello, me ha indicado algunos aspectos que podría mejorar, como por ejemplo añadir un registro _DMARC_, así como un encabezado de anulación de suscripción a la lista, principalmente pensado para aquellas personas que envíen correos de forma masiva a través de _newsletters_, por lo que me voy con un muy buen sabor de boca.