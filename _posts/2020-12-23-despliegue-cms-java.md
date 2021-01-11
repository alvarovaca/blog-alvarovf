---
layout: post
title:  "Despliegue de un CMS Java"
banner: "/assets/images/banners/java.png"
date:   2020-12-23 18:58:00 +0200
categories: aplicaciones
---
La finalidad de este artículo es la de desplegar un CMS **Java**, concretamente una herramienta de nombre **Apache Guacamole** que admite el uso de los principales protocolos de administración remota como **VNC**, **RDP** y **SSH**, de manera que podremos unificar la gestión de todos nuestros servidores a través de su interfaz web, con las ventajas en cuanto a comodidad que ello nos ofrece.

Sigue la filosofía de desarrollo del código abierto y es software libre, además de contar una gran comunidad de desarrolladores detrás. Utiliza un conjunto de APIs que están ampliamente documentadas, incluyendo tutoriales básicos y descripciones conceptuales en su [manual en línea](https://guacamole.apache.org/doc/gug/installing-guacamole.html).

Para este caso, he instalado previamente una máquina virtual con Debian Buster que es la que actuará como servidora (con dirección IP **192.168.50.2**) y sobre la que instalaremos la correspondiente aplicación. Considero que la explicación de dicha instalación se sale del objetivo de este artículo, así que vamos a obviarla.

El primer paso consiste en instalar las herramientas y dependencias necesarias para el correcto funcionamiento de nuestra aplicación, entre las que se encuentran el soporte para múltiples protocolos, ya que como se puede intuir, existe una gran cantidad de protocolos soportados y seremos nosotros quiénes decidan, mediante las dependencias, cúales se podrán usar y cuáles no.

En este caso, no he escatimado y he instalado el soporte para todos los protocolos admitidos por la aplicación, no sin antes actualizar toda la paquetería instalada en la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@java:~# apt update && apt upgrade && apt install build-essential libcairo2-dev libjpeg62-turbo-dev libpng-dev libtool-bin libossp-uuid-dev libavcodec-dev libavformat-dev libswscale-dev freerdp2-dev libpango1.0-dev libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev libpulse-dev libvorbis-dev libwebp-dev
{% endhighlight %}

Una vez instaladas las herramientas y dependencias necesarias, podremos proceder con la instalación del CMS.

Para esta ocasión, vamos a utilizar la última versión de Apache Guacamole (1.2.0), que podremos descargar desde la [página oficial](https://guacamole.apache.org/), concretamente desde [aquí](http://archive.apache.org/dist/guacamole/1.2.0/source/guacamole-server-1.2.0.tar.gz). Para ello, haremos uso de `wget` para descargar el paquete comprimido de Guacamole desde dicha web:

{% highlight shell %}
root@java:~# wget http://archive.apache.org/dist/guacamole/1.2.0/source/guacamole-server-1.2.0.tar.gz
--2020-12-18 10:58:57--  http://archive.apache.org/dist/guacamole/1.2.0/source/guacamole-server-1.2.0.tar.gz
Resolviendo archive.apache.org (archive.apache.org)... 138.201.131.134, 2a01:4f8:172:2ec5::2
Conectando con archive.apache.org (archive.apache.org)[138.201.131.134]:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1048759 (1,0M) [application/x-gzip]
Grabando a: “guacamole-server-1.2.0.tar.gz”

guacamole-server-1.2.0.tar.gz                100%[==========================>]  1,00M  2,06MB/s    en 0,5s

2020-12-18 10:58:58 (1.88 MB/s) - ‘guacamole-server-1.2.0.tar.gz’ saved [1048759/1048759]
{% endhighlight %}

Para verificar que la descarga se ha llevado a cabo correctamente, vamos a listar el contenido del directorio actual, ejecutando para ello el comando:

{% highlight shell %}
root@java:~# ls -l
total 1028
-rw-r--r-- 1 root root 1048759 Jul  3 03:57 guacamole-server-1.2.0.tar.gz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**guacamole-server-1.2.0.tar.gz**" con un peso total de **1 MB** (1048759 bytes).

Al estar comprimido el fichero, no podemos llevar a cabo la instalación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@java:~# tar -zxf guacamole-server-1.2.0.tar.gz && rm guacamole-server-1.2.0.tar.gz
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
root@java:~# ls -l
total 1032
drwxr-xr-x 8 1001 users    4096 Jun 26 02:49 guacamole-server-1.2.0
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido en un directorio de nombre **guacamole-server-1.2.0**, tal y como queríamos, de manera que nos moveremos dentro del mismo para visualizar su contenido, ejecutando para ello el comando:

{% highlight shell %}
root@java:~# cd guacamole-server-1.2.0/
{% endhighlight %}

Tras ello, listaremos el contenido existente una vez más para verificar que el fichero se ha descomprimido correctamente:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# ls -l
total 736
-rw-r--r--  1 root root  55997 jun 26  2020 aclocal.m4
drwxr-xr-x  2 root root   4096 jun 26  2020 bin
drwxr-xr-x  2 root root   4096 jun 26  2020 build-aux
-rw-r--r--  1 root root   6229 jun 26  2020 config.h.in
-rwxr-xr-x  1 root root 552590 jun 26  2020 configure
-rw-r--r--  1 root root  34761 jun 26  2020 configure.ac
-rw-r--r--  1 root root   1984 ene 26  2020 CONTRIBUTING
drwxr-xr-x  2 root root   4096 jun 26  2020 doc
-rw-r--r--  1 root root   4116 jun 24  2020 Dockerfile
-rw-r--r--  1 root root  11358 ene 26  2020 LICENSE
drwxr-xr-x  2 root root   4096 jun 26  2020 m4
-rw-r--r--  1 root root   2612 jun 20  2020 Makefile.am
-rw-r--r--  1 root root  30855 jun 26  2020 Makefile.in
-rw-r--r--  1 root root    165 jun 20  2020 NOTICE
-rw-r--r--  1 root root   6108 jun 20  2020 README
drwxr-xr-x 12 root root   4096 jun 26  2020 src
drwxr-xr-x  2 root root   4096 jun 26  2020 util
{% endhighlight %}

Una vez conocido el contenido de dicho directorio, considero necesario hacer un pequeño inciso para explicar qué vamos a continuación, ya que Apache Guacamole está dividido en dos subproyectos:

* **guacamole-client**: Es la aplicación web que sirve el cliente Guacamole a los clientes.
* **guacamole-server**: Es el proxy de escritorio remoto con el que se comunica la aplicación web.

Actualmente, estamos llevando a cabo la instalación de **guacamole-server**, cuyo requisito es que debe ser compilado e instalado a mano, a diferencia de **guacamole-client**, tal y como veremos a continuación.

Explicar con detenimiento cómo funciona una compilación sería bastante extenso, por lo que [aquí](https://www.alvarovf.com/sistemas/2020/10/31/compilacion-usando-makefile.html) se puede encontrar otro artículo de mi blog en el que se explica y muestra un ejemplo práctico, aunque no es un requisito conocer cómo funciona para hacerlo.

En resumen, para la compilación del mismo, se nos ofrece un script de nombre **configure**, cuyo propósito es el de determinar las librerías instaladas en el sistema y en base a ello, seleccionar los componentes apropiados para la construcción del proxy de escritorio remoto. Lo ejecutaremos haciendo para ello uso del comando:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# ./configure --with-init-dir=/etc/init.d
...
------------------------------------------------
guacamole-server version 1.2.0
------------------------------------------------

   Library status:

     freerdp2 ............ yes
     pango ............... yes
     libavcodec .......... yes
     libavformat.......... yes
     libavutil ........... yes
     libssh2 ............. yes
     libssl .............. yes
     libswscale .......... yes
     libtelnet ........... yes
     libVNCServer ........ yes
     libvorbis ........... yes
     libpulse ............ yes
     libwebsockets ....... yes
     libwebp ............. yes
     wsock32 ............. no

   Protocol support:

      Kubernetes .... yes
      RDP ........... yes
      SSH ........... yes
      Telnet ........ yes
      VNC ........... yes

   Services / tools:

      guacd ...... yes
      guacenc .... yes
      guaclog .... yes

   FreeRDP plugins: /usr/lib/x86_64-linux-gnu/freerdp2
   Init scripts: /etc/init.d
   Systemd units: no

Type "make" to compile guacamole-server.
{% endhighlight %}

Donde:

* **--with-init-dir**: Prepara la compilación para que se instale un script de inicio para **guacd** en **/etc/init.d**, permitiendo posteriormente levantar el servicio durante el arranque de la máquina de una forma mucho más sencilla.

Como se puede apreciar, el script ha encontrado todas las librerías necesarias, tanto obligatorias como opcionales, soportando por tanto un total de 5 protocolos. Ya se han seleccionado los componentes apropiados, de manera que faltará compilarlos e instalarlos en los correspondientes directorios de la máquina, ejecutando para ello los comandos:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# make
root@java:~/guacamole-server-1.2.0# make install
{% endhighlight %}

Cuando **guacamole-server** haya sido instalado en la máquina, tendremos que crear los enlaces para las nuevas librerías compartidas (_shared libraries_) instaladas en el sistema, además de volver a cargar todas las unidades de **systemd** con la finalidad de poder hacer uso del nuevo servicio, haciendo para ello uso de los comandos:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# ldconfig
root@java:~/guacamole-server-1.2.0# systemctl daemon-reload
{% endhighlight %}

Tras ello, todo estará listo para administrar el demonio de nombre **guacd** en la máquina, que debemos arrancar y habilitar, de manera que se inicie cada vez que se arranque la máquina, ejecutando para ello los comandos:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# systemctl start guacd.service
root@java:~/guacamole-server-1.2.0# systemctl enable guacd.service
{% endhighlight %}

Una vez habilitado y arrancado el demonio, vamos a comprobar el estado del mismo para asegurarnos de que no ha habido ningún problema durante su inicio, haciendo para ello uso del comando:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# systemctl status guacd.service
● guacd.service - LSB: Guacamole proxy daemon
   Loaded: loaded (/etc/init.d/guacd; generated)
   Active: active (running) since Wed 2020-12-23 10:41:36 GMT; 7s ago
     Docs: man:systemd-sysv-generator(8)
    Tasks: 1 (limit: 544)
   Memory: 11.3M
   CGroup: /system.slice/guacd.service
           └─8638 /usr/local/sbin/guacd -p /var/run/guacd.pid

Dec 23 10:41:36 java systemd[1]: Starting LSB: Guacamole proxy daemon...
Dec 23 10:41:36 java guacd[8636]: Guacamole proxy daemon (guacd) version 1.2.0 started
Dec 23 10:41:36 java guacd[8635]: Starting guacd: guacd[8636]: INFO:        Guacamole proxy daemon (guacd) version 1.2.0 started
Dec 23 10:41:36 java guacd[8635]: SUCCESS
Dec 23 10:41:36 java systemd[1]: Started LSB: Guacamole proxy daemon.
Dec 23 10:41:36 java guacd[8638]: Listening on host 127.0.0.1, port 4822
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el servicio se encuentra activo y escuchando peticiones en un socket TCP/IP alojado en **localhost**, concretamente en el puerto **4822**. Dicha información nos será necesario para comunicar la aplicación web con el mismo.

La instalación del componente servidor de Apache Guacamole ha finalizado, de manera que vamos a proceder a instalar la aplicación web que se comunicará con el proxy de escritorio remoto. En este caso, podríamos compilarla e instalarla a mano, pero no es necesario, ya el servidor de aplicaciones que vamos a instalar, de nombre **Tomcat** nos proporciona muchas facilidades para ello.

Para entender qué vamos a hacer, necesitamos entender tres conceptos esenciales sobre Java:

* **Servlet**: Programa que se ejecuta en el servidor y que genera HTML. Es similar a una aplicación en _Flask_ o en _Django_, con plantillas que generarán HTML.
* **Tomcat**: Lo necesitaremos para servir un programa escrito en Java (Servlet). Es el servidor de aplicaciones y tiene un gran uso a día de hoy. Se suele conectar mediante el protocolo HTTP o AJP con un servidor web que actúe como proxy inverso, redirigiendo las peticiones a Tomcat. Escucha por defecto en el puerto 8080.
* **Fichero WAR (_Web Application Archive_)**: Ficheros comprimidos que contienen una aplicación web Java para desplegar en Tomcat.

Como hemos mencionado, además del servidor de aplicaciones, necesitaremos un servidor web que será el que atienda peticiones en el puerto 80 por parte de los clientes, actuando como proxy inverso y redirigiéndolas en caso de ser necesario, a Tomcat. Por ello, tendremos que instalar ambos servicios, ejecutando para ello el comando:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# apt install tomcat9 apache2
{% endhighlight %}

Una vez que los paquetes hayan sido correctamente instalados, todo estará listo para comenzar con la instalación y configuración de la aplicación web, no sin antes verificar que se están escuchando peticiones en los puertos **4822** (**guacd**), **80** (**apache2**) y **8080** (**tomcat**), haciendo para ello uso del comando:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:4822          0.0.0.0:*               LISTEN      8638/guacd          
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      369/sshd            
tcp6       0      0 :::8080                 :::*                    LISTEN      11048/java          
tcp6       0      0 :::80                   :::*                    LISTEN      9469/apache2        
tcp6       0      0 :::22                   :::*                    LISTEN      369/sshd
{% endhighlight %}

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los sockets que están actualmente escuchando peticiones (State = LISTEN).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho socket.

Efectivamente el proceso **guacd** está escuchando peticiones en el puerto **4822**, el proceso **java** en el puerto **8080** y el proceso **apache2**, en el puerto **80**, tal y como debería.

Implantar una aplicación web Java no es una tarea compleja, ya que por defecto, cualquier fichero **.war** que se copie o mueva dentro del directorio **/var/lib/tomcat9/webapps/** se desplegaría automáticamente. Es importante mencionar que en mucha documentación, la variable de entorno **$CATALINA_HOME** hace referencia al directorio en cuestión, que podrá variar dependiendo de la instalación. Para seguir trabajando, nos moveremos dentro de dicho directorio ejecutando para ello el comando:

{% highlight shell %}
root@java:~/guacamole-server-1.2.0# cd /var/lib/tomcat9/webapps/
{% endhighlight %}

Una vez dentro del mismo, tendremos que descargar el fichero **.war** correspondiente a la última versión de Apache Guacamole (1.2.0), que podremos descargar [aquí](http://archive.apache.org/dist/guacamole/1.2.0/binary/guacamole-1.2.0.war), haciendo para ello uso, una vez más, de `wget`:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# wget -O 'guacamole.war' 'http://archive.apache.org/dist/guacamole/1.2.0/binary/guacamole-1.2.0.war'
--2020-12-14 07:44:16--  http://archive.apache.org/dist/guacamole/1.2.0/binary/guacamole-1.2.0.war
Resolviendo archive.apache.org (archive.apache.org)... 138.201.131.134, 2a01:4f8:172:2ec5::2
Conectando con archive.apache.org (archive.apache.org)[138.201.131.134]:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 12249847 (12M)
Grabando a: “guacamole.war”

guacamole.war                100%[==========================>]  11.68M  5.38MB/s    in 2.2s    

2020-12-14 07:44:19 (5.38 MB/s) - ‘guacamole.war’ saved [12249847/12249847]
{% endhighlight %}

Donde:

* **wget -O**

Para verificar que la descarga se ha llevado a cabo correctamente y con el nombre adecuado, vamos a listar el contenido del directorio actual, ejecutando para ello el comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# ls -l
total 11968
-rw-r--r-- 1 root root 12249847 Jul  3 03:57 guacamole.war
drwxr-xr-x 3 root root     4096 Dec 18 09:14 ROOT
{% endhighlight %}

Efectivamente, se ha descargado un fichero de nombre "**guacamole.war**" con un peso total de **11.68 MB** (12249847 bytes).

Tras ello, esperaremos unos segundos y volvemos a listar el contenido del directorio actual, haciendo uso una vez más del mismo comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# ls -l
total 11972
drwxr-x--- 11 tomcat tomcat     4096 Dec 18 09:16 guacamole
-rw-r--r--  1 root   root   12249847 Jul  3 03:57 guacamole.war
drwxr-xr-x  3 root   root       4096 Dec 18 09:14 ROOT
{% endhighlight %}

Como se puede apreciar, un nuevo directorio de nombre **guacamole** ha aparecido, resultado de desplegar la aplicación web contenida en el fichero **guacamole.war**. En un principio, la aplicación sin ningún tipo de configuración ya sería accesible a través del puerto **8080**, ya que Tomcat está escuchando peticiones en el mismo, sin embargo, no es lo que buscamos.

En este caso, tenemos un servidor web **apache2**, que servirá como proxy inverso para escuchar las peticiones en el puerto **80** y redirigirlas, en caso de ser necesario, al servidor de aplicaciones Tomcat, el cuál generará el código HTML que será devuelto a los clientes. Para que nuestra estructura funcione de esta manera, será necesario llevar a cabo unas modificaciones en el servidor web, para así conectarlo con Tomcat.

Para ello, dado que _apache2_ viene con un **VirtualHost** configurado, vamos a modificar el mismo adaptándolo a nuestras necesidades, en lugar de crear uno nuevo. Explicar con detalle cómo funcionan los VirtualHost en _apache2_ puede ser bastante extenso, además de salirse del objetivo de este articulo, de manera que [aquí](https://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html) podrás encontrar otro _post_ en el que se explica con mayor detalle.

Realmente, podríamos crear un VirtualHost nuevo, en otra ruta diferente, pero ya que _apache2_ nos ofrece dicha facilidad, vamos a aprovecharla. Para ello, vamos a proceder a modificar el fichero de configuración del VirtualHost por defecto ubicado en **/etc/apache2/sites-available/**, de nombre **000-default**, haciendo para ello uso del comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# nano /etc/apache2/sites-available/000-default.conf
{% endhighlight %}

Dentro del mismo, encontraremos la siguiente configuración por defecto (tras eliminar líneas comentadas para así dejar una salida más limpia):

{% highlight shell %}
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
{% endhighlight %}

La primera modificación a llevar a cabo en la configuración por defecto consiste en establecer correctamente el **DocumentRoot**, pues ha pasado de ser **/var/www/html** a ser **/var/lib/tomcat9/webapps/guacamole**.

Tras ello, tendremos que añadir nuevas directivas para hacer que actúe como proxy inverso, así como indicar la ruta de la aplicación web a partir del directorio **/var/lib/tomcat9/webapps**. El resultado final del VirtualHost sería:

{% highlight shell %}
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/lib/tomcat9/webapps/guacamole

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Proxy *>
                Order deny,allow
                Allow from all
        </Proxy>

        ProxyRequests Off
        ProxyPass / http://localhost:8080/guacamole/
        ProxyPassReverse / http://localhost:8080/guacamole/
</VirtualHost>
{% endhighlight %}

Sin embargo, no basta con modificar el fichero de configuración del VirtualHost para permitir que actúe como un proxy inverso en este caso, sino que tenemos que habilitar manualmente el módulo pensado para ello, de nombre **proxy_http** (en este caso, ya que estamos utilizando el protocolo HTTP). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# a2enmod proxy_http
Considering dependency proxy for proxy_http:
Enabling module proxy.
Enabling module proxy_http.
To activate the new configuration, you need to run:
  systemctl restart apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, el módulo ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que reiniciar el servicio _apache2_, haciendo para ello uso del comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# systemctl restart apache2
{% endhighlight %}

El servicio **apache2** ha sido correctamente reiniciado y por tanto, el módulo **proxy_http** ha sido habilitado.

Por defecto, **Guacamole** espera encontrar todos sus ficheros de configuración dentro del directorio **/etc/guacamole**, de manera que vamos a ubicar todos los ficheros correspondientes a la aplicación web dentro de dicha ruta. Lo primero que haremos será crear dicho directorio, haciendo para ello uso de `mkdir`:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# mkdir /etc/guacamole
{% endhighlight %}

Una vez el directorio haya sido generado, nos moveremos dentro del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@java:/var/lib/tomcat9/webapps# cd /etc/guacamole/
{% endhighlight %}

El primer fichero que vamos a generar es aquel de nombre **guacamole.properties**, en el que definiremos cómo se va a comunicar con el _daemon_ **guacd**, haciendo para ello uso del comando:

{% highlight shell %}
root@java:/etc/guacamole# nano guacamole.properties
{% endhighlight %}

Dentro del mismo tendremos que indicar la dirección del servidor en el que se encuentra en ejecución el demonio, así como el puerto que está utilizando y la ruta del fichero en el que posteriormente vamos a definir los usuarios que podrán utilizar la aplicación y las máquinas que se podrán utilizar, quedando en mi caso, de la siguiente forma:

{% highlight shell %}
guacd-hostname: localhost
guacd-port:     4822

auth-provider: net.sourceforge.guacamole.net.basic.BasicFileAuthenticationProvider
basic-user-mapping: /etc/guacamole/user-mapping.xml
{% endhighlight %}

Como se puede apreciar en la directiva **basic-user-mapping**, el fichero que se va a utilizar para _mapear_ los usuarios será **user-mapping.xml**, así que tendremos que generarlo, no sin antes encriptar una contraseña que utilizará el usuario de prueba que vamos a generar. Para ello, vamos a utilizar el algoritmo **md5**, encriptando en este caso, la contraseña **admin**, ejecutando para ello el comando:

{% highlight shell %}
debian@vps:~$ echo -n "admin" | openssl md5
(stdin)= 21232f297a57a5a743894a0e4a801fc3
{% endhighlight %}

Donde:

* **-n**: Evitamos que añada un salto de línea al final de la cadena introducida.

El resultado de haber encriptado la contraseña **admin** con el algoritmo **md5** es **21232f297a57a5a743894a0e4a801fc3**, por lo que la copiaremos en el portapapeles ya que nos será necesario a continuación.

El segundo fichero que vamos a generar es aquel de nombre **user-mapping.xml**, en el que definiremos, como acabamos de mencionar, los usuarios que podrán hacer uso de la aplicación web y las máquinas a las que se podrán conectar, haciendo para ello uso del comando:

{% highlight shell %}
root@java:/etc/guacamole# nano user-mapping.xml
{% endhighlight %}

Dentro del mismo tendremos que indicar al menos un nombre de usuario y la contraseña encriptada asociada, que en mi caso será un usuario de nombre **admin**. Además, añadiré en mi caso dos máquinas de prueba y configuraré las directivas necesarias para hacer uso del protocolo SSH (aunque podría haber utilizado cualquier otro, como VNC), quedando en mi caso, de la siguiente forma:

{% highlight shell %}
<user-mapping>
    <authorize
         username="admin"
         password="21232f297a57a5a743894a0e4a801fc3"
         encoding="md5">
      
       <connection name="Anfitrión">
         <protocol>ssh</protocol>
         <param name="hostname">192.168.1.134</param>
         <param name="port">22</param>
         <param name="username">alvaro</param>
         <param name="password">[contraseña]</param>
       </connection>

       <connection name="Servidor">
         <protocol>ssh</protocol>
         <param name="hostname">192.168.1.135</param>
         <param name="port">22</param>
         <param name="username">alvaro</param>
         <param name="password">[contraseña]</param>
       </connection>
    </authorize>
</user-mapping>
{% endhighlight %}

Tras ello, guardaremos los cambios en el fichero y procederemos a reiniciar los servicios **tomcat9** y **guacd** para que la nueva configuración surta efecto, ejecutando para ello el comando:

{% highlight shell %}
root@java:/etc/guacamole# systemctl restart tomcat9 guacd
{% endhighlight %}

Cuando el correspondiente reinicio de los servicios haya finalizado, accederemos desde el navegador web al puerto **80** de la máquina **192.168.50.2**, obteniendo el siguiente resultado:

![guacamole1](https://i.ibb.co/Dgk8F0r/Captura-de-pantalla-de-2020-12-23-11-48-18.png "Log-in")

Como se puede apreciar, el servidor web **apache2** ha sido capaz de servir correctamente la aplicación **Guacamole**, por lo que podemos concluir que está haciendo correctamente la función de proxy inverso y tiene comunicación con el servidor de aplicaciones **Tomcat**.

Tras loguearme con el usuario **admin** y la contraseña **admin**, se me ha mostrado lo siguiente:

![guacamole2](https://i.ibb.co/K6mVpfD/Captura-de-pantalla-de-2020-12-23-11-48-08.png "Inicio")

En este caso, el acceso ha funcionado correctamente y en la parte inferior se muestran todas las conexiones disponibles, referentes a las dos únicas máquinas que he configurado para ello. Para hacer la prueba, me he conectado a la primera de ellas, de nombre "**Anfitrión**", obteniendo el siguiente resultado:

![guacamole3](https://i.ibb.co/WBnhVjZ/Captura-de-pantalla-de-2020-12-23-11-48-26.png "Conexión a máquina")

La conexión a mi máquina anfitriona ha funcionado a la perfección, pudiendo ejecutar comandos como si de una conexión SSH normal y corriente se tratase.

Es importante mencionar que dadas las "limitaciones" de dicho protocolo, no es posible mostrar la interfaz gráfica de la máquina, pero si se hubiese hecho uso de **VNC**, por ejemplo, sí lo hubiese sido.