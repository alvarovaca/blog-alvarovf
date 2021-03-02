---
layout: post
title:  "Proxy inverso con Docker y nginx"
banner: "/assets/images/banners/docker.jpg"
date:   2021-03-02 17:33:00 +0200
categories: servicios
---
Instalamos **docker.io**, **docker-compose** y **nginx**:

{% highlight shell %}
root@debian:~# apt update && apt install docker.io docker-compose nginx
{% endhighlight %}

Creamos un fichero de nombre **docker-compose.yaml** en el que definimos los contenedores que existirán en el escenario, dos para la aplicación **joomla** y otros dos para **nextcloud** (base de datos y aplicación):

{% highlight shell %}
root@debian:~/proxyinverso# nano docker-compose.yaml

version: '3.1'

services:

  joomla:
    container_name: servidor_joomla
    image: joomla
    restart: always
    environment:
      JOOMLA_DB_HOST: dbjoomla
      JOOMLA_DB_USER: user_joomla
      JOOMLA_DB_PASSWORD: asdasd
      JOOMLA_DB_NAME: bd_joomla
    ports:
      - 8081:80

  dbjoomla:
    container_name: db_joomla
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: bd_joomla
      MYSQL_USER: user_joomla
      MYSQL_PASSWORD: asdasd
      MYSQL_ROOT_PASSWORD: asdasd

  nextcloud:
    container_name: servidor_nextcloud
    image: nextcloud
    restart: always
    environment:
      MYSQL_HOST: dbnextcloud
      MYSQL_USER: user_nextcloud
      MYSQL_PASSWORD: asdasd
      MYSQL_DATABASE: bd_nextcloud
    ports:
      - 8082:80

  dbnextcloud:
    container_name: db_nextcloud
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: bd_nextcloud
      MYSQL_USER: user_nextcloud
      MYSQL_PASSWORD: asdasd
      MYSQL_ROOT_PASSWORD: asdasd
{% endhighlight %}

Levantamos el escenario que acabamos de definir:

{% highlight shell %}
root@debian:~/proxyinverso# docker-compose up -d
Creating servidor_nextcloud ... done
Creating db_nextcloud       ... done
Creating db_joomla          ... done
Creating servidor_joomla    ... done
{% endhighlight %}

Comprobamos que las aplicaciones se están sirviendo en los puertos **8081** y **8082** respectivamente, sin hacer todavía ningún proxy inverso:

![nginx1](https://i.ibb.co/VDYKkjp/Captura-de-pantalla-de-2021-02-26-15-40-15.png "Joomla")

![nginx2](https://i.ibb.co/CsP5xGH/Captura-de-pantalla-de-2021-02-26-15-40-19.png "Nextcloud")

Una vez que son accesibles, es hora de generar dos nuevos VirtualHost en los que haremos proxy inverso desde dos nombres de dominio distintos para acceder a dichas aplicaciones (**www.app1.org** y **www.app2.org**):

{% highlight shell %}
root@debian:~/proxyinverso# cp /etc/nginx/sites-available/default /etc/nginx/sites-available/app1
root@debian:~/proxyinverso# cp /etc/nginx/sites-available/default /etc/nginx/sites-available/app2

root@debian:~/proxyinverso# nano /etc/nginx/sites-available/app1

server {
        listen 80;
        listen [::]:80;

        index index.html index.htm index.nginx-debian.html;

        server_name www.app1.org;

        location / {
                proxy_pass http://localhost:8081;
        }
}

root@debian:~/proxyinverso# nano /etc/nginx/sites-available/app2

server {
        listen 80;
        listen [::]:80;

        index index.html index.htm index.nginx-debian.html;

        server_name www.app2.org;

        location / {
                proxy_pass http://localhost:8082;
        }
}
{% endhighlight %}

Habilitamos dichos VirtualHost creando los correspondientes enlaces simbólicos:

{% highlight shell %}
root@debian:~/proxyinverso# ln -s /etc/nginx/sites-available/app1 /etc/nginx/sites-enabled/
root@debian:~/proxyinverso# ln -s /etc/nginx/sites-available/app2 /etc/nginx/sites-enabled/

root@debian:~/proxyinverso# systemctl reload nginx
{% endhighlight %}

Modificamos el fichero de resolución estática de nombres **/etc/hosts** para poder acceder a través de las cabeceras de las peticiones HTTP a los VirtualHost:

{% highlight shell %}
root@debian:~/proxyinverso# nano /etc/hosts

127.0.0.1       localhost
127.0.1.1       debian
127.0.0.1       www.app1.org
127.0.0.1       www.app2.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Si tratamos de acceder ahora a **www.app1.org** y **www.app2.org**, respectivamente, podremos apreciar lo siguiente:

![nginx3](https://i.ibb.co/1q0TqGN/Captura-de-pantalla-de-2021-02-26-15-51-46.png "Joomla")

![nginx4](https://i.ibb.co/BTVts8D/Captura-de-pantalla-de-2021-02-26-15-53-17.png "Nextcloud")

Ahora, en lugar de utilizar dos VirtualHost (uno para cada aplicación), vamos a utilizar el mismo para ambas (**www.servidor.org**), diferenciándolas mediante la ruta que solicitamos tras la **/**, de manera que es hora de generar un nuevo VirtualHost con dichas características:

{% highlight shell %}
root@debian:~/proxyinverso# cp /etc/nginx/sites-available/app1 /etc/nginx/sites-available/servidor

root@debian:~/proxyinverso# nano /etc/nginx/sites-available/servidor

server {
        listen 80;
        listen [::]:80;

        index index.html index.htm index.nginx-debian.html;

        server_name www.servidor.org;

        location /app1/ {
                proxy_pass http://localhost:8081/;
        }

        location /app2/ {
                proxy_pass http://localhost:8082/;
        }
}
{% endhighlight %}

Habilitamos el VirtualHost creando el correspondiente enlace simbólico:

{% highlight shell %}
root@debian:~/proxyinverso# ln -s /etc/nginx/sites-available/servidor /etc/nginx/sites-enabled/

root@debian:~/proxyinverso# systemctl reload nginx
{% endhighlight %}

Modificamos el fichero de resolución estática de nombres **/etc/hosts** para poder acceder a través de las cabeceras de las peticiones HTTP al VirtualHost:

{% highlight shell %}
root@debian:~/proxyinverso# nano /etc/hosts

127.0.0.1       localhost
127.0.1.1       debian
127.0.0.1       www.app1.org
127.0.0.1       www.app2.org
127.0.0.1       www.servidor.org

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Si tratamos de acceder ahora a **www.servidor.org/app1** y **www.servidor.org/app2**, respectivamente, podremos apreciar lo siguiente:

![nginx5](https://i.ibb.co/kyvk8q7/Captura-de-pantalla-de-2021-03-02-19-17-29.png "Joomla")

![nginx6](https://i.ibb.co/QD1C5DD/Captura-de-pantalla-de-2021-03-02-19-17-36.png "Nextcloud")

El proxy inverso ha funcionado correctamente, sin embargo, al estar haciendo uso de Docker y de rutas virtuales para acceder a las aplicaciones, el contenido estático no se está sirviendo como debería, ya que no es capaz de encontrarlo en la ruta especificada.

Existen determinadas aplicaciones que vienen preparadas para, mediante variables de entorno, indicarles desde dónde se pretende acceder a las mismas, y por tanto, permiten servir el contenido estático sin mayor complicación, pero este no es el caso.