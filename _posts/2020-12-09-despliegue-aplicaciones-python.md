---
layout: post
title:  "Despliegue de aplicaciones web Python"
banner: "/assets/images/banners/django.jpg"
date:   2020-12-09 21:13:00 +0200
categories: aplicaciones
---
La finalidad de este artículo es la de desplegar una aplicación web **Python** que haga uso del _framework_ **Django**, tratando de simular en todo momento una situación real, en la que existirán dos entornos claramente diferenciados: **desarrollo** y **producción**.

La premisa que debemos mantener en todo momento es que en el entorno de producción nunca se modifica el código de la aplicación, sino que se hace en desarrollo, y posteriormente, se despliegan los cambios en el entorno de producción cuando verifiquemos que todo funciona correctamente. Dichos entornos deberán permanecer en todo momento lo más similares posibles (en cuanto a estructura se refiere, pues lógicamente, en producción tendremos una base de datos mucho más poblada que en el entorno de desarrollo, en el cuál utilizaremos registros de prueba).

En este caso, vamos a hacer uso de la aplicación que se nos proporciona en el [tutorial](https://docs.djangoproject.com/en/3.1/intro/tutorial01/) introductorio de Django, pero como la intención no es enseñar a crear aplicaciones Django (sino desplegarlas), vamos a utilizar una aplicación que se encuentra ya un poco modificada, que podremos encontrar [aquí](https://github.com/josedom24/django_tutorial).

Todavía no hemos llegado al punto anteriormente mencionado consistente en desplegar los cambios del entorno de desarrollo en producción, pero considero que es necesario conocer el _modus operandi_ antes de empezar. Dado que vamos a desplegar una aplicación relativamente "pequeña", podríamos hacer uso de un comando como `scp` para transferir los ficheros que hayan sido modificados del entorno de desarrollo al entorno de producción, pero dicha técnica se vuelve totalmente ineficiente conforme la aplicación va creciendo en tamaño y el número de ficheros modificados es mayor.

Es por ello que haremos uso del sistema `git`, pues nos permitirá transferir de una forma muy cómoda y automatizada a nuestro repositorio común únicamente los ficheros que hayan sufrido modificaciones, aportando además de ello un control de versiones que nos puede venir muy bien en determinadas ocasiones. No pretendo enseñar a usar _git_, ya que para ello tenéis magníficas [guías](https://rogerdudler.github.io/git-guide/index.es.html), sino que parto de un punto en el que se da por hecho.

La aplicación que vamos a desplegar se encuentra actualmente alojada en [GitHub](https://github.com/josedom24/django_tutorial), pero dicho repositorio no nos pertenece, por lo que no podremos subir nuestros cambios. Para solventar este problema, existen los _fork_, también conocidos como bifurcaciones, consistentes en hacer una copia exacta en crudo de dicho repositorio, convirtiéndonos a nosotros en el propietario del nuevo repositorio, por lo que podremos trabajar con el mismo sin ningún tipo de limitación.

## Tarea 1: Entorno de desarrollo

El primer paso será clonar el repositorio de GitHub tras haber hecho el _fork_, de manera que en mi caso, me he movido al directorio **GitHub/** (haciendo uso de `cd`), pues es el que uso para alojar los repositorios que clono. Es importante mencionar que el paquete **git** no viene instalado por defecto, por lo que es necesario instalarlo manualmente.

Para la clonación tenemos a nuestra disposición 3 protocolos diferentes de los que podemos hacer uso. En mi caso, por comodidad, voy a utilizar el protocolo **SSH**, ya que tengo introducida mi clave pública en mi cuenta de GitHub, pero en caso contrario, también se podría utilizar **HTTPS**. Cuando hayamos obtenido la URI de descarga, se la pasaremos a `git clone`:

{% highlight shell %}
alvaro@debian:~/GitHub$ git clone git@github.com:alvarovaca/django_tutorial.git
Clonando en 'django_tutorial'...
remote: Enumerating objects: 37, done.
remote: Counting objects: 100% (37/37), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 129 (delta 4), reused 24 (delta 3), pack-reused 92
Recibiendo objetos: 100% (129/129), 4.25 MiB | 970.00 KiB/s, listo.
Resolviendo deltas: 100% (28/28), listo.
{% endhighlight %}

Tras haberme solicitado la frase de paso de mi clave privada asociada a la clave pública que tengo definida en GitHub, se habrá procedido a la clonación del repositorio (en caso de haber hecho uso de **HTTPS**, no se habrá pedido ningún tipo de credenciales), de manera que vamos a listar el contenido del directorio actual para así verificar que la clonación ha sido efectiva:

{% highlight shell %}
alvaro@debian:~/GitHub$ ls
blog-alvarovf  django_tutorial
{% endhighlight %}

Efectivamente, el repositorio de nombre **django_tutorial** ha sido correctamente clonado en nuestra máquina local y podremos empezar a hacer uso del mismo.

El siguiente paso será crear un entorno virtual de Python en nuestra máquina anfitriona, en el cuál instalaremos los paquetes necesarios para que la aplicación funcione. Para la correspondiente creación de un entorno virtual, debemos instalar previamente el paquete necesario (**python3-venv**), ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/GitHub$ sudo apt install python3-venv
{% endhighlight %}

En mi caso, me he movido al directorio **virtualenv/** (haciendo uso de `cd`), pues es donde tengo alojados todos los entornos virtuales que creo. En esta ocasión, el nombre que le voy a asignar al entorno virtual es **despliegue**, así que lo generaré ejecutando el comando:

{% highlight shell %}
alvaro@debian:~/virtualenv$ python3 -m venv despliegue
{% endhighlight %}

Una vez creado, tendremos que iniciarlo haciendo uso de `source` con el binario **activate** que se encuentra contenido en el directorio **bin/**:

{% highlight shell %}
alvaro@debian:~/virtualenv$ source despliegue/bin/activate
{% endhighlight %}

Nuestro entorno virtual habrá sido habilitado y en consecuencia, podremos apreciar que el _prompt_ ha cambiado.

Ya está todo listo para instalar los paquetes necesarios, entre los que se encuentra el propio _framework_, pero en lugar de hacerlo manualmente con `pip`, vamos a utilizar un fichero **requirements.txt** en el que se encuentra indicada la lista de dependencias necesarias, la cuál podremos instalar en su totalidad de forma automatizada. Dicho fichero se encuentra en el interior del repositorio que hemos clonado, por lo que nos moveremos al mismo haciendo uso de `cd`:

{% highlight shell %}
(despliegue) alvaro@debian:~/virtualenv$ cd ../GitHub/django_tutorial/
{% endhighlight %}

Cuando nos encontremos correctamente ubicados, procederemos a listar el contenido del directorio actual, para así comprobar los ficheros existentes, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ ls
django_tutorial  manage.py  polls  README.md  requirements.txt
{% endhighlight %}

Vamos a proceder a explicar qué son cada uno de los ficheros y directorios existentes:

* **django_tutorial/**: Es el propio proyecto Django, pues en este caso, al tratarse de un despliegue, viene ya creado. El contenido del mismo lo veremos a continuación.
* **manage.py**: Utilidad de línea de comandos que usaremos para interactuar con nuestro proyecto Django.
* **polls/**: Directorio generado como consecuencia de existir una aplicación de nombre **polls** instalada en nuestro proyecto Django. Lo veremos con más detenimiento a continuación.
* **README.md**: Fichero generado para mostrar información acerca del repositorio GitHub. No es relevante en este caso.
* **requirements.txt**: Fichero que contiene la lista de dependencias necesarias para el correcto funcionamiento de la aplicación. Debemos instalar su contenido en nuestro entorno virtual.

Tal y como hemos mencionado, el fichero que actualmente nos interesa es **requirements.txt**, así que antes de instalar el contenido de su interior, vamos a visualizarlo para entender con más profundidad qué estamos haciendo. Para ello, ejecutaremos el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ cat requirements.txt 
asgiref==3.3.0
Django==3.1.3
pytz==2020.4
sqlparse==0.4.1
{% endhighlight %}

Como se puede apreciar, existen un total de 4 paquetes necesarios para el funcionamiento de la aplicación web, así que instalaremos las versiones específicas que se nos han indicado, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ pip install -r requirements.txt 
Collecting asgiref==3.3.0 (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/c0/e8/578887011652048c2d273bf98839a11020891917f3aa638a0bc9ac04d653/asgiref-3.3.0-py3-none-any.whl
Collecting Django==3.1.3 (from -r requirements.txt (line 2))
  Downloading https://files.pythonhosted.org/packages/7f/17/16267e782a30ea2ce08a9a452c1db285afb0ff226cfe3753f484d3d65662/Django-3.1.3-py3-none-any.whl (7.8MB)
    100% |████████████████████████████████| 7.8MB 259kB/s 
Collecting pytz==2020.4 (from -r requirements.txt (line 3))
  Using cached https://files.pythonhosted.org/packages/12/f8/ff09af6ff61a3efaad5f61ba5facdf17e7722c4393f7d8a66674d2dbd29f/pytz-2020.4-py2.py3-none-any.whl
Collecting sqlparse==0.4.1 (from -r requirements.txt (line 4))
  Downloading https://files.pythonhosted.org/packages/14/05/6e8eb62ca685b10e34051a80d7ea94b7137369d8c0be5c3b9d9b6e3f5dae/sqlparse-0.4.1-py3-none-any.whl (42kB)
    100% |████████████████████████████████| 51kB 3.0MB/s 
Installing collected packages: asgiref, pytz, sqlparse, Django
Successfully installed Django-3.1.3 asgiref-3.3.0 pytz-2020.4 sqlparse-0.4.1
{% endhighlight %}

Donde:

* **-r**: Indicamos que vamos a hacer uso de un fichero que contendrá una lista de las dependencias a instalar.

El proceso de instalación ha sido muy sencillo, ya que nos hemos limitado a indicarle en qué fichero se encuentran contenidos los paquetes, recorriendo automáticamente la lista e instalando uno por uno todos ellos.

Como anteriormente hemos mencionado, existe un subdirectorio **django_tutorial/** correspondiente al propio proyecto Django, por lo vamos a proceder a listar su contenido para así ver qué contiene y explicar por tanto para qué sirven cada uno de los ficheros:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ ls django_tutorial/
asgi.py  __init__.py  settings.py  urls.py  wsgi.py
{% endhighlight %}

Vamos a proceder a explicar qué son cada uno de los ficheros existentes:

* **asgi.py**: Fichero utilizado como punto de entrada para servidores web compatibles con ASGI para servir nuestro proyecto.
* **__init__.py**: Fichero vacío que sirve para indicar a Python que el directorio actual debe utilizado como un paquete Python.
* **settings.py**: Fichero utilizado para indicar la configuración de nuestro proyecto, como por ejemplo la base de datos a utilizar.
* **urls.py**: Fichero que contiene las declaraciones de URL virtuales para el proyecto, ya que en Python se trabaja con rutas virtuales.
* **wsgi.py**: Fichero utilizado como punto de entrada para servidores web compatibles con WSGI para servir nuestro proyecto. Lo trataremos con mayor detenimiento más adelante.

Existe un problema que no hemos planteado hasta ahora, y es que si lo pensamos, gracias a `git` podremos desplegar en producción aquellos ficheros que hayan sido modificados, ¿pero qué pasaría si la nueva versión de la aplicación incluye modificaciones en la estructura de la base de datos? No podríamos hacer una copia de seguridad de la base de datos en desarrollo para posteriormente importarla en el otro entorno, ya que los registros existentes en ambos entornos son totalmente distintos. Para ello, vamos a hacer uso de una gran característica que nos aporta Django, las **migraciones**.

Gracias a dicho mecanismo, almacenaremos en un fichero los cambios producidos en la estructura de la base de datos, para posteriormente reproducir dichos cambios en producción. Además, por si no era suficiente, nos aporta una capa de abstracción, ya que el fichero resultante es genérico, es decir, no importa la base de datos que tengamos debajo, suponiendo por tanto que en desarrollo podemos utilizar una base de datos pequeña como puede ser un fichero **sqlite3**, mientras que en producción podemos utilizar una base de datos **MySQL**, teniendo en todo momento la seguridad de que los cambios los podremos importar indistintamente de la diferencia de gestor.

Para demostrarlo, vamos a leer el contenido del fichero **settings.py** existente en el subdirectorio **django_tutorial/** previamente mencionado, para así visualizar la sección referente al uso de la base de datos, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ cat django_tutorial/settings.py
{% endhighlight %}

Dentro del mismo, encontraremos una sección de nombre **DATABASES**, que por defecto tendrá la siguiente forma:

{% highlight shell %}
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
{% endhighlight %}

Como se puede apreciar, la base de datos es de tipo **sqlite3**, un tipo de base de datos muy pequeña, tan pequeña que se encuentra contenida en un fichero, que todavía no se encuentra generado. Para generarlo, tendremos que aplicar las migraciones que todavía no han sido aplicadas, es decir, los proyectos Django están compuestos de aplicaciones, que hacen uso a su vez de determinadas tablas en una base de datos, pero al no existir todavía dichas tablas, tendremos que crearlas.

No tendría sentido crearlas a mano, ya que para ello se nos proporcionan los correspondientes ficheros de migraciones con las instrucciones a ejecutar para reproducir de forma exacta la base de datos que necesitamos. Es normal que sea algo confuso, pero al final de este artículo vamos a crear nuestras propias migraciones para así aclarar conceptos. Para aplicar dichas migraciones y generar la base de datos, ejecutaremos el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying polls.0001_initial... OK
  Applying sessions.0001_initial... OK
{% endhighlight %}

Como se puede apreciar, hemos hecho uso del fichero **manage.py** para gestionar así nuestro proyecto y aplicar las migraciones de todas las aplicaciones existentes (**admin**, **auth**, **polls**...), generando por tanto, las tablas necesarias en la base de datos para que dichas aplicaciones puedan funcionar tal y como deberían.

Para verificar que el fichero que contiene la base de datos **sqlite3** ha sido correctamente generado, listaremos una vez más el contenido del directorio actual, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ ls
db.sqlite3  django_tutorial  manage.py  polls  README.md  requirements.txt
{% endhighlight %}

Efectivamente, un fichero de nombre **db.sqlite3** ha sido generado, que contendrá todas las tablas necesarias.

La aplicación se encuentra ya instalada en su totalidad, de manera que vamos a proceder a crear un usuario administrador de dicho proyecto, que tendrá privilegios suficientes para acceder a la zona de administración, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py createsuperuser
Username (leave blank to use 'alvaro'): 
Email address: avacaferreras@gmail.com
Password: 
Password (again): 
Superuser created successfully.
{% endhighlight %}

Se nos ha solicitado un nombre de usuario para el mismo, un correo electrónico y una contraseña, así que una vez creado, vamos a proceder a arrancar el servidor web ligero que nos proporciona Django, pensado para el desarrollo del proyecto, haciendo uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
November 26, 2020 - 12:36:55
Django version 3.1.3, using settings 'django_tutorial.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
{% endhighlight %}

Como podemos apreciar en la salida, el servidor web de desarrollo está actualmente ejecutándose por defecto en un _socket TCP/IP_ en la interfaz _loopback_ (**127.0.0.1**), concretamente en el puerto **8000** (ambos parámetros son modificables), así que abriremos un navegador web y accederemos al mismo, concretamente a la zona de administración (**/admin**):

![admin1](https://i.ibb.co/wY3JTNH/Captura-de-pantalla-de-2020-11-26-13-39-21.png "Zona de administración")

Una vez hayamos accedido a la zona de administración, se nos solicitarán los credenciales del usuario administrador que acabamos de generar, así que tras introducirlos, pulsaremos en **Log in**:

![admin2](https://i.ibb.co/Xj9xVJK/Captura-de-pantalla-de-2020-11-26-13-39-27.png "Zona de administración")

El acceso a la zona de administración ha funcionado correctamente, pudiendo interactuar con determinadas aplicaciones, como por ejemplo, aquella de autenticación y autorización, que nos permitirá añadir nuevos usuarios y grupos a la base de datos, e incluso una aplicación de encuestas que ha sido instalada de forma manual por la persona propietaria del repositorio del que hemos hecho _fork_ (este es el motivo por el que existe un subdirectorio de nombre **polls/**).

Para comprobar el funcionamiento de esta última aplicación, vamos a añadir dos nuevas preguntas, pulsando para ello en **Add** en el apartado **Questions** de la sección **POLLS**:

![admin3](https://i.ibb.co/pWMv7GM/Captura-de-pantalla-de-2020-11-26-13-41-35.png "Zona de administración")

En mi caso, he creado la pregunta "**¿Cuál es tu lenguaje de programación favorito?**" con 4 posibles respuestas, indicando a su vez la fecha y hora de inicio de la encuesta. Cuando hayamos finalizado, pulsaremos en **Save and add another** para así añadir una segunda encuesta de prueba:

![admin4](https://i.ibb.co/Lz8xsPD/Captura-de-pantalla-de-2020-11-26-13-42-39.png "Zona de administración")

El título de esta segunda pregunta es "**¿Qué fabricante de procesadores prefieres?**", existiendo en esta ocasión un total de 2 posibles respuestas. Por último, pulsaremos en **SAVE** para añadirla y volver a la zona de administración:

![admin5](https://i.ibb.co/5Gjc1c7/Captura-de-pantalla-de-2020-11-26-13-42-47.png "Zona de administración")

Como se puede apreciar, las dos preguntas han sido correctamente generadas, así que accederemos tras ello a la ruta **/polls** para así hacer uso de dicha aplicación, en la que se mostrarán las encuestas existentes y nos permitirá votar entre las diferentes posibilidades:

![polls1](https://i.ibb.co/sws6xS8/Captura-de-pantalla-de-2020-11-26-13-42-55.png "Aplicación de encuestas")

Efectivamente, ambas encuestas han sido mostradas, de manera que accederemos a la primera de ellas para verificar que todo funciona como debería:

![polls2](https://i.ibb.co/PQczBNK/Captura-de-pantalla-de-2020-11-26-13-43-08.png "Aplicación de encuestas")

Las 4 posibles respuestas han sido también mostradas, por lo que seleccionaré al protagonista de este artículo, **Python**, y pulsaré en **Vote** para así enviar mi voto:

![polls3](https://i.ibb.co/Qk5d0Lq/Captura-de-pantalla-de-2020-11-26-13-43-16.png "Aplicación de encuestas")

Efectivamente, el voto ha sido correctamente registrado y los resultados actuales, mostrados.

Vamos a comprobar qué ha ocurrido de una forma más interna, volviendo a la terminal y utilizando un cliente **sqlite3** que tenía instado con anterioridad, para así hacer uso de la base de datos existente. El comando a ejecutar sería:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ sqlite3 db.sqlite3 
SQLite version 3.27.2 2019-02-25 16:06:06
Enter ".help" for usage hints.
{% endhighlight %}

Una vez dentro del cliente con la correspondiente base de datos abierta, vamos a ejecutar la instrucción necesaria para listar las tablas existentes en la misma:

{% highlight shell %}
sqlite> .tables
auth_group                  django_admin_log          
auth_group_permissions      django_content_type       
auth_permission             django_migrations         
auth_user                   django_session            
auth_user_groups            polls_choice              
auth_user_user_permissions  polls_question
{% endhighlight %}

En la salida de la instrucción se han mostrado todas las tablas resultantes de haber aplicado las correspondientes migraciones, siendo **polls_question** aquella en la que se almacenarán las dos preguntas que hemos creado, así como **polls_choice** aquella en la que se almacenarán las posibles respuestas y los votos realizados para cada una de ellas. Vamos a proceder a comprobar el contenido de ambas, ejecutando para ello las instrucciones:

{% highlight shell %}
sqlite> SELECT * FROM polls_question;
1|¿Cuál es tu lenguaje de programación favorito?|2020-11-26 12:41:31
2|¿Qué fabricante de procesadores prefieres?|2020-11-26 12:42:38

sqlite> SELECT * FROM polls_choice;
1|Python|1|1
2|Java|0|1
3|C++|0|1
4|PHP|0|1
5|AMD|0|2
6|Intel|0|2
{% endhighlight %}

Efectivamente, la tabla **polls_question** contiene un total de 3 columnas por cada registro, almacenando la primera de ellas un identificador único, la segunda el título de la pregunta y la tercera, la fecha de publicación.

De otro lado, la tabla **polls_choice**, contiene un total de 4 columnas por cada registro, almacenando la primera de ellas un identificador único, la segunda el título de la respuesta, la tercera el número de votos, y la cuarta, el identificador de la pregunta a la que corresponde mediante una clave foránea de la tabla anteriormente mencionada.

En mi opinión, el contenido de la base de datos ha quedado bastante claro, de manera que saldremos del cliente para proceder a configurar ahora, el entorno de producción. Para ello, haremos uso de la instrucción:

{% highlight shell %}
sqlite> .quit
{% endhighlight %}

Tras ello, habremos salido del cliente **sqlite3** y nos encontraremos de vuelta en nuestra _shell_.

## Tarea 2: Entorno de producción

Nuestro entorno de desarrollo se encuentra ya totalmente instalado y operativo, pero nos falta la que posiblemente sea la parte más importante, el entorno de producción.

Dicho entorno se encontrará alojado en esta ocasión en una máquina virtual que he creado, y en el mismo, debemos instalar una serie de paquetes necesarios para poder servir dicha aplicación, ya que a diferencia del entorno de desarrollo, no vamos a utilizar el servidor web que incluye el proyecto Django, sino que vamos a utilizar un servidor web más potente como es _apache2_.

Aquí es cuando llega el problema, ¿cómo podemos hacer que un servidor web como _apache2_ sea capaz de servir una aplicación escrita en Python? Para ello, haremos uso de un protocolo que nos permite comunicar al servidor web con la aplicación web, de nombre **WSGI** (_Web Server Gateway Interface_).

Dicho protocolo define las reglas para que el servidor web se comunique con la aplicación web, es decir, cuando al servidor _apache2_ (en este caso) le llega una petición que tenemos que mandar a la aplicación web Python, se reenviará dicha petición a un fichero de entrada conocido como **fichero WSGI**, en el que estará nombrada la aplicación. En este caso, dicho fichero ya se nos ha proporcionado, de nombre **wsgi.py**.

Para que _apache2_ sea capaz de hacer uso de dicho protocolo, tendremos que instalar (además del propio paquete **apache2**) un módulo que aporta dicha funcionalidad, de nombre **libapache2-mod-wsgi-py3**, consiguiendo por tanto que el servidor de aplicaciones que interpretará el código para generar HTML sea el propio servidor web.

Esto no es todo, ya que la aplicación necesita una base de datos en la que almacenar sus correspondientes tablas, por lo que al tratarse del entorno de producción, en lugar de hacer uso de una base de datos de tipo _sqlite3_, la cuál se encuentra muy limitada, haremos uso de MySQL, por lo que instalaremos los paquetes **mariadb-server** y **mariadb-client**. Necesitaremos instalar además un módulo de Python para que le permita interactuar con una base de datos MySQL, de nombre **python3-mysqldb**.

Por último, instalaremos el paquete **git** para poder clonar el repositorio que contiene el proyecto Django y el paquete **python3-venv** para poder crear el correspondiente entorno virtual, no sin antes actualizar toda la paquetería instalada en la máquina. El comando final a ejecutar sería (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
root@django:~# apt update && apt upgrade && apt install apache2 libapache2-mod-wsgi-py3 mariadb-client mariadb-server python3-mysqldb git python3-venv
{% endhighlight %}

Una vez instalados los paquetes necesarios, vamos a clonar el repositorio al igual que hicimos en el entorno de desarrollo, llevandola a cabo en este caso dentro del directorio **/srv**, por lo que me he movido dentro del mismo haciendo uso de `cd`. Para esta ocasión, utilizaremos el protocolo **HTTPS**, ya que en esta nueva máquina virtual no tengo un par de claves SSH configurado para realizar la clonación mediante dicho protocolo. Para ello, haremos uso del comando:

{% highlight shell %}
root@django:/srv# git clone https://github.com/alvarovaca/django_tutorial.git
Cloning into 'django_tutorial'...
remote: Enumerating objects: 37, done.
remote: Counting objects: 100% (37/37), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 129 (delta 4), reused 24 (delta 3), pack-reused 92
Receiving objects: 100% (129/129), 4.25 MiB | 5.77 MiB/s, done.
Resolving deltas: 100% (28/28), done.
{% endhighlight %}

Como consecuencia de haber hecho uso de **HTTPS**, no se habrá pedido ningún tipo de credenciales para proceder con la clonación del repositorio. Tras ello, vamos a listar el contenido del directorio actual para así verificar que la clonación ha sido efectiva:

{% highlight shell %}
root@django:/srv# ls
django_tutorial
{% endhighlight %}

Efectivamente, el repositorio de nombre **django_tutorial** ha sido correctamente clonado en nuestra máquina virtual y podremos empezar a hacer uso del mismo.

Al igual que en el entorno de desarrollo, generaremos un entorno virtual en el que instalaremos las dependencias necesarias para el funcionamiento de la aplicación web, que en esta ocasión he decidido que tendrá también el nombre **despliegue** y estará ubicado en el directorio personal del usuario **debian**, por lo que para llevar a cabo la creación, ejecutaremos el comando:

{% highlight shell %}
root@django:/srv# python3 -m venv /home/debian/despliegue
{% endhighlight %}

Una vez creado, tendremos que iniciarlo haciendo uso de `source` con el binario **activate** que se encuentra contenido en el directorio **bin/**:

{% highlight shell %}
root@django:/srv# source /home/debian/despliegue/bin/activate
{% endhighlight %}

Nuestro entorno virtual habrá sido habilitado y en consecuencia, podremos apreciar que el _prompt_ ha cambiado.

Ya está todo listo para instalar los paquetes necesarios, haciendo uso una vez más del fichero **requirements.txt** en el que se encuentra indicada la lista de dependencias necesarias. Dicho fichero se encuentra en el interior del repositorio que hemos clonado, por lo que nos moveremos al mismo haciendo uso de `cd`:

{% highlight shell %}
(despliegue) root@django:/srv# cd django_tutorial/
{% endhighlight %}

Una vez dentro de dicho directorio, procederemos con la instalación haciendo uso de `pip`, al igual que en el entorno de desarrollo:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# pip install -r requirements.txt
Collecting asgiref==3.3.0 (from -r requirements.txt (line 1))
  Using cached https://files.pythonhosted.org/packages/c0/e8/578887011652048c2d273bf98839a11020891917f3aa638a0bc9ac04d653/asgiref-3.3.0-py3-none-any.whl
Collecting Django==3.1.3 (from -r requirements.txt (line 2))
  Using cached https://files.pythonhosted.org/packages/7f/17/16267e782a30ea2ce08a9a452c1db285afb0ff226cfe3753f484d3d65662/Django-3.1.3-py3-none-any.whl
Collecting pytz==2020.4 (from -r requirements.txt (line 3))
  Using cached https://files.pythonhosted.org/packages/12/f8/ff09af6ff61a3efaad5f61ba5facdf17e7722c4393f7d8a66674d2dbd29f/pytz-2020.4-py2.py3-none-any.whl
Collecting sqlparse==0.4.1 (from -r requirements.txt (line 4))
  Using cached https://files.pythonhosted.org/packages/14/05/6e8eb62ca685b10e34051a80d7ea94b7137369d8c0be5c3b9d9b6e3f5dae/sqlparse-0.4.1-py3-none-any.whl
Installing collected packages: asgiref, pytz, sqlparse, Django
Successfully installed Django-3.1.3 asgiref-3.3.0 pytz-2020.4 sqlparse-0.4.1
{% endhighlight %}

Además, del módulo **python3-mysqldb** instalado con anterioridad, debemos instalar un paquete adicional con `pip` para permitir que Python trabaje con MySQL, de nombre **mysql-connector-python**, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# pip install mysql-connector-python
Collecting mysql-connector-python
  Using cached https://files.pythonhosted.org/packages/8f/eb/4b449ee81b14aada746aa292481d3d522aa20b41cb124b24d6205251dcce/mysql_connector_python-8.0.22-cp37-cp37m-manylinux1_x86_64.whl
Collecting protobuf>=3.0.0 (from mysql-connector-python)
  Using cached https://files.pythonhosted.org/packages/e0/dd/5c5d156ee1c4dba470d76dac5ae57084829b4e17547f28e9f636ce3fa54b/protobuf-3.14.0-cp37-cp37m-manylinux1_x86_64.whl
Collecting six>=1.9 (from protobuf>=3.0.0->mysql-connector-python)
  Using cached https://files.pythonhosted.org/packages/ee/ff/48bde5c0f013094d729fe4b0316ba2a24774b3ff1c52d924a8a4cb04078a/six-1.15.0-py2.py3-none-any.whl
Installing collected packages: six, protobuf, mysql-connector-python
Successfully installed mysql-connector-python-8.0.22 protobuf-3.14.0 six-1.15.0
{% endhighlight %}

Toda la paquetería necesaria para el correcto funcionamiento de la aplicación web en el entorno de producción ha sido ya instalada, de manera que el siguiente paso será crear una base de datos MySQL y un usuario con privilegios sobre la misma, que será utilizada por las aplicaciones para generar sus tablas.

Lo primero que debemos hacer es acceder al servidor **mysql**, ejecutando para ello el comando (cuando nos pida una contraseña, no introduciremos nada, simplemente pulsaremos ENTER):

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 38
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Donde:

* **-u**: Indica el usuario con el que nos vamos a conectar, en este caso, **root**.
* **-p**: Indicamos la contraseña, en este caso, no introducimos ninguna.

Una vez dentro del servidor de bases de datos, tendremos que crear una nueva base de datos, valga la redundancia. En este caso, para identificarla fácilmente, de nombre **bd_django**. Para ello, ejecutamos el comando:

{% highlight sql %}
MariaDB [(none)]> CREATE DATABASE bd_django;
Query OK, 1 row affected (0.001 sec)
{% endhighlight %}

Para verificar que la base de datos ha sido correctamente creada, haremos uso de la instrucción:

{% highlight sql %}
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| bd_django          |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.001 sec)
{% endhighlight %}

La base de datos que usará la aplicación web se encuentra ya correctamente creada, pero de nada nos sirve tener una base de datos si no tenemos un usuario que pueda acceder a la misma.

En este caso, vamos a crear un usuario de nombre "**usuario**" que tenga permitido el acceso desde **localhost** (es decir, desde la máquina local) y cuya contraseña sea "**usuario**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso del comando:

{% highlight sql %}
MariaDB [(none)]> CREATE USER 'usuario'@'localhost' IDENTIFIED BY 'usuario';
Query OK, 0 rows affected (0.010 sec)
{% endhighlight %}

El usuario ya ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos el comando:

{% highlight sql %}
MariaDB [(none)]> GRANT ALL PRIVILEGES ON bd_django.* TO 'usuario'@'localhost';
Query OK, 0 rows affected (0.001 sec)
{% endhighlight %}

Una vez realizadas todas las modificaciones oportunas, podremos salir de _mysql_ haciendo uso del comando:

{% highlight sql %}
MariaDB [(none)]> quit
Bye
{% endhighlight %}

**Nota**: No es necesario hacer uso del comando `FLUSH PRIVILEGES;`, a diferencia de varios artículos que he estado leyendo en Internet, que usan dicho comando muy a menudo sin necesidad alguna. Dicho comando, lo que hace es recargar la caché de las tablas GRANT que se encuentran en memoria, recarga que se hace de forma automática al hacer uso de una sentencia **GRANT**, de manera que no es necesario hacerlo manualmente. Para aclararlo, dicho comando únicamente debe utilizarse tras modificar las tablas GRANT de manera indirecta, es decir, tras usar sentencias INSERT, UPDATE o DELETE.

La base de datos y el usuario con permisos sobre la misma ya han sido generados, pero si recordamos, el fichero de configuración **django_tutorial/settings.py** viene por defecto establecido para utilizar una base de datos **sqlite3**, de manera que tendremos que realizar las modificaciones oportunas para que haga uso de la nueva base de datos **MySQL**. Para modificar dicho fichero, ejecutaremos el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# nano django_tutorial/settings.py
{% endhighlight %}

Dentro de la sección **DATABASES** tendremos que establecer la siguiente configuración:

* El motor (**ENGINE**): **mysql.connector.django**
* El nombre de la base de datos (**NAME**): **bd_django**
* El usuario con permisos sobre dicha base de datos (**USER**): **usuario**
* La contraseña de dicho usuario (**PASSWORD**): **usuario**
* La dirección de la máquina en la que se encuentra la base de datos (**HOST**): **localhost**
* El puerto en el que está escuchando peticiones (**PORT**): **3306**

El resultado final sería:

{% highlight shell %}
DATABASES = {
    'default': {
        'ENGINE': 'mysql.connector.django',
        'NAME': 'bd_django',
        'USER': 'usuario',
        'PASSWORD': 'usuario',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
{% endhighlight %}

La configuración de dicho fichero no ha finalizado todavía, ya que tendremos que permitir además el nombre de dominio a través del cuál vamos a acceder a la aplicación, que posteriormente tendremos que configurar en el VirtualHost que creemos. En mi caso, he decidido que el nombre de dominio será **www.alvarodjango.com**, por lo que lo añadiremos a la directiva **ALLOWED_HOSTS**, quedando finalmente de la siguiente forma:

{% highlight shell %}
ALLOWED_HOSTS = ['www.alvarodjango.com']
{% endhighlight %}

Tras ello, guardaremos los cambios, de manera que una vez que la base de datos haya sido correctamente configurada para su uso, tendremos que aplicar las migraciones no aplicadas, al igual que hicimos en el entorno de desarrollo, para así generar en la base de datos las correspondientes tablas de las que harán uso las aplicaciones existentes, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying polls.0001_initial... OK
  Applying sessions.0001_initial... OK
{% endhighlight %}

Vamos a volver a acceder al servidor de bases de datos para así verificar que las tablas han sido correctamente generadas, ejecutando para conectarnos el comando visto con anterioridad:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 52
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Al no haber especificado explícitamente a qué base de datos nos queremos conectar, no nos ha conectado a ninguna, por lo que haremos uso de la instrucción `USE` para cambiarnos a la base de datos **bd_django**:

{% highlight sql %}
MariaDB [(none)]> USE bd_django;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Cuando nos encontremos haciendo uso de la base de datos correcta, vamos a proceder a listar las tablas existentes, ejecutando para ello la instrucción:

{% highlight sql %}
MariaDB [bd_django]> SHOW TABLES;
+----------------------------+
| Tables_in_bd_django        |
+----------------------------+
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
| polls_choice               |
| polls_question             |
+----------------------------+
12 rows in set (0.001 sec)
{% endhighlight %}

Efectivamente, las correspondientes tablas a ser usadas por las aplicaciones han sido correctamente generadas, por lo que podremos salir del servidor de bases de datos ejecutando una vez más la instrucción `quit`.

Nos queda un último paso antes de proceder a configurar el servidor web, consistente en crear un usuario administrador para dicho proyecto Django, que tendrá privilegios suficientes para acceder a la zona de administración, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# python3 manage.py createsuperuser
Username (leave blank to use 'root'): alvaro
Email address: avacaferreras@gmail.com
Password: 
Password (again): 
Superuser created successfully.
{% endhighlight %}

Al igual que en la anterior ocasión, se nos ha solicitado un nombre de usuario para el mismo, un correo electrónico y una contraseña.

Tras ello, todo estará listo para proceder a crear el correspondiente fichero de configuración para el VirtualHost que será utilizado para la aplicación web, que en este caso, tendrá nombre **django**, así que generaremos un fichero de nombre **django.conf** dentro de **/etc/apache2/sites-available/** con el siguiente contenido:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# nano /etc/apache2/sites-available/django.conf
{% endhighlight %}

{% highlight shell %}
<VirtualHost *:80>
    ServerName www.alvarodjango.com

    ServerAdmin webmaster@localhost
    DocumentRoot /srv/django_tutorial
    WSGIDaemonProcess flask_temp user=www-data group=www-data processes=1 threads=5 python-path=/srv/django_tutorial:/home/debian/despliegue/lib/python3.7/site-packages
    WSGIScriptAlias / /srv/django_tutorial/django_tutorial/wsgi.py

    Alias /static/ /srv/django_tutorial/static/

    <Directory /srv/django_tutorial>
      WSGIProcessGroup flask_temp
      WSGIApplicationGroup %{GLOBAL}
      Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
{% endhighlight %}

Donde:

* **ServerName**: Indicamos el nombre de dominio a través del cuál accederemos al servidor web, que tendrá que coincidir con aquel especificado en la directiva **ALLOWED_HOSTS**. En este caso, **www.alvarodjango.com**.
* **DocumentRoot**: Directorio en el que se encuentran contenidos los ficheros de la aplicación web. Realmente, el servidor web siempre va a llamar al fichero WSGI, pero es necesario indicarlo por si hay contenido estático a servir. En este caso, **/srv/django_tutorial**.
* **WSGIDaemonProcess**: Se define un grupo de procesos que se van a encargar de ejecutar la aplicación (servidor de aplicaciones). A dicho grupo se le asigna un nombre, en este caso **flask_temp** y un usuario (**user**) y grupo (**group**) a utilizar, que en este caso es el mismo que el del servidor web, **www-data**. Se indica además el número de procesos (**processes**) e hilos (**threads**) que va a tener cada proceso. Por último, se indica en la directiva **python-path** el directorio en el que se encuentran los ficheros de la aplicación web (en este caso, **/srv/django_tutorial**), así como el directorio donde se encuentran los paquetes en el entorno virtual (en este caso, **/home/debian/despliegue/lib/python3.7/site-packages**), separados por dos puntos.
* **WSGIScriptAlias**: Nos permite indicar la ruta del programa que se va a ejecutar, es decir, el fichero WSGI del que se va a hacer uso cuando se haga una petición. En este caso, **/srv/django_tutorial/django_tutorial/wsgi.py**.
* **Alias**: Establecemos un Alias para poder servir el contenido estático de la aplicación web. Lo veremos con más detalle a continuación.
* **Directory**: Nos permite asignar el grupo de procesos creado anteriormente de nombre **flask_temp** al directorio en el que se encuentra contenida nuestra aplicación.

Si no has entendido parte de la configuración llevada a cabo, es porque todavía no has leído el artículo [VirtualHosting con Apache](https://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html). Tras ello, tendremos que habilitar el nuevo VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# a2ensite django
Enabling site django.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

Como hemos mencionado, la creación del Alias ha sido necesaria para poder servir el contenido estático de las aplicaciones existentes, ya que por defecto, trata de buscar dicho contenido (como puede ser la hoja de estilo) dentro del directorio **static/**, de manera que tendremos que generar un directorio con dicho nombre para introducir en el mismo subdirectorios con el contenido estático a servir para cada una de las aplicaciones.

En este caso, las aplicaciones que necesitan servir contenido estático son **admin** y **polls**, por lo que crearemos subdirectorios con dichos nombres, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# mkdir -p static/{admin,polls}
{% endhighlight %}

Donde:

* **-p**: Indica que se cree el directorio padre en caso de no existir, en este caso, se creará el directorio **static/**.

Para verificar que la generación de dichos directorios se ha llevado a cabo correctamente, listaremos el contenido de **static/** de forma gráfica y recursiva haciendo uso de `tree`:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# tree static/
static/
├── admin
└── polls

2 directories, 0 files
{% endhighlight %}

Efectivamente, la estructura ha sido generada, así que únicamente nos quedaría introducir el contenido necesario a servir.

* En el caso de la aplicación **admin**, se sirve un contenido estático propio del proyecto Django, es decir, se encuentra dentro de los ficheros del entorno virtual (concretamente en **/home/debian/despliegue/lib/python3.7/site-packages/django/contrib/admin/static/admin/**).
* En el caso de la aplicación **polls**, el contenido estático a servir se encuentra en el subdirectorio generado por la propia aplicación, concretamente en **polls/static/polls/**.

Tendremos que copiar dichos contenidos dentro de los directorios correspondientes, ejecutando para ello los comandos:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# cp -r /home/debian/despliegue/lib/python3.7/site-packages/django/contrib/admin/static/admin/* static/admin/
(despliegue) root@django:/srv/django_tutorial# cp -r polls/static/polls/* static/polls/
{% endhighlight %}

Donde:

* **-r**: Indica que la copia se lleve a cabo de forma recursiva, copiando los directorios y el contenido del interior de los mismos.

El último paso será volver a cargar la configuración de _apache2_ en memoria para que así surtan efecto los nuevos cambios realizados, entre los que se encuentran el nuevo VirtualHost que hemos generado. Para ello, haremos uso del comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# systemctl reload apache2
{% endhighlight %}

Una vez cargada la nueva configuración, volveremos a la máquina anfitriona y configuraremos la resolución estática de nombres en la misma, para así poder realizar la traducción del nombre a la IP de la máquina y que así podamos acceder, gracias a la cabecera Host de la petición HTTP. Para llevar a cabo dicha configuración, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** para el nuevo sitio que hemos creado, siendo **[IP]** la dirección IP de la máquina servidora (en este caso **172.22.200.186**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente, de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
172.22.200.186  www.alvarodjango.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para así verificar que la aplicación web se encuentra operativa, accediendo concretamente a la zona de administración (**/admin**):

![admin6](https://i.ibb.co/M290MGd/servidor1.jpg "www.alvarodjango.com")

Como podemos apreciar, se ha podido acceder sin problema alguno, de manera que nos _loguearemos_ para comprobar que el acceso a la base de datos MySQL también funciona como debería (ya que internamente consultará la tabla referente a los usuarios):

![admin7](https://i.ibb.co/nr4NhRY/servidor2.jpg "www.alvarodjango.com")

Efectivamente, su funcionamiento es el esperado, así que he creado manualmente una nueva encuesta y he accedido a **/polls**, para asegurarme al igual que con **/admin**, que el contenido estático está siendo correctamente servido:

![polls4](https://i.ibb.co/9wRdcVq/servidor3.jpg "www.alvarodjango.com")

La aplicación **polls** también está funcionando correctamente, a la vez que siendo correctamente servido su contenido estático, por lo que podemos concluir que el despliegue ha sido efectivo.

Antes de pasar a la tercera y última parte del artículo, me gustaría deshabilitar el modo **DEBUG**, para que así los posibles errores de ejecución no den información sensible sobre la aplicación, pues podría ser algo peligroso al tratarse del entorno de producción.

Para ello, volveremos a la máquina virtual y modificaremos el valor de la directiva **DEBUG** de **True** a **False** en el fichero **django_tutorial/settings.py**, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# sed -i 's/DEBUG = True/DEBUG = False/g' django_tutorial/settings.py
{% endhighlight %}

Para verificar que el cambio se ha producido correctamente, vamos a visualizar el contenido de dicho fichero, estableciendo a su vez un filtro por nombre:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# cat django_tutorial/settings.py | egrep 'DEBUG'
DEBUG = False
{% endhighlight %}

Como era de esperar, el valor de la directiva **DEBUG** es ahora **False**, de manera que tendremos que volver a cargar la configuración del servicio en memoria para que el nuevo cambio surta efecto:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# systemctl reload apache2
{% endhighlight %}

El modo **DEBUG** ha sido ya desactivado, por lo que podremos dar paso a la siguiente parte del artículo.

## Tarea 3: Modificación de nuestra aplicación

Ya tenemos los dos entornos totalmente operativos, de manera que vamos a llevar a cabo una tarea que simula la realidad a la perfección, en la que un grupo de desarrolladores (en este caso, yo) crean una nueva versión de la aplicación y es necesario desplegarla en producción tras haber verificado que el funcionamiento es el correcto y que no hay errores.

Para ello, vamos a llevar a cabo una serie de modificaciones de ejemplo, que como varias veces he mencionado, serán en el entorno de desarrollo, ya que en el de producción lo tenemos prohibido.

La primera de ellas consistirá en modificar la imagen de fondo que se muestra en la aplicación **polls**, por lo que tendremos que modificar el contenido estático que se sirve para la misma. Si recordamos, el contenido estático de dicha aplicación se encuentra en un subdirectorio de nombre **static/** en el entorno de producción, mientras que en el entorno de desarrollo, sigue estando en el propio subdirectorio de la aplicación **polls/**, de manera que a la hora de llevar los cambios al entorno de producción, tendremos que mover el fichero manualmente al directorio que está siendo utilizado para contener los ficheros a servir de manera estática, o bien, tratar de conseguir que el entorno de producción y de desarrollo tuviesen en todo momento la estructura más similar posible, para así evitar este problema.

En el entorno de desarrollo, la imagen de fondo de la aplicación **polls** se encuentra contenida en **polls/static/polls/images/** con el nombre **background.jpg**, de manera que haremos uso de `wget` para descargar una imagen cualquiera en dicho directorio con el mismo nombre, para así sobreescribirla:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ wget -P polls/static/polls/images/ -O background.jpg 'https://hackercar.com/wp-content/uploads/2020/04/encuestas-coronavirus.jpg'
--2020-12-08 12:17:05--  https://hackercar.com/wp-content/uploads/2020/04/encuestas-coronavirus.jpg
Resolviendo hackercar.com (hackercar.com)... 35.214.200.43
Conectando con hackercar.com (hackercar.com)[35.214.200.43]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 154188 (151K) [image/jpeg]
Grabando a: “background.jpg”

                                 100%[==========================>]  25,89K  --.-KB/s    en 0,01s

2020-12-08 12:17:05 (2,53 MB/s) - “background.jpg” guardado [154188/154188]
{% endhighlight %}

Donde:

* **-P**: Indicamos la ruta en la que se va a llevar a cabo la descarga. En este caso, **polls/static/polls/images/**.
* **-O**: Indicamos el nombre del archivo al que vamos a renombrar la descarga. En este caso, **background.jpg**.

Para verificar que la descarga se ha llevado a cabo correctamente y con el nombre adecuado, vamos a listar el contenido de dicho directorio, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ ls polls/static/polls/images/
background.gif  background.jpg
{% endhighlight %}

Efectivamente, el fichero descargado ha sobreescrito al anterior, por lo que la imagen de fondo a servir ha sido modificada.

La segunda modificación que llevaremos a cabo será también sobre la aplicación **polls**, añadiendo nuestro nombre a la página que se nos muestra al acceder a **/polls**, de manera que tendremos que modificar el fichero de nombre **index.html** en el subdirectorio **polls/templates/**, es decir, aquel directorio utilizado para albergar las plantillas que se utilizan para generar el HTML que será devuelto al cliente a la hora de la petición. Para ello, ejecutaremos el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ nano polls/templates/polls/index.html
{% endhighlight %}

Dentro de dicho fichero se hacen uso de determinadas sentencias como bucles o condiciones, utilizadas para generar el código HTML de forma dinámica. En mi caso, he añadido una etiqueta **h2** en la que he contenido mi nombre de la siguiente forma:

{% highlight html %}
<h2>Álvaro Vaca Ferreras</h2>
{% endhighlight %}

Cuando hayamos finalizado, guardaremos los cambios realizados y procederemos a comprobar que todo funciona como debería, de manera que vamos a arrancar el servidor web ligero que nos proporciona Django para así ver el resultado final, ejecutando para ello el comando visto con anterioridad:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
December 08, 2020 - 11:18:12
Django version 3.1.3, using settings 'django_tutorial.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
{% endhighlight %}

Como podemos apreciar en la salida, el servidor web de desarrollo está actualmente ejecutándose en la interfaz _loopback_ (**127.0.0.1**), concretamente en el puerto **8000**, así que abriremos un navegador web y accederemos al mismo, concretamente a la aplicación de encuestas (**/polls**):

![polls5](https://i.ibb.co/D4NVMgV/Captura-de-pantalla-de-2020-12-08-12-18-44.png "Aplicación de encuestas")

Efectivamente, los cambios han surtido efecto, pues actualmente se está mostrando mi nombre en dicha página web, además de haber sido modificada la imagen de fondo servida estáticamente.

Ya hemos llevado a cabo dos modificaciones, así que vamos a realizar una última, que va a ser la más compleja, ya que vamos a modificar la estructura de la base de datos en desarrollo, concretamente creando una nueva tabla, para posteriormente desplegar dichos cambios en producción, que como ya hemos mencionado, sería una tarea muy compleja si no contásemos con el mecanismo de **migraciones** de Django.

La aplicación que vamos a modificar va a ser **polls**, añadiendo a la base de datos una nueva tabla de la que hará uso dicha aplicación.

Para ello, lo primero que tendremos que hacer será añadir un nuevo modelo al fichero **polls/models.py**, fichero en el que definiremos las diferentes clases (tablas) de las que hará uso la aplicación, junto con los atributos (columnas) que contendrá dicha tabla.

No es necesario conocer la estructura de la tabla que vamos a añadir, ya que únicamente queremos mostrar el proceso de despliegue. Para modificar dicho fichero, haremos uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ nano polls/models.py
{% endhighlight %}

La clase a añadir será la siguiente:

{% highlight shell %}
class Categoria(models.Model):
    Abr = models.CharField(max_length=4)
    Nombre = models.CharField(max_length=50)

    def __str__(self):
        return self.Abr+" - "+self.Nombre
{% endhighlight %}

Ya hemos añadido un nuevo modelo, pero eso no significa que la nueva tabla ya haya sido creada en la base de datos en desarrollo, ya que para ello, tendremos que generar un fichero con las instrucciones genéricas a reproducir para generarla.

Es por ello que introducimos **makemigrations**, un comando que hará uso del modelo que acabamos de introducir para así generar un fichero de migración que posteriormente aplicaremos con el comando **migrate**, con la finalidad de aplicar dichos cambios a la base de datos. Para crear la nueva migración, ejecutaremos por tanto el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py makemigrations
Migrations for 'polls':
  polls/migrations/0002_categoria.py
    - Create model Categoria
{% endhighlight %}

Como se puede apreciar, ha detectado el nuevo modelo que hemos introducido y ha generado el correspondiente fichero de migración en **polls/migrations/**, con el nombre **0002_categoria.py** que posteriormente aplicaremos con **migrate** para así reproducir los cambios correspondientes en la base de datos:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying polls.0002_categoria... OK
{% endhighlight %}

Efectivamente, la migración no aplicada ha sido ahora aplicada y por tanto, se habrá generado la correspondiente tabla en la base de datos de desarrollo.

Sin embargo, todavía no hemos finalizado, ya que es muy posible que queramos manipular el modelo que acabamos de generar desde la zona de administración, por lo que tendremos que notificar el nuevo cambio en el fichero **polls/admin.py**, para que así aparezca en la misma. Para modificar dicho fichero haremos uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ nano polls/admin.py
{% endhighlight %}

Dentro del mismo, encontraremos una línea en la que se importan aquellos modelos existentes (actualmente únicamente aparecen **Choice** y **Question**, que si recordamos, son las dos únicas tablas existentes en la base de datos pertenecientes a dicha aplicación), de la siguiente forma:

{% highlight shell %}
from .models import Choice, Question
{% endhighlight %}

Como es lógico, tendremos que añadir el modelo **Categoria** que acabamos de crear a dicha línea, siendo el resultado final:

{% highlight shell %}
from .models import Choice, Question, Categoria
{% endhighlight %}

Además de añadirlo a la lista, tendremos que introducir una nueva línea en el final del fichero para que así se muestre en la zona de administración, de la siguiente forma:

{% highlight shell %}
admin.site.register(Categoria)
{% endhighlight %}

Tras ello, guardaremos los cambios, de manera que ya habremos llevado a cabo todas las modificaciones que había planteado, por lo que una vez que el equipo de desarrollo haya comprobado con detalle que no hay ningún _bug_, es hora de desplegar dichos cambios en el entorno de producción, que como anteriormente hemos mencionado, haremos uso de **git**.

Lo primero que haremos será mostrar el estado actual de nuestra rama local existente en el entorno de desarrollo con respecto a aquella de nuestro repositorio GitHub, para que así nos notifique de los cambios que hemos llevado a cabo y que podemos potencialmente subir. Para ello, ejecutaremos el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ git status
En la rama master
Tu rama está actualizada con 'origin/master'.

Cambios no rastreados para el commit:
  (usa "git add <archivo>..." para actualizar lo que será confirmado)
  (usa "git checkout -- <archivo>..." para descartar los cambios en el directorio de trabajo)

	modificado:     polls/admin.py
	modificado:     polls/models.py
	modificado:     polls/static/polls/images/background.jpg
	modificado:     polls/templates/polls/index.html

Archivos sin seguimiento:
  (usa "git add <archivo>..." para incluirlo a lo que se será confirmado)

	polls/migrations/0002_categoria.py

sin cambios agregados al commit (usa "git add" y/o "git commit -a")
{% endhighlight %}

Como se puede apreciar, se han mostrado aquellos ficheros que hemos modificado o añadido, de manera que tras verificar que son correctos, tendremos que marcarlos para así subir dichos cambios. Para ello, haremos uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ git add .
{% endhighlight %}

Todos los ficheros habrán sido incluidos para el **commit** que vamos a llevar a cabo, es decir, vamos a hacer una especie de "foto" del estado actual de los ficheros marcados (control de versiones), para así subirla posteriormente a GitHub, de manera que en un futuro, en caso de ser necesario, podríamos volver atrás en las versiones. Para ello, ejecutaremos el comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ git commit -am "Migración."
[master 80551be] Migración.
 5 files changed, 33 insertions(+), 2 deletions(-)
 create mode 100644 polls/migrations/0002_categoria.py
 rewrite polls/static/polls/images/background.jpg (99%)
{% endhighlight %}

En el **commit** hemos establecido el mensaje "**Migración.**", que servirá para diferenciarlo del resto, por lo que siempre es recomendable que sea descriptivo (no como en este caso).

Nos queda el último paso, subir dichas modificaciones a nuestro repositorio GitHub, ya que actualmente, se encuentran almacenadas de forma local. Para ello, haremos uso del comando:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/django_tutorial$ git push
Enumerando objetos: 26, listo.
Contando objetos: 100% (26/26), listo.
Compresión delta usando hasta 12 hilos
Comprimiendo objetos: 100% (12/12), listo.
Escribiendo objetos: 100% (14/14), 27.18 KiB | 4.53 MiB/s, listo.
Total 14 (delta 4), reusado 0 (delta 0)
remote: Resolving deltas: 100% (4/4), completed with 4 local objects.
To github.com:alvarovaca/django_tutorial.git
   c10187a..80551be  master -> master
{% endhighlight %}

Nuestra labor en el entorno de desarrollo habrá finalizado, así que volveremos por último al entorno de producción, con la intención de desplegar los cambios realizados en la aplicación.

Como es lógico, el primer paso será descargar dichas modificaciones llevadas a cabo en el entorno de desarrollo en la máquina actual, que se encuentran actualmente disponibles en nuestro repositorio común, por lo que tendremos que llevar a cabo un **pull**, ejecutando para el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# git pull
remote: Enumerating objects: 26, done.
remote: Counting objects: 100% (26/26), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 14 (delta 4), reused 14 (delta 4), pack-reused 0
Unpacking objects: 100% (14/14), done.
From https://github.com/alvarovaca/django_tutorial
 + 80551be...70db539 master     -> origin/master
Updating c10187a..70db539
Fast-forward
 polls/admin.py                           |   3 ++-
 polls/migrations/0002_categoria.py       |  21 +++++++++++++++++++++
 polls/models.py                          |   7 +++++++
 polls/static/polls/images/background.jpg | Bin 46418 -> 26508 bytes
 polls/templates/polls/index.html         |   4 +++-
 5 files changed, 33 insertions(+), 2 deletions(-)
 create mode 100644 polls/migrations/0002_categoria.py
{% endhighlight %}

Como se puede apreciar, se han descargado únicamente aquellos ficheros que han sido modificados o añadidos en el entorno de desarrollo, evitando así transferencias innecesarias, además de haberse mostrado "gráficamente" cuáles han sido los que más modificaciones han sufrido, algo que en nuestro caso, no nos es relevante.

Una vez que todos los ficheros hayan sido descargados, tendremos que aplicar las migraciones no aplicadas (en este caso, aquella que acabamos de crear en el entorno de desarrollo), para así generar en nuestra base de datos MySQL del entorno de producción la tabla en cuestión de la que hará uso la aplicación **polls**, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying polls.0002_categoria... OK
{% endhighlight %}

Vamos a volver a acceder al servidor de bases de datos para así verificar que la tabla ha sido correctamente generada, ejecutando para conectarnos el comando visto con anterioridad:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 202
Server version: 10.3.25-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
{% endhighlight %}

Al no haber especificado explícitamente a qué base de datos nos queremos conectar, no nos ha conectado a ninguna, por lo que haremos uso de la instrucción `USE` para cambiarnos a la base de datos **bd_django**:

{% highlight sql %}
MariaDB [(none)]> USE bd_django;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
{% endhighlight %}

Cuando nos encontremos haciendo uso de la base de datos correcta, vamos a proceder a listar las tablas existentes, ejecutando para ello la instrucción:

{% highlight sql %}
MariaDB [bd_django]> SHOW TABLES;
+----------------------------+
| Tables_in_bd_django        |
+----------------------------+
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
| polls_categoria            |
| polls_choice               |
| polls_question             |
+----------------------------+
13 rows in set (0.000 sec)
{% endhighlight %}

Efectivamente, la tabla **polls_categoria**, consecuencia de la nueva migración, ha sido correctamente generada, por lo que podremos salir del servidor de bases de datos ejecutando una vez más la instrucción `quit`.

Si recordamos, en el entorno de producción el contenido estático a servir lo encontramos dentro de **static/**, mientras que a la hora de hacer el **pull**, la imagen de fondo de la aplicación **polls** que previamente hemos modificado se habrá descargado en **polls/static/polls/images/**, por lo que la copiaremos al directorio correcto para que así pueda ser servida, ejecutando para ello el comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# cp polls/static/polls/images/background.jpg static/polls/images/
{% endhighlight %}

Listo, ya hemos finalizado la migración, a falta de volver a cargar la configuración del servicio _apache2_ en memoria para que así surtan efecto los cambios realizados, haciendo para ello uso del comando:

{% highlight shell %}
(despliegue) root@django:/srv/django_tutorial# systemctl reload apache2
{% endhighlight %}

Tras ello, podremos acceder al navegador e introducir el nombre de dominio para así verificar que la aplicación web se encuentra operativa tras la migración y que los cambios han surtido efecto, accediendo concretamente a la aplicación de encuestas (**/polls**):

![polls6](https://i.ibb.co/99Sd0Q7/Captura-de-pantalla-de-2020-12-08-16-04-54.png "www.alvarodjango.com")

Efectivamente, los cambios han surtido efecto, pues actualmente se está mostrando mi nombre en dicha página web, además de haber sido modificada la imagen de fondo servida estáticamente.

Por último, accederemos a la zona de administración (**/admin**) y nos _loguearemos_, para así verificar además que el nuevo modelo se está mostrando en dicha aplicación, de manera que podríamos manipularlo desde ahí:

![admin8](https://i.ibb.co/tmyk2tV/Captura-de-pantalla-de-2020-12-08-16-05-01.png "www.alvarodjango.com")

Como se puede apreciar, dentro de la sección **Polls** se ha mostrado el nuevo modelo **Categorias**, de manera que podemos asegurar que la migración se ha completado satisfactoriamente y que toda la configuración llevada a cabo es correcta.