---
layout: post
title:  "Configuración LDAPs"
banner: "/assets/images/banners/ldap.jpg"
date:   2020-12-17 12:06:00 +0200
categories: sistemas openstack
---
Se recomienda realizar una previa lectura del _post_ [Instalación y configuración inicial de OpenLDAP](https://www.alvarovf.com/sistemas/openstack/2020/12/14/instalacion-configuracion-openldap.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una modificación sobre el servidor LDAP de la máquina **Freston**, habilitando el uso del protocolo **ldaps://**.

Hasta ahora hemos hecho uso de **Freston** como cliente y servidor LDAP, de manera que el cifrado de la conexión no era tan importante al llevarse a cabo de manera local en la misma máquina, reduciendo por tanto los riesgos, pero si queremos que clientes remotos hagan uso del servidor, es sumamente importante asegurar que la conexión se encuentra cifrada entre ambos extremos para así implementar un nivel de seguridad mínimo en las comunicaciones haciendo uso de **SSL/TLS**.

El protocolo del que se hace uso por defecto es **ldap://** (puerto **389/TCP**), que como se puede suponer, no se encuentra cifrado. Además de dicho protocolo, también se utiliza **ldapi://**, un _socket UNIX_ que se abre en la máquina y que permite el acceso localmente a través del mismo, sin necesidad de establecer conexiones TCP/IP. Este último es de gran importancia, ya que la configuración de LDAP se lleva a cabo exclusivamente a través del mismo.

Para la correspondiente implementación del protocolo seguro necesitamos disponer de un certificado firmado por una autoridad certificadora. En mi caso, dispongo de un certificado **_wildcard_** previamente [generado](https://www.alvarovf.com/seguridad/openstack/2020/12/15/certificado-wildcard-https.html) y con validez para todas las máquinas y servicios existentes en el subdominio ***.alvaro.gonzalonazareno.org**, en el que se encuentra también nombrada la máquina servidora LDAP. Es importante mencionar que es posible crear nuestra propia autoridad certificadora y autofirmarnos el certificado, pero no es el objetivo de este artículo.

Además del certificado firmado y la clave privada asociada al mismo, necesitaremos disponer del certificado de la autoridad certificadora firmante, el cuál contendrá su clave pública necesaria para verificar la firma realizada con su clave privada sobre nuestro certificado, que en mi caso, se trata de la autoridad certificadora del **IES Gonzalo Nazareno**.

Dicho certificado fue anteriormente generado en **Quijote**, una máquina totalmente ajena al servidor LDAP, pero que al tratarse de un certificado **_wildcard_**, también nos servirá para este servidor. Lo primero que haremos será transferir los 3 ficheros necesarios a **Freston**, ejecutando para ello los comandos:

{% highlight shell %}
[root@quijote ~]# scp -p /etc/ssl/certs/gonzalonazareno.crt debian@freston:
[root@quijote ~]# scp -p /etc/ssl/certs/openstack.crt debian@freston:
[root@quijote ~]# scp -p /etc/ssl/private/openstack.key debian@freston:
{% endhighlight %}

Donde:

* **-p**: Indica que se mantengan las marcas temporales y permisos de los ficheros tras la transferencia.

Los ficheros ya han sido transferidos al directorio personal de **debian** en la máquina **Freston**, así que vamos a proceder a listar el contenido de dicho directorio para así verificar que las transferencias se han completado satisfactoriamente, haciendo para ello uso del comando:

{% highlight shell %}
debian@freston:~$ ls
gonzalonazareno.crt  openstack.crt  openstack.key
{% endhighlight %}

Como es lógico, esa no es la ruta apropiada para almacenar certificados ni claves privadas, de manera que moveremos los certificados (**gonzalonazareno.crt** y **openstack.crt**) al directorio **/etc/ssl/certs/**, y la clave privada (**openstack.key**) al directorio **/etc/ssl/private/**, ejecutando para ello los comandos (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
root@freston:~# mv /home/debian/gonzalonazareno.crt /etc/ssl/certs/
root@freston:~# mv /home/debian/openstack.crt /etc/ssl/certs/
root@freston:~# mv /home/debian/openstack.key /etc/ssl/private/
{% endhighlight %}

Una vez movidos los correspondientes ficheros, vamos a comenzar por listar el contenido del directorio **/etc/ssl/certs/**, estableciendo a su vez un filtro por nombre para así comprobar que los certificados están ahora ubicados en el mismo, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# ls -l /etc/ssl/certs/ | egrep '(openstack|gonzalonazareno)'
-rw-r--r-- 1 root root   3634 Dec 15 16:22 gonzalonazareno.crt
-rw-r--r-- 1 root root  10097 Dec 15 13:27 openstack.crt
{% endhighlight %}

Efectivamente, los certificados se encuentran ahí ubicados con los permisos correctos. Es importante mencionar que el usuario y grupo propietario de los mismos ha de ser **root**, ya que en mi caso, al haber realizado la transferencia al directorio personal de **debian**, se modificaron los propietarios a dicho nombre, de manera que tuve que hacer uso de `chown` para modificarlos de vuelta.

Por último, listaremos también el contenido del directorio **/etc/ssl/private/** para así comprobar que la clave privada asociada al certificado firmado está ahora ubicada en el mismo, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ls -l /etc/ssl/private/
total 4
-r-------- 1 root root 3247 Dec 14 11:53 openstack.key
{% endhighlight %}

Como era de esperar, la clave privada se encuentra ahí ubicada con los permisos correctos. Al igual que ocurría con los certificados, es importante que el usuario y grupo propietario de la misma sea **root**, pero teniendo a la vez en cuenta un aspecto de gran importancia.

El usuario que ejecuta por defecto el servicio **slapd** tiene nombre **openldap**, usuario que deberá tener acceso a dicha clave privada para así permitir el correcto funcionamiento del protocolo **ldaps://**, ya que si nos fijamos, actualmente no cuenta con permisos de lectura sobre dicho fichero, al encontrarse limitados para el usuario propietario, es decir, para **root**.

Lo primero que se nos puede venir a la cabeza es modificar los permisos para que cualquier usuario existente en el sistema pueda leer el contenido de dicho fichero, acción que supondría por tanto un gran agujero de seguridad.

Para solventar el inconveniente que acabamos de mencionar, haremos uso de las **ACL**, un mecanismo complementario a los permisos de los ficheros UNIX que permite llevar a cabo dicha gestión de una manera mucho más precisa, dando permisos a un único usuario o grupo a un recurso concreto. Al tratarse de un tema bastante extenso, explicarlo con detalle supondría que el artículo se saliese de su objetivo, pues en Internet se puede encontrar mucha documentación al respecto.

Como se puede suponer, vamos a otorgarle permisos sobre dicho fichero y directorio que lo contiene al usuario **openldap**, haciendo para ello uso de los comandos:

{% highlight shell %}
root@freston:~# setfacl -m u:openldap:r-x /etc/ssl/private
root@freston:~# setfacl -m u:openldap:r-x /etc/ssl/private/openstack.key
{% endhighlight %}

Donde:

* **-m**: Indicamos que vamos a modificar las ACL de un determinado fichero o directorio.

Una vez que los permisos han sido correctamente ajustados, todo está listo para comenzar con la configuración del demonio, no sin antes verificar los puertos en los que el servicio **slapd** está escuchando, que en un principio únicamente debe ser el correspondiente al protocolo **ldap://**, es decir, el puerto **389/TCP**, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# netstat -tlnp | egrep 'slapd'
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      495/slapd
tcp6       0      0 :::389                  :::*                    LISTEN      495/slapd
{% endhighlight %}

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho _socket_.

Efectivamente, el proceso **slapd** está escuchando peticiones para el protocolo no seguro tal y como debería, así que podemos proceder con la configuración para el protocolo **ldaps://**.

La configuración de LDAP es bastante peculiar, ya que se encuentra contenida en una rama aparte dentro del propio directorio, característica que se implementó hace varios años con la finalidad de evitar tener que llevar a cabo un reinicio del servicio para así cargar la nueva configuración en memoria tras una modificación del hipotético fichero contenido en el directorio **/etc/** (al que estamos acostumbrados), pues estamos hablando de un servicio del que dependen muchos otros, que dejarían de funcionar a su vez.

Dado que vamos a tratar los parámetros de configuración como si fuesen atributos de un objeto en el directorio, tendremos que crear un fichero de extensión **.ldif** que contenga las modificaciones a llevar a cabo, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# nano ldaps.ldif
{% endhighlight %}

Dentro del mismo, modificaremos en este caso un total de tres atributos, uno por cada uno de los ficheros de los que disponemos, quedando de la siguiente manera:

{% highlight shell %}
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/gonzalonazareno.crt           
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/openstack.key
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/openstack.crt
{% endhighlight %}

* En la primera línea (directiva **dn**) hemos indicado el nombre distintivo del objeto sobre el que queremos realizar modificaciones dentro de la jerarquía, es decir, aquel correspondiente a la configuración.
* En la segunda línea (directiva **changetype**) hemos indicado el tipo de cambio a llevar a cabo, que en este caso, se trata de una modificación.
* A partir de aquí tendremos que indicar pares de directivas (atributo a modificar seguido del nuevo valor asignado), separadas entre ellas con un guión (**-**), siendo el primer par el correspondiente al certificado de la autoridad certificadora, el segundo a nuestra clave privada y el tercero, al certificado firmado.

Una vez definido el fichero **.ldif**, tendremos que importarlo para así modificar los atributos del objeto correspondiente a la configuración, haciendo para ello uso de `ldapmodify`, pero utilizando en este caso unos parámetros concretos, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f ldaps.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
{% endhighlight %}

Donde:

* **-Y**: Especificamos un mecanismo SASL a utilizar para la autenticación, en este caso, **EXTERNAL**.
* **-H**: Indicamos la URI de conexión al servidor LDAP, haciendo uso en este caso del _socket UNIX_ (**ldapi:///**) para así contar con los privilegios necesarios para llevar a cabo modificaciones en la configuración.
* **-f**: Indicamos el fichero en el que se encuentran definidos los cambios a realizar.

Como se puede apreciar, la entrada **cn=config** ha sido correctamente modificada dado que contamos con los privilegios necesarios para ello al haber hecho uso del _socket UNIX_, además de tener el usuario **openldap** los permisos necesarios en los correspondientes ficheros.

De otro lado, el propio _daemon_ incluye una serie de parámetros que no se pueden modificar internamente en el directorio, como por ejemplo el usuario y grupo que ejecuta el proceso, e incluso los servicios que se arrancan inicialmente. A nosotros nos interesa esta última configuración, para así poder habilitar el protocolo **ldaps://** para que escuche en el puerto **636/TCP**.

El fichero en cuestión donde se encuentran asignados los valores de dichos parámetros es **/etc/default/slapd**, de manera que lo modificaremos haciendo uso del comando:

{% highlight shell %}
root@freston:~# nano /etc/default/slapd
{% endhighlight %}

Dentro del mismo, tendremos que buscar la directiva **SLAPD_SERVICES**, que tendrá la siguiente forma:

{% highlight shell %}
SLAPD_SERVICES="ldap:/// ldapi:///"
{% endhighlight %}

Como se puede apreciar, el protocolo **ldaps://** no se encuentra contemplado en dicha directiva, por lo que tendremos que incluirlo manualmente, quedando de la siguiente manera:

{% highlight shell %}
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
{% endhighlight %}

Tras ello, guardaremos los cambios y dado que hemos modificado el fichero de configuración de un servicio, tendremos que reiniciarlo para así cargar la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# systemctl restart slapd
{% endhighlight %}

Cuando el reinicio del servicio haya concluido, comprobaremos el estado del mismo para así verificar que no ha habido ningún fallo en la configuración y que por tanto, el protocolo **ldaps://** se encuentra ahora operativo, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# systemctl status slapd
● slapd.service - LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)
   Loaded: loaded (/etc/init.d/slapd; generated)
   Active: active (running) since Thu 2020-12-17 10:56:42 CET; 4s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 2447 ExecStart=/etc/init.d/slapd start (code=exited, status=0/SUCCESS)
    Tasks: 3 (limit: 562)
   Memory: 3.3M
   CGroup: /system.slice/slapd.service
           └─2456 /usr/sbin/slapd -h ldap:/// ldapi:/// ldaps:/// -g openldap -u openldap -F /etc/ldap/slapd.d

Dec 17 10:56:42 freston systemd[1]: Starting LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)...
Dec 17 10:56:42 freston slapd[2452]: @(#) $OpenLDAP: slapd  (Nov 17 2020 01:23:45) $
                                             Debian OpenLDAP Maintainers <pkg-openldap-devel@lists.alioth.debian.org>
Dec 17 10:56:42 freston slapd[2456]: slapd starting
Dec 17 10:56:42 freston slapd[2447]: Starting OpenLDAP: slapd.
Dec 17 10:56:42 freston systemd[1]: Started LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol).
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el servicio se encuentra actualmente en ejecución y no se puede apreciar ningún fallo en los _logs_, además de haberse percatado del nuevo protocolo que le hemos indicado que utilice. Si todo ha funcionado como debería, el proceso estará ahora escuchando peticiones en el puerto **636/TCP**, por lo que lo comprobaremos ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# netstat -tlnp | egrep 'slapd'
tcp        0      0 0.0.0.0:636             0.0.0.0:*               LISTEN      2456/slapd          
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      2456/slapd          
tcp6       0      0 :::636                  :::*                    LISTEN      2456/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      2456/slapd
{% endhighlight %}

Efectivamente, se ha establecido un _socket TCP/IP_ en el correspondiente puerto para todas las interfaces de la máquina, de manera que todo está listo para hacer uso del nuevo protocolo configurado en lo que al servidor respecta.

Sin embargo, ahora pasamos al lado del cliente (a pesar de que ambos extremos se encuentran unificados en **Freston**, sería aplicable a una situación normal en la que el cliente y el servidor sean máquinas distintas), en el que al igual que importamos certificados de autoridades certificadoras en el navegador para poder hacer uso del protocolo HTTPS, tendremos que hacerlo para aquellas aplicaciones manejadas desde línea de comandos, que hacen uso de otros protocolos cifrados, como por ejemplo **ldaps://**.

Es importante mencionar que los certificados que importemos haciendo uso del método que mencionaré a continuación son totalmente ajenos a aquellos importados en el navegador.

El paquete encargado de instalar y mantener actualizada la lista de autoridades certificadoras es **ca-certificates**, que usa la lista de autoridades de confianza de **Mozilla Firefox** y las instala en el sistema para que sean utilizadas por el resto de aplicaciones, almacenándolos en **/etc/ssl/certs/** mediante un enlace simbólico a los mismos.

Como resultado, se genera dentro del directorio anteriormente mencionado un fichero de nombre **ca-certificates.crt**, que es una concatenación de todos esos certificados, fichero del que harán uso dichas aplicaciones de línea de comandos. Como se puede suponer, el certificado que importemos deberá estar reflejado en el mismo.

El directorio **/etc/ssl/certs/** no está pensado para ubicar los certificados de las autoridades certificadoras que nosotros instalemos localmente, sino que para ello existe el directorio **/usr/local/share/ca-certificates/** en el que deberé copiar, en mi caso, el certificado de la autoridad certificadora **IES Gonzalo Nazareno**, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# cp /etc/ssl/certs/gonzalonazareno.crt /usr/local/share/ca-certificates/
{% endhighlight %}

El certificado se encuentra ya correctamente ubicado en el directorio pensado para ello, sin embargo, por el simple hecho de encontrarse ahí no será suficiente para hacer uso del mismo con la finalidad de comprobar firmas por parte de dicha autoridad certificadora, sino que tendremos que crear el enlace simbólico al mismo en el directorio **/etc/ssl/certs/**, además de concatenarlo en el fichero **ca-certificates.crt**. Para ello, se nos proporciona el binario **update-ca-certificates**, que llevará a cabo dichas tareas de forma automática:

{% highlight shell %}
root@freston:~# update-ca-certificates 
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping duplicate certificate in gonzalonazareno.pem
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, un nuevo certificado ha sido añadido, de manera que podemos asegurar que el certificado de la autoridad certificadora **IES Gonzalo Nazareno** ha sido correctamente instalado en la máquina.

Tras ello, haremos uso de `ldapsearch` para así ejecutar una búsqueda anónima sobre el directorio. En este caso, no quiero establecer ningún tipo de filtro, sino listar todos los objetos existentes en la estructura actualmente definida pero haciendo uso de LDAP sobre **SSL/TLS** para la conexión, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org" -H ldaps://localhost:636

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

# admin, alvaro.gonzalonazareno.org
dn: cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# Personas, alvaro.gonzalonazareno.org
dn: ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Personas

# Grupos, alvaro.gonzalonazareno.org
dn: ou=Grupos,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Grupos

# search result
search: 2
result: 0 Success
{% endhighlight %}

Donde:

* **-x**: Especificamos que se haga uso de la autenticación simple en lugar de SASL.
* **-b**: Indicamos la base a partir de la cuál queremos mostrar la información. En este caso, **dc=alvaro,dc=gonzalonazareno,dc=org**.
* **-H**: Especificamos la URI de conexión al servidor LDAP, que en este caso hará uso de **ldaps://** y se conectará a **localhost** en el puerto **636/TCP**.

Como se puede apreciar, la consulta al directorio ha funcionado correctamente y ha devuelto la información que debería. Sin embargo, la configuración del cliente no está establecida para hacer uso del protocolo **ldaps://** por defecto, sino para utilizar **ldap://**. Por ello, tendremos que llevar a cabo las modificaciones oportunas para que el comportamiento no sea ese.

En el lado del cliente encontramos un fichero **/etc/ldap/ldap.conf** que contiene definidos algunos parámetros a utilizar por defecto en las conexiones, como por ejemplo la base del directorio LDAP (que podríamos configurar para así evitar tener que indicar la opción **-b** en cada consulta) e incluso la URI de la que se hará uso, de manera que vamos a proceder a modificar dicho fichero ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano /etc/ldap/ldap.conf
{% endhighlight %}

Dentro del mismo, tendremos que buscar la directiva **URI**, que tendrá la siguiente forma:

{% highlight shell %}
#URI     ldap://ldap.example.com ldap://ldap-master.example.com:666
{% endhighlight %}

Como se puede apreciar, el protocolo **ldaps://** no se encuentra contemplado en dicha directiva, por lo que tendremos que descomentarla y establecer dicho protocolo para que se conecte a **localhost** de manera predeterminada, quedando de la siguiente manera:

{% highlight shell %}
URI     ldaps://localhost
{% endhighlight %}

En esta ocasión no he especificado el puerto **636/TCP**, ya que al estar haciendo uso del puerto por defecto para dicho protocolo no es necesario indicarlo explícitamente.

Para llevar a cabo la correspondiente prueba, he deshabilitado el protocolo **ldap://** en el lado del servidor, modificando para ello la directiva **SLAPD_SERVICES** del fichero **/etc/default/slapd**, de manera que el puerto **389/TCP** se encuentra ahora cerrado y por tanto, únicamente se podrá acceder mediante **ldaps://**.

Tras ello, repetiremos la consulta anteriormente ejecutada pero sin indicar ahora de forma explícita la URI a utilizar para la conexión:

{% highlight shell %}
root@freston:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org"

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

# admin, alvaro.gonzalonazareno.org
dn: cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# Personas, alvaro.gonzalonazareno.org
dn: ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Personas

# Grupos, alvaro.gonzalonazareno.org
dn: ou=Grupos,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Grupos

# search result
search: 2
result: 0 Success
{% endhighlight %}

Como era de esperar, la consulta se ha llevado ahora a cabo haciendo uso del protocolo **ldaps://** de forma predeterminada, de manera que podemos asegurar que toda la configuración llevada a cabo funciona correctamente.

Por último, volví a habilitar el uso del protocolo **ldap://** para que aquellos clientes que no dispongan del certificado correctamente configurado o bien existiese algún tipo de problema, siempre se pueda acceder al servidor haciendo uso de dicho protocolo no cifrado, con las consecuentes penalizaciones en cuanto a seguridad.