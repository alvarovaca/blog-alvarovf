---
layout: post
title:  "Aumentar el rendimiento de servidores web"
banner: "/assets/images/banners/pxe.jpg"
date:   2021-02-18 09:01:00 +0200
categories: servicios
---
Instalamos **ansible**:

{% highlight shell %}
alvaro@debian:~$ sudo apt update && sudo apt install ansible
{% endhighlight %}

Clonamos el repositorio con la **receta** en su interior:

{% highlight shell %}
alvaro@debian:~/GitHub$ git clone https://github.com/josedom24/ansible_nginx_fpm_php.git
Clonando en 'ansible_nginx_fpm_php'...
remote: Enumerating objects: 40, done.
remote: Counting objects: 100% (40/40), done.
remote: Compressing objects: 100% (27/27), done.
remote: Total 40 (delta 0), reused 36 (delta 0), pack-reused 0
Desempaquetando objetos: 100% (40/40), listo.
{% endhighlight %}

Listamos el contenido del repositorio clonado:

{% highlight shell %}
alvaro@debian:~/GitHub/ansible_nginx_fpm_php$ ls -l
total 44
-rw-r--r-- 1 alvaro alvaro    76 feb 10 18:00 ansible.cfg
drwxr-xr-x 2 alvaro alvaro  4096 feb 10 18:00 group_vars
-rw-r--r-- 1 alvaro alvaro    98 feb 10 18:00 hosts
-rw-r--r-- 1 alvaro alvaro 18092 feb 10 18:00 LICENSE
-rw-r--r-- 1 alvaro alvaro    84 feb 10 18:00 README.md
drwxr-xr-x 6 alvaro alvaro  4096 feb 10 18:00 roles
-rw-r--r-- 1 alvaro alvaro   264 feb 10 18:00 site.yaml
{% endhighlight %}

Editamos el fichero **hosts** y establecemos la dirección IP de la máquina que vamos a utilizar para las pruebas:

{% highlight shell %}
alvaro@debian:~/GitHub/ansible_nginx_fpm_php$ nano hosts

[servidores_web]
nodo1 ansible_ssh_host=192.168.1.136 ansible_python_interpreter=/usr/bin/python3
{% endhighlight %}

Ejecutamos el **playbook** de **ansible** para así realizar las modificaciones iniciales en la máquina que utilizaremos para las pruebas:

{% highlight shell %}
alvaro@debian:~/GitHub/ansible_nginx_fpm_php$ ansible-playbook site.yaml 

PLAY [servidores_web] **********************************************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [nodo1]

TASK [nginx : install nginx, php-fpm] ******************************************************************
changed: [nodo1]

TASK [nginx : Copy info.php] ***************************************************************************
changed: [nodo1]

TASK [nginx : Copy virtualhost default] ****************************************************************
changed: [nodo1]

RUNNING HANDLER [nginx : restart nginx] ****************************************************************
changed: [nodo1]

PLAY [servidores_web] **********************************************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [nodo1]

TASK [mariadb : ensure mariadb is installed] ***********************************************************
changed: [nodo1]

TASK [mariadb : ensure mariadb binds to internal interface] ********************************************
changed: [nodo1]

RUNNING HANDLER [mariadb : restart mariadb] ************************************************************
changed: [nodo1]

PLAY [servidores_web] **********************************************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [nodo1]

TASK [wordpress : install unzip] ***********************************************************************
changed: [nodo1]

TASK [wordpress : download wordpress] ******************************************************************
changed: [nodo1]

TASK [wordpress : unzip wordpress] *********************************************************************
changed: [nodo1]

TASK [wordpress : create database wordpress] ***********************************************************
changed: [nodo1]

TASK [wordpress : create user mysql wordpress] *********************************************************
changed: [nodo1] => (item=localhost)

TASK [wordpress : copy wp-config.php] ******************************************************************
changed: [nodo1]

RUNNING HANDLER [wordpress : restart nginx] ************************************************************
changed: [nodo1]

PLAY RECAP *********************************************************************************************
nodo1                      : ok=17   changed=14   unreachable=0    failed=0
{% endhighlight %}

Accedemos desde el navegador web a http://**IP**/wordpress y procedemos con la instalación del CMS, quedando finalmente así:

![wordpress1](https://i.ibb.co/B2VBgv1/Captura-de-pantalla-de-2021-02-10-18-43-42.png "Wordpress")

Nos conectamos a la máquina de pruebas e instalamos **apache2-utils**, que contiene el comando **ab** que utilizaremos para los benchmarks:

{% highlight shell %}
debian@varnish:~$ sudo apt install apache2-utils
{% endhighlight %}

Realizamos diferentes pruebas de rendimiento, alternando entre diversos niveles de concurrencia, para conocer el número de peticiones que es capaz de responder. Entre cada prueba, reiniciamos el servicio para que los resultados no se vean afectados:

{% highlight shell %}
debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    164.29 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    164.80 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    163.25 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    153.69 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    154.68 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    157.09 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    11170.51 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    12488.91 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    11782.17 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    14262.15 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    14513.48 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    14268.24 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx
{% endhighlight %}

Una vez conocido los resultados, vamos a hacer uso de un proxy inverso caché conocido como **varnish**, que cacheará las respuestas del servidor web para así ofrecerlas de una forma mucho más rápida. Para que los clientes no noten diferencia, dicho proxy inverso se encontrará escuchando peticiones en el puerto 80, en lugar de **nginx**, de manera que modificaremos el VirtualHost para así cambiar el puerto de nginx a por ejemplo, el 8080:

{% highlight shell %}
root@varnish:~# nano /etc/nginx/sites-available/default

listen 8080 default_server;
listen [::]:8080 default_server;

root@varnish:~# systemctl restart nginx
{% endhighlight %}

Nos aseguramos que ahora está escuchando peticiones en el 8080:

{% highlight shell %}
root@varnish:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN     
tcp6       0      0 :::8080                 :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN
{% endhighlight %}

Instalamos **varnish**:

{% highlight shell %}
root@varnish:~# apt install varnish
{% endhighlight %}

Realizamos las correspondientes modificaciones en los ficheros de configuración para que el servicio escuche en el puerto 80 y redirija las peticiones al puerto 8080 (nginx), en caso de ser necesario:

{% highlight shell %}
root@varnish:~# nano /etc/varnish/default.vcl

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}

root@varnish:~# nano /etc/default/varnish

DAEMON_OPTS="-a :80 \
             -T localhost:6082 \
             -f /etc/varnish/default.vcl \
             -S /etc/varnish/secret \
             -s malloc,256m"

root@varnish:~# nano /lib/systemd/system/varnish.service

ExecStart=/usr/sbin/varnishd -j unix,user=vcache -F -a :80 -T localhost:6082 -f /etc/varnish/default.vcl -S /etc/varnish/secret -s malloc,256m

root@varnish:~# systemctl daemon-reload

root@varnish:~# systemctl restart varnish
{% endhighlight %}

Nos aseguramos que ahora está escuchando peticiones en el 80:

{% highlight shell %}
root@varnish:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:6082          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN     
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::8080                 :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:6082                :::*                    LISTEN
{% endhighlight %}

Repetimos las pruebas anteriores para así verificar que el número de peticiones respondidas por segundo es ahora mucho mayor, al tener un proxy inverso caché por delante:

{% highlight shell %}
debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    37704.80 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    38217.53 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 50 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    37134.25 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    37765.69 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    36442.75 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 100 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    36487.91 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    34797.91 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    35886.95 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 250 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    33152.41 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    31376.60 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    31074.93 [#/sec] (mean)
...

debian@varnish:~$ sudo systemctl restart nginx

debian@varnish:~$ ab -t 10 -c 500 -k http://127.0.0.1/wordpress/index.php
...
Requests per second:    32747.87 [#/sec] (mean)
...
{% endhighlight %}

Efectivamente, el número de peticiones respondidas por segundo ha aumentado considerablemente ya que ahora no hay que realizar todo el proceso que estamos acostumbrados (la petición llega al servidor web, posteriormente al servidor de aplicaciones y el HTML es generado...) sino que ahora, únicamente la primera petición es la que llega al servidor web y por consiguiente, al servidor de aplicaciones, ya que el contenido devuelto es cacheado por el proxy inverso varnish.

Podemos comprobar en los logs de apache que únicamente se ha recibido una petición:

{% highlight shell %}
debian@varnish:~$ sudo cat /var/log/nginx/access.log

127.0.0.1 - - [10/Feb/2021:17:42:05 +0000] "GET /wordpress/index.php HTTP/1.1" 301 5 "-" "ApacheBench/2.3"
{% endhighlight %}