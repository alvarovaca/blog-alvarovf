---
layout: post
title:  "Instalación de aplicaciones web Python"
banner: "/assets/images/banners/django.jpg"
date:   2021-01-12 23:42:00 +0200
categories: aplicaciones openstack
---
Creamos y activamos el entorno virtual en el que instalaremos **Mezzanine**:

{% highlight shell %}
alvaro@debian:~/virtualenv$ python3 -m venv despliegue

alvaro@debian:~/virtualenv$ source despliegue/bin/activate
{% endhighlight %}

Instalamos **mezzanine** mediante **pip** y creamos un proyecto "**cms**", que generará su directorio con los ficheros necesarios:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub$ pip install mezzanine

(despliegue) alvaro@debian:~/GitHub$ mezzanine-project cms
{% endhighlight %}

Listamos el contenido del nuevo directorio para apreciar los ficheros generados:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ ls -l
total 40
drwxr-xr-x 2 alvaro alvaro  4096 feb  1 11:06 cms
drwxr-xr-x 2 alvaro alvaro  4096 feb  1 11:06 deploy
-rw-r--r-- 1 alvaro alvaro 22057 feb  1 11:06 fabfile.py
-rw-r--r-- 1 alvaro alvaro   367 feb  1 11:06 manage.py
-rw-r--r-- 1 alvaro alvaro    17 feb  1 11:06 requirements.txt
{% endhighlight %}

Listamos el contenido del subdirectorio **cms** para apreciar los ficheros generados:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ ls -l cms/
total 28
-rw-r--r-- 1 alvaro alvaro     0 feb  1 11:06 __init__.py
-rw-r--r-- 1 alvaro alvaro  1643 feb  1 11:06 local_settings.py
-rw-r--r-- 1 alvaro alvaro 11778 feb  1 11:06 settings.py
-rw-r--r-- 1 alvaro alvaro  4385 feb  1 11:06 urls.py
-rw-r--r-- 1 alvaro alvaro   483 feb  1 11:06 wsgi.py
{% endhighlight %}

Modificamos el fichero **cms/settings.py** para utilizar una base de datos sqlite3, ya que estamos en desarrollo:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ nano cms/settings.py

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": "",
        "USER": "",
        "PASSWORD": "",
        "HOST": "",
        "PORT": "",
    }
}
{% endhighlight %}

Hacemos una migración para así generar las tablas en la base de datos sqlite3:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, blog, conf, contenttypes, core, django_comments, forms, galleries, generic, pages, redirects, sessions, sites, twitter
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  ...
  Applying pages.0004_auto_20170411_0504... OK
  Applying redirects.0001_initial... OK
  Applying sessions.0001_initial... OK
  Applying sites.0002_alter_domain_unique... OK
  Applying twitter.0001_initial... OK
{% endhighlight %}

Creamos un superusuario en la aplicación para así poder gestionarla:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ python3 manage.py createsuperuser
Username (leave blank to use 'alvaro'): 
Email address: avacaferreras@gmail.com
Password: 
Password (again): 
Superuser created successfully.
{% endhighlight %}

Ejecutamos el servidor web incluido en el framework Django para así servir de forma local la aplicación:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ python3 manage.py runserver
              .....
          _d^^^^^^^^^b_
       .d''           ``b.
     .p'                `q.
    .d'                   `b.
   .d'                     `b.   * Mezzanine 4.3.1
   ::                       ::   * Django 1.11.29
  ::    M E Z Z A N I N E    ::  * Python 3.7.3
   ::                       ::   * SQLite 3.27.2
   `p.                     .q'   * Linux 5.7.0-0.bpo.2-amd64
    `p.                   .q'
     `b.                 .d'
       `q..          ..p'
          ^q........p^
              ''''

Performing system checks...

System check identified no issues (0 silenced).
February 01, 2021 - 10:08:21
Django version 1.11.29, using settings 'cms.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
{% endhighlight %}

Accedemos a **localhost** en el puerto **8000**, que es donde se está sirviendo la aplicación:

![mezzanine1](https://i.ibb.co/nn43VKJ/Captura-de-pantalla-de-2021-02-01-10-54-01.png "mezzanine")

Nos logueamos y personalizamos la página, cambiando el nombre del blog y añadiendo un nuevo artículo al blog, por ejemplo:

![mezzanine2](https://i.ibb.co/Y8k22Xg/Captura-de-pantalla-de-2021-02-01-11-00-56.png "mezzanine")

Antes de realizar la migración al entorno de producción, haremos un `dumpdata` para guardar la información de la base de datos en un fichero que posteriormente importaremos en el entorno de producción:

{% highlight shell %}
(despliegue) alvaro@debian:~/GitHub/cms$ python3 manage.py dumpdata > backup.json
{% endhighlight %}

Para mover los ficheros, creamos un repositorio vacío en GitHub y subimos los mismos, haciendo para ello uso de los comandos:

{% highlight shell %}
alvaro@debian:~/GitHub/cms$ git init
Inicializado repositorio Git vacío en /home/alvaro/GitHub/cms/.git/

alvaro@debian:~/GitHub/cms$ git add .

alvaro@debian:~/GitHub/cms$ git commit -am "Inserción de ficheros."
[master (commit-raíz) ad372a8] Inserción de ficheros.
 15 files changed, 1304 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 .hgignore
 create mode 100644 backup.json
 create mode 100644 cms/__init__.py
 create mode 100644 cms/settings.py
 create mode 100644 cms/urls.py
 create mode 100644 cms/wsgi.py
 create mode 100644 deploy/crontab.template
 create mode 100644 deploy/gunicorn.conf.py.template
 create mode 100644 deploy/local_settings.py.template
 create mode 100644 deploy/nginx.conf.template
 create mode 100644 deploy/supervisor.conf.template
 create mode 100644 fabfile.py
 create mode 100644 manage.py
 create mode 100644 requirements.txt

alvaro@debian:~/GitHub/cms$ git remote add origin git@github.com:alvarovaca/mezzanine_django.git

alvaro@debian:~/GitHub/cms$ git push -u origin master
Enumerando objetos: 19, listo.
Contando objetos: 100% (19/19), listo.
Compresión delta usando hasta 12 hilos
Comprimiendo objetos: 100% (17/17), listo.
Escribiendo objetos: 100% (19/19), 18.75 KiB | 6.25 MiB/s, listo.
Total 19 (delta 0), reusado 0 (delta 0)
To github.com:alvarovaca/mezzanine_django.git
 * [new branch]      master -> master
Rama 'master' configurada para hacer seguimiento a la rama remota 'master' de 'origin'.
{% endhighlight %}

Accedemos a **Freston** para crear un nuevo registro DNS "**python**" que nombre el nuevo servicio:

{% highlight shell %}
debian@freston:~$ sudo nano /var/cache/bind/db.alvaro.interna

dulcinea        IN      A       10.0.1.9
sancho  IN      A       10.0.1.4
quijote IN      A       10.0.2.6
freston IN      A       10.0.1.7
www     IN      CNAME   quijote
bd      IN      CNAME   sancho
python  IN      CNAME   quijote

debian@freston:~$ sudo nano /var/cache/bind/db.alvaro.dmz

dulcinea        IN      A       10.0.2.12
sancho  IN      A       10.0.1.4
quijote IN      A       10.0.2.6
freston IN      A       10.0.1.7
www     IN      CNAME   quijote
bd      IN      CNAME   sancho
python  IN      CNAME   quijote

debian@freston:~$ sudo nano /var/cache/bind/db.alvaro.externa

dulcinea        IN      A       172.22.200.134
www     IN      CNAME   dulcinea
python  IN      CNAME   dulcinea

debian@freston:~$ sudo systemctl restart bind9
{% endhighlight %}

Accedemos a **Sancho** para crear una nueva base de datos en MariaDB y un usuario con privilegios para acceder a la misma:

{% highlight shell %}
ubuntu@sancho:~$ sudo mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 37
Server version: 10.3.25-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE mezzanine;
Query OK, 1 row affected (0.016 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON mezzanine.* to 'quijote'@'10.0.2.6';
Query OK, 0 rows affected (0.029 sec)
{% endhighlight %}

Accedemos a **Quijote** para realizar el despliegue de nuestra aplicación, instalando primeramente las dependencias necesarias para que la aplicación funcione:

{% highlight shell %}
[root@quijote ~]# dnf install virtualenv git python3-mod_wsgi gcc python3-devel mysql-devel
{% endhighlight %}

Creamos y activamos el entorno virtual en el que instalaremos las dependencias de la aplicación:

{% highlight shell %}
[root@quijote virtualenv]# python3 -m venv despliegue

[root@quijote virtualenv]# source despliegue/bin/activate
{% endhighlight %}

Clonamos el repositorio de GitHub previamente creado, que contiene nuestra aplicación y el backup de la base de datos:

{% highlight shell %}
(despliegue) [root@quijote www]# git clone https://github.com/alvarovaca/mezzanine_django.git
{% endhighlight %}

Listamos el contenido del mismo para verificar que tenemos todos los ficheros necesarios:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# ls -l
total 52
-rw-r--r--. 1 root root 19638 Feb  1 12:29 backup.json
drwxr-xr-x. 2 root root    74 Feb  1 12:29 cms
drwxr-xr-x. 2 root root   156 Feb  1 12:29 deploy
-rw-r--r--. 1 root root 22057 Feb  1 12:29 fabfile.py
-rw-r--r--. 1 root root   367 Feb  1 12:29 manage.py
-rw-r--r--. 1 root root    17 Feb  1 12:29 requirements.txt
{% endhighlight %}

Instalamos con **pip** los requerimientos y algunos paquetes necesarios para el despliegue de la aplicación:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# pip install -r requirements.txt

(despliegue) [root@quijote mezzanine_django]# pip install mysql-connector-python uwsgi mysqlclient
{% endhighlight %}

Modificamos el fichero **cms/settings.py** para utilizar una base de datos mysql, ya que estamos en producción, así como habilitar el acceso a la misma desde **localhost**:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# vi cms/settings.py

ALLOWED_HOSTS = ['python.alvaro.gonzalonazareno.org']

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "mezzanine",
        "USER": "quijote",
        "PASSWORD": "quijote",
        "HOST": "bd.alvaro.gonzalonazareno.org",
        "PORT": "",
    }
}
{% endhighlight %}

Hacemos una migración para así generar las tablas en la base de datos mysql:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, blog, conf, contenttypes, core, django_comments, forms, galleries, generic, pages, redirects, sessions, sites, twitter
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
...
  Applying pages.0002_auto_20141227_0224... OK
  Applying pages.0003_auto_20150527_1555... OK
  Applying pages.0004_auto_20170411_0504... OK
  Applying redirects.0001_initial... OK
  Applying sessions.0001_initial... OK
  Applying sites.0002_alter_domain_unique... OK
  Applying twitter.0001_initial... OK
{% endhighlight %}

Una vez creadas las tablas, importamos el backup generado con la información que teníamos en la base de datos en desarrollo:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# python3 manage.py loaddata backup.json
Installed 150 object(s) from 1 fixture(s)
{% endhighlight %}

Generamos el contenido estático que deberá servirse:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# python manage.py collectstatic
{% endhighlight %}

Creamos un nuevo VirtualHost en el que definiremos la nueva aplicación:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# vi /etc/httpd/sites-available/mezzanine.conf

<VirtualHost *:80>
    ServerName python.alvaro.gonzalonazareno.org

    Redirect 301 / https://python.alvaro.gonzalonazareno.org/

    ErrorLog /var/www/mezzanine_django/log/error.log
    CustomLog /var/www/mezzanine_django/log/requests.log combined
</VirtualHost>

<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerName python.alvaro.gonzalonazareno.org
        DocumentRoot /var/www/mezzanine_django

        <Proxy "unix:/run/php-fpm/www.sock|fcgi://php-fpm">
            ProxySet disablereuse=off
        </Proxy>

        <FilesMatch \.php$>
            SetHandler proxy:fcgi://php-fpm
        </FilesMatch>

        Alias /static "/var/www/python-cms-mezzanine/mezzanine_openstack/static"
    
        <Directory /var/www/mezzanine_django/static>
          Require all granted
          Options FollowSymlinks
        </Directory>

        ProxyPass /static !
        ProxyPass / http://127.0.0.1:8080/

        ErrorLog /var/www/mezzanine_django/log/error.log
        CustomLog /var/www/mezzanine_django/log/requests.log combined

        SSLEngine on

        SSLCertificateFile      /etc/ssl/certs/openstack.crt
        SSLCertificateKeyFile   /etc/ssl/private/openstack.key
        SSLCACertificateFile    /etc/ssl/certs/gonzalonazareno.crt
    </VirtualHost>
</IfModule>
{% endhighlight %}

Creamos los directorios en los que se ubicarán los logs, damos los permisos necesarios para que la aplicación pueda ser servida y habilitamos el sitio:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# mkdir log

(despliegue) [root@quijote mezzanine_django]# touch log/{error.log,requests.log}

(despliegue) [root@quijote mezzanine_django]# chown -R apache:apache ../mezzanine_django/

(despliegue) [root@quijote mezzanine_django]# ln -s /etc/httpd/sites-available/mezzanine.conf /etc/httpd/sites-enabled/

(despliegue) [root@quijote mezzanine_django]# systemctl restart httpd
{% endhighlight %}

Configuramos el servidor de aplicaciones **uwsgi** para que escuche peticiones en el puerto 8080 y pueda así, comunicarse con el servidor web:

{% highlight shell %}
(despliegue) [root@quijote mezzanine_django]# nano uwsgi.ini

[uwsgi]
http = :8080
chdir = /var/www/mezzanine_django
wsgi-file = /var/www/mezzanine_django/cms/wsgi.py
master
{% endhighlight %}

A pesar de todos los pasos llevados a cabo, la aplicación no ha logrado funcionar en el entorno de desarrollo. He comparado el proceso y los ficheros actualmente existentes con algunos de mis compañeros y no logro determinar el fallo, pues debería funcionar correctamente. Cuando tenga más tiempo lo miraré con mayor profundidad.