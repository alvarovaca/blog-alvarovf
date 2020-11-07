---
layout: post
title:  "Servidor Web Nginx"
banner: "/assets/images/banners/nginx.jpg"
date:   2020-11-07 13:56:00 +0200
categories: servicios
---
## VirtualHosting

Gracias a la funcionalidad "**VirtualHost**", también conocida como "**Server Blocks**" del servidor web _nginx_ podemos hacer que una misma máquina con una única interfaz de red (es decir, con una única dirección IP) escuchando en un único puerto (generalmente el puerto **80** para HTTP y **443** para HTTPS) pueda servir diferentes páginas web que se puedan acceder de forma independiente. Hoy en día, _nginx_ también se utiliza como proxy inverso, balanceador de carga HTTP y proxy de correo electrónico para IMAP, POP3 y SMTP, aunque no lo trataremos en este artículo.

Para ello, se hace uso de las cabeceras que contiene la petición HTTP que recibe el servidor, que contiene un parámetro "**Host**" que indica el nombre del servidor al que se le hace la petición (por ejemplo, **www.prueba1.com** o **www.prueba2.com**).

Al igual que se puede especificar que la separación se haga mediante los nombres de dominio, también se puede hacer basada en la dirección IP, de manera que cada sitio tenga una dirección IP diferente, pero no es algo tan "sorprendente", por lo que no lo trataremos en este _post_.

Es muy probable que hasta el día de hoy no supieses de la existencia de dicha funcionalidad, y pensabas que por cada sitio web que tuviese que servir _nginx_, había que tener una máquina (ya sea física o virtual) para cada uno de ellos, pero gracias a esta funcionalidad, se ahorrarán costes tanto monetarios como computacionales, al servir todos los sitios web desde la misma máquina, sin necesidad de ejecutar ninguna máquina virtual.

Pero si yo no he configurado nada, ¿cómo es posible que al acceder a **localhost:80**, me sirva automáticamente el contenido que hay en **/var/www/html**? Bien, la respuesta es muy sencilla, y es que _nginx_ se instala por defecto con un VirtualHost configurado, de nombre "**default**", sin especificar el ServerName desde el que es accesible, es decir, significando que podemos entrar con cualquier nombre, siendo el DocumentRoot (directorio donde se encuentran alojados los ficheros a servir por el sitio web) "**/var/www/html**". Además, en el fichero de configuración se indica que se escuchan las peticiones desde el puerto __80__, por lo que se puede acceder desde cualquier dirección IP, siempre y cuando sea al puerto **80**.

Podemos encontrar los ficheros de configuración de todos los VirtualHost creados en _nginx_ en **/etc/nginx/sites-available/**, pero eso no significa que todos los que se encuentren ahí estén habilitados y actualmente activos. Los sitios actualmente activos los encontramos en **/etc/nginx/sites-enabled/**, que al fin y al cabo son un enlace simbólico al fichero de configuración existente en _/etc/nginx/sites-available/_.

El objetivo de esta práctica es el de construir en nuestro servidor web _nginx_ dos sitios web con nombres distintos pero compartiendo dirección IP y puerto, sobre los que realizaremos determinadas configuraciones posteriormente:

* El primero de ellos tendrá nombre de dominio **www.iesgn.org**, con DocumentRoot en **/srv/www/iesgn** y cuyo contenido será una página llamada **index.html**, donde se verá una bienvenida a la página del instituto Gonzalo Nazareno.
* El segundo de ellos tendrá nombre de dominio **departamentos.iesgn.org**, con DocumentRoot en **/srv/www/departamentos** y cuyo contenido será una página llamada **index.html**, donde se verá una bienvenida a la página de los departamentos del instituto.

Para la práctica he creado una máquina virtual en el cloud (**OpenStack**), de las siguientes características:

* **Instance Name**: nginx
* **Source**: Debian Buster 10.6
* **Sabor**: m1.mini

Además, nos tendremos que asegurar que le asociemos nuestro par de claves (en caso de no tenerlo, debemos crearlo), puesto que el primer inicio se tendrá que llevar a cabo mediante un par de claves, pues no tiene ninguna contraseña configurada.

Dado que en OpenStack tenemos una red privada con direccionamiento **10.0.0.0/24**, tendremos que tener un router virtual que nos permita hacer DNAT a dicho direccionamiento. Por ello, tendremos que añadirle una IP flotante, de la siguiente forma:

![openstack1](https://i.ibb.co/dfFQ26V/7.jpg "Asociación IP flotante")

Tras la creación, obtendremos el siguiente resultado:

![openstack2](https://i.ibb.co/sj0KBCb/8.jpg "Creación instancia")

Como se puede apreciar, tenemos una máquina virtual con dirección IP **10.0.0.3/24**, que es accesible haciendo DNAT a dicha dirección desde un router virtual con dirección **172.22.200.108/16**.

Si lo pensamos, al estar trabajando con un router que hace Destination NAT (DNAT), tendremos que abrir el correspondiente puerto para servir páginas web mediante **HTTP** (**80**), ya que de lo contrario, no podremos acceder a dicho puerto de la máquina. Para ello, nos iremos al apartado **Acceso y seguridad** y pulsaremos en **Agregar regla**, agregando las correspondientes reglas **TCP** para el puerto **80** (y para el puerto **22**, para llevar a cabo la conexión **SSH**, en caso de que no existiesen con anterioridad), quedando de la siguiente manera:

![openstack3](https://i.ibb.co/RcXsH94/9.jpg "Abriendo puertos")

Tras ello, ya estaremos preparados para llevar a cabo la conexión SSH a la máquina, pero antes de ello, vamos a añadir la clave privada que vamos a utilizar para conectarnos a la máquina virtual a nuestro agente de claves de la sesión, pues nos será necesario más adelante (lo explicaremos con más detalle llegado el momento). Para ello, tendremos que hacer uso de `ssh-add`, indicando la ruta de la clave privada que vamos a utilizar para la autentificación:

{% highlight shell %}
alvaro@debian:~$ ssh-add .ssh/linux.pem 
Identity added: .ssh/linux.pem (.ssh/linux.pem)
{% endhighlight %}

Como se puede apreciar, la clave privada ha sido correctamente añadida a nuestro agente, así que ahora sí, vamos a proceder a realizar la conexión SSH a la dirección IP del router virtual que hará DNAT, es decir, a **172.22.200.108**, pero indicando una opción concreta:

{% highlight shell %}
alvaro@debian:~$ ssh -A debian@172.22.200.108
Linux nginx 4.19.0-11-cloud-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Donde:

* **-A**: Indica a SSH que herede el agente con las claves privadas almacenadas, que podrán ser utilizadas dentro de dicha máquina virtual. Lo trataremos con más detalle a continuación.

Una vez que nos hemos conseguido conectar correctamente, procedemos a configurar el servidor web _nginx_. Para ello, lo primero que haremos será instalar el paquete necesario (**nginx**), no sin antes upgradear los paquetes instalados, ya que la máquina virtual se va quedando desactualizada con el tiempo. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@nginx:~# apt update && apt upgrade && apt install nginx
{% endhighlight %}

Una vez instalado el paquete, vamos a modificar la página inicial de _nginx_ para verificar que está funcionando correctamente (se encuentra alojada en **/var/www/html/index.nginx-debian.html**, como consecuencia del VirtualHost generado por defecto). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@nginx:~# nano /var/www/html/index.nginx-debian.html
{% endhighlight %}

El contenido existente por defecto es el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <title>Welcome to nginx!</title>
        <style>
            body {
                width: 35em;
                margin: 0 auto;
                font-family: Tahoma, Verdana, Arial, sans-serif;
            }
        </style>
    </head>
    <body>
        <h1>Welcome to nginx!</h1>
        <p>If you see this page, the nginx web server is successfully installed and
        working. Further configuration is required.</p>

        <p>For online documentation and support please refer to
        <a href="http://nginx.org/">nginx.org</a>.<br/>
        Commercial support is available at
        <a href="http://nginx.com/">nginx.com</a>.</p>

        <p><em>Thank you for using nginx.</em></p>
    </body>
</html>
{% endhighlight %}

En mi caso, he eliminado todo el contenido y he añadido un código HTML bastante básico, quedando de la siguiente forma:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Prueba nginx</title>
    </head>
    <body>
        <h2>El servidor nginx ha sido correctamente configurado.</h2>
    </body>
</html>
{% endhighlight %}

Tras ello, guardaremos los cambios y accederemos a la dirección IP del router virtual desde el navegador para verificar que está funcionando como debería:

![nginx1](https://i.ibb.co/RjDMF4N/10.jpg "Modificación HTML")

Efectivamente, el contenido por defecto de la página web ha sido modificado, así que ya estamos en posición de crear los nuevos DocumentRoot para ambos sitios dentro de **/srv/www**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:~# mkdir -p /srv/www/{iesgn,departamentos}
{% endhighlight %}

Donde:

* **-p**: Indicamos que cree el directorio padre en caso de no existir.

Para verificar que la creación se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:~# tree /srv/
/srv/
└── www
    ├── departamentos
    └── iesgn

3 directories, 0 files
{% endhighlight %}

Efectivamente, han sido creados, así que tras ello, tendremos que crear los correspondientes ficheros de configuración para los sitios web, que se encuentran en **/etc/nginx/sites-available**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@nginx:~# cd /etc/nginx/sites-available/
{% endhighlight %}

Tras ello, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ls
default
{% endhighlight %}

En este caso, existe un único fichero, **default**, así que para no complicarnos, vamos a copiar dicho fichero dos veces para tener una plantilla base que posteriormente modificaremos y adaptaremos a cada uno de los sitios web que vamos a crear. Para ello, ejecutaremos los comandos:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# cp default iesgn
root@nginx:/etc/nginx/sites-available# cp default departamentos
{% endhighlight %}

Para verificar que los ficheros se han copiado correctamente, listaremos el contenido de dicho directorio haciendo uso de `ls`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ls
default  departamentos  iesgn
{% endhighlight %}

Empezaré modificando el fichero **iesgn**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano iesgn
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

* La primera de las líneas indica que está escuchando peticiones en el puerto **80 IPv4**. Dado que no se especifica ninguna dirección IP, escuchará en todas las interfaces. Hay un dato importante, y es que contiene el valor "**default_server**", significando que en caso de hacer una petición HTTP al servidor web en la que no se especifique ningún nombre de dominio coincidente con algún VirtualHost en la cabecera Host (como por ejemplo cuando accedemos a través de la dirección IP), éste será el sitio web que se servirá por defecto. Como es lógico, únicamente podrá haber un VirtualHost con dicha funcionalidad activada, así que en este caso, la deshabilitaremos para que el sitio por defecto siga siendo **default**. Es un gran punto a favor para _nginx_ el hecho de poder seleccionar el VirtualHost por defecto, ya que _apache2_, se limita a ordenar alfabéticamente el nombre de los VirtualHost y mostrar el primero de ellos.

* La segunda linea, es similar a la anterior pero con IPv6, de manera que también deshabilitaremos la funcionalidad "**default_server**".

* La tercera de ellas, tiene una gran importancia, ya que tendremos que indicar el DocumentRoot de dicho VirtualHost, es decir, el directorio que contendrá los ficheros a servir por el servidor web. En este caso, será **/srv/www/iesgn**, así que lo modificaremos. Es equivalente a la directiva _DocumentRoot_ de _apache2_.

* La cuarta línea establece la lista de recursos que se deben buscar cuando el cliente solicita el índice de un directorio del sitio (es decir, no ha solicitado ningún recurso concreto), siguiendo un determinado orden de prioridad, de manera que el primero fichero en servirse sería el **index.html**, seguido del **index.htm**, seguido de **index.nginx-debian.html**... No es necesario realizar modificación alguna. Es equivalente a la directiva _DirectoryIndex_ de _apache2_.

* La quinta línea tiene también gran importancia, pues especificaremos el nombre de dominio a través del cuál accederemos al VirtualHost, que se encontrará contenido en la cabecera Host de la petición HTTP, de manera que encontrará una coincidencia y nos mostrará el sitio en cuestión. En este caso, será **www.iesgn.org**. Es equivalente a la directiva _ServerName_ de _apache2_.

* Por último, encontramos una sección **location**, donde se especificarán las instrucciones para resolver una petición (lo veremos con más detalle a continuación). En este caso, se ha indicado que cuando se lleve a cabo la petición de un recurso, primero se compruebe si existe como fichero, en caso de que no, compruebe si existe como directorio y si tampoco existe, devuelva un error 404 (Not Found). No será necesario llevar a cabo ninguna modificación. Es equivalente a la directiva _Directory_ de _apache2_.

El resultado final, tras llevar a cabo todas las modificaciones necesarias, es el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/iesgn;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

Tras ello, repetiremos el mismo proceso en el fichero **departamentos**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano departamentos
{% endhighlight %}

Dentro del mismo, deshabilitaremos la directiva **default_server**, indicaremos como **DocumentRoot** el directorio **/srv/www/departamentos** y estableceremos como **ServerName** el nombre **departamentos.iesgn.org**, quedando de la siguiente forma:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/departamentos;

        index index.html index.htm index.nginx-debian.html;

        server_name departamentos.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

Una vez que los ficheros de configuración hayan sido creados y correctamente modificados, podremos proceder a crear el fichero **index.html** que servirá cada uno de los sitios web. En este caso, he creado un fichero muy simple para cada uno de ellos:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/iesgn/index.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html lang="es">
    <head>
        <meta charset="utf-8">
        <title>Iesgn</title>
    </head>
    <body>
        <h2>Bienvenidos a Iesgn</h2>
    </body>
</html>
{% endhighlight %}

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/departamentos/index.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html lang="es">
    <head>
        <meta charset="utf-8">
        <title>Departamentos Iesgn</title>
    </head>
    <body>
        <h2>Bienvenidos a los departamentos de Iesgn</h2>
    </body>
</html>
{% endhighlight %}

Para verificar que dichos ficheros se han generado correctamente en sus correspondientes directorios, volveremos a listar el contenido del directorio **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        └── index.html

3 directories, 2 files
{% endhighlight %}

Efectivamente, los ficheros se han generado correctamente. Listo, toda la configuración necesaria ya ha sido realizada, así que únicamente queda habilitar dichos sitios y probar que realmente funcionan. Para activar los sitios, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/nginx/sites-available** dentro de **/etc/nginx/sites-enabled** de forma manual. Para ello, ejecutamos los comandos:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ln -s /etc/nginx/sites-available/iesgn /etc/nginx/sites-enabled/
root@nginx:/etc/nginx/sites-available# ln -s /etc/nginx/sites-available/departamentos /etc/nginx/sites-enabled/
{% endhighlight %}

Al parecer, los sitios han sido correctamente habilitados, pero para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

Una vez que la configuración del servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/nginx/sites-enabled** para verificar que los correspondientes enlaces simbólicos han sido correctamente creados. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 34 Nov  3 07:30 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 40 Nov  3 08:16 departamentos -> /etc/nginx/sites-available/departamentos
lrwxrwxrwx 1 root root 32 Nov  3 08:16 iesgn -> /etc/nginx/sites-available/iesgn
{% endhighlight %}

Efectivamente, los tres sitios se encuentran actualmente activos, así que es hora de realizar las correspondientes pruebas de acceso. Para ello, volveremos a la máquina anfitriona y configuraremos la resolución estática de nombres en la misma, para así poder realizar la traducción del nombre. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** por cada uno de los sitios que hayamos creado, siendo **[IP]** la dirección IP de la máquina servidora (en ambos casos **172.22.200.108**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
172.22.200.108  www.iesgn.org
172.22.200.108  departamentos.iesgn.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para que se lleve a cabo la resolución estática de nombres, que tiene prioridad sobre las peticiones al servidor DNS. Al primero que intentaré acceder será a **www.iesgn.org**:

![iesgn1](https://i.ibb.co/4pj5VP3/11.jpg "www.iesgn.org")

Como podemos apreciar, se ha podido acceder sin problema, ya que hemos realizado correctamente la configuración del **ServerName** en el fichero de configuración y a la hora de comparar la cabecera **Host** existente en la petición HTTP, ha encontrado coincidencia con dicho VirtualHost. Por último, probaré también con **departamentos.iesgn.org**:

![departamentos1](https://i.ibb.co/g9Vn8Vr/12.jpg "departamentos.iesgn.org")

Efectivamente, también ha sido accesible, por lo que podemos concluir que hemos conseguido alojar dos sitios web en una única máquina, siendo accesibles ambos de ellos desde la misma dirección IP y puerto.

## Mapeo de URL

Vamos a llevar a cabo algunas configuraciones de ejemplo para profundizar un poco más en el uso de _nginx_, en este caso, sobre mapeo de URL, que llevaremos a cabo en el VirtualHost **www.iesgn.org**.

### Ejemplo 1

Lo primero que trataremos de hacer será una redirección, de manera que cuando se acceda a la dirección **www.iesgn.org**, se redireccione automáticamente a **www.iesgn.org/principal**, donde se mostrará el mensaje de bienvenida. En dicho directorio **principal/**, vamos a restringir además el listado de ficheros y el seguimiento de enlaces simbólicos.

Lo primero que tendremos que hacer por tanto, será crear dicho directorio dentro del DocumentRoot, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# mkdir /srv/www/iesgn/principal
{% endhighlight %}

Para verificar que la creación de dicho directorio se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        ├── index.html
        └── principal

4 directories, 2 files
{% endhighlight %}

Efectivamente, el directorio ha sido correctamente generado, así que podremos proceder a editar el fichero de configuración del VirtualHost, para así llevar a cabo las modificaciones necesarias, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano iesgn
{% endhighlight %}

En este caso, queremos hacer una redirección que nos permita pedir al cliente que haga otra petición a una URL diferente (indicada en la cabecera de la respuesta HTTP junto a un código de estado **3xx**). Existen las redirecciones **Permanentes** (301) y **Temporales** (302). Hay que tener cuidado con las cachés de los navegadores ya que puede ser que se aprendan dichas redirecciones para así evitar preguntar al servidor nuevamente.

Podemos redireccionar a una URL en un **host diferente** (deberá ser **absoluta**, como por ejemplo de **"/service"** a **"http://foo2.example.com/service"**) o a una URL dentro del **mismo host** (deberá comenzar por **/**, como por ejemplo de **"/one"** a **"/two"**). En este caso, vamos a redireccionar a una URL del mismo host.

La sintaxis para crear una redirección es:

{% highlight shell %}
rewrite ^<pathURL>$ <URL> <tipo>;
{% endhighlight %}

Donde:

* **pathURL**: El recurso "virtual" que solicitaremos en la URL a la hora de hacer la petición al servidor. En este caso, la petición la haríamos a www.iesgn.org**/**, por lo que pathURL sería **/**.
* **URL**: La ruta real a la que se va a mapear el pathURL, es decir, la URL a la que deseamos redirigir al cliente. En este caso, la redirección la queremos hacer a www.iesgn.org**/principal**, por lo que al tratarse de una redirección dentro del mismo host, la URL sería **/principal**.
* **tipo**: El tipo de redirección que queremos llevar a cabo, ya sea temporal (**redirect**) o permanente (**permanent**). En este caso, es indiferente, así que usaremos una redirección temporal.

En resumen, la redirección que queremos crear tendrá la siguiente forma:

{% highlight shell %}
rewrite ^/$ /principal redirect;
{% endhighlight %}

Genial, la redirección ya está declarada, pero todavía no hemos restringido las funcionalidades para dicho directorio. Recalco lo de "dicho directorio" ya que queremos que la prohibición del listado de ficheros y el seguimiento de enlaces simbólicos únicamente se produzca en el directorio **principal/**, no en todo el VirtualHost, de manera que tendremos que crear una nueva sección **location** que contenga dichas directivas que afecten únicamente a **principal/**.

En _apache2_, la opción para habilitar el listado de ficheros era **Indexes** (de manera que si se eliminaba, no se permitiría dicho listado), mientras que en _nginx_, se utiliza **autoindex off**. De la misma manera, se utilizaba la opción **FollowSymLinks** para habilitar el seguimiento de enlaces simbólicos en _apache2_ (de manera que si se eliminaba, no se permitiría dicho seguimiento), mientras que en _nginx_, se utiliza **disable_symlinks on**. El resultado final sería el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/iesgn;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }

        rewrite ^/$ /principal redirect;

        location /principal {
                autoindex off;
                disable_symlinks on;
        }
}
{% endhighlight %}

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

En un principio, la nueva configuración ya está habilitada, pero para comprobar que lo que hemos hecho es correcto, vamos a crear un fichero de nombre **prueba.txt** (por ejemplo, aunque se podría haber usado cualquier otro nombre) con un contenido cualquiera en el directorio personal (**/home/debian/**), al que posteriormente crearemos un enlace simbólico para así verificar que el seguimiento de enlaces simbólicos se encuentra deshabilitado. Para generar dicho fichero, ejecutaremos el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# echo "Prueba SymLink" > /home/debian/prueba.txt
{% endhighlight %}

El fichero ya se encuentra generado dentro del directorio personal, así que ahora, crearemos un enlace simbólico dentro del directorio **principal/**, que apunte a dicho fichero que acabamos de generar. Para generar el enlace simbólico, haremos uso de `ln -s`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ln -s /home/debian/prueba.txt /srv/www/iesgn/principal/
{% endhighlight %}

Donde:

* **-s**: Indica que se cree un enlace simbólico (ya que por defecto, **ln** crea enlaces duros).

Para verificar que la creación de dicho enlace simbólico se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        ├── index.html
        └── principal
            └── prueba.txt -> /home/debian/prueba.txt

4 directories, 3 files
{% endhighlight %}

Efectivamente, el enlace simbólico con nombre **prueba.txt** se encuentra correctamente generado y apuntando a **/home/debian/prueba.txt**, de manera que para comprobar todos los cambios hechos hasta ahora, vamos a tratar de acceder desde el navegador a **www.iesgn.org**:

![iesgn2](https://i.ibb.co/sHD7XWQ/13.jpg "www.iesgn.org/principal")

Como se puede apreciar, la redirección de **www.iesgn.org** a **www.iesgn.org/principal** ha funcionado correctamente, a pesar de que haya mostrado un error 403 (Forbidden). Esto último es debido a que no ha encontrado ninguno de los ficheros a servir por defecto (**index**) y además, hemos deshabilitado la posibilidad de listar el contenido del mismo.

Sin embargo, todavía tenemos que comprobar si podemos seguir el enlace simbólico que hemos generado, así que solicitaremos el recurso **prueba.txt**, que es el enlace simbólico en cuestión, accediendo por tanto a la dirección **www.iesgn.org/principal/prueba.txt**:

![iesgn3](https://i.ibb.co/nbWtsQY/14.jpg "www.iesgn.org/principal/prueba.txt")

Como era de esperar, ha devuelto de nuevo un error 403 (Forbidden), ya que no tiene permisos para seguir dicho enlace simbólico.

Por último, vamos a mover el fichero **index.html** actualmente ubicado en **/srv/www/iesgn/** a **/srv/www/iesgn/principal/**, de manera que al acceder a **www.iesgn.org**, se lleve a cabo la redirección a **www.iesgn.org/principal** y se muestre por tanto dicho fichero, gracias a la directiva **index**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# mv /srv/www/iesgn/index.html /srv/www/iesgn/principal/
{% endhighlight %}

De nuevo, volveremos a acceder a **www.iesgn.org**:

![iesgn4](https://i.ibb.co/jgFDLR8/post.jpg "www.iesgn.org/principal/index.html")

Como era de esperar, la redirección ha vuelto a funcionar correctamente, y además, dado que hemos movido el fichero **index.html** al directorio **principal/**, ha sido capaz de mostrarlo.

### Ejemplo 2

Lo siguiente que trataremos de hacer será un alias, de manera que cuando se acceda a la dirección **www.iesgn.org/principal/documentos**, se muestre el contenido de **/srv/doc**. En dicho recurso virtual, vamos a permitir el listado de ficheros y el seguimiento de enlaces simbólicos, siempre y cuando el dueño del enlace simbólico y el del fichero o directorio al que apunta, sea el mismo.

Lo primero que tendremos que hacer por tanto, será crear dicho directorio real que va a contener los ficheros a servir fuera del DocumentRoot (**/srv/doc/**), ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# mkdir /srv/doc
{% endhighlight %}

Para verificar que la creación de dicho directorio se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
├── doc
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        └── principal
            ├── index.html
            └── prueba.txt -> /home/debian/prueba.txt

5 directories, 3 files
{% endhighlight %}

Efectivamente, el directorio ha sido correctamente generado, así que podremos proceder a editar el fichero de configuración del VirtualHost, para así llevar a cabo las modificaciones necesarias, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano iesgn
{% endhighlight %}

En este caso, queremos crear un alias que nos permita mapear URLs a ubicaciones de un sistema de ficheros, o en otras palabras, que nos permita que el servidor nos sirva ficheros que no estén necesariamente situados dentro del DocumentRoot.

Extrapolando dicha definición a este ejemplo, podemos hacer que al acceder a **/principal/documentos**, nos muestre el contenido que existe dentro de **/srv/doc**, que como se puede intuir, no se encuentra dentro del DocumentRoot, pero aun así, podemos hacer que sea accesible.

La sintaxis para crear un alias (que en _nginx_ debe estar contenido dentro de una sección **location**) es:

{% highlight shell %}
alias <pathFS>;
{% endhighlight %}

Siendo:

* **pathFS**: La ruta real a la que se va a mapear el recurso virtual, es decir, la ruta del sistema de ficheros a la que deseamos apuntar. En este caso, el pathFS sería **/srv/doc**, pues es donde deseamos apuntar.

En resumen, el alias que queremos crear tendrá la siguiente forma:

{% highlight shell %}
alias /srv/doc;
{% endhighlight %}

Como acabo de mencionar, dicha directiva deberá estar dentro de una sección **location**, que afectará al recurso virtual, en este caso, a **/principal/documentos**. Además, tendremos que habilitar el listado de ficheros, gracias a la directiva **autoindex on**, y permitir únicamente el seguimiento de enlaces simbólicos si el dueño del enlace y del fichero o directorio al que apunta es el mismo, gracias a la directiva **disable_symlinks if_not_owner**. El resultado final sería el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/iesgn;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }

        rewrite ^/$ /principal redirect;

        location /principal {
                autoindex off;
                disable_symlinks on;
        }

        location /principal/documentos {
                alias /srv/doc;
                autoindex on;
                disable_symlinks if_not_owner;
        }
}
{% endhighlight %}

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

En un principio, la nueva configuración ya está habilitada, pero para comprobar que lo que hemos hecho es correcto, vamos a crear otro enlace simbólico dentro de **/srv/doc/** (de nombre **symlink.html**, por ejemplo) que apunte al fichero anteriormente generado en el directorio personal, para así verificar que el seguimiento de enlaces simbólicos se encuentra ahora habilitado. Para generar el enlace simbólico, haremos uso de `ln -s`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# ln -s /home/debian/prueba.txt /srv/doc/symlink.html
{% endhighlight %}

Para verificar que la creación de dicho enlace simbólico se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
├── doc
│   └── symlink.html -> /home/debian/prueba.txt
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        └── principal
            ├── index.html
            └── prueba.txt -> /home/debian/prueba.txt

5 directories, 4 files
{% endhighlight %}

Efectivamente, el enlace simbólico con nombre **symlink.html** se encuentra correctamente generado y apuntando a **/home/debian/prueba.txt**. Sin embargo, el usuario creador y el grupo correspondiente a todos los ficheros y directorios creados hasta ahora es **root**, cosa que jamás debe ocurrir ya que el usuario propietario por defecto a través del cuál se sirven las páginas web es **www-data**, de la misma manera, necesitaremos que el fichero al que se apunta tenga dicho propietario, para que así se pueda seguir el enlace simbólico, por lo que procedemos a cambiar dicho propietario y grupo de forma recursiva, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# chown -R www-data:www-data /srv/*
{% endhighlight %}

El fichero anteriormente generado con nombre **prueba.txt** en **/home/debian/** también tiene a **root** como propietario, pero para hacer la prueba, no vamos a modificar todavía dicho propietario, para así verificar que no podemos acceder. Lo primero que haremos será tratar de acceder a **www.iesgn.com/principal/documentos** desde el navegador, para ver si el alias funciona correctamente y se lista el contenido del directorio **/srv/doc/**:

![iesgn5](https://i.ibb.co/ZXhmQZL/15.jpg "www.iesgn.org/principal/documentos")

Efectivamente, el alias ha funcionado correctamente y está mostrando el contenido del directorio **/srv/doc/**, a pesar de estar accediendo a **principal/documentos**. Además, el enlace simbólico se muestra entre los ficheros existentes, de manera que pulsaremos en el mismo para ver si tenemos permisos para seguirlo:

![iesgn6](https://i.ibb.co/2sVCRS6/16.jpg "www.iesgn.org/principal/documentos/symlink.html")

Como era de esperar, nos ha devuelto un error 403 (Forbidden), dado que el propietario del enlace simbólico y del fichero al que apunta no coinciden. Vamos a proceder a solucionar esto, cambiando para ello el propietario del fichero existente en el directorio personal a **www-data**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# chown www-data:www-data /home/debian/prueba.txt
{% endhighlight %}

Tras ello, actualizaremos la página para que vuelva a solicitar el recurso:

![iesgn7](https://i.ibb.co/wBrpBVj/17.jpg "www.iesgn.org/principal/documentos/symlink.html")

Tal y como se puede apreciar, el alias está funcionando correctamente, además del listado de ficheros en el directorio y el correspondiente seguimiento de los enlaces simbólicos siempre y cuando el propietario coincida con el fichero o directorio al que apuntan.

### Ejemplo 3

Para terminar con los ejemplos de mapeo de URL, vamos a crear mensajes de error personalizados para todo el VirtualHost para los errores de objeto no encontrado (**404**) y no permitido (**403**). Para ello, tendremos que escribir el HTML de cada error dentro de un fichero, en este caso, dentro de un nuevo directorio de nombre **error/**.

Lo primero que tendremos que hacer será generar dicho directorio dentro del DocumentRoot, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# mkdir /srv/www/iesgn/error
{% endhighlight %}

Tras ello, generaremos dentro del mismo un fichero por cada uno de los errores a gestionar. En este caso, voy a empezar por el error **404**, que estará contenido en un fichero de nombre **404.html**, por ejemplo:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/iesgn/error/404.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>No encontrado</title>
    </head>
    <body>
        <h2>El recurso solicitado no existe (404).</h2>
        <h3>Contacta con el administrador para más información.</h3>
    </body>
</html>
{% endhighlight %}

Tras ello, guardaremos los cambios y procederemos a generar el fichero para el error **403**, que estará contenido en un fichero de nombre **403.html**, por ejemplo:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/iesgn/error/403.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Prohibido</title>
    </head>
    <body>
        <h2>No tienes permiso para acceder al recurso solicitado (403).</h2>
        <h3>Contacta con el administrador para más información.</h3>
    </body>
</html>
{% endhighlight %}

Para verificar que la creación de dichos ficheros de error se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
├── doc
│   └── symlink.html -> /home/debian/prueba.txt
└── www
    ├── departamentos
    │   └── index.html
    └── iesgn
        ├── error
        │   ├── 403.html
        │   └── 404.html
        └── principal
            ├── index.html
            └── prueba.txt -> /home/debian/prueba.txt

6 directories, 6 files
{% endhighlight %}

Efectivamente, ambos ficheros de error se encuentran correctamente generados dentro de **/srv/www/iesgn/error/**, pero el hecho de que estén generados no implica que _nginx_ vaya a hacer uso de los mismos cuando se produzca un error, pues se lo tendremos que indicar explícitamente, así que podremos proceder a editar el fichero de configuración del VirtualHost, para así llevar a cabo las modificaciones necesarias, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano iesgn
{% endhighlight %}

La sintaxis para definir errores personalizados es:

{% highlight shell %}
error_page <error> <ruta>;
{% endhighlight %}

Siendo:

* **error**: El código del error que queremos gestionar. En este caso, serían **404** y **403**.
* **ruta**: La ruta del fichero que contiene el HTML del error. En este caso, serían **/error/404.html** y **/error/403.html**.

En resumen, la configuración para que haga uso de los ficheros de errores tendrá la siguiente forma:

{% highlight shell %}
error_page 404      /error/404.html;
error_page 403      /error/403.html;
{% endhighlight %}

Dado que queremos que dichos mensajes personalizados de error se muestren en todo el VirtualHost y no para un directorio en concreto, no meteremos dicha configuración en ninguna sección **location**, sino que lo dejaremos indicado en la configuración general, quedando de la siguiente forma:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/iesgn;

        index index.html index.htm index.nginx-debian.html;

        server_name www.iesgn.org;

        error_page 404      /error/404.html;
        error_page 403      /error/403.html;

        location / {
                try_files $uri $uri/ =404;
        }

        rewrite ^/$ /principal redirect;

        location /principal {
                autoindex off;
                disable_symlinks on;
        }

        location /principal/documentos {
                alias /srv/doc;
                autoindex on;
                disable_symlinks if_not_owner;
        }
}
{% endhighlight %}

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

En un principio, la nueva configuración ya está habilitada, así que para comprobar que los nuevos mensajes de error están operativos, vamos a forzar dichos errores. Para el primero de ellos (**404**), vamos a solicitar un recurso que sabemos con certeza que no existe, por ejemplo, **principal/asd**:

![iesgn8](https://i.ibb.co/7jCFb4j/18.jpg "www.iesgn.org/principal/asd")

Efectivamente, el nuevo mensaje de error se ha mostrado correctamente como consecuencia de no haber encontrado el recurso solicitado (**404**). Por último, vamos a forzar un error **403**, solicitando por ejemplo, el enlace simbólico existente en **principal/prueba.txt**, pues si recordamos, dicho directorio tiene deshabilitado el seguimiento de enlaces simbólicos, por lo que va a devolver un error:

![iesgn9](https://i.ibb.co/3N8kfRq/19.jpg "www.iesgn.org/principal/prueba.txt")

Como se puede apreciar, el nuevo mensaje de error se ha mostrado correctamente como consecuencia de no tener permisos para mostrar el recurso solicitado (**403**), ya que se ha deshabilitado el seguimiento de enlaces simbólicos, por lo que podemos concluir que la configuración referente a los mensajes de error personalizados ha funcionado a la perfección.

## Autentificación, autorización y control de acceso

Para este apartado del artículo vamos a crear una nueva máquina virtual en el cloud, que no será accesible desde el exterior, es decir, no tendrá un router virtual haciendo DNAT a la IP del rango privado **10.0.0.0/24**.

Para ello, he creado una máquina virtual en el cloud (**OpenStack**), de las siguientes características:

* **Instance Name**: nginx2
* **Source**: Debian Buster 10.6
* **Sabor**: m1.mini

Además, nos tendremos que asegurar que le asociemos nuestro par de claves usado con anterioridad en la otra máquina, puesto que el primer inicio se tendrá que llevar a cabo mediante un par de claves, pues no tiene ninguna contraseña configurada.

El resultado final deberá ser el siguiente:

![openstack4](https://i.ibb.co/ScPP5jk/24.jpg "Creación instancia")

Como se puede apreciar, tenemos una nueva máquina virtual con dirección IP **10.0.0.13/24**, que es accesible únicamente desde la red privada.

### Ejemplo 1

Para el primer ejemplo, vamos a hacer uso del VirtualHost **departamentos.iesgn.org**, en el cuál crearemos tres directorios:

* **intranet/**: Haremos que únicamente sea accesible desde la red interna **10.0.0.0/24**.
* **internet/**: Haremos que únicamente sea accesible desde la red externa **172.22.0.0/16**.
* **secreto/**: Lo trataremos más adelante.

Para crear dichos directorios dentro del DocumentRoot ejecutaremos el siguiente comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# mkdir /srv/www/departamentos/{intranet,internet,secreto}
{% endhighlight %}

Para verificar que la creación de dichos directorios se ha llevado a cabo correctamente, listaremos el contenido de **/srv/** de forma recursiva y gráfica, haciendo uso del comando `tree`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# tree /srv/
/srv/
├── doc
│   └── symlink.html -> /home/debian/prueba.txt
└── www
    ├── departamentos
    │   ├── index.html
    │   ├── internet
    │   ├── intranet
    │   └── secreto
    └── iesgn
        ├── error
        │   ├── 403.html
        │   └── 404.html
        └── principal
            ├── index.html
            └── prueba.txt -> /home/debian/prueba.txt

9 directories, 6 files
{% endhighlight %}

Efectivamente, los directorios se han generado correctamente, así que para poder hacer la prueba de una manera más visual, vamos a crear ficheros **index.html** que serán servidos automáticamente al acceder a cada uno de los subdirectorios. La estructura de los mismos será muy sencilla, así que empezaremos por generar dicho fichero en el subdirectorio **intranet/**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/departamentos/intranet/index.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Intranet</title>
    </head>
    <body>
        <h2>Red interna.</h2>
    </body>
</html>
{% endhighlight %}

Tras ello, generaremos el fichero en el subdirectorio **internet/**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/departamentos/internet/index.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Internet</title>
    </head>
    <body>
        <h2>Red pública.</h2>
    </body>
</html>
{% endhighlight %}

Por último, generaremos el fichero en el subdirectorio **secreto/**, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano /srv/www/departamentos/secreto/index.html
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Secreto</title>
    </head>
    <body>
        <h2>Esto es un secreto.</h2>
    </body>
</html>
{% endhighlight %}

Ya tenemos todos los ficheros **index.html** generados dentro de los correspondientes directorios, así que iremos accediendo desde el navegador a todos ellos para verificar que el fichero se está sirviendo y mostrando correctamente, empezando por ejemplo, por **departamentos.iesgn.org/intranet**:

![departamentos2](https://i.ibb.co/cQ3z011/25.jpg "departamentos.iesgn.org/intranet")

Como se puede apreciar, el fichero ha sido correctamente servido, gracias a la directiva **index** configurada en el servidor _nginx_, que establece la lista de ficheros que han de servirse por defecto a la hora de acceder a un directorio. La siguiente prueba la haremos para **departamentos.iesgn.org/internet**:

![departamentos3](https://i.ibb.co/t46wLN3/26.jpg "departamentos.iesgn.org/internet")

De nuevo, el fichero ha sido correctamente servido, así que vamos a pasar al último directorio, **departamentos.iesgn.org/secreto**:

![departamentos4](https://i.ibb.co/cyY7z25/27.jpg "departamentos.iesgn.org/secreto")

Genial, todos los ficheros se están sirviendo correctamente, pero todavía no hemos hecho lo que se nos pide, ya que estamos accediendo sin problema desde el navegador de la máquina anfitriona a los tres directorios, y lo que queremos es que únicamente podamos acceder a **internet/** desde la máquina anfitriona y a **intranet/** desde la máquina cliente conectada a la red local (el directorio **secreto/** lo trataremos más adelante).

En este caso, queremos limitar el acceso por IP, así que podremos proceder a editar el fichero de configuración del VirtualHost, para así llevar a cabo las modificaciones necesarias, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano departamentos
{% endhighlight %}

Gracias a la directiva de control de acceso **allow** de _nginx_, podremos filtrar dichas conexiones, y permitir únicamente aquellas que provengan de una determinada red (también podríamos haber especificado la dirección de una máquina concreta en lugar de una red), seguido de la directiva **deny all**, de manera que todos aquellos que no cumplan la anterior condición no tendrán permitido el acceso. Para ello, tendremos que crear una sección **location** para cada uno de los directorios que queramos configurar.

El resultado del fichero sería el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/departamentos;

        index index.html index.htm index.nginx-debian.html;

        server_name departamentos.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }

        location /intranet {
                allow 10.0.0.0/24;
                deny all;
        }

        location /internet {
                allow 172.22.0.0/16;
                deny all;
        }
}
{% endhighlight %}

Como se puede apreciar, el directorio **intranet/** ha sido configurado para ser únicamente accesible desde la red **10.0.0.0/24**, y el directorio **internet/** ha sido configurado para ser únicamente accesible desde la red **172.22.0.0/16**.

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

De nuevo, vamos a realizar las correspondientes pruebas para verificar que el control de acceso se ha configurado correctamente. Lo primero que haremos será tratar de acceder desde el navegador de la máquina anfitriona al directorio **internet/** (recordemos que utiliza la red **172.22.0.0/16**):

![departamentos5](https://i.ibb.co/cy6X2RM/Captura-de-pantalla-de-2020-11-04-13-07-33.png "departamentos.iesgn.org/internet")

Como era de esperar, el recurso ha sido correctamente devuelto, ya que dicho directorio tenía permitido el acceso desde la red **172.22.0.0/16**. Sin embargo, vamos a tratar de acceder esta vez al directorio **intranet/**:

![departamentos6](https://i.ibb.co/ynCxcRY/Captura-de-pantalla-de-2020-11-04-13-07-39.png "departamentos.iesgn.org/intranet")

En este caso, dado que dicho directorio únicamente tiene permitido el acceso desde la red **10.0.0.0/24**, no nos ha sido posible acceder, pues no pertenecemos a dicha red, por lo que nos ha devuelto un error Forbidden (403).

Ya hemos llevado a cabo las correspondientes pruebas en la máquina anfitriona, pero todavía nos queda probar en la máquina cliente, que se encuentra conectada a la red interna. Lo primero que haremos será conectarnos a la misma.

Si recordamos, a la hora de conectarnos a la primera máquina virtual (la que contiene el servidor web _nginx_), hicimos uso de la opción **-A** de SSH. Esto tiene un motivo, y es que dado que la segunda máquina virtual no es accesible desde el exterior, no podremos llevar a cabo una conexión SSH a la misma de forma directa desde la máquina anfitriona, sino que tendremos que utilizar la primera máquina virtual como "puente".

Llegamos a este punto, se nos plantea otro problema, y es que el primer acceso a la máquina virtual ha de llevarse a cabo haciendo uso de un par de claves, el cuál no se encuentra disponible para su uso en la primera máquina virtual, así que gracias a la opción anteriormente mencionada, heredaremos el agente con las claves privadas a dicha máquina virtual, de manera que podremos utilizarlo para conectarnos a otras máquinas que tengan permitido el acceso con dicha clave pública desde la misma.

En este caso, el comando a ejecutar sería (desde la primera máquina virtual):

{% highlight shell %}
debian@nginx:~$ ssh debian@10.0.0.13
Linux nginx2 4.19.0-11-cloud-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Dado que la segunda máquina virtual es una máquina totalmente ajena a la anfitriona, la resolución estática de nombres no se ha configurado todavía, así que tendremos que volver a hacerlo, para así poder realizar la traducción del nombre a la IP de la máquina y que así podamos acceder, gracias a la cabecera Host de la petición HTTP. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@nginx2:~# nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir las líneas anteriormente añadidas al fichero **/etc/hosts** de la máquina anfitriona, pero ésta vez, en lugar de apuntar a la dirección IP **172.22.200.108**, tendremos que apuntar a la dirección IP de la interfaz de red de la máquina servidora perteneciente a la red interna, es decir, la **10.0.0.3**, ya que de lo contrario no sería accesible, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.1.1 nginx2.novalocal nginx2
127.0.0.1 localhost
10.0.0.3  www.iesgn.org
10.0.0.3  departamentos.iesgn.org

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
{% endhighlight %}

Listo. La resolución estática de nombres ha sido correctamente configurada, pero dado que la máquina cliente no tiene interfaz gráfica, tendremos que hacer uso de un paquete que simula un navegador en la terminal, para así hacerlo más similar a la realidad. En este caso, haremos uso de **elinks**, paquete que debemos instalar, no sin antes upgradear los paquetes instalados, ya que la máquina virtual se va quedando desactualizada con el tiempo. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@nginx2:~# apt update && apt upgrade && apt install elinks
{% endhighlight %}

Tras ello, únicamente tendremos que ejecutar el comando `elinks` seguido de la dirección del sitio al que nos queremos conectar, en este caso, empezaremos por acceder al directorio **intranet/**:

{% highlight shell %}
root@nginx2:~# elinks departamentos.iesgn.org/intranet
{% endhighlight %}

![departamentos7](https://i.ibb.co/dWYn3XG/Captura-de-pantalla-de-2020-11-04-13-24-57.png "departamentos.iesgn.org/intranet")

Como era de esperar, el recurso ha sido correctamente devuelto, ya que dicho directorio tenía permitido el acceso desde la red **10.0.0.0/24**. Sin embargo, vamos a tratar de acceder esta vez al directorio **internet/**, haciendo uso de la sintaxis anteriormente mencionada:

{% highlight shell %}
root@nginx2:~# elinks departamentos.iesgn.org/internet
{% endhighlight %}

![departamentos8](https://i.ibb.co/8dZgYyp/Captura-de-pantalla-de-2020-11-04-13-25-27.png "departamentos.iesgn.org/internet")

En este caso, dado que dicho directorio únicamente tiene permitido el acceso desde la red **172.22.0.0/16**, no nos ha sido posible acceder, pues no pertenecemos a dicha red, por lo que nos ha devuelto un error Forbidden (403).

### Ejemplo 2

Para el siguiente ejemplo, vamos a configurar una autentificación básica para el directorio **secreto/**, de manera que nos pida un usuario y una contraseña para mostrar el contenido del mismo.

Para ello, tendremos que crear una sección **location** en la máquina servidora para el nuevo directorio que vamos a configurar, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano departamentos
{% endhighlight %}

En este caso, vamos a hacer uso de dos directivas para la **autentificación básica** del mismo:

* **auth_basic**: Le damos un nombre al área protegida con contraseña, además de habilitarla. En caso de poner **off**, la deshabilitaríamos.
* **auth_basic_user_file**: Indicaremos la ruta del fichero que almacenará las credenciales de acceso de los usuarios. Debe estar fuera del DocumentRoot, para que no sea accesible.

El resultado del fichero sería el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/departamentos;

        index index.html index.htm index.nginx-debian.html;

        server_name departamentos.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }

        location /intranet {
                allow 10.0.0.0/24;
                deny all;
        }

        location /internet {
                allow 172.22.0.0/16;
                deny all;
        }

        location /secreto {
                auth_basic      "Acceso de administrador";
                auth_basic_user_file    /etc/nginx/.htpasswd;
        }
}
{% endhighlight %}

Como se puede apreciar, el fichero que almacenará las credenciales de acceso se encuentra en **/etc/nginx/.htpasswd** y el texto que se mostrará en la ventana emergente será "**Acceso de administrador**".

En realidad, el fichero que contendrá las credenciales de acceso todavía no se encuentra generado. En este caso, haremos uso de la utilidad `htpasswd`, que se encuentra contenida en el paquete **apache2-utils**, el cuál debemos instalar previamente, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# apt install apache2-utils
{% endhighlight %}

Para hacer uso de la utilidad, tendremos que indicar la ruta en cuestión y el usuario que queremos añadir. En este caso, crearé un usuario "**prueba**", cuya contraseña será "**password123**", ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# htpasswd -c /etc/nginx/.htpasswd prueba
New password: 
Re-type new password: 
Adding password for user prueba
{% endhighlight %}

Donde:

* **-c**: Indica que se lleve a cabo la creación del fichero de credenciales. Únicamente debemos hacer uso de ésta opción la primera vez, pues de lo contrario, sobreescribirá el fichero existente.

Por curiosidad, vamos a comprobar el contenido de dicho fichero generado, haciendo uso de `cat`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# cat /etc/nginx/.htpasswd 
prueba:$apr1$FRflZ9BZ$2fFLCS6GsDivacZDC5mGa/
{% endhighlight %}

Como se puede apreciar, existe un único usuario **prueba** cuya contraseña se encuentra encriptada como resultado de la aplicación de una función hash con algoritmo MD5. En caso de querer eliminar un usuario, es decir, hacer que deje de tener acceso, bastaría con eliminar la línea correspondiente al mismo.

Dado que anteriormente hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

Tras ello, trataremos de acceder al directorio **secreto/** desde el navegador de la máquina anfitriona (por ejemplo, aunque desde la máquina cliente funcionaría exactamente de la misma forma):

![departamentos9](https://i.ibb.co/DQ6SNnQ/Captura-de-pantalla-de-2020-11-04-13-32-13.png "departamentos.iesgn.org/secreto")

Tal y como era de esperar, el navegador ha mostrado una ventana emergente solicitando las credenciales de acceso. Tras introducir el usuario **prueba** y la contraseña **password123**, pulsaremos en **Iniciar sesión** y veremos que hemos podido acceder correctamente.

![departamentos10](https://i.ibb.co/cyY7z25/27.jpg "departamentos.iesgn.org/secreto")

Realmente, la parte importante que quería mostrar no era cómo acceder, sino lo que ocurre durante dicho acceso. Para ello, he dejado **Wireshark** capturando paquetes con un filtro **HTTP** en segundo plano, para posteriormente analizar con detalle qué es lo que ha ocurrido. En Internet se puede encontrar documentación sobre cómo descargar e instalar Wireshark y capturar paquetes usando un determinado filtro, pues dicha explicación se saldría del objetivo de este _post_. Estos son los paquetes capturados:

![departamentos11](https://i.ibb.co/1GHfrHX/wireshark.jpg "Captura Wireshark")

Explicar con detalle todo el proceso de negociación entre ambas máquinas sería algo extenso, así que voy a tratar de resumirlo de la mejor manera posible:

* **Paquete 1**: La máquina anfitriona hace una petición GET a la máquina servidora solicitando el recurso **/secreto/**.
* **Paquete 2**: La máquina servidora le devuelve una respuesta Unauthorized (401) ya que el recurso está protegido, por lo que le pide las credenciales de acceso.
* **Paquete 3**: La máquina anfitriona envía las credenciales introducidas en la ventana emergente a la máquina servidora, volviendo a solicitar el mismo recurso.
* **Paquete 4**: La máquina servidora verifica que las credenciales sean correctas, y tras ello, devuelve el recurso solicitado (en este caso, el **index.html**), devolviendo un código de estado 200.
* **Paquete 5**: La máquina anfitriona desea mostrar todo el contenido posible, así que solicita también el favicon, indicando de nuevo las credenciales de acceso.
* **Paquete 6**: La máquina servidora devuelve un código de estado 404 ya que no ha encontrado el recurso solicitado.

Esa sería la explicación superficial de todo lo que ha ocurrido durante este proceso, pero realmente, lo importante se encuentra en el **Paquete 3**, cuando se envían las credenciales al servidor. Si nos volvemos a fijar en la imagen anteriormente mostrada, en la cabecera de la petición existe un apartado **Authorization**. Dentro del mismo, se encuentra el **usuario** y la **contraseña** en "texto plano". Lo pongo entre comillas porque no va en texto plano como tal, sino usando un sistema de numeración posicional **base64**, pero que realmente no nos impide nada, ya que lo podemos descifrar haciendo uso de cualquier [utilidad](https://www.base64decode.org/) (aunque Wireshark ya lo ha hecho automáticamente, en el apartado **Credentials**):

![departamentos12](https://i.ibb.co/KqYQ7Kk/decode.jpg "Decode BASE64")

Como podemos apreciar, este método de autentificación es poco seguro. Es por ello, que tenemos (al menos) dos alternativas. La primera de ellas sería utilizar _nginx_ con SSL (_Secure Sockets Layer_), de manera que permite cifrar el tráfico de datos entre el navegador web y el sitio web (o entre dos servidores web), protegiendo así la conexión. De otro lado, podríamos usar un método de autentificación más seguro, como _digest_.

### Ejemplo 3

Para el último ejemplo, vamos a combinar el control de acceso y la autentificación, de manera que a dicho directorio **secreto/**, se pueda acceder de forma directa desde la intranet (**10.0.0.0/24**) y desde internet pida autentificarse haciendo uso del fichero anteriormente generado. Para ello, tendremos que hacer uso de las políticas de acceso de _nginx_, concretamente de **satisfy any**, ya que debe cumplirse al menos una de las dos condiciones existentes en el bloque. En caso de necesitar que se cumpliesen todas, haríamos uso de **satisfy all**, pero no es el caso.

Para llevar a cabo la configuración, tendremos que modificar el fichero de configuración del VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# nano departamentos
{% endhighlight %}

El resultado del fichero sería el siguiente:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        root /srv/www/departamentos;

        index index.html index.htm index.nginx-debian.html;

        server_name departamentos.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }

        location /intranet {
                allow 10.0.0.0/24;
                deny all;
        }

        location /internet {
                allow 172.22.0.0/16;
                deny all;
        }

        location /secreto {
                satisfy any;

                allow 10.0.0.0/24;
                deny all;

                auth_basic      "Acceso de administrador";
                auth_basic_user_file    /etc/nginx/.htpasswd;
        }
}
{% endhighlight %}

Dado que hemos modificado un fichero de configuración de un servicio, tendremos que volver a cargar la configuración del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

Tras ello, trataremos de acceder al directorio **secreto/** desde el navegador de la máquina anfitriona (conexión que se llevaría a cabo desde la red **172.22.0.0/16**, por lo que debería solicitar las credenciales de acceso):

![departamentos13](https://i.ibb.co/DQ6SNnQ/Captura-de-pantalla-de-2020-11-04-13-32-13.png "departamentos.iesgn.org/secreto")

Como era de esperar, dado que el acceso se ha realizado desde la red externa, se han solicitado las credenciales, así que las introduciremos y veremos si se muestra el contenido:

![departamentos14](https://i.ibb.co/cyY7z25/27.jpg "departamentos.iesgn.org/secreto")

Efectivamente, así ha sido. Todavía nos queda ver la otra _cara de la moneda_, así que trataremos de acceder con **elinks** desde la segunda máquina virtual (conexión que se llevaría a cabo desde la red **10.0.0.0/24**, por lo que no debería solicitar las credenciales de acceso):

{% highlight shell %}
debian@nginx2:~$ elinks departamentos.iesgn.org/secreto
{% endhighlight %}

![departamentos15](https://i.ibb.co/1qzbJFD/Captura-de-pantalla-de-2020-11-04-13-41-57.png "departamentos.iesgn.org/secreto")

Como era de esperar, dado que el acceso se ha realizado desde la red interna, no se han solicitado las credenciales, mostrándose el contenido directamente. Por lo tanto, podemos asegurar que la configuración ha funcionado correctamente.

Por último, me gustaría dejar una pequeña anotación. Tal y como hemos mencionado anteriormente, el usuario a través del cuál se sirven las páginas web en _nginx_ es **www-data** (en este caso no habría inconveniente ya que el grupo "otros", al que pertenece dicho usuario tiene permisos de **lectura**, que es lo único que necesitamos para servir el fichero, pero en caso de necesitar escribir en dichos ficheros, no contaría con dichos permisos de **escritura**), por lo que podemos proceder a cambiar dicho propietario y grupo de forma recursiva, tal y como hemos hecho con anterioridad, para que así afecte a los nuevos ficheros generados, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@nginx:/etc/nginx/sites-available# chown -R www-data:www-data /srv/*
{% endhighlight %}

Gracias al comando ejecutado con anterioridad, todos los directorios y ficheros hijos de **/srv** serían ahora propiedad de **www-data** y haciendo uso del grupo **www-data**.