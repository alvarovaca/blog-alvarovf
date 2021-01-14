---
layout: post
title:  "Certificado wildcard y HTTPS"
banner: "/assets/images/banners/https.jpg"
date:   2020-12-16 14:05:00 +0200
categories: seguridad openstack
---
Se recomienda realizar una previa lectura del _post_ [Servidor DNS, Web y Base de Datos](https://www.alvarovf.com/servicios/openstack/2020/12/15/servidor-dns-web-bbdd.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una modificación sobre el servidor web de la máquina **Quijote**, habilitando el uso del protocolo **https://**. Es sumamente importante asegurar que la conexión se encuentra cifrada entre ambos extremos para así implementar un nivel de seguridad mínimo en las comunicaciones haciendo uso de **SSL/TLS**.

Para la correspondiente implementación del protocolo seguro necesitamos disponer de un certificado firmado por una autoridad certificadora. En mi caso, voy a generar un certificado **_wildcard_** que cuente con validez para todas las máquinas y servicios existentes en el subdominio ***.alvaro.gonzalonazareno.org**, en el que se encuentra también nombrada la máquina que actúa como servidor web.

Además del certificado firmado y la clave privada asociada al mismo, necesitaremos disponer del certificado de la autoridad certificadora firmante, el cuál contendrá su clave pública necesaria para verificar la firma realizada con su clave privada sobre nuestro certificado, que en mi caso, se trata de la autoridad certificadora del **IES Gonzalo Nazareno**.

En este caso, vamos a realizar el procedimiento con **openssl**, pero se podría hacer con otras múltiples opciones de software.

El primer paso será generar una clave privada RSA de 4096 bits, que será almacenada en **/etc/ssl/private/**, directorio que todavía no se encuentra generado, por lo que deberemos hacerlo manualmente, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# mkdir /etc/ssl/private
{% endhighlight %}

Dado que el contenido que va a poseer dicho directorio es bastante sensible, es muy recomendable modificar sus permisos a sólo lectura por parte del propietario, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# chmod 700 /etc/ssl/private/
{% endhighlight %}

Todo está listo para llevar a cabo la generación de la clave privada RSA de 4096 bits, de nombre **openstack.key**, por ejemplo, de manera que ejecutaremos para ello el comando:

{% highlight shell %}
[root@quijote ~]# openssl genrsa 4096 > /etc/ssl/private/openstack.key
Generating RSA private key, 4096 bit long modulus (2 primes)
........++++
.......................................................++++
e is 65537 (0x010001)
{% endhighlight %}

Una vez generado, cambiaremos los permisos de la clave privada que acabamos de generar a **400**, de manera que únicamente el propietario pueda leer el contenido, pues se trata de una clave privada. Este paso no es obligatorio pero sí recomendable por seguridad. Para ello, haremos uso de `chmod`:

{% highlight shell %}
[root@quijote ~]# chmod 400 /etc/ssl/private/openstack.key
{% endhighlight %}

Para verificar que los permisos han sido correctamente modificados, listaremos el contenido de dicho directorio haciendo uso de `ls -l`:

{% highlight shell %}
[root@quijote ~]# ls -l /etc/ssl/private
total 4
-r--------. 1 root root 3247 Dec 14 11:53 openstack.key
{% endhighlight %}

Efectivamente, los permisos han sido correctamente modificados a sólo lectura por parte del propietario.

Tras ello, crearemos un fichero **.csr** de solicitud de firma de certificado para que sea firmado por la autoridad certificadora (CA) del IES Gonzalo Nazareno, que estará asociado a la clave privada que acabamos de generar:

{% highlight shell %}
[root@quijote ~]# openssl req -new -key /etc/ssl/private/openstack.key -out /root/openstack.csr
{% endhighlight %}

Durante la ejecución, nos pedirá una serie de valores para identificar al certificado, que tendremos que rellenar de la siguiente manera:

{% highlight shell %}
Country Name (2 letter code) [XX]:ES
State or Province Name (full name) []:Sevilla
Locality Name (eg, city) [Default City]:Dos Hermanas
Organization Name (eg, company) [Default Company Ltd]:IES Gonzalo Nazareno
Organizational Unit Name (eg, section) []:Informatica
Common Name (eg, your name or your server's hostname) []:*.alvaro.gonzalonazareno.org
Email Address []:avacaferreras@gmail.com
{% endhighlight %}

Todos los valores son genéricos excepto los dos últimos:

* **Common Name (CN)**: Indica el nombre _FQDN_ para el que queremos generar el certificado. Al tratarse de un certificado wildcard, tendremos que introducir __*__ (haciendo referencia a todas las máquinas existentes) seguido del nombre de dominio en cuestión, en este caso, **alvaro.gonzalonazareno.org**.
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
[root@quijote ~]# ls -l /root/ | egrep 'openstack'
-rw-r--r--. 1 root root 1809 Dec 14 11:57 openstack.csr
{% endhighlight %}

Como se puede apreciar, el fichero se encuentra generado en el directorio **/root**, con nombre **openstack.csr**.

El siguiente paso será subir el fichero de solicitud de firma mediante la aplicación [Gestiona](https://dit.gonzalonazareno.org/gestiona/cert/), para que sea firmado y así obtengamos el certificado que necesitamos, con extensión **.crt** o **.pem**. El fichero debe subirse en el apartado "**Certificados de Equipos**":

![cert1](https://i.ibb.co/PZd3P8t/Captura-de-pantalla-de-2020-11-02-10-31-53.png "Subida del fichero")

Una vez subido, tendremos que esperar unas horas (pues la firma del mismo se lleva a cabo manualmente) y tras ello, nos aparecerá de la siguiente forma:

![cert2](https://i.ibb.co/6HhHJHr/Captura-de-pantalla-de-2020-12-15-13-17-41.png "Certificado firmado")

Como se puede apreciar, nos aparece para **descargar** o **revocar** el certificado firmado, con extensión **.crt**. En este caso, lo descargaremos.

Tras ello, tendremos que descargar además el certificado de la propia autoridad certificadora del IES Gonzalo Nazareno dentro de **/etc/ssl/certs/**, de manera que se pueda llevar a cabo la comprobación de la firma y así poder verificar que quién nos lo ha firmado es quien realmente dice ser. Para ello, podemos hacer uso de la utilidad `wget` para descargar el fichero pasándole una [URL de descarga](https://dit.gonzalonazareno.org/gestiona/info/documentacion/doc/gonzalonazareno.crt) o bien, transferirlo de otra máquina en la que ya se encuentre descargado, que es lo que yo haré:

{% highlight shell %}
[root@quijote ~]# scp alvaro@172.22.9.241:openstack.crt /etc/ssl/certs/
{% endhighlight %}

Para verificar que el certificado de la CA Gonzalo Nazareno y nuestro certificado firmado han sido correctamente transferidos, vamos a listar el contenido del directorio anteriormente mencionado, estableciendo de nuevo, un filtro por nombre:

{% highlight shell %}
[root@quijote ~]# ls -l /etc/ssl/certs/ | egrep '(openstack|gonzalonazareno)'
-rw-r--r--. 1 root root  3634 Dec 15 16:22 gonzalonazareno.crt
-rw-r--r--. 1 root root 10097 Dec 15 13:27 openstack.crt
{% endhighlight %}

Como se puede apreciar, existe un fichero de nombre **openstack.crt** que es el resultado de la firma de la solicitud de firma de certificado que previamente hemos enviado, y otro de nombre **gonzalonazareno.crt**, que es el certificado de la entidad certificadora, con el que posteriormente se comprobará la firma de la autoridad certificadora sobre dicho certificado del servidor.

Al igual que anteriormente hemos configurado un VirtualHost en _httpd_ para las peticiones entrantes por el puerto 80 (**HTTP**), tendremos que configurar otro bloque para las peticiones entrantes por el puerto 443 (**HTTPS**), así que procederemos a añadirlo en el fichero de configuración ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/httpd/sites-available/sitioweb.conf
{% endhighlight %}

Dentro del mismo, tendremos que -crear- dos directivas **VirtualHost**, una para el VirtualHost accesible en el puerto 80 HTTP, dentro del cuál únicamente asignaremos el ServerName correspondiente y una redirección permanente para así forzar HTTPS. En la otra, para el VirtualHost accesible en el puerto 443 HTTPS, tendremos que configurar las siguientes directivas:

* **ServerName**: Al igual que en el VirtualHost anterior, tendremos que indicar el nombre de dominio a través del cuál accederemos al servidor.
* **SSLEngine**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **SSLCertificateFile**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/ssl/certs/openstack.crt**.
* **SSLCertificateKeyFile**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/ssl/private/openstack.key**.
* **SSLCACertificateFile**: Indicamos la ruta del certificado de la _CA_ con el que comprobaremos la firma de nuestro certificado. En este caso, **/etc/ssl/certs/gonzalonazareno.crt**.

Como es lógico, toda aquella configuración previamente existente en el bloque del VirtualHost accesible en el puerto 80 HTTP, pasará ahora al bloque del VirtualHost accesible en el puerto 443 HTTPS, de manera que el resultado final sería:

{% highlight shell %}
<VirtualHost *:80>
    ServerName www.alvaro.gonzalonazareno.org

    Redirect 301 / https://www.alvaro.gonzalonazareno.org/

    ErrorLog /var/www/alvaro/log/error.log
    CustomLog /var/www/alvaro/log/requests.log combined
</VirtualHost>

<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
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

        SSLEngine on

        SSLCertificateFile      /etc/ssl/certs/openstack.crt
        SSLCertificateKeyFile   /etc/ssl/private/openstack.key
        SSLCACertificateFile    /etc/ssl/certs/gonzalonazareno.crt
    </VirtualHost>
</IfModule>
{% endhighlight %}

Si nos fijamos en el contenido final del fichero de configuración, podremos apreciar que se hace uso de la directiva **IfModule**, es decir, se requiere que el módulo **ssl** se encuentre habilitado.

Para verificar si dicho módulo se encuentra ya instalado, listaremos el contenido del directorio **/etc/httpd/modules/**, estableciendo a su vez el correspondiente filtro por nombre:

{% highlight shell %}
[root@quijote ~]# ls /etc/httpd/modules/ | egrep 'ssl'
{% endhighlight %}

Como se puede apreciar, el filtro no ha devuelto ningún resultado, por lo que podemos asegurar que el módulo no se encuentra instalado, de manera que lo instalaremos manualmente haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# dnf install mod_ssl
{% endhighlight %}

De otro lado, en CentOS 8 viene habilitado por defecto **SELinux**, un módulo de seguridad para el kernel Linux que proporciona el mecanismo para implementar una políticas de control de acceso para las aplicaciones, los procesos y los archivos dentro de un sistema. Para evitar problemas con los certificados que anteriormente hemos añadido, tendremos que restablecer el contexto de seguridad de los mismos, ejecutando para ello los comandos:

{% highlight shell %}
[root@quijote ~]# restorecon /etc/ssl/certs/openstack.crt 
[root@quijote ~]# restorecon /etc/ssl/certs/gonzalonazareno.crt
{% endhighlight %}

Tras ello, será de suma importancia verificar que el proceso correspondiente esté escuchando peticiones en el puerto 443 TCP, ya que por defecto únicamente lo hace en el puerto 80. Para ello, vamos a modificar el fichero de configuración del servicio, haciendo para ello uso del comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/httpd/conf/httpd.conf
{% endhighlight %}

Dentro del mismo, tendremos que buscar las directivas **Listen**, existiendo en mi caso, únicamente la siguiente:

{% highlight shell %}
Listen 80
{% endhighlight %}

Como es lógico, tendremos que añadir una nueva directiva **Listen** para el puerto 443 TCP, quedando de la siguiente forma:

{% highlight shell %}
Listen 80
Listen 443
{% endhighlight %}

Para que todos los cambios llevados a cabo hasta ahora surtan efecto, tendremos que reiniciar el servicio _httpd_, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# systemctl restart httpd
{% endhighlight %}

Todo está listo para llevar a cabo la prueba de funcionamiento, no sin antes verificar los puertos en los que el servicio **httpd** está escuchando. Si todo ha funcionado como debería, el proceso estará ahora escuchando peticiones en los puertos -**80/TCP** y **443/TCP**-, por lo que lo comprobaremos ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# netstat -tlnp | egrep 'httpd'
tcp6       0      0 :::80                   :::*                    LISTEN      1436/httpd
tcp6       0      0 :::443                  :::*                    LISTEN      1436/httpd
{% endhighlight %}

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho _socket_.

Efectivamente, el proceso **httpd** está escuchando peticiones en los puertos **80/TCP** y **443/TCP**, para los protocolos **HTTP** y **HTTPS**, respectivamente.

Tras ello, ya estará todo listo para acceder a **www.alvaro.gonzalonazareno.org** desde el navegador, ya que en el anterior artículo añadí la correspondiente regla DNAT para que las peticiones entrantes en los puertos 80 y 443 se reenviasen a **Quijote**. El resultado sería el siguiente:

![https1](https://i.ibb.co/LQhR78V/Captura-de-pantalla-de-2020-12-15-17-08-18.png "HTTPS")

Como se puede apreciar, se nos ha mostrado una advertencia de seguridad, por lo que podemos asegurar que la redirección de **http://** a **https://** ha funcionado correctamente.

Esta advertencia es debida a que el navegador no ha podido comprobar la firma por parte de la _CA_ en el certificado recibido por el servidor, básicamente porque no tiene la clave pública o certificado de la _CA_, por lo que debemos importarlo manualmente en el navegador. En una situación cotidiana, esto no suele hacerse, ya que las claves públicas de las _CA_ más conocidas vienen importadas por defecto, pero al ser una entidad certificadora sin reputación alguna, los navegadores como es obvio, no la han importado. De igual manera, la conexión sigue estando cifrada ya que el certificado ha sido correctamente transferido por parte del servidor, lo único es que no se ha podido comprobar la firma de la _CA_.

Para importar el certificado de la _CA_ en un navegador _Firefox_, tendremos que pulsar en el icono de las **3 barras** en la barra superior, seguidamente en **Preferencias** y una vez ahí, buscar el apartado **Privacidad & Seguridad**. Si nos desplazamos hasta la parte inferior del mismo, encontraremos la sección **Certificados**, en la que debemos pulsar en **Ver certificados**. Acto seguido, nos moveremos al apartado **Autoridades** para que así, pulsando en **Importar**, nos permita importar el certificado de una autoridad certificadora.

Una vez seleccionado el certificado a importar, se nos mostrará el siguiente mensaje:

![https2](https://i.ibb.co/ft0TY8Y/Captura-de-pantalla-de-2020-12-15-17-05-46.png "HTTPS")

Tendremos que marcar, como se puede apreciar, la casilla **Confiar en esta CA para identificar sitios web.**, de manera que todos los certificados de servidores que hayan sido firmados por dicha autoridad certificadora, serán marcados de confianza y por tanto, no se mostrará la advertencia.

Posteriormente, recargaremos el navegador para así volver a mostrar la página **www.alvaro.gonzalonazareno.org**, accediendo desde HTTPS:

![https3](https://i.ibb.co/brmyh9d/Captura-de-pantalla-de-2020-12-15-17-06-17.png "HTTPS")

En esta ocasión, no se ha mostrado ninguna advertencia de seguridad, y como se puede apreciar al lado de la URL, se muestra un candado cerrado, indicando que la identidad del servidor ha podido ser correctamente validada. Si pulsamos en el mismo, se nos mostrará más información referente:

![https4](https://i.ibb.co/YkhF8VB/Captura-de-pantalla-de-2020-12-15-17-06-29.png "HTTPS")

En el mismo, se muestra mayor información técnica sobre el certificado, así como el nombre de la entidad certificadora, fecha de expiración, tipo de cifrado... De manera que podemos concluir que la configuración del protocolo HTTPS en la máquina **Quijote** ha funcionado correctamente.