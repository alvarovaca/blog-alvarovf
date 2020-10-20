---
layout: post
title:  "Mapear URL en Apache"
banner: "/assets/images/banners/mapear.jpg"
date:   2020-10-20 12:32:00 +0200
categories: servicios
---
Este _post_ es una continuación de [VirtualHosting con Apache](http://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html), por lo que es recomendable realizar su previa lectura, pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Tarea 1: Crea un nuevo host virtual que es accedido con el nombre www.mapeo.com, cuyo DocumentRoot sea /srv/mapeo.

En esta tarea se pide que el DocumentRoot sea **/srv/mapeo** y si recordamos, en el anterior _post_, configuramos la sección **Directory** para incluir los DocumentRoot de los diferentes VirtualHost dentro de **/srv/www**. Dadas dichas circunstancias, los nuevos DocumentRoot los vamos a seguir alojando dentro de **/srv**, pero no dentro de **/srv/www**, por lo que tendremos que realizar la modificación correspondiente en dicho fichero de configuración.

En este caso, lo que debemos hacer es subir un nivel de directorios (es decir, en lugar de configurar el Directory para **/srv/www**, configurarlo para **/srv**), de manera que dichos permisos que se configuren se heredarán a todos los hijos de dicho directorio padre.

Para ello, debemos modificar el fichero de configuración **/etc/apache2/apache2.conf**, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
root@apache:~# nano /etc/apache2/apache2.conf
{% endhighlight %}

Dentro del mismo, tendremos que buscar la sección **Directory** previamente configurada:

{% highlight shell %}
<Directory /srv/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

Como se puede apreciar, el directorio afectado es **/srv/www/**, y por tanto, todos sus directorios hijos heredan dichos permisos, pero no nos serviría para alojar un DocumentRoot en otro directorio que no sea hijo de **/srv/www/**, por lo que cambiaremos dicho directorio por **/srv/**, quedando de la siguiente forma:

{% highlight shell %}
<Directory /srv/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

Tras ello, guardaremos los cambios y podremos proceder a cargar la nueva configuración del servicio _apache2_, ejecutando para ello el comando (no es necesario reiniciar el servicio al completo):

{% highlight shell %}
root@apache:~# systemctl reload apache2
{% endhighlight %}

La nueva configuración ya se encuentra cargada, así que podemos proceder a crear el nuevo VirtualHost (estos pasos los llevaré a cabo explicándolo todo de una manera más superficial, ya que como se ha mencionado anteriormente, se ha tratado con profundidad en el anterior _post_). El primer paso será crear el DocumentRoot para el mismo, que tal y como se pide, será en **/srv/mapeo**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:~# mkdir /srv/mapeo
{% endhighlight %}

Para verificar que dicho directorio se ha creado junto al directorio **www/** previamente existente, haremos uso del comando `ls`:

{% highlight shell %}
root@apache:~# ls /srv/
mapeo  www
{% endhighlight %}

Efectivamente, así ha sido, por lo que únicamente nos queda crear el nuevo fichero de configuración dentro de **/etc/apache2/sites-available/**, así que nos moveremos a dicho directorio ejecutando el comando:

{% highlight shell %}
root@apache:~# cd /etc/apache2/sites-available/
{% endhighlight %}

Una vez dentro del mismo, vamos a utilizar como plantilla el fichero de configuración del VirtualHost que trae _apache2_ configurado por defecto, **000-default.conf**, así que copiaremos dicho fichero a uno de nombre **mapeo.conf** (por ejemplo, aunque se podría haber usado cualquier otro nombre). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# cp 000-default.conf mapeo.conf
{% endhighlight %}

Para verificar que el nuevo fichero de configuración se ha generado correctamente, volveremos a ejecutar el comando `ls`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls
000-default.conf  default-ssl.conf  departamentos.conf	iesgn.conf  mapeo.conf
{% endhighlight %}

Como era de esperar, el fichero se encuentra generado, así que lo único que queda es realizar las modificaciones oportunas al mismo, tales como indicar **www.mapeo.com** como **ServerName** y **/srv/mapeo** como **DocumentRoot**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano mapeo.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
{% endhighlight %}

Esa sería la apariencia del fichero de configuración hasta ahora, así que vamos a pasar al siguiente ejercicio para seguir realizando las modificaciones oportunas.

## Tarea 2: Crea un alias en el host virtual del ejercicio anterior que permita entrar en la URL http://www.mapeo.com/documentos y visualice los ficheros del /home/usuario/Documentos. En la sección Directory pon las mismas directivas que tiene la sección Directory del fichero /etc/apache2/apache2.conf.

Los alias de _apache2_ nos permiten mapear URLs a ubicaciones de un sistema de ficheros, o en otras palabras, nos permite que el servidor nos sirva ficheros que no estén necesariamente situados dentro del DocumentRoot.

Extrapolando dicha definición a este ejemplo, podemos hacer que al acceder a **/documentos**, nos muestre el contenido que existe dentro de **/home/vagrant/Documentos**, que como se puede intuir, no se encuentra dentro del DocumentRoot, pero aun así, podemos hacer que sea accesible.

Existe un requisito indispensable para poder crear dichos alias, y es que se necesita dar permisos al directorio al que estamos apuntando, ya que por defecto, hay un Directory que es el padre de todos (contenido en **/etc/apache2/apache2.conf**), que tiene el acceso denegado a **/**, por lo que afecta de forma recursiva a todos sus hijos, es decir, a todos los directorios y ficheros existentes en el sistema. Dicha configuración se puede contrarrestar especificando unos permisos sobre un directorio hijo concreto (tal y como hemos estado haciendo hasta ahora), ya que los permisos sobre un directorio concreto tienen prioridad sobre aquellos que se heredan, es decir, si no hemos creado una sección **Directory** para **/home/vagrant/Documentos**, dicho directorio heredará los permisos del padre (**/**), mientras que sí hemos creado una sección Directory concreta para el mismo, los permisos especificados ahí tienen prioridad que aquellos que pudiese heredar del padre.

La sintaxis para crear un alias es:

{% highlight shell %}
Alias "<pathURL>" "<pathFS>"
{% endhighlight %}

Siendo:

* **pathURL**: El recurso "virtual" que solicitaremos en la URL a la hora de hacer la petición al servidor. En este caso, la petición la haríamos a www.mapeo.com**/documentos**, por lo que pathURL sería **/documentos**.
* **pathFS**: La ruta real a la que se va a mapear el pathURL, es decir, la ruta del sistema de ficheros a la que deseamos apuntar. En este caso, el pathFS sería **/home/vagrant/Documentos**, pues es donde deseamos apuntar.

En resumen, el alias que queremos crear tendrá la siguiente forma:

{% highlight shell %}
Alias "/documentos" "/home/vagrant/Documentos"
{% endhighlight %}

Además, tal y como se ha mencionado antes, tendremos que dar permisos a dicho directorio al que apuntamos (crear un nuevo apartado **Directory**, pero esta vez, dentro del fichero de configuración del VirtualHost, de manera que dicho directorio sea accesible únicamente por este VirtualHost y no por el resto), además, en este caso se ha pedido que dichas directivas sean las mismas que la sección **Directory** existente en el fichero **/etc/apache2/apache2.conf**, quedando de la siguiente manera:

{% highlight shell %}
<Directory "/home/vagrant/Documentos">
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
{% endhighlight %}

Tras ello, tendremos que añadir ambos bloques al fichero de configuración del VirtualHost (haciendo uso de `nano`), quedando de la siguiente manera:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        Alias "/documentos" "/home/vagrant/Documentos"
        <Directory "/home/vagrant/Documentos">
                Options Indexes FollowSymLinks
                AllowOverride None
                Require all granted
        </Directory>
</VirtualHost>
{% endhighlight %}

Antes de volver a cargar la nueva configuración, tendremos que asegurarnos que el directorio al que estamos apuntando en el alias realmente exista, ya que en este caso, la box de Vagrant no trae un directorio de nombre **Documentos** dentro del **home** de **Vagrant**. La solución es sencilla, crearemos manualmente el directorio, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# mkdir /home/vagrant/Documentos
{% endhighlight %}

Una vez creado, vamos a generar un fichero con un pequeño texto en su interior (de nombre **prueba.txt**, por ejemplo) para posteriormente poder comprobar que el alias está funcionando, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# echo "Esto es una prueba." > /home/vagrant/Documentos/prueba.txt
{% endhighlight %}

Tras ello, ya podremos habilitar el VirtualHost para verificar si funciona o no, haciendo uso del comando `a2ensite`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# a2ensite mapeo
Enabling site mapeo.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Una vez activado el VirtualHost, tendremos que volver a cargar la configuración del servicio, tal y como se ha solicitado, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Genial, el VirtualHost ya debería estar en funcionamiento, así que para comprobarlo, vamos a listar el contenido de **/etc/apache2/sites-enabled/** para verificar que se ha generado el correspondiente enlace simbólico al fichero de configuración en **/etc/apache2/sites-available/**, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls -l /etc/apache2/sites-enabled/
total 0
lrwxrwxrwx 1 root root 35 Oct 15 07:47 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root 37 Oct 15 08:00 departamentos.conf -> ../sites-available/departamentos.conf
lrwxrwxrwx 1 root root 29 Oct 15 08:00 iesgn.conf -> ../sites-available/iesgn.conf
lrwxrwxrwx 1 root root 29 Oct 18 07:28 mapeo.conf -> ../sites-available/mapeo.conf
{% endhighlight %}

Efectivamente, el enlace simbólico se ha generado correctamente, así que antes de acceder desde el navegador, tendremos que añadir la correspondiente línea al **/etc/hosts** de la máquina anfitriona para que realice la resolución estática de nombres, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

La dirección IP de la máquina servidora es **192.168.50.2** y el nombre de dominio al que apuntar, **www.mapeo.com**, por lo que el resultado sería el siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.50.2    www.iesgn.org
192.168.50.2    www.departamentosgn.org
192.168.50.2    www.mapeo.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Listo, ya está todo listo para probar a acceder al servidor desde el navegador para ver si el alias funciona correctamente, así que trataremos de acceder a **www.mapeo.com/documentos**:

![alias](https://i.ibb.co/dQBGdjr/Captura-de-pantalla-de-2020-10-18-09-41-52.png "Alias")

Como se puede apreciar, nos ha listado el contenido de **/home/vagrant/Documentos** pero accediendo a través de un alias, por lo que podemos asegurar que funciona correctamente.

## Tarea 3: Determina para que sirven las siguientes opciones de funcionamiento:

* **FollowSymLinks**: Permite seguir un enlace simbólico existente dentro del DocumentRoot. No se suele utilizar. No es necesario crear un Directory para el recurso al que apuntamos.
* **SymLinksIfOwnerMatch**: Similar a la opción anterior pero en este caso, el enlace únicamente es válido si el propietario del fichero/directorio al que apunta es el mismo que el propietario del enlace simbólico.
* **Indexes**: En caso de acceder a un directorio y no existir ningún recurso predeterminado (especificado en la directiva **DirectoryIndex**, como por ejemplo el index.html), en lugar de mostrar un error 404, el módulo **mod_autoindex** mostrará una lista de los recursos disponibles en dicho directorio. Se suele deshabilitar por seguridad.
* **MultiViews**: Habilita la negociación de contenidos mediante el módulo **mod_negotiation**, es decir, dependiendo de algunas configuraciones del cliente, el servidor servirá un contenido u otro. Un ejemplo que se ve muy claro es cuando tienes el navegador configurado en un determinado idioma y el servidor te devuelve el recurso en ese mismo idioma. Actualmente no tiene mucho sentido su uso, ya que dicha funcionalidad se puede hacer a nivel de aplicación.
* **ExecCGI**: Cuando no existían lenguajes de programación, se hacía un programa compilado (conocido como programa CGI o script CGI) que se ponía en el servidor, el cuál era el encargado de recoger información de programas externos de generación de contenido y procesar la petición gracias al módulo **mod_cgi**, con la finalidad de mostrar contenido dinámico en el sitio web. Hoy en día está en desuso.
* **All**: Habilita todas las opciones excepto MultiViews. Es la opción por defecto.

Tal y como se ha mencionado anteriormente, todas las opciones que indiquemos en una sección Directory afectarán a dicho directorio y a todos sus hijos, por lo que los DocumentRoot de los VirtualHost que creemos deben estar dentro de dicho Directory, de manera que sean afectados de forma recursiva.

En caso de querer usar una configuración específica para un DocumentRoot determinado, tendremos que crear un apartado Directory en concreto para dicho DocumentRoot, cuyas opciones tendrán prioridad sobre aquellas heredadas, y podremos elegir entre indicar unas opciones que sobreescriban las del padre o bien, usar unas opciones relativas, es decir, realizar algunas modificaciones sustanciales sobre las heredadas (haciendo uso de los operadores **+** y **-**), pudiendo añadir nuevas opciones o quitar las que no queramos.

Por ejemplo, supongamos que tenemos el siguiente apartado **Directory** en el fichero **/etc/apache2/apache2.conf**:

{% highlight shell %}
<Directory /srv/>
        Options Indexes FollowSymLinks
        ...
</Directory>
{% endhighlight %}

Un ejemplo de sección Directory que sobreescribiría aquellas opciones heredadas por el padre sería:

{% highlight shell %}
<Directory /srv/prueba1/>
        Options MultiViews SymLinksIfOwnerMatch
        ...
</Directory>
{% endhighlight %}

Un ejemplo de sección Directory que modificaría sustancialmente aquellas opciones heredadas por el padre sería:

{% highlight shell %}
<Directory /srv/prueba1/>
        Options -Indexes +MultiViews
        ...
</Directory>
{% endhighlight %}

### Crea un enlace directo dentro de /home/usuario/Documentos y comprueba si es posible seguirlo. Cambia las opciones del directorio para que no siga los enlaces símbolicos.

En este caso, voy a crear un enlace simbólico de nombre **link.html** que apunte al fichero de texto **prueba.txt** anteriormente creado. Para ello haremos uso de la sintaxis:

{% highlight shell %}
ln -s <rutafichero> <nombreenlace>
{% endhighlight %}

Siendo:

* **-s**: Indica que crear un enlace simbólico o _softlink_ (ya que por defecto crea enlaces duros o _hardlinks_).
* **rutafichero**: Ruta del fichero al que queremos apuntar.
* **nombreenlace**: Nombre del enlace que queremos crear.

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ln -s /home/vagrant/Documentos/prueba.txt /home/vagrant/Documentos/link.html
{% endhighlight %}

Tras ello, vamos a recargar la página del navegador para ver si el enlace simbólico ha aparecido:

![followsymlinks1](https://i.ibb.co/qWNPq0B/Captura-de-pantalla-de-2020-10-18-09-48-17.png "Opción FollowSymLinks")

Como se puede apreciar, dicho recurso es ahora visible en la estructura del directorio devuelta, por lo que pulsaremos en él para ver si redirige al fichero especificado:

![followsymlinks2](https://i.ibb.co/M22hk0V/Captura-de-pantalla-de-2020-10-18-09-48-22.png "Opción FollowSymLinks")

Efectivamente, el enlace simbólico ha redirigido correctamente al fichero **prueba.txt** creado con anterioridad y el contenido del mismo es totalmente visible, gracias a la opción **FollowSymLinks** de la sección **Directory** que afecta a **/home/vagrant/Documentos**.

Para deshabilitar dicha funcionalidad, tendremos que volver a acceder al fichero de configuración del VirtualHost (**mapeo.conf**) haciendo uso del comando `nano` y eliminar la opción **FollowSymLinks**, quedando de la siguiente forma:

{% highlight shell %}
<Directory "/home/vagrant/Documentos">
    Options Indexes
    AllowOverride None
    Require all granted
</Directory>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez recargada la configuración, trataremos de recargar la página del navegador que tiene abierto el recurso del enlace simbólico (puede que sea necesario hacer uso de **CTRL + F5** para limpiar la caché del navegador, ya que puede ser que no esté haciendo la petición al servidor, sino cargando el contenido almacenado en caché):

![followsymlinks3](https://i.ibb.co/0srb80B/Captura-de-pantalla-de-2020-10-18-09-51-23.png "Opción FollowSymLinks")

Como se puede apreciar, nos ha devuelto un error 403, que indica que no tenemos permiso para acceder a dicho recurso (Forbidden), pues la opción que permitía seguir los enlaces simbólicos ha sido deshabilitada.

### Deshabilita la opción de que se listen los archivos existentes en la carpeta cuando no existe un fichero definido en la directiva DirectoryIndex.

La directiva **DirectoryIndex** establece la lista de recursos que se deben buscar cuando el cliente solicita el índice de un directorio del sitio, siguiendo un determinado orden de prioridad, por ejemplo:

{% highlight shell %}
DirectoryIndex index.php index.html
{% endhighlight %}

Esta directiva haría que al acceder al directorio, primero se busque el recurso de nombre **index.php** y en caso de no existir, se busque el **index.html**. ¿Pero qué ocurre si no existe ninguno de los ficheros indicados en la lista? Bien, pueden ocurrir dos cosas:

* Que se encuentre habilitada la opción **Indexes**, por lo que se mostrará una lista de los recursos disponibles en dicho directorio.
* Que no se encuentre habilitada la opción **Indexes**, por lo que se devolverá un error 404 indicando que el recurso solicitado no ha sido encontrado.

En este momento, la opción **Indexes** se encuentra habilitada, y es la causante de que se listen dichos archivos existentes, por lo que para deshabilitar dicha funcionalidad, tendremos que volver a acceder al fichero de configuración del VirtualHost (**mapeo.conf**) haciendo uso del comando `nano` y eliminar la opción **Indexes**, quedando de la siguiente forma:

{% highlight shell %}
<Directory "/home/vagrant/Documentos">
    Options None
    AllowOverride None
    Require all granted
</Directory>
{% endhighlight %}

Muy importante prestar atención al detalle de que en lugar de eliminar la línea **Options** o dejarla vacía, he especificado la opción **None**. Esto se debe a que en caso de no indicar nada, se habilitaría automáticamente la opción **All**, al estar marcada como "por defecto". Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez recargada la configuración, trataremos de recargar la página del navegador que tiene abierto el directorio:

![indexes1](https://i.ibb.co/R6HYBrF/Captura-de-pantalla-de-2020-10-18-09-55-43.png "Opción Indexes")

Como se puede apreciar, nos ha devuelto un error 403, que indica que no tenemos permiso para acceder a dicho recurso (Forbidden), pues la opción que permitía listar el los recursos existentes en el directorio ha sido deshabilitada.

Sin embargo, si en la URL solicitamos explícitamente el recurso **prueba.txt**, por ejemplo, el recurso sigue siendo accesible, pues lo único que ha variado es que ahora no se muestran listados los recursos existentes en caso de que no exista ningún fichero de los indicados en la directiva **DirectoryIndex**:

![indexes2](https://i.ibb.co/M6H1WDm/Captura-de-pantalla-de-2020-10-18-09-55-49.png "Opción Indexes")

### Realiza un fichero de bienvenida en Español e Inglés y comprueba como se visualiza haciendo uso de la directiva MultiViews.

En _apache2_ existe un fichero de configuración global de negociación de contenido (**/etc/apache2/mods-available/negotiation.conf**), pero como en este caso queremos hacerlo de forma concreta en cada VirtualHost, vamos a comentar todo el contenido de dicho fichero, para así poder hacerlo de una forma más concreta y "granular". Para ello, ejecutaremos el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /etc/apache2/mods-available/negotiation.conf
{% endhighlight %}

Tras ello, ya podremos generar los ficheros que se servirán en cada uno de los idiomas. En este caso, vamos a generar un fichero index.html para el idioma **Español**, con su correspondiente extensión .es (**index.html.es**) y otro para el idioma **Inglés**, con su correspondiente extensión .en (**index.html.en**). Ambos ficheros tendrán una estructura muy sencilla, y se encontrarán alojados dentro de **/home/vagrant/Documentos**. El primero que voy a crear es el fichero en Español, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /home/vagrant/Documentos/index.html.es
{% endhighlight %}

El contenido es el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Español</title>
    </head>
    <body>
        <h2>Bienvenidos a la web en Español.</h2>
    </body>
</html>
{% endhighlight %}

Tras ello, crearé el fichero en Inglés, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /home/vagrant/Documentos/index.html.en
{% endhighlight %}

El contenido es el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>English</title>
    </head>
    <body>
        <h2>Welcome to the website in English.</h2>
    </body>
</html>
{% endhighlight %}

Para verificar que ambos ficheros se han generado correctamente, vamos a listar el contenido de **/home/vagrant/Documentos** haciendo uso del comando `ls`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# ls /home/vagrant/Documentos
index.html.en  index.html.es  link.html  prueba.txt
{% endhighlight %}

El último paso que nos falta para poder utilizar dicha funcionalidad es activar la opción **MultiViews**, la cuál permite que en caso de no existir un recurso solicitado (por ejemplo **index.html**) en el directorio, se busquen automáticamente todos los recursos que tengan dicho nombre pero con una extensión (por ejemplo **index.html.es** o **index.html.en**).

Para habilitar dicha funcionalidad, tendremos que volver a acceder al fichero de configuración del VirtualHost (**mapeo.conf**) haciendo uso del comando `nano` y añadir la opción **MultiViews**, quedando de la siguiente forma:

{% highlight shell %}
<Directory "/home/vagrant/Documentos">
    Options MultiViews
    AllowOverride None
    Require all granted
</Directory>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez recargada la configuración, vamos a acceder a la configuración del navegador para configurar el idioma del mismo y así verificar que funciona. Para acceder a la configuración de idioma en **Firefox**, tendremos que irnos a las **3 barras** que se encuentran arriba a la derecha, para posteriormente pulsar en **Preferencias** y buscamos el apartado **Idioma y apariencia**. En este caso, la configuración que tengo es la siguiente:

![multiviews1](https://i.ibb.co/7JDcnDv/Captura-de-pantalla-de-2020-10-18-10-31-06.png "Opción MultiViews")

El idioma configurado es el **Español**, por lo que si tratamos de acceder a **www.mapeo.com/documentos** debería devolver el fichero **index.html.es**:

![multiviews2](https://i.ibb.co/k6RgpM8/Captura-de-pantalla-de-2020-10-18-10-36-36.png "Opción MultiViews")

Como se puede apreciar, nos ha devuelto correctamente dicho fichero, coincidente con el idioma que tenemos configurado en el navegador, gracias a la negociación de contenidos. Para poder mostrar de una forma un poco más profunda y detallada el funcionamiento de la negociación, he dejado **Wireshark** capturando paquetes con un filtro **HTTP** en segundo plano, para posteriormente analizar con detalle qué es lo que ha ocurrido. En Internet se puede encontrar documentación sobre cómo descargar e instalar Wireshark y capturar paquetes usando un determinado filtro, pues dicha explicación se saldría del objetivo de este _post_. Estas son las cabeceras de la petición HTTP realizada:

![multiviews3](https://i.ibb.co/H4yk0RW/Captura-de-pantalla-de-2020-10-18-10-36-39.png "Opción MultiViews")

Una de las cabeceras que podemos apreciar es la cabecera **Host**, que contiene el nombre de dominio del servidor al que hemos hecho la petición, el **User-Agent**, que es el navegador con el que hemos accedido... pero la cabecera que realmente nos importa es aquella de nombre **Accept-Language**, que contiene las preferencias de lenguaje por parte del navegador, que son aquellas que hemos indicado con anterioridad. Como se puede ver, el lenguaje más preferente es el Español, seguido del Inglés.

Vamos a volver a hacer la prueba pero esta vez, modificando el idioma del navegador al **Inglés**, quedando de la siguiente forma:

![multiviews4](https://i.ibb.co/wSKnFRh/Captura-de-pantalla-de-2020-10-18-10-37-30.png "Opción MultiViews")

El idioma configurado es el **Inglés**, por lo que si tratamos de acceder a **www.mapeo.com/documentos** debería devolver el fichero **index.html.en**:

![multiviews5](https://i.ibb.co/P6kd6Rs/Captura-de-pantalla-de-2020-10-18-10-37-34.png "Opción MultiViews")

Como se puede apreciar, nos ha devuelto correctamente dicho fichero, coincidente con el idioma que tenemos configurado en el navegador, gracias a la negociación de contenidos. De nuevo, volveremos a ver las cabeceras de la petición HTTP capturada por **Wireshark**:

![multiviews6](https://i.ibb.co/ncTnvPf/Captura-de-pantalla-de-2020-10-18-10-37-56.png "Opción MultiViews")

En este caso, la cabecera **Accept-Language** tiene el Inglés como el lenguaje con mayor preferencia, explicando por tanto por qué nos ha devuelto el **index.html** en Inglés en lugar de hacerlo en Español.

## Tarea 4: Realiza una redirección que permita que cuando se acceda al servidor http://www.mapeo.com, salte a http://www.mapeo.com/web.

Las redirecciones de _apache2_ nos permiten pedir al cliente que haga otra petición a una URL diferente (indicada en la cabecera de la respuesta HTTP junto a un código de estado **3xx**). Se suelen usar cuando el recurso al que intentamos acceder ha cambiado de localización.

Existen las redirecciones **Permanentes** (301) y **Temporales** (302) (son las que se usan por defecto, en caso de no indicar lo contrario). Hay que tener cuidado con las cachés de los navegadores ya que puede ser que se aprendan dichas redirecciones para así evitar preguntar al servidor nuevamente.

Podemos redireccionar a una URL en un **host diferente** (deberá ser **absoluta**, como por ejemplo de **"/service"** a **"http://foo2.example.com/service"**) o a una URL dentro del **mismo host** (deberá comenzar por **/**, como por ejemplo de **"/one"** a **"/two"**). En este caso, vamos a redireccionar a una URL del mismo host.

La sintaxis para crear una redirección es:

{% highlight shell %}
Redirect "<pathURL>" "<URL>"
{% endhighlight %}

Siendo:

* **pathURL**: El recurso "virtual" que solicitaremos en la URL a la hora de hacer la petición al servidor. En este caso, la petición la haríamos a www.mapeo.com**/**, por lo que pathURL sería **/**.
* **URL**: La ruta real a la que se va a mapear el pathURL, es decir, la URL a la que deseamos redirigir al cliente. En este caso, la redirección la queremos hacer a www.mapeo.com**/web**, por lo que al tratarse de una redirección dentro del mismo host, la URL sería **/web**.

Existe un inconveniente o limitación, y es que la instrucción **Redirect** busca coincidencias parciales comenzando por la izquierda de **pathURL** en **URL**, es decir, que si quisiéramos redirigir desde **/** (pathURL) a **/web** (URL), se va a crear un bucle infinito, pues la expresión **/** se encuentra contenida en **/**web y cuando redirija a dicha URL, volverá a encontrar la misma coincidencia de forma indefinida. Lo mismo ocurriría si por ejemplo escribiésemos en la URL **/hola**, que a pesar de no haber introducido el **pathURL** de forma concreta (**/**), se va a realizar la redirección, ya que encuentra una coincidencia por la izquierda en dicha URL.

Para ello, en lugar de utilizar la instrucción **Redirect**, usaremos la instrucción **RedirectMatch**, que permite usar expresiones regulares, de manera que podremos especificar de forma más concreta el **pathURL**, evitando así dicho bucle infinito. En este caso, el pathURL sería **^/$** y la URL sería **/web**, por lo que gracias al carácter **^** indicamos que la coincidencia tendrá que empezar por **/** y gracias al carácter **$**, indicamos que tendrá que terminar por eso mismo, de manera que cuando se redirija a **/web**, no se volverá a encontrar ninguna coincidencia más, de manera que el bucle no se creará.

En resumen, la redirección que queremos crear tendrá la siguiente forma:

{% highlight shell %}
RedirectMatch ^/$ /web
{% endhighlight %}

Para ello, tendremos que introducir dicha instrucción al fichero de configuración del VirtualHost (haciendo uso de `nano`), quedando de la siguiente manera:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        Alias "/documentos" "/home/vagrant/Documentos"
        <Directory "/home/vagrant/Documentos">
                Options MultiViews
                AllowOverride None
                Require all granted
        </Directory>

        RedirectMatch ^/$ /web
</VirtualHost>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez cargada la nueva configuración, volveremos al navegador y trataremos de acceder a **www.mapeo.com**:

![redirect](https://i.ibb.co/r0Nrw2D/Captura-de-pantalla-de-2020-10-18-10-48-49.png "Redirect")

Efectivamente, la redirección ha funcionado y al acceder a **www.mapeo.com** se ha llevado a cabo de forma automática una redirección a **www.mapeo.com/web**, aunque es normal que nos de un error 404, pues no existe el recurso solicitado.

## Tarea 5: Con la directiva ErrorDocument se pueden crear respuestas de error personalizadas. Todo esto se puede llevar a cabo en el fichero /etc/apache2/conf-available/localized-error-pages.conf. Después de leer sobre el tema realiza los siguientes ejercicios:

### Cuando no se encuentre una página (error 404), personaliza un mensaje de error.

Los documentos de error personalizados se configuran mediante la directiva **ErrorDocument**, haciendo uso de la sintaxis:

{% highlight shell %}
ErrorDocument <códigoerror> <acción>
{% endhighlight %}

Siendo:

* **códigoerror**: El [código](https://developer.mozilla.org/es/docs/Web/HTTP/Status) de estado del error a gestionar. Por ejemplo, si queremos gestionar un error _Not Found_ tendríamos que introducir _404_.
* **acción**: Se podrá redireccionar a una URL del mismo host (si la acción empieza con "/"), a una URL de distinto host (si la acción es una URL válida) o un texto para mostrar, el cuál deberá estar entrecomillado si contiene más de una palabra.

En este caso, el **códigoerror** que queremos gestionar es un 404, además de indicar un texto para mostrar en la **acción**, por lo que la directiva quedaría de la siguiente forma:

{% highlight shell %}
ErrorDocument 404 "La página que has solicitado no se ha encontrado. Lo sentimos."
{% endhighlight %}

Para ello, tendremos que introducir dicha instrucción al fichero de configuración del VirtualHost (haciendo uso de `nano`), quedando de la siguiente manera:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        Alias "/documentos" "/home/vagrant/Documentos"
        <Directory "/home/vagrant/Documentos">
                Options MultiViews
                AllowOverride None
                Require all granted
        </Directory>

        RedirectMatch ^/$ /web

        ErrorDocument 404 "La página que has solicitado no se ha encontrado. Lo sentimos."
</VirtualHost>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez cargada la nueva configuración, volveremos al navegador y trataremos de acceder a **www.mapeo.com**:

![errordocument1](https://i.ibb.co/kHCTTFx/Captura-de-pantalla-de-2020-10-18-10-59-22.png "ErrorDocument")

Efectivamente, la redirección ha vuelto a funcionar y al acceder a **www.mapeo.com** se ha llevado a cabo de forma automática una redirección a **www.mapeo.com/web**, mostrando en esta ocasión el error 404 personalizado en forma de texto que hemos creado, pues el recurso **/web** no existe.

### Crea un alias llamado error que corresponda a /srv/mapeo/error. Dentro de ese directorio crea páginas personalizadas para visualizar cuando se produzca un error 404 y cuando se produzca un Forbidden (403). Configura el sistema para que se redireccione a estas páginas cuando se produce un error.

Para llevar a cabo esta tarea vamos a hacer uso de la misma sintaxis que el ejercicio anterior, pero en esta ocasión, en lugar de indicar un texto que se mostrará, indicaremos un fichero al que se redirigirá, además de crear un alias tal y como se ha solicitado, quedando las directivas de la siguiente forma:

{% highlight shell %}
Alias "/error" "/srv/mapeo/error"
ErrorDocument 404 /error/error_404.html
ErrorDocument 403 /error/error_403.html
{% endhighlight %}

Además, para que no se generen conflictos, eliminaremos la directiva **ErrorDocument** anteriormente configurada para introducir las nuevas.

Para ello, tendremos que llevar a cabo dichas modificaciones en el fichero de configuración del VirtualHost (haciendo uso de `nano`), quedando de la siguiente manera:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        Alias "/documentos" "/home/vagrant/Documentos"
        <Directory "/home/vagrant/Documentos">
                Options MultiViews
                AllowOverride None
                Require all granted
        </Directory>

        RedirectMatch ^/$ /web

        Alias "/error" "/srv/mapeo/error"
        ErrorDocument 404 /error/error_404.html
        ErrorDocument 403 /error/error_403.html
</VirtualHost>
{% endhighlight %}

Antes de volver a cargar la configuración del servicio, es recomendable crear el correspondiente directorio con dichos ficheros en su interior. Para crear el directorio, ejecutaremos el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# mkdir /srv/mapeo/error
{% endhighlight %}

Tras ello, ya podremos generar los ficheros que se servirán para cada uno de los errores. En este caso, vamos a generar un fichero **error_404.html** para el error Not Found y otro de nombre **error_403.html** para el error Forbidden, y se encontrarán alojados dentro de **/srv/mapeo/error**. El primero que voy a crear es el fichero para el error 404, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /srv/mapeo/error/error_404.html
{% endhighlight %}

El contenido es el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Español</title>
    </head>
    <body>
        <h2>El recurso solicitado no existe (404).</h2>
        <h3>Contacta con el administrador para más información.</h3>
    </body>
</html>
{% endhighlight %}

Tras ello, crearé el fichero para el error 403, ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /srv/mapeo/error/error_403.html
{% endhighlight %}

El contenido es el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Español</title>
    </head>
    <body>
        <h2>No tienes permiso para acceder al recurso solicitado (403).</h2>
        <h3>Contacta con el administrador para más información.</h3>
    </body>
</html>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servicio haciendo uso del comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl reload apache2
{% endhighlight %}

Una vez cargada la nueva configuración, volveremos al navegador y trataremos de acceder a **www.mapeo.com**:

![errordocument2](https://i.ibb.co/YbXD0Z1/Captura-de-pantalla-de-2020-10-18-11-16-31.png "ErrorDocument")

Efectivamente, la redirección ha vuelto a funcionar y al acceder a **www.mapeo.com** se ha llevado a cabo de forma automática una redirección a **www.mapeo.com/web**, mostrando en esta ocasión el error 404 personalizado en forma de fichero que hemos creado, pues el recurso **/web** no existe.

Para comprobar que el error personalizado para el Forbidden también funciona, podemos intentar acceder al enlace simbólico que hemos creado con anterioridad (**www.mapeo.com/documentos/link.html**), pues al haber desactivado la opción **FollowSymLinks** previamente, no tendremos permisos para ello:

![errordocument3](https://i.ibb.co/j5GBBc6/Captura-de-pantalla-de-2020-10-18-11-16-54.png "ErrorDocument")

Como era de esperar, se ha mostrado el error 403 personalizado en forma de fichero que hemos generado, pues no tenemos permisos para acceder al recurso solicitado.

### Descomenta en el fichero localized-error-pages.conf las líneas adecuadas para tener los mensajes de error traducidos a los diferentes idiomas.

Gracias al fichero situado en **/etc/apache2/conf-available/localized-error-pages.conf** podemos activar la negociación de contenidos de manera que devolverá las páginas de error en el idioma preferido del navegador.

Antes de empezar a configurar dicho fichero, vamos a eliminar las directivas creadas en el ejercicio anterior para evitar así conflictos.

Para ello, tendremos que llevar a cabo dichas modificaciones en el fichero de configuración del VirtualHost (haciendo uso de `nano`), quedando de la siguiente manera:

{% highlight shell %}
<VirtualHost *:80>
        ServerName www.mapeo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /srv/mapeo

        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #Include conf-available/serve-cgi-bin.conf

        Alias "/documentos" "/home/vagrant/Documentos"
        <Directory "/home/vagrant/Documentos">
                Options MultiViews
                AllowOverride None
                Require all granted
        </Directory>

        RedirectMatch ^/$ /web
</VirtualHost>
{% endhighlight %}

No es necesario recargar la configuración del servicio ya que posteriormente se nos pedirá que lo reiniciemos, así que nos podemos ahorrar el paso. Para modificar el fichero **/etc/apache2/conf-available/localized-error-pages.conf** tendremos que ejecutar el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# nano /etc/apache2/conf-available/localized-error-pages.conf
{% endhighlight %}

Dentro del mismo, debemos descomentar el siguiente bloque:

{% highlight shell %}
<IfModule mod_negotiation.c>
        <IfModule mod_include.c>
                <IfModule mod_alias.c>

                        Alias /error/ "/usr/share/apache2/error/"

                        <Directory "/usr/share/apache2/error">
                                Options IncludesNoExec
                                AddOutputFilter Includes html
                                AddHandler type-map var
                                Order allow,deny
                                Allow from all
                                LanguagePriority en cs de es fr it nl sv pt-br ro
                                ForceLanguagePriority Prefer Fallback
                        </Directory>

                        ErrorDocument 400 /error/HTTP_BAD_REQUEST.html.var
                        ErrorDocument 401 /error/HTTP_UNAUTHORIZED.html.var
                        ErrorDocument 403 /error/HTTP_FORBIDDEN.html.var
                        ErrorDocument 404 /error/HTTP_NOT_FOUND.html.var
                        ErrorDocument 405 /error/HTTP_METHOD_NOT_ALLOWED.html.var
                        ErrorDocument 408 /error/HTTP_REQUEST_TIME_OUT.html.var
                        ErrorDocument 410 /error/HTTP_GONE.html.var
                        ErrorDocument 411 /error/HTTP_LENGTH_REQUIRED.html.var
                        ErrorDocument 412 /error/HTTP_PRECONDITION_FAILED.html.var
                        ErrorDocument 413 /error/HTTP_REQUEST_ENTITY_TOO_LARGE.html.var
                        ErrorDocument 414 /error/HTTP_REQUEST_URI_TOO_LARGE.html.var
                        ErrorDocument 415 /error/HTTP_UNSUPPORTED_MEDIA_TYPE.html.var
                        ErrorDocument 500 /error/HTTP_INTERNAL_SERVER_ERROR.html.var
                        ErrorDocument 501 /error/HTTP_NOT_IMPLEMENTED.html.var
                        ErrorDocument 502 /error/HTTP_BAD_GATEWAY.html.var
                        ErrorDocument 503 /error/HTTP_SERVICE_UNAVAILABLE.html.var
                        ErrorDocument 506 /error/HTTP_VARIANT_ALSO_VARIES.html.var
                </IfModule>
        </IfModule>
</IfModule>
{% endhighlight %}

Si nos fijamos en las tres primeras líneas del mismo, podremos apreciar que es necesario tener activos los módulos **negotiation** (activo por defecto), **include** (hay que activarlo) y **alias** (activo por defecto).

Para activar dicho módulo, haremos uso del comando `a2enmod`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# a2enmod include
Considering dependency mime for include:
Module mime already enabled
Enabling module include.
To activate the new configuration, you need to run:
  systemctl restart apache2
{% endhighlight %}

Como se puede apreciar en los mensajes devueltos, el módulo ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que reiniciar el servicio _apache2_ (no es suficiente con volver a cargar la configuración), ejecutando para ello el comando:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# systemctl restart apache2
{% endhighlight %}

En este caso, tengo el navegador configurado para que utilice el idioma Español por defecto, así que volveremos a acceder a **www.mapeo.com** de manera que nos redirija a **www.mapeo.com/web** y nos muestre un error 404 (Not Found), para así verificar que nos devuelve el error en el idioma deseado:

![errordocument4](https://i.ibb.co/qMytb6g/Captura-de-pantalla-de-2020-10-18-11-41-35.png "ErrorDocument")

Efectivamente, el error se ha mostrado en Español, así que tras ello, modifiqué el idioma del navegador al Inglés, para volver a hacer la petición y ver si en esta ocasión se muestra en el nuevo idioma configurado:

![errordocument5](https://i.ibb.co/Tr5Frvg/Captura-de-pantalla-de-2020-10-18-11-41-47.png "ErrorDocument")

Como era de esperar, el error se ha devuelto en Inglés, dada la configuración actual del navegador.

Por último, me gustaría dejar una pequeña anotación. Por defecto, el usuario a través del cuál se sirven las páginas web en _apache2_ es **www-data** (en este caso no habría inconveniente ya que el grupo "otros", al que pertenece dicho usuario tiene permisos de **lectura**, que es lo único que necesitamos para servir el fichero, pero en caso de necesitar escribir en dichos ficheros, no contaría con dichos permisos de **escritura**), por lo que podemos proceder a cambiar dicho propietario y grupo de forma recursiva, en caso de así desearlo, ya que el usuario y grupo actual es **root**, haciendo para ello uso del comando `chown -R`:

{% highlight shell %}
root@apache:/etc/apache2/sites-available# chown -R www-data:www-data /srv/*
{% endhighlight %}

Gracias al comando ejecutado con anterioridad, todos los directorios y ficheros hijos de **/srv** serían ahora propiedad de **www-data** y haciendo uso del grupo **www-data**.