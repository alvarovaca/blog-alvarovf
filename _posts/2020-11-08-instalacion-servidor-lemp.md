---
layout: post
title:  "Instalación de un servidor LEMP"
banner: "/assets/images/banners/lemp.jpg"
date:   2020-11-08 12:41:00 +0200
categories: servicios vps
---
Este es el primer artículo referente a la configuración del VPS, una máquina virtual contratada en OVH que cuenta con un direccionamiento público **51.210.109.246**. Además, se ha contratado una zona DNS en la que nombraremos dicha máquina junto a los servicios que despleguemos, en el dominio **iesgn19.es**.

La intención es la de ir desarrollando a lo largo del curso una serie de _posts_ en los que se detallen las configuraciones llevadas a cabo en dicho VPS, así como el mantenimiento de las mismas. En el día de hoy, vamos a _sentar las bases_ del proyecto, instalando para ello un servidor **LEMP**, que nos será necesario más adelante.

Se recomienda realizar una previa lectura del _post_ [Servidor Web Nginx](http://www.alvarovf.com/servicios/2020/11/07/servidor-web-nginx.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Tarea 1: Instalación de la pila LEMP.

Un servidor **LEMP** contiene una unión de diferentes tecnologías, con la que podremos definir la infraestructura de un servidor web. Las tecnologías implicadas son las siguientes:

* **L**inux, el sistema operativo (en este caso, Debian Buster).
* **E**ngine-X, el servidor web (en este caso, _nginx 1.14.2_).
* **M**ySQL/**M**ariaDB, el gestor de bases de datos.
* **P**HP, el lenguaje de programación.

Para servir las páginas web dinámicas escritas en PHP que posteriormente alojemos, necesitamos un **servidor de aplicaciones** capaz de interpretar dicho código PHP y convertirlo en HTML, un lenguaje legible por el navegador web, de manera que utilizaremos _php-fpm_. Además, necesitamos un **servidor web** que devuelva el HTML a los clientes tras las correspondientes peticiones, para ello, vamos a utilizar _nginx_, que se comunicará con el servidor de aplicaciones. Las páginas guardan sus datos en una base de datos, es decir, el código PHP que se ejecuta, accede a la base de datos mediante llamadas a la misma, por lo que también necesitaremos un **servidor de bases de datos**.

Los paquetes que tendremos que instalar para cada una de las tecnologías implicadas son:

* **L**inux - Suponemos que ya se encuentra instalado.
* **E**ngine-X - **nginx**.
* **M**ySQL/**M**ariaDB - **mariadb-client** y **mariadb-server**.
* **P**HP - **php**, **php-mysql** y **php-fpm**.

Esos son los paquetes a instalar para tener una pila LEMP totalmente operativa, no sin antes upgradear los paquetes instalados, para asegurarnos que tenemos todo al día. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@vps:~# apt update && apt upgrade && apt install nginx mariadb-client mariadb-server php php-mysql php-fpm
{% endhighlight %}

Listo, todos los paquetes se encuentran ya instalados.

Antes de continuar, hay que tener en cuenta un punto bastante importante. Este VPS está expuesto a Internet, es decir, tiene un direccionamiento accesible desde el exterior (sin necesidad de hacer uso de un mecanismo NAT), lo que significa que debemos extremar las precauciones, pues dentro del mismo, vamos a tener una base de datos en producción (situación poco típica, ya que en situaciones reales, la base de datos suele estar en una red interna que no sea accesible desde el exterior).

Para aumentar la seguridad de dicha base de datos, vamos a hacer uso de un _script_ que proporciona MariaDB, que llevará a cabo una configuración inicial pensada para ello:

* Estableceremos una contraseña para el administrador de MariaDB.
* Eliminaremos el usuario anónimo que viene creado por defecto.
* Desactivaremos la conexión remota a _root_, es decir, únicamente se podrá llevar a cabo desde _localhost_.
* Eliminaremos la base de datos de pruebas que viene creada por defecto.
* Volveremos a cargar las tablas de privilegios, para que los cambios surtan efecto.

Lo primero que preguntará el _script_ será la contraseña actual del usuario **root** de MariaDB. Lo dejaremos vacío y pulsaremos **ENTER**, ya que por ahora no tiene ninguna contraseña configurada. Tras ello, tendremos que ir introduciendo "**Y**" a las opciones que nos vayan saliendo, para así llevar a cabo toda la configuración anteriormente mencionada. Para ejecutar dicho _script_, haremos uso del comando:

{% highlight shell %}
root@vps:~# mysql_secure_installation

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
 - Removing privileges on tes

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

## Tarea 2: Creación de un VirtualHost al que vamos acceder con el nombre www.iesgn19.es.

El primer paso será crear el DocumentRoot para dicho VirtualHost. En este caso, he decidido que se encontrará en **/srv/iesgn19**, así que para llevar a cabo la creación de dicho directorio, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# mkdir /srv/iesgn19
{% endhighlight %}

Para verificar que la creación se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@vps:~# tree /srv/
/srv/
└── iesgn19

1 directory, 0 files
{% endhighlight %}

Efectivamente, el directorio ha sido generado, así que tras ello, tendremos que crear el correspondiente fichero de configuración para el sitio web, que se encuentra en **/etc/nginx/sites-available**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@vps:~# cd /etc/nginx/sites-available/
{% endhighlight %}

Tras ello, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls
default
{% endhighlight %}

En este caso, existe un único fichero, **default**, así que para no complicarnos, vamos a copiar dicho fichero para tener una plantilla base que posteriormente modificaremos y adaptaremos al sitio web que vamos a crear. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# cp default iesgn19
{% endhighlight %}

Para verificar que el fichero se ha copiado correctamente, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls
default  iesgn19
{% endhighlight %}

Ya está todo listo para modificar el fichero **iesgn19**, que contendrá la configuración de nuestro nuevo VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano iesgn19
{% endhighlight %}

El contenido de dicho fichero es el siguiente (eliminando líneas comentadas para dejar una salida más limpia):

{% highlight shell %}
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

En este caso, tendremos que eliminar las directivas **default_server**, pues queremos que el VirtualHost que se sirva por defecto (en caso de no encontrar coincidencia con el ServerName de ningún VirtualHost, como por ejemplo, cuando accedemos a través de la IP) siga siendo **default**. Explicaremos el por qué a continuación.

Dentro del mismo, tendremos que modificar además el **DocumentRoot** para hacer uso del nuevo directorio que hemos generado, **/srv/iesgn19**. Como es lógico, también tendremos que modificar el **ServerName** y poner el nombre de dominio a través del cuál queremos acceder, en este caso, **www.iesgn19.es**.

La apariencia actual del fichero sería:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/iesgn19;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn19.es;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

Posiblemente os estéis preguntando, ¿por qué hemos dejado el VirtualHost **default** como sitio por defecto? Bien, tiene una sencilla explicación, y es que si hubiésemos cambiado el VirtualHost por defecto al que acabamos de crear, si accediésemos mediante la IP (por ejemplo), aparecería todo el rato la dirección IP en la barra de búsqueda, en lugar de mostrar el nombre de dominio **www.iesgn19.es**.

Quizás, es un detalle sin importancia, pero afecta a la experiencia visual, así que para solucionarlo, vamos a añadir dentro del VirtualHost por defecto una directiva que permita hacer una redirección al nombre de dominio anteriormente mencionado. Para modificar el fichero de configuración de dicho sitio, ejecutaremos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano default
{% endhighlight %}

En este caso, queremos hacer una redirección que nos permita pedir al cliente que haga otra petición a una URL diferente (indicada en la cabecera de la respuesta HTTP junto a un código de estado **3xx**). Existen las redirecciones **Permanentes** (301) y **Temporales** (302). En este caso, haremos uso de una redirección permanente para que los navegadores se aprendan dicha redirección y no pregunten siempre al servidor, llevándola a cabo de forma automática.

La sintaxis para crear una redirección con _return_ es:

{% highlight shell %}
return <codigo> <URL>;
{% endhighlight %}

Donde:

* **codigo**: El código de estado que se va a devolver. Como hemos mencionado con anterioridad, **301** para redirecciones permanentes y **302** para redirecciones temporales. En este caso, haremos uso de una redirección permanente.
* **URL**: La ruta a la que deseamos redirigir al cliente. En este caso, la redirección la queremos hacer a **http://www.iesgn19.es**, pero además, en caso de que hayan solicitado un recurso, mantendremos dicho recurso en la URL, haciendo uso de la variable **$request_uri**, quedando finalmente **http://www.iesgn19.es$request_uri**.

En resumen, la redirección que queremos crear tendrá la siguiente forma:

{% highlight shell %}
return 301 http://www.iesgn19.es$request_uri;
{% endhighlight %}

El fichero de configuración final del VirtualHost por defecto tendrá el siguiente aspecto:

{% highlight shell %}
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }

        return 301 http://www.iesgn19.es$request_uri;
}
{% endhighlight %}

Tras ello, guardaremos los cambios en el mismo.

## Tarea 3: Cuando se acceda a www.iesgn19.es se nos redigirá a la página www.iesgn19.es/principal, en la que se mostrará una página web estática que mostrará nuestro nombre y una lista de enlaces a las aplicaciones que vamos a ir desplegando posteriormente.

Lo primero que tendremos que hacer será crear el nuevo directorio al que vamos a redirigir (**principal/**) dentro del DocumentRoot, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# mkdir /srv/iesgn19/principal
{% endhighlight %}

Para verificar que la creación se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# tree /srv/
/srv/
└── iesgn19
    └── principal

2 directories, 0 files
{% endhighlight %}

En este caso, y a diferencia de la redirección creada con anterioridad, queremos hacerlo de una forma más "granular", es decir, únicamente queremos se redirija en caso de acceder a **/**, mientras que si se accede a cualquier otro recurso, no. Es por ello, que en lugar de hacer uso de **return**, haremos uso de **rewrite**, pues permite el uso de expresiones regulares. Siempre que sea posible, se recomienda hacer uso de _return_, pues su procesamiento es más rápido, al no tener que evaluar expresiones regulares, pero no es el caso. Para modificar el fichero de configuración del VirtualHost, con la intención de definir la nueva redirección, ejecutaremos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano iesgn19
{% endhighlight %}

La sintaxis para crear una redirección con _rewrite_ es:

{% highlight shell %}
rewrite <pathURL> <URL> <tipo>;
{% endhighlight %}

Donde:

* **pathURL**: El recurso "virtual" que solicitaremos en la URL a la hora de hacer la petición al servidor. En este caso, la petición la haríamos a www.iesgn19.es**/**, por lo que pathURL sería **^/$**.
* **URL**: La ruta real a la que se va a mapear el pathURL, es decir, la URL a la que deseamos redirigir al cliente. En este caso, la redirección la queremos hacer a www.iesgn19.es**/principal**, por lo que al tratarse de una redirección dentro del mismo host, la URL sería **/principal**.
* **tipo**: El tipo de redirección que queremos llevar a cabo, ya sea temporal (**redirect**) o permanente (**permanent**). En este caso, es indiferente, así que usaremos una redirección temporal.

En resumen, la redirección que queremos crear tendrá la siguiente forma:

{% highlight shell %}
rewrite ^/$ /principal redirect;
{% endhighlight %}

El fichero de configuración del VirtualHost tendrá la siguiente forma:

{% highlight shell %}
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /srv/iesgn19;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn19.es;

        rewrite ^/$ /principal redirect;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

La nueva redirección ya ha sido declarada, pero todavía no tenemos un contenido existente que mostrar cuando se acceda a **principal/**, por lo que nos moveremos dentro de dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# cd /srv/iesgn19/principal/
{% endhighlight %}

Una vez dentro de dicho directorio, haremos uso de `wget` para descargar la [plantilla](https://www.html5webtemplates.co.uk/wp-content/uploads/2020/templates/simplestyle_8/index.html) para la página web estática que usaremos, cuyo enlace de descarga se encuentra [aquí](https://www.html5webtemplates.co.uk/wp-content/uploads/2020/05/simplestyle_8.zip). Para llevar a cabo la descarga, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# wget https://www.html5webtemplates.co.uk/wp-content/uploads/2020/05/simplestyle_8.zip
--2020-11-03 17:11:34--  https://www.html5webtemplates.co.uk/wp-content/uploads/2020/05/simplestyle_8.zip
Resolving www.html5webtemplates.co.uk (www.html5webtemplates.co.uk)... 185.151.30.151, 2a07:7800::151
Connecting to www.html5webtemplates.co.uk (www.html5webtemplates.co.uk)|185.151.30.151|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 28372 (28K) [application/zip]
Saving to: ‘simplestyle_8.zip’

simplestyle_8.zip                100%[==========================>]  27.71K  --.-KB/s    in 0s

2020-11-03 17:11:34 (121 MB/s) - ‘simplestyle_8.zip’ saved [28372/28372]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# ls -l
total 28
-rw-r--r-- 1 root root 28372 May  6 13:20 simplestyle_8.zip
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**simplestyle_8.zip**" con un peso total de **27.71 KB** (28372 bytes).

Al estar comprimido el fichero, no podemos llevar a cabo la visualización de la plantilla hasta que no hagamos una extracción de los ficheros contenidos. Dado que está comprimido en **.zip**, tendremos que hacer uso de la herramienta `unzip`, que no viene instalada por defecto, así que la instalaremos ejecutando el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# apt install unzip
{% endhighlight %}

Tras ello, ya podremos llevar a cabo la descompresión, haciendo uso de `unzip`:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# unzip simplestyle_8.zip
Archive:  simplestyle_8.zip
   creating: simplestyle_8/
  inflating: simplestyle_8/another_page.html
  inflating: simplestyle_8/contact.html
  inflating: simplestyle_8/examples.html
  inflating: simplestyle_8/index.html
  inflating: simplestyle_8/page.html
   creating: simplestyle_8/READ_ME/
  inflating: simplestyle_8/READ_ME/HTML5WebTemplates.co.uk.url
  inflating: simplestyle_8/READ_ME/PLEASE READ.txt
  inflating: simplestyle_8/READ_ME/Remove the footer link.URL
   creating: simplestyle_8/style/
  inflating: simplestyle_8/style/bullet.png
  inflating: simplestyle_8/style/graphic.png
  inflating: simplestyle_8/style/paperclip.png
  inflating: simplestyle_8/style/pattern.png
  inflating: simplestyle_8/style/style.css
  inflating: simplestyle_8/style/Thumbs.db
  inflating: simplestyle_8/style/transparent.png
{% endhighlight %}

Como se puede apreciar, ha mostrado por pantalla todos los ficheros que ha extraído, pero aun así, vamos a ejecutar el comando `ls -l` para ver la estructura de directorios existente:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# ls -l
total 32
drwxr-xr-x 4 root root  4096 Aug 12  2014 simplestyle_8
-rw-r--r-- 1 root root 28372 May  6 13:20 simplestyle_8.zip
{% endhighlight %}

Como suponía, ha creado un directorio en el que se encuentran todos los ficheros extraídos. Esto no es lo que queremos, ya que para visualizar la página web, tendríamos que acceder a **principal/simplestyle_8/**, en lugar de acceder simplemente a **principal/**. Es por ello, que moveremos todo el contenido de dicho directorio al directorio en el que nos encontramos actualmente, además de eliminar el directorio **simplestyle_8/** y el fichero comprimido **simplestyle_8.zip** para liberar espacio. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# mv simplestyle_8/* ./ && rm -r simplestyle_8 simplestyle_8.zip
{% endhighlight %}

Para verificar que los ficheros se han movido correctamente y que tanto el directorio como el comprimido han sido eliminados, volveremos a listar el contenido del directorio actual:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# ls -l
total 44
-rw-r--r-- 1 root root 5753 Nov 11  2013 another_page.html
-rw-r--r-- 1 root root 4050 Nov 11  2013 contact.html
-rw-r--r-- 1 root root 6768 Nov 11  2013 examples.html
-rw-r--r-- 1 root root 4504 Nov 11  2013 index.html
-rw-r--r-- 1 root root 5746 Nov 11  2013 page.html
drwxr-xr-x 2 root root 4096 Aug 12  2014 READ_ME
drwxr-xr-x 2 root root 4096 Nov 11  2013 style
{% endhighlight %}

Efectivamente, los ficheros han sido correctamente movidos y el directorio y el fichero comprimido, eliminados.

En mi caso, dado que no me gustaba el diseño de la página web, realicé una serie de modificaciones (cuya explicación se sale del objetivo de este _post_) en el HTML para adaptarlo a lo que se pide y dejarlo más atractivo y sencillo visualmente. En el siguiente apartado, veremos el resultado.

Una vez realizadas todas las configuraciones oportunas, es hora de habilitar el VirtualHost. Para ello, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/nginx/sites-available** dentro de **/etc/nginx/sites-enabled** de forma manual. Para ello, ejecutamos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# ln -s /etc/nginx/sites-available/iesgn19 /etc/nginx/sites-enabled/
{% endhighlight %}

Al parecer, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# systemctl reload nginx
{% endhighlight %}

Una vez que la configuración del servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/nginx/sites-enabled** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 34 Nov  3 15:11 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 34 Nov  3 16:14 iesgn19 -> /etc/nginx/sites-available/iesgn19
{% endhighlight %}

Efectivamente, los dos sitios se encuentran actualmente activos.

## Tarea 4: Realiza las modificaciones oportunas en la zona DNS y comprueba el acceso.

Antes de tratar de acceder al sitio, tendremos que llevar a cabo las correspondientes modificaciones en la zona DNS para que se pueda llevar a cabo la resolución. Para ello tendremos que tener la máquina virtual (VPS) nombrada con un **registro A**, que traduce nombres a direcciones IPv4 (en mi caso, la he nombrado como **vps**), de la siguiente forma:

![dns1](https://i.ibb.co/kMgcZG8/Captura-de-pantalla-de-2020-11-08-12-13-53.png "Registro A")

Tras ello, crearemos un **registro CNAME** para nombrar el servicio que apunte al **registro A** anteriormente creado, en este caso, con nombre **www**, de la siguiente forma:

![dns2](https://i.ibb.co/k9b55rQ/Captura-de-pantalla-de-2020-11-08-12-13-38.png "Registro CNAME")

Como se puede apreciar, las máquinas las nombraremos con **registros A** y tras ello, crearemos **registros CNAME** que apunten a dichas máquinas para nombrar a los servicios, de manera que en caso de que se lleve a cabo un cambio de dirección IP, tendremos que modificar un único registro A para la máquina, y no para todos los registros de los servicios que se encuentren alojados en la misma, ya que estarán apuntando a dicho registro A, por lo que el cambio se efectúa en todas ellas.

Es posible que la propagación del cambio tarde hasta 24 horas, pues necesario que expire la caché de los servidores DNS que conocían dicho nombre de dominio, para que así vuelvan a realizar la petición y reciban la nueva dirección IP.

Tras esperar un rato, accedí a **www.iesgn19.es** desde el navegador y el resultado fue el siguiente:

![iesgn1](https://i.ibb.co/4T1Cbq2/7.jpg "Página estática")

Como era de esperar, la redirección ha funcionado correctamente y nos ha mostrado el contenido de **www.iesgn19.es/principal**, por lo que podemos concluir también que la página estática se ha servido correctamente y muestra nuestro nombre y una lista de enlaces de prueba que he configurado.

## Tarea 5: Configura el nuevo VirtualHost para que pueda ejecutar PHP.

Como hemos mencionado anteriormente, necesitamos un servidor de aplicaciones para interpretar el código PHP y transformarlo en HTML para que pueda ser servido por _nginx_. Si recordamos, en _apache2_, gracias a un módulo que podíamos instalar, podíamos unificar el servidor web y el servidor de aplicaciones, cosa que no ocurre en _nginx_, por lo que tendremos que hacer uso de un servicio externo.

En este caso, hemos instalado **php-fpm**, pero todavía no lo hemos configurado para que tenga comunicación con el servidor web (pues han de ser capaces de comunicarse entre ellos para resolver correctamente las peticiones). Es por ello, que tenemos dos alternativas para comunicarlos:

* **Socket UNIX**: Permiten un intercambio de datos eficiente entre procesos que se ejecutan y comunican de manera local (no-remota). Al fin y al cabo, es un tipo de fichero en el que un proceso escribe y el otro lee, actuando como pasarela, al que se le aplican los permisos de ficheros UNIX, de manera que se pueden restringir qué procesos pueden leer y escribir en el mismo. Es más seguro, al no estar expuestos a Internet y únicamente ser accesibles de manera local. Cuentan con un mayor rendimiento.
* **Socket TCP/IP**: Identifica un servidor basado en una dirección IP y un puerto, por ejemplo, cuando alojamos un servidor web y lo hacemos en la dirección **192.168.1.2** y en el puerto **80** y a dicha dirección y puerto se conecta una máquina cliente con dirección **192.168.1.3** a través de un puerto no privilegiado, como el **3412**, entre los que existe una comunicación bidireccional, de manera que suele utilizarse para conexiones remotas. Es más inseguro, al ser accesible por cualquier persona, a no ser que se implemente un firewall o un método de autentificación. Cuentan con menor rendimiento.

En este caso, tras comparar ambas posibilidades, y dadas las circunstancias, haremos uso de un **Socket UNIX**, pues aumentaremos así la seguridad y el rendimiento. En realidad, también podríamos haber alojado un Socket TCP/IP en la dirección local (**127.0.0.1**), pero contaríamos con las desventajas del mismo.

El primer paso será configurar el método de escucha del servicio **php-fpm**, configuración que se llevará a cabo en un archivo del grupo de recursos, pues dicho servicio puede ejecutar múltiples grupos de procesos con diferentes configuraciones, pero en este caso, haremos uso del que viene por defecto, **www.conf**. Para comprobar qué configuración está usando actualmente, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# cat /etc/php/7.3/fpm/pool.d/www.conf | egrep 'listen ='
listen = /run/php/php7.3-fpm.sock
{% endhighlight %}

Como se puede apreciar, hemos leído el contenido de dicho fichero, filtrando por una cadena que se usa para establecer el método de escucha del servicio. En este caso, estamos usando por defecto un **Socket UNIX** alojado en **/run/php/php7.3-fpm.sock**, por lo que no tendremos que llevar a cabo ninguna modificación a lo que respecta.

Como hemos visto anteriormente, a los ficheros tipo _Socket UNIX_ se les puede cambiar los permisos, por lo que tendremos que verificar que los permisos de lectura y escritura son los correctos, de manera que el usuario **www-data**, que es el usuario que usa _nginx_ por defecto, tenga los permisos necesarios. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# cat /etc/php/7.3/fpm/pool.d/www.conf | egrep 'listen\.'
;listen.backlog = 511
listen.owner = www-data
listen.group = www-data
;listen.mode = 0660
; When set, listen.owner and listen.group are ignored
;listen.acl_users =
;listen.acl_groups =
;listen.allowed_clients = 127.0.0.1
{% endhighlight %}

De nuevo, hemos vuelto a leer el contenido del fichero y a filtrar por una cadena, que nos ha devuelto el propietario, el grupo y los permisos de dicho fichero, que como se puede apreciar, son los correctos.

Ya hemos recorrido la mitad del camino, pues hemos configurado correctamente el servicio **php-fpm**, pero todavía nos queda configurar **nginx** para indicarle la ruta de dicho _Socket UNIX_. Para ello, procedemos a modificar el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# nano /etc/nginx/sites-available/iesgn19
{% endhighlight %}

Dentro del mismo, encontraremos un bloque **location** comentado, que afecta a **~ \.php$**, es decir, a todos los ficheros **.php**. Tendremos que descomentar las líneas **include** y **fastcgi_pass**, indicando en ésta última la cadena **unix:** seguido de la ruta del _Socket UNIX_ (**/run/php/php7.3-fpm.sock**). Además de ello, tendremos que añadir el fichero **index.php** a la lista de la directiva **index**, para que así también se haga la correspondiente búsqueda para dicho fichero y pueda ser servido. El resultado final sería:

{% highlight shell %}
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /srv/iesgn19;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name www.iesgn19.es;

        rewrite ^/$ /principal redirect;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.3-fpm.sock;
        }
}
{% endhighlight %}

Como se puede apreciar, hemos incluido los **snippets** (pequeñas partes reutilizables de código fuente, que en este caso hemos incluido para el correcto funcionamiento del servicio), además de indicar la ruta del _Socket UNIX_ que ha de utilizar. Tras ello, guardaremos los cambios.

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# systemctl reload nginx
{% endhighlight %}

En un principio, _nginx_ ya tiene comunicación con _php-fpm_, pero para comprobarlo, vamos a generar un fichero PHP de nombre **info.php** que devuelva la información de PHP, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# echo "<?php phpinfo(); ?>" > ./info.php
{% endhighlight %}

Tras ello, podremos solicitar dicho recurso desde el navegador, obteniendo el siguiente resultado:

![php1](https://i.ibb.co/8BjtrSY/Captura-de-pantalla-de-2020-11-05-13-53-11.png "phpinfo")

Como se puede apreciar, el recurso ha sido correctamente generado y devuelto en HTML para su correspondiente visualización, por lo que podemos concluir que el servidor de apliciones **php-fpm** y el servidor web **nginx** tienen conectividad y están funcionando correctamente.

Por último, me gustaría dejar una pequeña anotación. Por defecto, el usuario a través del cuál se sirven las páginas web en _apache2_ es **www-data** (en este caso no habría inconveniente ya que el grupo "otros", al que pertenece dicho usuario tiene permisos de **lectura**, que es lo único que necesitamos para servir el fichero, pero en caso de necesitar escribir en dichos ficheros, no contaría con dichos permisos de **escritura**), por lo que podemos proceder a cambiar dicho propietario y grupo de forma recursiva, en caso de así desearlo, ya que el usuario y grupo actual es **root**, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@vps:/srv/iesgn19/principal# chown -R www-data:www-data /srv/iesgn19/
{% endhighlight %}

Gracias al comando ejecutado con anterioridad, todos los directorios y ficheros hijos de **/srv/iesgn19**, incluido el propio directorio, serían ahora propiedad de **www-data** y haciendo uso del grupo **www-data**.