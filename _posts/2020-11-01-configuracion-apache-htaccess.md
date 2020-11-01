---
layout: post
title:  "Configuración de Apache mediante .htaccess"
banner: "/assets/images/banners/htaccess.png"
date:   2020-11-01 18:38:00 +0200
categories: servicios
---
Se recomienda realizar una previa lectura de los siguientes _posts_, pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad:

* [VirtualHosting con Apache](http://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html)
* [Mapear URL en Apache](http://www.alvarovf.com/servicios/2020/10/20/mapear-url-en-apache.html)
* [Control de acceso, autentificación y autorización en Apache](http://www.alvarovf.com/servicios/2020/11/01/control-acceso-apache.html)

Supongamos que tenemos contratado un _hosting_ compartido para hacer uso del servicio _apache2_ y servir páginas web. Si quisiésemos cambiar alguna configuración de nuestro VirtualHost, no sería posible, ya que no tenemos acceso al fichero de configuración correspondiente ni el administrador nos va a proporcionar la contraseña para ello.

Es por ello, que surgió una alternativa llamada **.htaccess** en _apache2_ (también conocido como archivo de configuración distribuida), un fichero que permite especificar la configuración para el **Directory** en el que nos encontramos, afectando de manera recursiva a los directorios hijos. Dicho fichero tiene el "poder" de sobreescribir aquella configuración especificada en el fichero de configuración del VirtualHost por el administrador.

Para que el fichero **.htaccess** tenga efecto, es necesario configurar una directiva en aquel Directory que contenga al DocumentRoot (bien desde el fichero de configuración general de _apache2_, es decir, **/etc/apache2/apache2.conf**, o desde el fichero de configuración concreto del VirtualHost, en **/etc/apache2/sites-available**). En este caso, dicha configuración tendrá que hacerla el administrador, pues nosotros, no tenemos permisos para ello.

Dicha directiva es **AllowOverride**, que por defecto, cuando instalamos el servicio, suele venir desactivada (_None_). Esta directiva puede tomar diferentes [valores](https://httpd.apache.org/docs/current/es/mod/core.html#allowoverride), con diferentes niveles de permisos:

* **AuthConfig**: Permite el uso de directivas de autorización (_AuthGroupFile_, _AuthName_, _AuthType_, _AuthUserFile_, _Require_...).
* **FileInfo**: Permite el uso de directivas de control de documentos (_ErrorDocument_, _LanguagePriority_...), de metadatos de documentos (_Header_, _RequestHeader_, _CookieExpires_...) y aquellas del módulo _mod_rewrite_ (_RewriteEngine_, _RewriteOptions_, _RewriteBase_...).
* **Indexes**: Permite el uso de directivas que controlen la indexación de directorios (_DirectoryIndex_, _HeaderName_, _IndexIgnore_...).
* **Limit**: Permite el uso de directivas que controlen el acceso al servidor (_Require ip_, _Require host_, _Require local_...).
* **Options**: Permite el uso de opciones específicas de directorios que se usan dentro de la directiva _Options_ (_Indexes_, _FollowSymLinks_, _MultiViews_...), que deben ser indicadas separadas por comas.
* **All**: Tiene habilitados todos los permisos anteriormente mencionados.
* **None**: Tiene deshabilitados todos los permisos anteriormente mencionados, o lo que es lo mismo, el fichero _.htaccess_ no se tiene en cuenta.

Vamos a proceder a registrarnos en un _hosting_ web gratuito para hacer algunas pruebas con el fichero **.htaccess**. Es muy importante buscar un _hosting_ que tenga la directiva **AllowOverride** habilitada, ya que de lo contrario, la configuración que hagamos no va a servir de nada. En este caso, he optado por hacer uso de [000webhost](https://www.000webhost.com/). No considero necesario explicar el proceso de registro ya que es algo totalmente trivial.

Una vez registrados, podremos gestionar los ficheros existentes desde un cliente FTP, como por ejemplo _FileZilla_ o bien, acceder al [gestor de ficheros](https://files.000webhost.com/) que nos proporciona la propia web, una opción que nos resultará mucho más cómoda. Dentro del mismo, veremos lo siguiente:

![estructura1](https://i.ibb.co/Kqxht00/1.jpg "Estructura de ficheros")

Como podemos apreciar, del directorio raíz (virtual), cuelgan dos directorios:

* **public_html/**: Directorio en el que se deben subir los ficheros para que sean servidos. Realmente no es necesario saberlo, pero por curiosidad, esto es resultado de estar usando un módulo llamado _userdir_, que permite que cada uno de los usuarios registrados en el _hosting_ tengan un directorio personal totalmente aislado del resto en el que podrán subir los ficheros oportunos (ya que no tienen permisos para subirlos en el propio DocumentRoot).
* **tmp/**: Directorio que contendrá ficheros temporales, por ejemplo, durante la instalación de un CMS, que posteriormente serán eliminados de forma automática.

Como se puede suponer, nosotros haremos uso del directorio **public_html/**, así que accederemos al mismo y encontraremos el siguiente contenido:

![estructura2](https://i.ibb.co/tBbnW2q/3.jpg "Estructura de ficheros")

Efectivamente, el único contenido existente es el fichero _.htaccess_, el cuál usaremos a partir de ahora.

## Tarea 1: Habilita el listado de ficheros en la URL http://host.dominio/nas.

En **000webhost** viene habilitado por defecto el listado de ficheros (_Options Indexes_), así que lo primero que haremos será desactivar dicha opción en el fichero _.htaccess_ que acabamos de ver.

Una peculiaridad de dicho fichero es que no soporta el uso de estructuras **Directory**, por lo que aquella configuración que hagamos afectará al directorio actual y a los subdirectorios existentes en el mismo.

En este caso, debemos añadir lo siguiente para deshabilitar el listado de ficheros:

{% highlight shell %}
Options -Indexes
{% endhighlight %}

Tras dicha modificación, el fichero debe quedar de la siguiente manera:

![htaccess1](https://i.ibb.co/QQ7DDJ8/10.jpg ".htaccess")

Para guardar los cambios, pulsaremos en **SAVE & CLOSE**.

Listo, ya tendremos la configuración inicial hecha, así que vamos a empezar a trabajar con el directorio **nas/**. Lo primero que tendremos que hacer será crearlo, pulsando para ello en la opción **New Folder** de la barra superior, quedando tras ello, de la siguiente manera:

![estructura3](https://i.ibb.co/VvJTnwv/5.jpg "Estructura de ficheros")

Una vez creado, abriremos una nueva pestaña en el navegador y trataremos de acceder a dicha ruta. El resultado debería ser el siguiente:

![acceso1](https://i.ibb.co/Sfsg2h0/7.jpg "Acceso prohibido")

Si recordamos, anteriormente he mencionado que el fichero _.htaccess_ afecta al directorio actual y a los directorios recursivos, es por ello que aquí tampoco tenemos dicho permiso. Dado que anteriormente hemos prohibido la opción _Indexes_, ahora nos muestra un error Forbidden (403), pues no tenemos permiso para ello, pero si lo pensamos, esto es justo lo contrario a lo que queremos, pues pretendemos permitir el listado del contenido de dicho directorio.

Ahora, tendremos que añadir dicha opción pero únicamente para el directorio **nas/**. Para ello, accederemos al mismo y crearemos un nuevo fichero _.htaccess_, pulsando para ello en la opción **New File** de la barra superior. Una vez creado, debemos introducir dentro del mismo la siguiente instrucción, para así habilitar dicha funcionalidad:

{% highlight shell %}
Options +Indexes
{% endhighlight %}

Tras dicha modificación, el fichero debe quedar de la siguiente manera:

![htaccess2](https://i.ibb.co/XCxfZRf/11.jpg ".htaccess")

De nuevo, volveremos a pulsar en **SAVE & CLOSE**.

Supuestamente, el listado de ficheros ya está habilitado, así que recargaremos la página en el navegador, debiendo obtener el siguiente resultado:

![acceso2](https://i.ibb.co/ssVM9wQ/6.jpg "Acceso permitido")

Efectivamente, el listado de ficheros ha sido habilitado únicamente para el directorio **/nas**.

## Tarea 2: Crea una redirección permanente, de manera que cuando entremos en http://host.dominio/google salte a www.google.es.

Para ello, tendremos que hacer uso de la directiva **Redirect**, pero no en el fichero _.htaccess_ del directorio **/nas**, sino en el fichero _.htaccess_ existente en el directorio padre, es decir, en **public_html/**.

Tendremos que añadir al mismo la siguiente directiva:

{% highlight shell %}
Redirect 301 "/google" "https://www.google.es"
{% endhighlight %}

Como se puede apreciar, vamos a crear una redirección 301 (permanente), para que cuando accedamos al recurso **/google**, nos redirija a **https://www.google.es**.

De nuevo, volveremos a pulsar en **SAVE & CLOSE**, para posteriormente acceder a dicho recurso desde el navegador y verificar que la redirección funciona:

![acceso3](https://i.ibb.co/KVydjKw/8.jpg "Google")

Efectivamente, la redirección ha funcionado, pero vamos a hacer uso del comando `curl` para ver la redirección de una forma más profunda, observando la respuesta por parte del servidor. Para ello, tendremos que indicarle la URL a la que hacer la petición:

{% highlight shell %}
alvaro@debian:~$ curl pruebaapache.000webhostapp.com/google/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="https://www.google.es/">here</a>.</p>
</body></html>
{% endhighlight %}

Como se puede apreciar, la respuesta del servidor es una _Moved Permanently_ (301), indicando la URL del servidor al que hacer la nueva petición.

## Tarea 3: Pedir autentificación para entrar en la URL http://host.dominio/prohibido.

Lo primero que tendremos que hacer será crear dicho directorio dentro de **public_html/**, ya que todavía no existe, pulsando para ello en la opción **New Folder** de la barra superior, quedando finalmente de la siguiente manera:

![estructura4](https://i.ibb.co/25xJCs5/9.jpg "Estructura de ficheros")

Para hacer la prueba de una forma un poco más visual, vamos a generar un fichero **index.html** que se sirva de forma automática al acceder al directorio (gracias a la directiva _DirectoryIndex_), pulsando para ello en la opción **New File** de la barra superior. El contenido del mismo será el siguiente:

{% highlight html %}
<!DOCTYPE html>

<html>
    <head>
        <meta charset="utf-8">
        <title>Prueba</title>
    </head>
    <body>
        <h2>Prueba 000webhost.</h2>
    </body>
</html>
{% endhighlight %}

Tras ello, tendremos que generar un fichero para almacenar las credenciales de los usuarios, que no debe ser accesible, por lo que no puede estar dentro del directorio **public_html/**. En este caso, lo he situado al mismo nivel que el directorio anteriormente mencionado, es decir, dentro de **/**. El fichero lo he creado con nombre **pass.txt**, pulsando para ello en la opción **New File** de la barra superior.

Como es lógico, dentro del mismo, tendremos que añadir la correspondencia entre usuarios y contraseñas, así que podemos hacer uso de `htpasswd` para cifrar dichas contraseñas (ya que vamos a configurar la autentificación básica). En este caso, voy a crear un usuario "**prueba**" con contraseña "**password123**" (obviamente, en un caso real, se utilizarían credenciales más seguras).

En caso de no tener dicha utilidad instalada, tendríamos que instalar el paquete **apache2-utils**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt install apache2-utils
{% endhighlight %}

Listo, ya podemos proceder a cifrar la contraseña, haciendo uso del comando anteriormente mencionado:

{% highlight shell %}
alvaro@debian:~$ htpasswd -n prueba
New password: 
Re-type new password: 
prueba:$apr1$hWjrFj7Q$hnzo5A7Lj/uG1tF3ABWa3/
{% endhighlight %}

Donde:

* **-n**: Indica que el resultado se muestre por pantalla en lugar de incluirlo en un fichero.

Tras ello, tendremos que incluir el resultado de la ejecución del comando dentro del fichero **pass.txt** generado con anterioridad, quedando de la siguiente manera:

![pass1](https://i.ibb.co/YZbG9sD/15.jpg "pass.txt")

De nuevo, volveremos a pulsar en **SAVE & CLOSE** para guardar los cambios en el fichero.

A la hora de llevar a cabo la configuración de la autentificación, tendremos que indicar la ruta de dicho fichero que contiene las contraseñas encriptadas. Tal y como he mencionado anteriormente, la ruta **/** realmente se trata de una ruta virtual, así que tendremos que ver a qué ruta real corresponde. Para ello, iremos a nuestra [lista de sitios](https://www.000webhost.com/members/website/list) del _hosting_ y pulsaremos en **Quick Actions** en el sitio en cuestión para seguidamente pulsar en **Details**.

Una vez ahí, tendremos que buscar el apartado **File Upload Details (FTP)**, y buscar el **Home Directory**, que será la equivalencia de la ruta virtual **/**:

![home1](https://i.ibb.co/FBgtfdP/16.jpg "Home directory")

Como se puede apreciar, la ruta virtual **/** equivale a **/storage/ssd5/556/15225556**.

Ahora, tendremos que añadir dicha opción de autentificación pero únicamente para el directorio **prohibido/**. Para ello, crearemos un nuevo fichero _.htaccess_, pulsando para ello en la opción **New File** de la barra superior. Una vez creado, debemos introducir dentro del mismo la siguiente instrucción, para así habilitar dicha funcionalidad:

{% highlight shell %}
AuthType Basic
AuthName "Acceso administrador"
AuthUserFile "/storage/ssd5/556/15225556/pass.txt"
Require valid-user
{% endhighlight %}

Donde:

* **AuthType**: Es el tipo de autentificación que vamos a usar. En este caso, **Basic**.
* **AuthName**: En teoría, es el texto informativo que se mostrará en la ventana emergente a la hora de solicitar las credenciales de acceso al usuario que trata de acceder, pero en este _hosting_, no funciona correctamente.
* **AuthUserFile**: Indicaremos la ruta del fichero que almacenará las credenciales de acceso de los usuarios (en caso de querer permitir el acceso mediante grupos en lugar de usuarios, tendríamos que hacer uso de **AuthGroupFile**).
* **Require**: Indicamos el método de verificación de acceso, que en este caso, será **valid-user**, de manera que dejará acceder a cualquier usuario existente en el fichero previamente mencionado que introduzca de forma correcta su contraseña.

Tras dicha modificación, el fichero debe quedar de la siguiente manera:

![htaccess3](https://i.ibb.co/7rLJRS6/12.jpg ".htaccess")

De nuevo, volveremos a pulsar en **SAVE & CLOSE** para guardar los cambios en el fichero.

Supuestamente, la autentificación ya se encuentra totalmente configurada, así que accederemos a dicho recurso, debiendo obtener el siguiente resultado:

![acceso4](https://i.ibb.co/Lzw7113/13.jpg "Solicita credenciales")

En este caso, introduciremos las credenciales anteriormente configuradas y pulsaremos en **Aceptar**, de manera que debería mostrar el contenido de la misma:

![acceso5](https://i.ibb.co/qgvpG6H/14.jpg "Acceso permitido")

Efectivamente, la configuración para la autentificación ha sido correctamente realizada y funciona a la perfección.