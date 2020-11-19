---
layout: post
title:  "Migración de aplicaciones web PHP"
banner: "/assets/images/banners/migracion.jpg"
date:   2020-11-19 12:07:00 +0200
categories: aplicaciones vps
---
Este es el segundo artículo referente a la configuración del VPS, una máquina virtual contratada en OVH que cuenta con un direccionamiento público **51.210.109.246**. Además, se ha contratado una zona DNS en la que nombraremos dicha máquina junto a los servicios que despleguemos, en el dominio **iesgn19.es**.

La intención es la de ir desarrollando a lo largo del curso una serie de _posts_ en los que se detallen las configuraciones llevadas a cabo en dicho VPS, así como el mantenimiento de las mismas. En el día de hoy, vamos a migrar un sitio _Drupal_ que fue anteriormente [instalado](http://www.alvarovf.com/aplicaciones/2020/10/29/instalacion-cms-php.html), para así desplegarlo en la VPS, además de un sitio _Nextcloud_, que instalaremos y migraremos.

Se recomienda realizar una previa lectura del _post_ [Instalación de un servidor LEMP](http://www.alvarovf.com/servicios/vps/2020/11/08/instalacion-servidor-lemp.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Tarea 1: Drupal

### Crea un nuevo VirtualHost accesible con el nombre portal.iesgn19.es.

El primer paso será crear el DocumentRoot para dicho VirtualHost. En este caso, he decidido que se encontrará en **/srv/drupal**, así que para llevar a cabo la creación de dicho directorio, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@vps:~# mkdir /srv/drupal
{% endhighlight %}

Tras ello, modificaremos el usuario y el grupo propietario de dicho directorio a **debian**, pues nos será necesario más adelante. Para ello, haremos uso de `chown`:

{% highlight shell %}
root@vps:~# chown debian:debian /srv/drupal/
{% endhighlight %}

Para verificar que la creación se ha llevado a cabo correctamente y que el usuario y el grupo propietario han sido modificados, listaremos el contenido de **/srv/**, haciendo uso del comando `ls -l`:

{% highlight shell %}
root@vps:~# ls -l /srv/
total 12
drwxr-xr-x 2 debian   debian   4096 Nov 11 10:19 drupal
drwxr-xr-x 3 www-data www-data 4096 Nov  3 16:24 iesgn19
{% endhighlight %}

Efectivamente, el directorio ha sido generado y su usuario y grupo propietario correctamente modificados, así que tras ello, tendremos que crear el correspondiente fichero de configuración para el sitio web, que se encuentra en **/etc/nginx/sites-available/**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@vps:~# cd /etc/nginx/sites-available/
{% endhighlight %}

Tras ello, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls
default  iesgn19
{% endhighlight %}

En este caso, y para no complicarnos, vamos a copiar el fichero **iesgn19** creado con anterioridad, para así tener una plantilla base que posteriormente modificaremos y adaptaremos al sitio web que vamos a crear. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# cp iesgn19 drupal
{% endhighlight %}

Para verificar que el fichero se ha copiado correctamente, volveremos a listar el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls
default  drupal  iesgn19
{% endhighlight %}

Ya está todo listo para modificar el fichero **drupal**, que contendrá la configuración de nuestro nuevo VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano drupal
{% endhighlight %}

Si recordamos, en _apache2_ hacíamos uso de un fichero **.htaccess** que contenía una determinada configuración para el VirtualHost que no había sido previamente definida en el propio fichero de configuración, pero ésta funcionalidad no existe en _nginx_, por lo que tendremos que buscar una alternativa para que las directivas que se incluían en el interior del **.htaccess** (necesarias para el correcto funcionamiento del CMS), sigan aplicándose.

Para ello, tras leer documentación al respecto, la propia página de _nginx_ nos proporciona una [plantilla](https://www.nginx.com/resources/wiki/start/topics/recipes/drupal/) de configuración para el VirtualHost en caso de que vayamos a alojar un sitio _Drupal_, que contiene aquellas directivas necesarias para solventar la falta del **.htaccess**, como por ejemplo evitar que se sirvan determinados ficheros sensibles, establecer las URL limpias...

Tras adaptar dicha plantilla al sitio que vamos a crear, el resultado sería (eliminando líneas comentadas para dejar una salida más limpia):

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/drupal;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name portal.iesgn19.es;

        location / {
            try_files $uri /index.php?$query_string;
        }

        location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        location ~* \.(txt|log)$ {
            deny all;
        }

        location ~ \..*/.*\.php$ {
            return 403;
        }

        location ~ ^/sites/.*/private/ {
            return 403;
        }

        location ~ ^/sites/[^/]+/files/.*\.php$ {
            deny all;
        }

        location ~* ^/.well-known/ {
            allow all;
        }

        location ~ (^|/)\. {
            return 403;
        }

        location @rewrite {
            rewrite ^ /index.php;
        }

        location ~ /vendor/.*\.php$ {
            deny all;
            return 404;
        }

        location ~* \.(engine|inc|install|make|module|profile|po|sh|.*sql|theme|twig|tpl(\.php)?|xtmpl|yml)(~|\.sw[op]|\.bak|\.orig|\.save)?$|/(\.(?!well-known).*|Entries.*|Repository|Root|Tag|Template|composer\.(json|lock)|web\.config)$|/#.*#$|\.php(~|\.sw[op]|\.bak|\.orig|\.save)$ {
            deny all;
            return 404;
        }

        location ~ '\.php$|^/update.php' {
            fastcgi_split_path_info ^(.+?\.php)(|/.*)$;

            try_files $fastcgi_script_name =404;

            include fastcgi_params;

            fastcgi_param HTTP_PROXY "";
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_param QUERY_STRING $query_string;
            fastcgi_intercept_errors on;

            fastcgi_pass unix:/run/php/php7.3-fpm.sock;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            try_files $uri @rewrite;
            expires max;
            log_not_found off;
        }

        location ~ ^/sites/.*/files/styles/ {
            try_files $uri @rewrite;
        }

        location ~ ^(/[a-z\-]+)?/system/files/ {
            try_files $uri /index.php?$query_string;
        }

        if ($request_uri ~* "^(.*/)index\.php/(.*)") {
            return 307 $1$2;
        }
}
{% endhighlight %}

Tras ello, guardaremos los cambios en el mismo.

Una vez realizadas todas las configuraciones oportunas, es hora de habilitar el VirtualHost. Para ello, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/nginx/sites-available/** dentro de **/etc/nginx/sites-enabled/** de forma manual. Para ello, ejecutamos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ln -s /etc/nginx/sites-available/drupal /etc/nginx/sites-enabled/
{% endhighlight %}

Al parecer, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

Una vez que la configuración del servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/nginx/sites-enabled/** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 34 Nov  3 15:11 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 33 Nov 11 10:21 drupal -> /etc/nginx/sites-available/drupal
lrwxrwxrwx 1 root root 34 Nov  3 16:14 iesgn19 -> /etc/nginx/sites-available/iesgn19
{% endhighlight %}

Efectivamente, los tres sitios se encuentran actualmente activos.

Antes de tratar de acceder al sitio, tendremos que llevar a cabo las correspondientes modificaciones en la zona DNS para que se pueda llevar a cabo la resolución. Para ello tendremos que tener la máquina virtual (VPS) nombrada con un **registro A**, que traduce nombres a direcciones IPv4 (en mi caso, ya la tenía previamente nombrada como **vps**), de la siguiente forma:

![dns1](https://i.ibb.co/kMgcZG8/Captura-de-pantalla-de-2020-11-08-12-13-53.png "Registro A")

Tras ello, crearemos un **registro CNAME** para nombrar el servicio que apunte al **registro A** anteriormente creado, en este caso, con nombre **portal**, de la siguiente forma:

![dns2](https://i.ibb.co/jf7Tgns/1.jpg "Registro CNAME")

Como se puede apreciar, las máquinas las nombraremos con **registros A** y tras ello, crearemos **registros CNAME** que apunten a dichas máquinas para nombrar a los servicios, de manera que en caso de que se lleve a cabo un cambio de dirección IP, tendremos que modificar un único registro A para la máquina, y no para todos los registros de los servicios que se encuentren alojados en la misma, ya que estarán apuntando a dicho registro A, por lo que el cambio se efectúa en todas ellas.

Es posible que la propagación del cambio tarde hasta 24 horas, pues necesario que expire la caché de los servidores DNS que conocían dicho nombre de dominio, para que así vuelvan a realizar la petición y reciban la nueva dirección IP.

Tras esperar un rato, accedí a **portal.iesgn19.es** desde el navegador y el resultado fue el siguiente:

![portal1](https://i.ibb.co/8P8KN1D/2.jpg "Drupal")

Como era de esperar, el nuevo VirtualHost ha funcionado correctamente, a pesar de habernos mostrado un error 403 (Forbidden), ya que actualmente no tiene disponible ningún fichero por defecto a servir (directiva **index**), ni tampoco tiene permiso para mostrar la lista de ficheros disponibles (directiva **autoindex**).

### Nombra y configura el servicio de base de datos en producción.

En este caso, y tal y como configuramos en el anterior artículo, la base de datos se va a encontrar en la misma máquina que el resto de servicios. Al tratarse de un servicio interno (que no pretendemos que sea accesible desde el exterior), no tendría sentido nombrarlo en la zona DNS, así que lo nombraremos utilizando resolución estática.

Realmente, podríamos hacer la tarea perfectamente sin nombrar dicho servicio de forma estática y en su lugar, utilizar la dirección IP, pero en un futuro, si el servidor de bases de datos cambiase de máquina o de dirección IP, bastaría con modificar la línea correspondiente a dicha resolución estática, al estar centralizado, en lugar de tener que ir realizando dicha modificación en todas aquellas aplicaciones que necesiten acceder a la base de datos en cuestión. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** para el nuevo sitio que hemos creado, siendo **[IP]** la dirección IP de la máquina servidora (en este caso, la dirección IP local, es decir, **127.0.0.1**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente (en este caso he usado **bd.iesgn19.es**), de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.0.1       bd.iesgn19.es

127.0.1.1       debian.example.com
127.0.1.1       packer-output-dd0d5e86-1b54-48ab-bc93-cb3ebe9f1b16      packer-output-dd0d5e86-1b54-48ab-bc93-cb3ebe9f1b16
127.0.1.1       vps.iesgn19.es  vps
127.0.1.1       vps-6ab4f9ac.vps.ovh.net.novalocal      vps-6ab4f9ac

::1     localhost       ip6-localhost   ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, probaremos a hacer un `ping` a dicho nombre, para así verificar que resuelve correctamente y alcanza a nuestra máquina de manera local, pues al fin y al cabo, sería lo mismo que tratar de alcanzar a **localhost**:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ping bd.iesgn19.es
PING bd.iesgn19.es (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.026 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.041 ms
64 bytes from localhost (127.0.0.1): icmp_seq=3 ttl=64 time=0.060 ms
^C
--- bd.iesgn19.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 0.026/0.042/0.060/0.014 ms
{% endhighlight %}

Efectivamente, la resolución se ha llevado a cabo correctamente, así que el siguiente paso será configurar lo necesario en la base de datos para alojar la aplicación _Drupal_.

Lo primero que debemos hacer es acceder al servidor **mysql**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 64
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Donde:

* **-u**: Indica el usuario con el que nos vamos a conectar, en este caso, **root**.
* **-p**: Indicamos la contraseña, en este caso, no introducimos ninguna, ya que vamos a acceder por Socket UNIX, tal y como hemos configurado en el anterior artículo.

Una vez dentro del servidor de bases de datos, tendremos que crear una nueva base de datos, valga la redundancia. En este caso, para identificarla fácilmente, de nombre **bd_drupal**. Para ello, ejecutamos el comando:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE bd_drupal;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

Para verificar que la nueva base de datos ha sido correctamente creada, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| bd_drupal          |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.001 sec)
{% endhighlight %}

La base de datos que usará _Drupal_ se encuentra ya creada, pero de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma. En este caso, vamos a crear un usuario de nombre "**user_drupal**" que tenga permitido el acceso desde **localhost** (es decir, desde la máquina local) y cuya contraseña sea "**pass_drupal**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso del comando:

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'user_drupal'@'localhost' IDENTIFIED BY 'pass_drupal';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

El usuario ya ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_drupal.* TO 'user_drupal'@'localhost';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

**Nota**: No es necesario hacer uso del comando `FLUSH PRIVILEGES;`, a diferencia de varios artículos que he estado leyendo en Internet, que usan dicho comando muy a menudo sin necesidad alguna. Dicho comando, lo que hace es recargar la caché de las tablas GRANT que se encuentran en memoria, recarga que se hace de forma automática al hacer uso de una sentencia **GRANT**, de manera que no es necesario hacerlo manualmente. Para aclararlo, dicho comando únicamente debe utilizarse tras modificar las tablas GRANT de manera indirecta, es decir, tras usar sentencias INSERT, UPDATE o DELETE.

### Realiza la migración de la aplicación.

Antes de comenzar a migrar _Drupal_, quiero dejar claro la estructura actualmente existente en mi entorno de desarrollo, compuesto por dos máquinas:

* **cms**: La máquina principal y expuesta, que tiene alojado un servidor web _apache2_ y un servidor de aplicaciones PHP, que se encuentra unificado con el servidor web, al tratarse de un módulo de _apache2_ que proporciona dicha funcionalidad.
* **backup**: La máquina no expuesta, que tiene alojado un servidor de bases de datos _MySQL/MariaDB_, a la que accede la máquina principal mediante peticiones a la misma, concretamente a una base de datos de nombre **drupalbackup**.

Lo primero que tendremos que hacer para llevar a cabo la migración es una copia de seguridad de la base de datos, que en el caso de _mysql_, se nos ofrece una funcionalidad que lleva a cabo dicha función, **mysqldump**. Gracias a la misma, se nos generará un fichero que contendrá todas las instrucciones necesarias para dejar la base de datos en cuestión tal y como está actualmente, fichero que posteriormente importaremos en la otra máquina, así que empezaremos trabajando con la máquina **backup**.

En este caso, voy a generar dentro del directorio raíz un fichero de nombre **backup-file.sql** que contenga la copia de seguridad de la base de datos **drupalbackup** (aunque en realidad podríamos hacerlo en cualquier otro directorio, intentando evitar siempre el DocumentRoot de la página web, ya que en ese caso, se podría solicitar ese recurso y sería servido por _nginx_, algo totalmente inseguro). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@backup:~# mysqldump drupalbackup > /backup-file.sql
{% endhighlight %}

Para verificar que la copia de seguridad se ha generado correctamente, vamos a listar el contenido de dicho directorio (raíz), filtrando por el nombre **backup**:

{% highlight shell %}
root@backup:~# ls -l / | egrep 'backup'
-rw-r--r--  1 root root 7960671 Nov 11 11:05 backup-file.sql
{% endhighlight %}

Efectivamente, la copia de seguridad se ha generado correctamente en dicho directorio, así que procederemos a transferirla a la VPS, haciendo uso de `scp`:

{% highlight shell %}
root@backup:~# scp /backup-file.sql debian@vps.iesgn19.es:
debian@vps.iesgn19.es's password:
backup-file.sql                          100% 7774KB  41.7KB/s   03:06
{% endhighlight %}

El fichero que contiene la copia de seguridad de la base de datos ya ha sido correctamente transferido al directorio personal de **debian** en la VPS, algo que sería totalmente inseguro en un entorno que no estuviese controlado, pero que en esta ocasión, no hay problema alguno. Ya hemos terminado el trabajo en la máquina **backup**, dando paso por lo tanto, a la máquina **cms**.

Dentro de ésta se encuentra la mayor parte de la migración, pues contiene el directorio con todos los ficheros y subdirectorios necesarios para el correcto funcionamiento de la aplicación, concretamente dentro de **/srv/drupal**, así que listaremos el contenido del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# ls -l
total 244
-rw-r--r--  1 www-data www-data    312 Oct  7 19:48 autoload.php
-rw-r--r--  1 www-data www-data   3044 Oct  7 19:48 composer.json
-rw-r--r--  1 www-data www-data 157644 Oct  7 19:48 composer.lock
drwxr-xr-x 12 www-data www-data   4096 Oct  7 19:48 core
-rw-r--r--  1 www-data www-data   1507 Oct  7 19:48 example.gitignore
-rw-r--r--  1 www-data www-data    549 Oct  7 19:48 index.php
-rw-r--r--  1 www-data www-data     95 Oct  7 19:48 INSTALL.txt
-rw-r--r--  1 www-data www-data  18092 Nov 16  2016 LICENSE.txt
drwxr-xr-x  3 www-data www-data   4096 Nov  5 17:35 modules
drwxr-xr-x  2 www-data www-data   4096 Oct  7 19:48 profiles
-rw-r--r--  1 www-data www-data   5971 Oct  7 19:48 README.txt
-rw-r--r--  1 www-data www-data   1594 Oct  7 19:48 robots.txt
drwxr-xr-x  3 www-data www-data   4096 Oct  7 19:48 sites
drwxr-xr-x  3 www-data www-data   4096 Nov  5 17:35 themes
-rw-r--r--  1 www-data www-data    848 Oct  7 19:48 update.php
drwxr-xr-x 19 www-data www-data   4096 Oct  7 19:48 vendor
-rw-r--r--  1 www-data www-data   4566 Oct  7 19:48 web.config
{% endhighlight %}

Podríamos copiar todos los ficheros y directorios de manera recursiva a la VPS, pero sería un trabajo poco organizado y lento, de manera que procederemos a comprimirlos, ya que los recursos del sistema se utilizan mucho mejor a la hora de transferir un único archivo, ya **scp** trabaja con **ssh**, un protocolo que cifra las transferencias, siendo mucho menos costoso computacionalmente cifrar un único fichero que cifrar miles. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/drupal# tar -zcf /drupal.tar.gz .
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para comprimir el fichero.
* **-c**: Indica a tar que cree un fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.

Para verificar que el fichero comprimido se ha generado correctamente, vamos a listar el contenido del directorio raíz, filtrando por el nombre **drupal**:

{% highlight shell %}
root@cms:/srv/drupal# ls -l / | egrep 'drupal'
-rw-r--r--  1 root root 20507617 Nov 11 11:42 drupal.tar.gz
{% endhighlight %}

Efectivamente, el fichero comprimido se ha generado correctamente en dicho directorio, así que procederemos a transferirlo a la VPS, haciendo uso de `scp`:

{% highlight shell %}
root@cms:/srv/drupal# scp /drupal.tar.gz debian@vps.iesgn19.es:/srv/drupal
debian@vps.iesgn19.es's password:
drupal.tar.gz                          100%   20MB  55.3KB/s   06:02
{% endhighlight %}

El fichero comprimido que contiene todos los ficheros y directorios del sitio _Drupal_ ya ha sido correctamente transferido al DocumentRoot del VirtualHost en el que se alojará dicha aplicación en la VPS, concretamente a **/srv/drupal**, transferencia que ha sido posible llevar a cabo gracias a que previamente hemos modificado el usuario y grupo propietario de dicho directorio a **debian**, consiguiendo así que tenga los privilegios necesarios para escribir, pues es el usuario a través del cuál nos estamos conectando para realizar la transferencia. Ya hemos terminado el trabajo en ambas máquinas, así que volveremos a la VPS para finalizar la migración.

Una vez dentro de la VPS, procederemos a restaurar la copia de seguridad de la base de datos, pero previamente, para verificar que el fichero con la copia de seguridad ha sido correctamente transferido, listaremos el contenido del directorio personal de **debian** haciendo uso de `ls -l` y estableciendo un filtro por nombre:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls -l /home/debian/ | egrep 'backup'
-rw-r--r-- 1 debian debian 7960671 Nov 11 11:10 backup-file.sql
{% endhighlight %}

Efectivamente, el fichero con la copia de seguridad ha sido correctamente transferido, por lo que ya podremos importarlo a la base de datos creada con anterioridad. Para ello, únicamente inyectaremos el contenido del fichero a _mysql_, indicando la base de datos que deseamos recuperar. Además, por seguridad, eliminaremos posteriormente dicho fichero con la copia de seguridad, quedando de la siguiente forma:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# mysql bd_drupal < /home/debian/backup-file.sql && rm /home/debian/backup-file.sql
{% endhighlight %}

Para verificar que el fichero ha sido correctamente eliminado, volveremos a listar el contenido del directorio personal de **debian** estableciendo a su vez un filtro por nombre, ejecutando para ello el comando visto con anteriordad:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls -l /home/debian/ | egrep 'backup'
{% endhighlight %}

Efectivamente, en ésta ocasión, el filtro no ha devuelto ningún resultado, ya que el fichero ha sido correctamente eliminado.

De nuevo, accederemos al servicio _mysql_ para verificar que la base de datos ha sido correctamente restaurada:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 66
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Tras ello, nos conectaremos a la base de datos en cuestión, con la instrucción `use`:

{% highlight shell %}
MariaDB [(none)]> use bd_drupal;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Como podemos apreciar en el último mensaje, nos hemos conectado a dicha base de datos, así que listaremos las tablas de la misma con la siguiente instrucción:

{% highlight shell %}
MariaDB [bd_drupal]> show tables;
+----------------------------------+
| Tables_in_bd_drupal              |
+----------------------------------+
| batch                            |
| block_content                    |
| ...                              |
| users_field_data                 |
| watchdog                         |
+----------------------------------+
77 rows in set (0.001 sec)
{% endhighlight %}

Efectivamente, la base de datos ha sido correctamente restaurada, pues contiene las 77 tablas previamente existentes en el entorno de desarrollo.

Ya hemos recorrido la mitad del camino, pues la base de datos está correctamente restaurada, pero todavía nos falta restaurar la aplicación como tal. Si recordamos, el DocumentRoot que iba a usar se encuentra en **/srv/drupal**, siendo éste el mismo directorio en el que se encuentra el fichero comprimido, así que nos moveremos dentro del mismo, haciendo uso de `cd`:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# cd /srv/drupal/
{% endhighlight %}

Antes de seguir, vamos a volver a modificar el usuario y grupo propietario del directorio actual a **www-data**, pues es el usuario por defecto a través del cuál se sirven las en _nginx_, para que posteriormente no se nos olvide, pudiendo ocasionar un gran agujero de seguridad. Para ello, volveremos a hacer uso de `chown`:

{% highlight shell %}
root@vps:/srv/drupal# chown www-data:www-data /srv/drupal/
{% endhighlight %}

Para verificar que el usuario y grupo propietario han sido modificados, volveremos a listar el contenido de **/srv/**, haciendo uso del comando:

{% highlight shell %}
root@vps:/srv/drupal# ls -l /srv/
total 12
drwxr-xr-x 8 www-data www-data 4096 Nov 11 11:50 drupal
drwxr-xr-x 3 www-data www-data 4096 Nov  3 16:24 iesgn19
{% endhighlight %}

Como era de esperar, la modificación se ha llevado a cabo tal y como hemos especificado. Tras ello, listaremos el contenido del directorio actual para así verificar que el fichero comprimido ha sido correctamente transferido, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/drupal# ls -l
total 20028
-rw-r--r-- 1 debian debian 20507617 Nov 11 11:49 drupal.tar.gz
{% endhighlight %}

Efectivamente, el fichero comprimido ha sido correctamente transferido, por lo que ya podremos descomprimirlo para así obtener exactamente la misma estructura anteriormente existente en el entorno de desarrollo. Además, eliminaremos tras ello el fichero comprimido para así liberar espacio, ya que no nos va a ser necesario. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/drupal# tar -zxf drupal.tar.gz && rm drupal.tar.gz
{% endhighlight %}

Donde:

* **-z**: Utiliza gzip para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.gz.

Para verificar que se ha descomprimido correctamente y que además se ha eliminado de dicho directorio, haremos uso de `ls -la`:

{% highlight shell %}
root@vps:/srv/drupal# ls -la
total 284
drwxr-xr-x  8 www-data www-data   4096 Nov 11 11:50 .
drwxr-xr-x  5 root     root       4096 Nov 11 10:19 ..
-rw-r--r--  1 www-data www-data    312 Oct  7 19:48 autoload.php
-rw-r--r--  1 www-data www-data   3044 Oct  7 19:48 composer.json
-rw-r--r--  1 www-data www-data 157644 Oct  7 19:48 composer.lock
drwxr-xr-x 12 www-data www-data   4096 Oct  7 19:48 core
-rw-r--r--  1 www-data www-data   1025 Oct  7 19:48 .csslintrc
-rw-r--r--  1 www-data www-data    357 Oct  7 19:48 .editorconfig
-rw-r--r--  1 www-data www-data    151 Oct  7 19:48 .eslintignore
-rw-r--r--  1 www-data www-data     41 Oct  7 19:48 .eslintrc.json
-rw-r--r--  1 www-data www-data   1507 Oct  7 19:48 example.gitignore
-rw-r--r--  1 www-data www-data   3858 Oct  7 19:48 .gitattributes
-rw-r--r--  1 www-data www-data   7567 Oct  7 19:48 .htaccess
-rw-r--r--  1 www-data www-data   2314 Oct  7 19:48 .ht.router.php
-rw-r--r--  1 www-data www-data    549 Oct  7 19:48 index.php
-rw-r--r--  1 www-data www-data     95 Oct  7 19:48 INSTALL.txt
-rw-r--r--  1 www-data www-data  18092 Nov 16  2016 LICENSE.txt
drwxr-xr-x  3 www-data www-data   4096 Nov  5 17:35 modules
drwxr-xr-x  2 www-data www-data   4096 Oct  7 19:48 profiles
-rw-r--r--  1 www-data www-data   5971 Oct  7 19:48 README.txt
-rw-r--r--  1 www-data www-data   1594 Oct  7 19:48 robots.txt
drwxr-xr-x  3 www-data www-data   4096 Oct  7 19:48 sites
drwxr-xr-x  3 www-data www-data   4096 Nov  5 17:35 themes
-rw-r--r--  1 www-data www-data    848 Oct  7 19:48 update.php
drwxr-xr-x 19 www-data www-data   4096 Oct  7 19:48 vendor
-rw-r--r--  1 www-data www-data   4566 Oct  7 19:48 web.config
{% endhighlight %}

Como se puede apreciar, el fichero se ha descomprimido correctamente y tras ello, ha sido eliminado. Si nos fijamos con detenimiento, existe un fichero **.htaccess** que no vamos a utilizar, ya que estamos utilizando un servidor web _nginx_ en lugar de _apache2_, de manera que podremos proceder a eliminarlo, pues no nos será necesario, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/drupal# rm .htaccess
{% endhighlight %}

Actualmente, la web no funcionaría todavía ya que la máquina a la que está configurada para hacer las peticiones SQL no se encuentra operativa, por lo que tendremos que llevar a cabo las correspondientes modificaciones en el fichero oportuno para indicarle la nueva base de datos a la que ha de realizar las peticiones. Éste fichero se encuentra en **/srv/drupal/sites/default/**, con nombre **settings.php**, que modificaremos con `nano`:

{% highlight shell %}
root@vps:/srv/drupal# nano sites/default/settings.php
{% endhighlight %}

Dentro del mismo, nos desplazaremos al final del documento y encontraremos un bloque como el siguiente:

{% highlight shell %}
$databases['default']['default'] = array (
  'database' => 'drupalbackup',
  'username' => 'backupuser',
  'password' => 'usuario',
  'prefix' => '',
  'host' => '10.0.0.5',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
);
{% endhighlight %}

Como se puede apreciar, ahí se encuentran las credenciales de acceso y la base de datos a la que acceder (correspondiente a la máquina **backup**). En este caso, el nombre de la base de datos ha cambiado de **drupalbackup** a **bd_drupal**, el usuario ha cambiado de **backupuser** a **user_drupal** y la contraseña ha cambiado de **usuario** a **pass_drupal**. Nos queda una última modificación por hacer, y es que la base de datos ya no se encuentra en **10.0.0.5**, sino en **bd.iesgn19.es**. El puerto en el que escucha las peticiones no ha variado, pues sigue siendo el **3306**. La configuración final quedaría así:

{% highlight shell %}
$databases['default']['default'] = array (
  'database' => 'bd_drupal',
  'username' => 'user_drupal',
  'password' => 'pass_drupal',
  'prefix' => '',
  'host' => 'bd.iesgn19.es',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
);
{% endhighlight %}

Tras ello, guardaremos los cambios, y antes de tratar de acceder al sitio _Drupal_, tendremos que instalar aquellas librerías y extensiones que se requerieron en la instalación de la aplicación en el entorno de desarollo, con la finalidad de intentar que ambos entornos (desarrollo y producción) sean en todo momento lo más similares posibles. En este caso, instalaremos dichas librerías y extensiones ejecutando el comando:

{% highlight shell %}
root@vps:/srv/drupal# apt install php-gd php-xml php-mbstring
{% endhighlight %}

Listo, ya hemos finalizado la migración, así que ha llegado _la hora de la verdad_. Para verificar si la migración ha sido realmente exitosa, accederemos desde el navegador al ServerName que hemos configurado para éste VirtualHost, es decir, a **portal.iesgn19.es**:

![portal2](https://i.ibb.co/6J7yd5B/Captura-de-pantalla-de-2020-11-19-11-19-26.png "Drupal")

Efectivamente, el sitio web se ha mostrado correctamente y hemos podido hacer uso de las funcionalidades de la aplicación, como por ejemplo, [autenticarnos](http://portal.iesgn19.es/user/login), por lo que podemos asegurar que la migración se ha llevado a cabo de forma exitosa.

## Tarea 2: Nextcloud

### Instala la aplicación web Nextcloud en tu entorno de desarrollo.

Como el propio enunciado indica, la instalación la vamos a llevar a cabo sobre nuestro entorno de desarrollo (máquinas **cms** y **backup**), de manera que cambiaremos el _chip_ para utilizar ahora _apache2_, hasta que realicemos la migración a la VPS, en la que volveremos a utilizar _nginx_.

En este caso, el nuevo VirtualHost lo voy a alojar dentro de **/srv/**, en un directorio de nombre **nextcloud/**, además de crear otro directorio fuera del DocumentRoot para que así no sea accesible, de nombre **nextclouddata/**, que contendrá los ficheros que se suban a dicha aplicación web, pues tiene la finalidad de una nube. Para llevar a cabo la creación de dichos directorios ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/drupal# mkdir /srv/{nextcloud,nextclouddata}
{% endhighlight %}

Una vez generados, tendremos que crear el fichero de configuración para dicho VirtualHost, así que para no complicarme, copiaré la plantilla del VirtualHost por defecto de _apache2_ de nombre **000-default**:

{% highlight shell %}
root@cms:/srv/drupal# cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/nextcloud.conf
{% endhighlight %}

Tras ello, lo modificaremos haciendo uso de `nano` y estableceremos el DocumentRoot **/srv/nextcloud** y el ServerName **www.alvaro-nextcloud.com**, por ejemplo:

{% highlight shell %}
root@cms:/srv/drupal# nano /etc/apache2/sites-available/nextcloud.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.alvaro-nextcloud.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/nextcloud

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
{% endhighlight %}

Una vez realizada la correspondiente configuración en el fichero, tendremos que habilitar el nuevo VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# a2ensite nextcloud
Enabling site nextcloud.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Tal y como se nos ha solicitado, vamos a volver a cargar la configuración del servicio _apache2_, haciendo uso del comando:

{% highlight shell %}
root@cms:/srv/drupal# systemctl reload apache2
{% endhighlight %}

El siguiente paso consiste en añadir la correspondiente línea al fichero **/etc/hosts** de la máquina anfitriona para que lleve a cabo la resolución estatica de nombres y así poder acceder al nuevo VirtualHost, alojado en la máquina **cms**. Para ello, ejecutamos el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Tras añadir la correspondiente línea, el resultado final es el siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
172.22.200.103  www.alvaro-nextcloud.com

::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

La máquina anfitriona ya es capaz de realizar la resolución del nombre **www.alvaro-nextcloud.com**, pero todavía no tenemos ninguno de los ficheros necesarios para la instalación, así que nos moveremos al DocumentRoot, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/drupal# cd /srv/nextcloud/
{% endhighlight %}

Una vez dentro del mismo, haremos uso de `wget` para descargar el paquete comprimido de [Nextcloud](https://nextcloud.com/), que podremos encontrar [aquí](https://download.nextcloud.com/server/releases/nextcloud-20.0.1.tar.bz2):

{% highlight shell %}
root@cms:/srv/nextcloud# wget https://download.nextcloud.com/server/releases/nextcloud-20.0.1.tar.bz2
--2020-11-16 08:35:33--  https://download.nextcloud.com/server/releases/nextcloud-20.0.1.tar.bz2
Resolving download.nextcloud.com (download.nextcloud.com)... 95.217.64.181, 2a01:4f9:2a:3119::181
Connecting to download.nextcloud.com (download.nextcloud.com)|95.217.64.181|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 120287967 (115M) [application/x-bzip2]
Saving to: ‘nextcloud-20.0.1.tar.bz2’

nextcloud-20.0.1.tar.bz2                100%[==========================>]  114.71M   175KB/s    in 13m 0s  

2020-11-16 08:48:34 (151 KB/s) - ‘nextcloud-20.0.1.tar.bz2’ saved [120287967/120287967]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`:

{% highlight shell %}
root@cms:/srv/nextcloud# ls -l
total 117472
-rw-r--r-- 1 root root 120287967 oct 24 10:41 nextcloud-20.0.1.tar.bz2
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**nextcloud-20.0.1.tar.bz2**" con un peso total de **114.71 MB** (120287967 _bytes_).

Al estar comprimido el fichero, no podemos llevar a cabo la instalación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello dicho fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# tar -xf nextcloud-20.0.1.tar.bz2 --strip 1 && rm nextcloud-20.0.1.tar.bz2
{% endhighlight %}

Donde:

* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.bz2.
* **--strip 1**: Saltamos el primer directorio, ya que dentro del comprimido hay un directorio padre que no necesitamos.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
root@cms:/srv/nextcloud# ls -l
total 148
drwxr-xr-x 41 nobody nogroup  4096 Oct 24 08:39 3rdparty
drwxr-xr-x 47 nobody nogroup  4096 Oct 24 08:38 apps
-rw-r--r--  1 nobody nogroup 17234 Oct 24 08:37 AUTHORS
drwxr-xr-x  2 nobody nogroup  4096 Oct 24 08:40 config
-rw-r--r--  1 nobody nogroup  3893 Oct 24 08:37 console.php
...
-rw-r--r--  1 nobody nogroup    26 Oct 24 08:37 robots.txt
-rw-r--r--  1 nobody nogroup  2379 Oct 24 08:37 status.php
drwxr-xr-x  3 nobody nogroup  4096 Oct 24 08:37 themes
drwxr-xr-x  2 nobody nogroup  4096 Oct 24 08:38 updater
-rw-r--r--  1 nobody nogroup   362 Oct 24 08:39 version.php
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos (en lugar de descomprimir un directorio de nombre **nextcloud** del que posteriormente tendríamos que mover los ficheros contenidos al directorio actual).

Ya tenemos todos los ficheros necesarios para llevar a cabo la instalación de _Nextcloud_, pero hay un pequeño detalle que todavía no hemos contemplado, pues el usuario y el grupo propietario correspondiente a dichos ficheros y directorios es **nobody** y **nogroup**, respectivamente, por lo que procedemos a cambiar dicho propietario y grupo de forma recursiva a **www-data**, haciendo para ello uso del comando `chown -R`, pues de otro modo no podría escribir en dichos ficheros durante la instalación:

{% highlight shell %}
root@cms:/srv/nextcloud# chown -R www-data:www-data /srv/nextcloud/
{% endhighlight %}

También es importante cambiar dichos permisos para el directorio **nextclouddata/**, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# chown www-data:www-data /srv/nextclouddata/
{% endhighlight %}

Para verificar que el cambio se ha llevado a cabo correctamente, volveremos a listar el contenido de **/srv**, haciendo uso de `ls -l`:

{% highlight shell %}
root@cms:/srv/nextcloud# ls -l /srv/
total 8
drwxr-xr-x  8 www-data www-data 4096 Nov  5 17:06 drupal
drwxr-xr-x 13 www-data www-data 4096 Nov 16 08:52 nextcloud
drwxr-xr-x  2 www-data www-data 4096 Nov 16 08:52 nextclouddata
{% endhighlight %}

Como era de esperar, el usuario y el grupo propietario de los directorios **nextcloud/** y **nextclouddata/** se han visto modificados, al igual que todos sus ficheros y directorios contenidos, al haberlo hecho de forma recursiva.

Nos falta un único paso para proceder con la instalación, y es que todavía no hemos creado la base de datos que utilizará, por lo que nos moveremos a la máquina **backup** para crearla, con nombre **bd_nextcloud**, por ejemplo:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE bd_nextcloud;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

La nueva base de datos se encuentra ya creada. Ésta vez no vamos a usar la base de datos de manera local, sino que queremos permitir el acceso a otra máquina (en concreto desde **cms**). En este caso, crearemos un usuario de nombre **user_nextcloud**, que tenga permitido el acceso desde **10.0.0.8**, es decir, desde la máquina que queremos que se pueda conectar (si quisiéramos permitir el acceso desde cualquier dirección IP, se podría poder **%**), y cuya contraseña será **pass_nextcloud**.

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'user_nextcloud'@'10.0.0.8' IDENTIFIED BY 'pass_nextcloud';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Tras ello, tendremos que otorgarle los privilegios necesarios sobre la base de datos que acabamos de crear, ejecutando para ello el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_nextcloud.* TO 'user_nextcloud'@'10.0.0.8';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Ya está todo listo para llevar a cabo la instalación, así que accederemos al navegador de la máquina anfitriona y escribiremos el nombre de dominio previamente configurado, **www.alvaro-nextcloud.com**:

![nextcloud1](https://i.ibb.co/bgKFRd7/Captura-de-pantalla-de-2020-11-16-09-53-30.png "Problemas de dependencias")

Como se puede apreciar, nos ha devuelto un error de falta de extensiones. Su solución es sencilla, pues únicamente tendremos que instalar los paquetes correspondientes en la máquina **cms**, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# apt install php-zip php-curl
{% endhighlight %}

Tras ello, tendremos que volver a cargar la configuración del servicio _apache2_ para que los cambios realizados surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# systemctl reload apache2
{% endhighlight %}

Una vez realizada la instalación de las extensiones y el correspondiente reinicio del servicio, volveremos al instalador y refrescaremos la página y podremos verificar que el problema se ha solucionado.

El siguiente y último paso nos pedirá indicar una cuenta de administrador, la ruta del directorio de datos (**/srv/nextclouddata/**), y la base de datos a la que accederá (alojada en la máquina **backup**), así como sus credenciales, que serán las siguientes:

* **Usuario**: user_nextcloud
* **Contraseña**: pass_nextcloud
* **Nombre**: bd_nextcloud
* **Dirección**: 10.0.0.5:3306

Una vez introducida dicha información, pulsaremos en **Completar la instalación** y tras unos segundos, ya tendremos la aplicación totalmente operativa:

![nextcloud2](https://i.ibb.co/Fxh8ZPT/Captura-de-pantalla-de-2020-11-16-10-09-56.png "Nextcloud")

Como se puede apreciar, la aplicación se ha instalado correctamente, junto a algunos ficheros que trae por defecto.

### Realiza la migración al servidor en producción.

En ésta ocasión, en lugar de crear un nuevo registro en la zona DNS de la VPS, con la finalidad de alojarlo en un VirtualHost distinto, vamos a utilizar uno de los VirtualHost previamente configurados, concretamente **www.iesgn19.es**, que contendrá un subdirectorio **cloud/** donde estará ubicada la aplicación _Nextcloud_.

Al igual que en la anterior migración, la propia página de _Nextcloud_ nos proporciona una [plantilla](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html) de configuración para el VirtualHost en caso de que vayamos a alojar un sitio _Nextcloud_ en un **subdirectorio**, que contiene aquellas directivas necesarias para solventar la falta del **.htaccess**, como por ejemplo evitar que se sirvan determinados ficheros sensibles, establecer las URL limpias...

Para modificar el fichero fichero de configuración de dicho VirtualHost (concretamente **iesgn19**), ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/drupal# nano /etc/nginx/sites-available/iesgn19
{% endhighlight %}

Tras adaptar dicha plantilla al sitio que vamos a crear, el resultado sería (eliminando líneas comentadas para dejar una salida más limpia):

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/iesgn19;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name www.iesgn19.es;

        rewrite ^/$ /principal redirect;

        location / {
                try_files $uri $uri/ =404;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        location /.well-known {
            rewrite ^/\.well-known/host-meta\.json  /cloud/public.php?service=host-meta-json    last;
            rewrite ^/\.well-known/host-meta        /cloud/public.php?service=host-meta         last;
            rewrite ^/\.well-known/webfinger        /cloud/public.php?service=webfinger         last;
            rewrite ^/\.well-known/nodeinfo         /cloud/public.php?service=nodeinfo          last;

            location = /.well-known/carddav   { return 301 /cloud/remote.php/dav/; }
            location = /.well-known/caldav    { return 301 /cloud/remote.php/dav/; }

            try_files $uri $uri/ =404;
        }

        location ^~ /cloud {
            client_max_body_size 512M;
            fastcgi_buffers 64 4K;

            gzip on;
            gzip_vary on;
            gzip_comp_level 4;
            gzip_min_length 256;
            gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
            gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

            add_header Referrer-Policy                      "no-referrer"   always;
            add_header X-Content-Type-Options               "nosniff"       always;
            add_header X-Download-Options                   "noopen"        always;
            add_header X-Frame-Options                      "SAMEORIGIN"    always;
            add_header X-Permitted-Cross-Domain-Policies    "none"          always;
            add_header X-Robots-Tag                         "none"          always;
            add_header X-XSS-Protection                     "1; mode=block" always;

            fastcgi_hide_header X-Powered-By;

            index index.php index.html /cloud/index.php$request_uri;

            expires 1m;

            location = /cloud {
                if ( $http_user_agent ~ ^DavClnt ) {
                    return 302 /cloud/remote.php/webdav/$is_args$args;
                }
            }

            location ~ ^/cloud/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)    { return 404; }
            location ~ ^/cloud/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

            location ~ \.php(?:$|/) {
                fastcgi_split_path_info ^(.+?\.php)(/.*)$;
                set $path_info $fastcgi_path_info;

                try_files $fastcgi_script_name =404;

                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $path_info;

                fastcgi_param modHeadersAvailable true;
                fastcgi_param front_controller_active true;
                fastcgi_pass unix:/run/php/php7.3-fpm.sock;

                fastcgi_intercept_errors on;
                fastcgi_request_buffering off;
            }

            location ~ \.(?:css|js|svg|gif)$ {
                try_files $uri /cloud/index.php$request_uri;
                expires 6M;
                access_log off;
            }

            location ~ \.woff2?$ {
                try_files $uri /cloud/index.php$request_uri;
                expires 7d;
                access_log off;
            }

            location /cloud {
                try_files $uri $uri/ /cloud/index.php$request_uri;
            }
        }
}
{% endhighlight %}

Tras ello, guardaremos los cambios en el mismo y procederemos a crear los directorios que utilizará la aplicación, uno que contendrá los ficheros y directorios necesarios para su funcionamiento (dentro de **/srv/iesgn19/cloud/**) y otro que contendrá aquellos ficheros que se suban a la nube, que debe estar fuera del DocumentRoot por seguridad (por ejemplo, en **/srv/nextclouddata**). Para crear dichos directorios ejecutaremos los comandos:

{% highlight shell %}
root@vps:/srv/drupal# mkdir /srv/iesgn19/cloud
root@vps:/srv/drupal# mkdir /srv/nextclouddata
{% endhighlight %}

Tras ello, modificaremos el usuario y el grupo propietario de dichos directorios a **debian**, pues nos será necesario más adelante. Para ello, haremos uso de `chown`:

{% highlight shell %}
root@vps:/srv/drupal# chown debian:debian /srv/iesgn19/cloud/
root@vps:/srv/drupal# chown debian:debian /srv/nextclouddata/
{% endhighlight %}

Para verificar que la creación se ha llevado a cabo correctamente y que usuario y el grupo propietario han sido modificados, listaremos el contenido de **/srv/** y de **/srv/iesgn19/**, haciendo uso del comando `ls -l`:

{% highlight shell %}
root@vps:/srv/drupal# ls -l /srv/
total 16
drwxr-xr-x 8 www-data www-data 4096 Nov 11 11:54 drupal
drwxr-xr-x 4 www-data www-data 4096 Nov 16 18:09 iesgn19
drwxr-xr-x 2 debian   debian   4096 Nov 16 18:09 nextclouddata

root@vps:/srv/drupal# ls -l /srv/iesgn19/
total 8
drwxr-xr-x 2 debian   debian   4096 Nov 16 18:09 cloud
drwxr-xr-x 3 www-data www-data 4096 Nov 16 11:35 principal
{% endhighlight %}

Efectivamente, los directorios han sido generados y su propietario y grupo correctamente modificado. Para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/drupal# systemctl reload nginx
{% endhighlight %}

Una vez cargada la nueva configuración, vamos a proceder a crear la base de datos que la aplicación va a utilizar, de manera que accederemos al servidor **mysql**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/drupal# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 64
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Una vez dentro del servidor de bases de datos, tendremos que crear una nueva base de datos, valga la redundancia. En este caso, para identificarla fácilmente, de nombre **bd_nextcloud**, ejecutando para ello el comando:

{% highlight shell %}
MariaDB [(none)]> CREATE DATABASE bd_nextcloud;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

La base de datos que usará _Nextcloud_ se encuentra ya creada, pero de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma. En este caso, vamos a crear un usuario de nombre "**user_nextcloud**" que tenga permitido el acceso desde **localhost** (es decir, desde la máquina local) y cuya contraseña sea "**pass_nextcloud**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso del comando:

{% highlight shell %}
MariaDB [(none)]> CREATE USER 'user_nextcloud'@'localhost' IDENTIFIED BY 'pass_nextcloud';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

El usuario ya ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos el comando:

{% highlight shell %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_nextcloud.* TO 'user_nextcloud'@'localhost';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight shell %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

Una vez creada la base de datos que va a utilizar la aplicación en la VPS, tendremos que crear una copia de seguridad de la base de datos del entorno de desarrollo, haciendo uso, al igual que en la anterior migración, de **mysqldump**, así que empezaremos trabajando con la máquina **backup**.

En este caso, voy a generar dentro del directorio raíz un fichero de nombre **backup-nextcloud.sql** que contenga la copia de seguridad de la base de datos **bd_nextcloud** (aunque en realidad podríamos hacerlo en cualquier otro directorio, intentando evitar siempre el DocumentRoot de la página web, ya que en ese caso, se podría solicitar ese recurso y sería servido por _nginx_, algo totalmente inseguro). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@backup:~# mysqldump bd_nextcloud > /backup-nextcloud.sql
{% endhighlight %}

Para verificar que la copia de seguridad se ha generado correctamente, vamos a listar el contenido de dicho directorio (raíz), filtrando por el nombre **backup**:

{% highlight shell %}
root@backup:~# ls -l / | egrep 'backup'
-rw-r--r--  1 root root 7960671 Nov 11 11:05 backup-file.sql
-rw-r--r--  1 root root  126035 Nov 16 13:06 backup-nextcloud.sql
{% endhighlight %}

Efectivamente, la copia de seguridad se ha generado correctamente en dicho directorio, así que procederemos a transferirla a la VPS, haciendo uso de `scp`:

{% highlight shell %}
root@backup:~# scp /backup-nextcloud.sql debian@vps.iesgn19.es:
debian@vps.iesgn19.es's password:
backup-nextcloud.sql                          100%  123KB  69.2KB/s   00:01
{% endhighlight %}

El fichero que contiene la copia de seguridad de la base de datos ya ha sido correctamente transferido al directorio personal de **debian** en la VPS, algo que sería totalmente inseguro en un entorno que no estuviese controlado, pero que en esta ocasión, no hay problema alguno. Ya hemos terminado el trabajo en la máquina **backup**, dando paso por lo tanto, a la máquina **cms**.

Dentro de ésta se encuentra la mayor parte de la migración, pues contiene el directorio con todos los ficheros y subdirectorios necesarios para el correcto funcionamiento de la aplicación, concretamente dentro de **/srv/nextcloud**, además de todos los ficheros que se han subido a la nube, concretamente dentro de **/srv/nextclouddata**. Dado que ya nos encontramos ubicados en el primero de ellos, podremos comenzar con la migración de dichos ficheros, creando al igual que en la anterior migración, un comprimido que posteriormente transferiremos. Para crear dicho fichero comprimido ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# tar -zcf /nextcloud.tar.gz .
{% endhighlight %}

Ya hemos terminado el trabajo en **/srv/nextcloud**, así que nos moveremos a **/srv/nextclouddata** para repetir exactamente el mismo procedimiento de compresión. Para ello ejecutaremos el comando:

{% highlight shell %}
root@cms:/srv/nextcloud# cd ../nextclouddata/
{% endhighlight %}

Una vez dentro del mismo, comprimiremos los ficheros existentes, haciendo uso del comando:

{% highlight shell %}
root@cms:/srv/nextclouddata# tar -zcf /nextclouddata.tar.gz .
{% endhighlight %}

Para verificar que los ficheros comprimidos se han generado correctamente, vamos a listar el contenido del directorio raíz, filtrando por el nombre **nextcloud**:

{% highlight shell %}
root@cms:/srv/nextclouddata# ls -l / | egrep 'nextcloud'
-rw-r--r--  1 root root  17039195 Nov 16 18:07 nextclouddata.tar.gz
-rw-r--r--  1 root root 131736076 Nov 16 18:07 nextcloud.tar.gz
{% endhighlight %}

Efectivamente, los ficheros comprimidos se han generado correctamente en dicho directorio, así que procederemos a transferirlos a la VPS, haciendo uso de `scp`:

{% highlight shell %}
root@cms:/srv/nextclouddata# scp /nextcloud.tar.gz debian@vps.iesgn19.es:/srv/iesgn19/cloud
debian@vps.iesgn19.es's password:
nextcloud.tar.gz                          100%  126MB  63.0KB/s   34:00

root@cms:/srv/nextclouddata# scp /nextclouddata.tar.gz debian@vps.iesgn19.es:/srv/nextclouddata
debian@vps.iesgn19.es's password:
nextclouddata.tar.gz                          100%   16MB  63.5KB/s   04:21
{% endhighlight %}

Los ficheros comprimidos que contienen todos los ficheros y directorios del sitio _Nextcloud_ ya han sido correctamente transferidos a los correspondientes directorios que utilizará dicha aplicación, concretamente a **/srv/iesgn19/cloud** y **/srv/nextclouddata**, transferencia que ha sido posible llevar a cabo gracias a que previamente hemos modificado el usuario y grupo propietario de dichos directorios a **debian**, consiguiendo así que tenga los privilegios necesarios para escribir, pues es el usuario a través del cuál nos estamos conectando para realizar la transferencia. Ya hemos terminado el trabajo en ambas máquinas, así que volveremos a la VPS para finalizar la migración.

Una vez dentro de la VPS, procederemos a restaurar la copia de seguridad de la base de datos, pero previamente, para verificar que el fichero con la copia de seguridad ha sido correctamente transferido, listaremos el contenido del directorio personal de **debian** haciendo uso de `ls -l` y estableciendo un filtro por nombre:

{% highlight shell %}
root@vps:/srv/drupal# ls -l /home/debian/ | egrep 'backup'
-rw-r--r-- 1 debian debian 126035 Nov 16 13:07 backup-nextcloud.sql
{% endhighlight %}

Efectivamente, el fichero con la copia de seguridad ha sido correctamente transferido, por lo que ya podremos importarlo a la base de datos creada con anterioridad. Para ello, únicamente inyectaremos el contenido del fichero a _mysql_, indicando la base de datos que deseamos recuperar. Además, por seguridad, eliminaremos posteriormente dicho fichero con la copia de seguridad, quedando de la siguiente forma:

{% highlight shell %}
root@vps:/srv/drupal# mysql bd_nextcloud < /home/debian/backup-nextcloud.sql && rm /home/debian/backup-nextcloud.sql
{% endhighlight %}

Para verificar que el fichero ha sido correctamente eliminado, volveremos a listar el contenido del directorio personal de **debian** estableciendo a su vez un filtro por nombre, ejecutando para ello el comando visto con anteriordad:

{% highlight shell %}
root@vps:/srv/drupal# ls -l /home/debian/ | egrep 'backup'
{% endhighlight %}

Efectivamente, en ésta ocasión, el filtro no ha devuelto ningún resultado, ya que el fichero ha sido correctamente eliminado.

De nuevo, accederemos al servicio _mysql_ para verificar que la base de datos ha sido correctamente restaurada:

{% highlight shell %}
root@vps:/srv/drupal# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 169
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Tras ello, nos conectaremos a la base de datos en cuestión, con la instrucción `use`:

{% highlight shell %}
MariaDB [(none)]> use bd_nextcloud;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Como podemos apreciar en el último mensaje, nos hemos conectado a dicha base de datos, así que listaremos las tablas de la misma con la siguiente instrucción:

{% highlight shell %}
MariaDB [bd_nextcloud]> show tables;
+-----------------------------+
| Tables_in_bd_nextcloud      |
+-----------------------------+
| oc_accounts                 |
| oc_activity                 |
| ...                         |
| oc_webauthn                 |
| oc_whats_new                |
+-----------------------------+
75 rows in set (0.002 sec)
{% endhighlight %}

Efectivamente, la base de datos ha sido correctamente restaurada, pues contiene las 75 tablas previamente existentes en el entorno de desarrollo.

Ya hemos recorrido la mitad del camino, pues la base de datos está correctamente restaurada, pero todavía nos falta restaurar la aplicación como tal. Si recordamos, el directorio que la aplicación iba a usar se encuentra en **/srv/iesgn19/cloud**, siendo éste el mismo directorio en el que se encuentra uno de los dos ficheros comprimidos, así que nos moveremos dentro del mismo, haciendo uso de `cd`:

{% highlight shell %}
root@vps:/srv/drupal# cd /srv/iesgn19/cloud/
{% endhighlight %}

Antes de seguir, vamos a volver a modificar el usuario y grupo propietario de los directorios **/srv/iesgn19/cloud** y **/srv/nextclouddata** a **www-data**, pues es el usuario por defecto a través del cuál se sirven las en _nginx_, para que posteriormente no se nos olvide, pudiendo ocasionar un gran agujero de seguridad. Para ello, volveremos a hacer uso de `chown`:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud/# chown www-data:www-data /srv/iesgn19/cloud/
root@vps:/srv/iesgn19/cloud/# chown www-data:www-data /srv/nextclouddata/
{% endhighlight %}

Para verificar que el usuario y grupo propietario han sido modificados, volveremos a listar el contenido de **/srv/** y de **/srv/iesgn19/**, haciendo uso del comando `ls -l`:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud/# ls -l /srv/
total 16
drwxr-xr-x 8 www-data www-data 4096 Nov 11 11:54 drupal
drwxr-xr-x 4 www-data www-data 4096 Nov 16 18:09 iesgn19
drwxrwx--- 4 www-data www-data 4096 Nov 16 18:57 nextclouddata

root@vps:/srv/iesgn19/cloud/# ls -l /srv/iesgn19/
total 8
drwxr-xr-x 14 www-data www-data 4096 Nov 16 18:57 cloud
drwxr-xr-x  3 www-data www-data 4096 Nov 16 11:35 principal
{% endhighlight %}

Como era de esperar, la modificación se ha llevado a cabo tal y como hemos especificado. Tras ello, listaremos el contenido del directorio actual para así verificar que el fichero comprimido ha sido correctamente transferido, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud# ls -l
total 128656
-rw-r--r-- 1 debian debian 131736076 Nov 16 18:46 nextcloud.tar.gz
{% endhighlight %}

Efectivamente, el fichero comprimido ha sido correctamente transferido, por lo que ya podremos descomprimirlo para así obtener exactamente la misma estructura anteriormente existente en el entorno de desarrollo. Además, eliminaremos tras ello el fichero comprimido para así liberar espacio, ya que no nos va a ser necesario. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud# tar -zxf nextcloud.tar.gz && rm nextcloud.tar.gz
{% endhighlight %}

Para verificar que se ha descomprimido correctamente y que además se ha eliminado de dicho directorio, haremos uso de `ls -la`:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud# ls -la
total 168
drwxr-xr-x 14 www-data www-data  4096 Nov 16 18:57 .
drwxr-xr-x  4 www-data www-data  4096 Nov 16 18:09 ..
drwxr-xr-x 41 www-data www-data  4096 Oct 24 08:39 3rdparty
drwxr-xr-x 48 www-data www-data  4096 Nov 16 09:09 apps
-rw-r--r--  1 www-data www-data 17234 Oct 24 08:37 AUTHORS
drwxr-xr-x  2 www-data www-data  4096 Nov 16 09:09 config
-rw-r--r--  1 www-data www-data  3893 Oct 24 08:37 console.php
-rw-r--r--  1 www-data www-data 34520 Oct 24 08:37 COPYING
drwxr-xr-x 22 www-data www-data  4096 Oct 24 08:39 core
-rw-r--r--  1 www-data www-data  5083 Oct 24 08:37 cron.php
drwxr-xr-x  2 www-data www-data  4096 Nov 16 09:07 data
-rw-r--r--  1 www-data www-data  3124 Nov 16 09:09 .htaccess
-rw-r--r--  1 www-data www-data   156 Oct 24 08:37 index.html
-rw-r--r--  1 www-data www-data  2960 Oct 24 08:37 index.php
drwxr-xr-x  6 www-data www-data  4096 Oct 24 08:37 lib
-rw-r--r--  1 www-data www-data   283 Oct 24 08:37 occ
drwxr-xr-x  2 www-data www-data  4096 Oct 24 08:37 ocm-provider
drwxr-xr-x  2 www-data www-data  4096 Oct 24 08:37 ocs
drwxr-xr-x  2 www-data www-data  4096 Oct 24 08:37 ocs-provider
-rw-r--r--  1 www-data www-data  3102 Oct 24 08:37 public.php
-rw-r--r--  1 www-data www-data  5332 Oct 24 08:37 remote.php
drwxr-xr-x  4 www-data www-data  4096 Oct 24 08:37 resources
-rw-r--r--  1 www-data www-data    26 Oct 24 08:37 robots.txt
-rw-r--r--  1 www-data www-data  2379 Oct 24 08:37 status.php
drwxr-xr-x  3 www-data www-data  4096 Oct 24 08:37 themes
drwxr-xr-x  2 www-data www-data  4096 Oct 24 08:38 updater
-rw-r--r--  1 www-data www-data   101 Oct 24 08:37 .user.ini
-rw-r--r--  1 www-data www-data   362 Oct 24 08:39 version.php
{% endhighlight %}

Como se puede apreciar, el fichero se ha descomprimido correctamente y tras ello, ha sido eliminado. Posteriormente, tendremos que repetir exactamente el mismo procedimiento en **/srv/nextclouddata** para así descomprimir el otro fichero, de manera que lo primero que haremos será movernos a dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/iesgn19/cloud# cd /srv/nextclouddata/
{% endhighlight %}

Una vez dentro del directorio, listaremos el contenido del mismo para así verificar que el fichero comprimido ha sido correctamente transferido, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/srv/nextclouddata# ls -l
total 16640
-rw-r--r-- 1 debian debian 17039195 Nov 16 18:51 nextclouddata.tar.gz
{% endhighlight %}

Efectivamente, el fichero comprimido ha sido correctamente transferido, por lo que ya podremos descomprimirlo para así obtener exactamente la misma estructura anteriormente existente en el entorno de desarrollo. Además, eliminaremos tras ello el fichero comprimido para así liberar espacio, ya que no nos va a ser necesario. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:/srv/nextclouddata# tar -zxf nextclouddata.tar.gz && rm nextclouddata.tar.gz
{% endhighlight %}

Para verificar que se ha descomprimido correctamente y que además se ha eliminado de dicho directorio, haremos uso de `ls -la`:

{% highlight shell %}
root@vps:/srv/nextclouddata# ls -la
total 96
drwxrwx---  4 www-data www-data  4096 Nov 16 18:57 .
drwxr-xr-x  6 root     root      4096 Nov 16 18:09 ..
drwxr-xr-x  4 www-data www-data  4096 Nov 16 18:02 alvaro
drwxr-xr-x 10 www-data www-data  4096 Nov 16 18:02 appdata_occ9n86f7dq2
-rw-r--r--  1 www-data www-data   542 Nov 16 09:09 .htaccess
-rw-r--r--  1 www-data www-data     0 Nov 16 09:09 index.html
-rw-r-----  1 www-data www-data 76310 Nov 16 18:04 nextcloud.log
-rw-r--r--  1 www-data www-data     0 Nov 16 09:09 .ocdata
{% endhighlight %}

Como se puede apreciar, el fichero se ha descomprimido correctamente y tras ello, ha sido eliminado.

Actualmente, la web no funcionaría todavía ya que la máquina a la que está configurada para hacer las peticiones SQL no se encuentra operativa, por lo que tendremos que llevar a cabo las correspondientes modificaciones en el fichero oportuno para indicarle la nueva base de datos a la que ha de realizar las peticiones. Éste fichero se encuentra en **/srv/iesgn19/cloud/config/**, con nombre **config.php**, que modificaremos con `nano`:

{% highlight shell %}
root@vps:/srv/nextclouddata# nano /srv/iesgn19/cloud/config/config.php
{% endhighlight %}

Dentro del mismo, encontraremos un bloque como el siguiente:

{% highlight shell %}
<?php
$CONFIG = array (
  'instanceid' => 'occ9n86f7dq2',
  'passwordsalt' => 'QsJmN6fF+Nrj8djQTRgjH/r65iRkZM',
  'secret' => 'smxsGYDQnn0kbvwxSsrRBFHc9nek1YTTFP16BSTP8qmehUF1',
  'trusted_domains' =>
  array (
    0 => 'www.alvaro-nextcloud.com',
  ),
  'datadirectory' => '/srv/nextclouddata',
  'dbtype' => 'mysql',
  'version' => '20.0.1.1',
  'overwrite.cli.url' => 'http://www.alvaro-nextcloud.com',
  'dbname' => 'bd_nextcloud',
  'dbhost' => '10.0.0.5:3306',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'user_nextcloud',
  'dbpassword' => 'pass_nextcloud',
  'installed' => true,
);
{% endhighlight %}

Como se puede apreciar, ahí se encuentran las credenciales de acceso y la base de datos a la que acceder (correspondiente a la máquina **backup**). En este caso, el nombre de la base de datos se ha mantenido como **bd_nextcloud**, el usuario se ha mantenido como **user_nextcloud** y la contraseña también se ha mantenido como **pass_nextcloud**. Además, la base de datos ya no se encuentra en **10.0.0.5:3306**, sino en **bd.iesgn19.es:3306**. Por último, tendremos que cambiar el dominio con confianza de **www.alvaro-nextcloud.com** a **www.iesgn19.es** y la URL donde se encuentra alojada la aplicación de **http://www.alvaro-nextcloud.com** a **http://www.iesgn19.es/cloud**, manteniéndose a su vez el directorio de datos en **/srv/nextclouddata**. La configuración final quedaría así:

{% highlight shell %}
<?php
$CONFIG = array (
  'instanceid' => 'occ9n86f7dq2',
  'passwordsalt' => 'QsJmN6fF+Nrj8djQTRgjH/r65iRkZM',
  'secret' => 'smxsGYDQnn0kbvwxSsrRBFHc9nek1YTTFP16BSTP8qmehUF1',
  'trusted_domains' =>
  array (
    0 => 'www.iesgn19.es',
  ),
  'datadirectory' => '/srv/nextclouddata',
  'dbtype' => 'mysql',
  'version' => '20.0.1.1',
  'overwrite.cli.url' => 'http://www.iesgn19.es/cloud',
  'dbname' => 'bd_nextcloud',
  'dbhost' => 'bd.iesgn19.es:3306',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'user_nextcloud',
  'dbpassword' => 'pass_nextcloud',
  'installed' => true,
);
{% endhighlight %}

Tras ello, guardaremos los cambios, y antes de tratar de acceder al sitio _Nextcloud_, tendremos que instalar aquellas librerías y extensiones que se requerieron en la instalación de la aplicación en el entorno de desarollo, con la finalidad de intentar que ambos entornos (desarrollo y producción) sean en todo momento lo más similares posibles. En este caso, instalaremos dichas librerías y extensiones ejecutando el comando:

{% highlight shell %}
root@vps:/srv/nextclouddata# apt install php-zip php-curl
{% endhighlight %}

Listo, ya hemos finalizado la migración, así que ha llegado _la hora de la verdad_. Para verificar si la migración ha sido realmente exitosa, accederemos desde el navegador a **www.iesgn19.es/cloud/**:

![nextcloud3](https://i.ibb.co/RhbjRMT/5.jpg "Nextcloud")

Efectivamente, el sitio web se ha mostrado correctamente y hemos podido hacer uso de las funcionalidades de la aplicación, como por ejemplo, autenticarnos, por lo que podemos asegurar que la migración se ha llevado a cabo de forma exitosa.

### Instala y configura en un ordenador el cliente de Nextcloud.

Para éste apartado, he vuelto a la máquina anfitriona, donde procederé a instalar dicho cliente, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt install nextcloud-desktop
{% endhighlight %}

Tras ello, simplemente ejecutaremos el siguiente comando para abrir la aplicación de escritorio:

{% highlight shell %}
alvaro@debian:~$ nextcloud
{% endhighlight %}

La configuración de la misma se compone de los siguientes pasos:

* Nos preguntará si queremos **registrarnos en un proveedor** o **entrar** a uno en el que ya estamos registrados, por lo que pulsaremos en **Entrar**.
* Tras ello nos preguntará la dirección del mismo, en este caso, **http://www.iesgn19.es/cloud/**.
* Tendremos que pulsar en **Iniciar sesión** e introducir nuestros credenciales para posteriormente **Conceder acceso** a la aplicación para acceder a nuestra cuenta.
* Establecemos qué es lo que queremos sincronizar y dónde queremos hacerlo y pulsaremos en **Conectar**.

Finalmente, la aplicación de escritorio se verá de la siguiente forma:

![nextcloud4](https://i.ibb.co/zP18vTV/12.jpg "Aplicación Nextcloud")

En mi caso, he decidido que la sincronización se lleve a cabo en el directorio **/home/alvaro/Nextcloud**, así que listaré el contenido del mismo para verificar que la sincronización se ha llevado a cabo correctamente:

{% highlight shell %}
alvaro@debian:~$ ls -l /home/alvaro/Nextcloud/
total 10328
drwxr-xr-x 2 alvaro alvaro    4096 nov 19 09:26  Documents
-rw-r--r-- 1 alvaro alvaro 3963036 nov 16 10:09 'Nextcloud intro.mp4'
-rw-r--r-- 1 alvaro alvaro 5745866 nov 16 10:09 'Nextcloud Manual.pdf'
-rw-r--r-- 1 alvaro alvaro   50598 nov 16 10:09  Nextcloud.png
drwxr-xr-x 2 alvaro alvaro    4096 nov 19 09:26  Photos
-rw-r--r-- 1 alvaro alvaro       1 nov 16 10:09  Readme.md
-rw-r--r-- 1 alvaro alvaro  791921 nov 16 10:09 'Reasons to use Nextcloud.pdf'
{% endhighlight %}

Efectivamente, todos los ficheros que se crearon automáticamente durante la instalación de _Nextcloud_ están ahora sincronizados, así que para hacer la prueba, vamos a generar un nuevo fichero en dicho directorio para comprobar que también se sincroniza sin problemas. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ echo "Hola" > /home/alvaro/Nextcloud/prueba.txt
{% endhighlight %}

Tras ello, accederemos a la aplicación desde el navegador web para ver si dicho fichero es accesible desde ahí:

![nextcloud5](https://i.ibb.co/9ttxk1S/14.jpg "Aplicación Nextcloud")

Como era de esperar, no ha habido ningún problema en la sincronización y todos los ficheros que creemos en **/home/alvaro/Nextcloud** serán accesibles desde la aplicación en el navegador web y viceversa.

Por último, y como curiosidad, me gustaría listar el contenido del directorio del usuario **alvaro** en **/srv/nextclouddata**, directorio alojado en la VPS, para así concienciar del gran cuidado que hay que tener como administrador de sistemas a la hora de administrar una nube:

{% highlight shell %}
root@vps:/srv/nextclouddata# ls -l /srv/nextclouddata/alvaro/files/
total 10328
drwxr-xr-x 2 www-data www-data    4096 Nov 16 09:09  Documents
-rw-r--r-- 1 www-data www-data 3963036 Nov 16 09:09 'Nextcloud intro.mp4'
-rw-r--r-- 1 www-data www-data 5745866 Nov 16 09:09 'Nextcloud Manual.pdf'
-rw-r--r-- 1 www-data www-data   50598 Nov 16 09:09  Nextcloud.png
drwxr-xr-x 2 www-data www-data    4096 Nov 16 09:09  Photos
-rw-r--r-- 1 www-data www-data       5 Nov 19 08:28  prueba.txt
-rw-r--r-- 1 www-data www-data       1 Nov 16 09:09  Readme.md
-rw-r--r-- 1 www-data www-data  791921 Nov 16 09:09 'Reasons to use Nextcloud.pdf'
{% endhighlight %}

Como se puede apreciar, todos los ficheros del usuario **alvaro** (junto a los del resto de usuarios) están aquí contenidos y son accesibles sin ningún tipo de cifrado, por lo que cualquier fallo de ajuste en los permisos UNIX supondría un gran agujero de seguridad.