---
layout: post
title:  "Recolección de métricas con Telegraf, InfluxDB y Grafana"
banner: "/assets/images/banners/metricas.jpg"
date:   2021-02-09 11:35:00 +0200
categories: sistemas openstack
---
El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una instalación de un sistema de recolección de métricas sobre el mismo, incluyendo a su vez, las métricas de la máquina VPS contratada en OVH que en otros artículos hemos tratado.

Dicho sistema nos permitirá recolectar, centralizar y filtrar determinados aspectos sobre el rendimiento de las máquinas para su posterior representación gráfica que permita controlar la evolución temporal de parámetros esenciales de todos los servidores, permitiendo por tanto detectar anomalías en los mismos.

El sistema de recolección de métricas que vamos a instalar es una pila compuesta por los siguientes elementos:

* **InfluxDB**: Base de datos basada en series de tiempo (_time-series database_), pensada para almacenar y tratar de la manera más eficiente posible una gran cantidad de información por segundo, permitiendo a su vez realizar análisis de dichos datos en tiempo real (medias, mínimos, máximos, búsquedas en el tiempo...).

* **Telegraf**: Servicio que recopila y envía datos de métricas de diferentes sistemas, como por ejemplo uso de disco, carga del sistema, uso de la memoria RAM, carga de la CPU... La salida de Telegraf, por norma general, se envía a una base de datos de tipo series de tiempo como **InfluxDB**. Cuenta con una enorme lista de _plugins_ para aumentar sus funcionalidades.

* **Grafana**: Servicio que permite crear cuadros de mando y gráficos a partir de múltiples fuentes, incluidas las bases de datos de series de tiempo como **InfluxDB**. Permite la creación de usuarios con diferentes privilegios dentro de la aplicación, por lo que es posible compartir con otros miembros los paneles generados.

![diagrama1](https://i.ibb.co/VVvBSqS/tig-monitor-logic.png "Diagrama sistema")

Sin embargo, en este caso contamos con dos escenarios existentes, uno privado y uno público:

* **OpenStack**: Escenario privado en el que se encuentran ubicadas las máquinas **Dulcinea** (Debian 10), **Sancho** (Ubuntu 20.04), **Quijote** (CentOS 8) y **Freston** (Debian 10), que como se puede suponer, cuentan con direccionamiento privado.
* **OVH**: Escenario público en el que se encuentra ubicada la **VPS** (Debian 10), que como se puede suponer, cuenta con direccionamiento público.

Para entender el problema existente, es necesario recalcar una vez más cómo funciona **Telegraf**. Dicho servicio recolecta las métricas especificadas y posteriormente abre una conexión con el servidor configurado que será aquel que esté ejecutando la base de datos en la que se va a almacenar la información. En otras palabras, el cliente Telegraf necesitará alcanzar a la máquina en la que se almacenarán las métricas.

Si nuestra intención fuese recolectar métricas únicamente en el escenario privado, no habría problema, sin embargo, la VPS no será capaz de alcanzar las máquinas ubicadas en el escenario OpenStack privado, al contar las mismas con un direccionamiento no alcanzable desde el exterior.

Para solventar esta situación, tenemos dos opciones:

* Instalar InfluxDB en una de las máquinas OpenStack y posteriormente configurar una VPN en la VPS para que se pueda alcanzar dicha máquina desde el exterior.
* Instalar InfluxDB en la VPS, que será alcanzable desde cualquiera de las máquinas OpenStack sin necesidad de llevar a cabo ninguna configuración adicional.

En mi caso, voy a optar por la segunda opción dada la sencillez que me aporta, evitando por tanto el hecho de tener que llevar a cabo configuraciones adicionales. Es por ello que la instalación de **InfluxDB**, **Telegraf** y **Grafana** se llevará a cabo en la VPS, mientras que en el resto de máquinas únicamente será necesario instalar **Telegraf** para así enviar dichas métricas a la base de datos alojada en la VPS.

## InfluxDB

El primer paso consistirá en instalar la paquetería necesaria en la VPS para poder trabajar con **InfluxDB**, los cimientos de este sistema, que como ya hemos mencionado, será el gestor de bases de datos que almacenará las métricas recolectadas.

Dado que dicho paquete cuenta con una gran diferencia de versión en los repositorios de Debian con respecto a _upstream_, procederemos a descargarlo de los repositorios oficiales de _InfluxData_.

Para añadir dicho repositorio a nuestra máquina, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# echo "deb https://repos.influxdata.com/debian buster stable" > /etc/apt/sources.list.d/influxdb.list
{% endhighlight %}

De otro lado, tendremos que añadir también la clave pública GPG de _InfluxData_ a nuestro anillo de claves para así poder verificar la integridad e instalar el paquete de InfluxDB que vamos a descargar, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# wget -qO- https://repos.influxdata.com/influxdb.key | apt-key add -
OK
{% endhighlight %}

Tras ello, podremos dar paso a la instalación de InfluxDB, no sin antes actualizar la lista de la paquetería disponible, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# apt update && apt install influxdb
{% endhighlight %}

El paquete **influxdb** ha sido instalado en la máquina, de manera que vamos a verificar el estado de dicho servicio para así comprobar que se encuentra actualmente en ejecución, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl status influxdb
● influxdb.service - InfluxDB is an open-source, distributed, time series database
   Loaded: loaded (/lib/systemd/system/influxdb.service; enabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: https://docs.influxdata.com/influxdb/
{% endhighlight %}

Como se puede apreciar, el servicio se encuentra habilitado para arrancar junto a la máquina (_enabled_), sin embargo, actualmente no se encuentra activo (_inactive_), de manera que procederemos a levantarlo manualmente, ejecutando para ello la siguiente instrucción:

{% highlight shell %}
root@vps:~# systemctl start influxdb
{% endhighlight %}

Si verificamos una vez más el estado del servicio, haciendo para ello uso del comando utilizado con anterioridad, se nos mostrará lo siguiente:

{% highlight shell %}
root@vps:~# systemctl status influxdb
● influxdb.service - InfluxDB is an open-source, distributed, time series database
   Loaded: loaded (/lib/systemd/system/influxdb.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2021-02-05 18:28:43 CET; 2s ago
     Docs: https://docs.influxdata.com/influxdb/
 Main PID: 5255 (influxd)
    Tasks: 7 (limit: 2318)
   Memory: 13.8M
   CGroup: /system.slice/influxdb.service
           └─5255 /usr/bin/influxd -config /etc/influxdb/influxdb.conf
{% endhighlight %}

Efectivamente, el servicio se encuentra ahora activo y listo para su uso.

Como consecuencia del proceso que está actualmente ejecutándose, se habrán abierto dos _sockets TCP/IP_ en los puertos por defecto de InfluxDB (**8086** y **8088**) que estarán escuchando peticiones, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tlnp | egrep 'influx'
tcp        0      0 127.0.0.1:8088          0.0.0.0:*               LISTEN      5255/influxd
tcp6       0      0 :::8086                 :::*                    LISTEN      5255/influxd
{% endhighlight %}

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho _socket_.

Efectivamente, el proceso **influxd** está escuchando peticiones en todas las interfaces de la máquina en el puerto **8086**, que es aquel que se expone al exterior para recibir peticiones HTTP y del que por tanto, haremos uso para recibir las métricas del resto de máquinas.

De otro lado, el puerto **8088** se encuenta limitado a la interfaz _loopback_ de la máquina, ya que es el que se utiliza para interactuar de forma directa con la base de datos en operaciones como copias de seguridad y restauraciones, que no debe ser accesible desde el exterior.

Tras ello, procederemos a abrir una _shell_ de **influx** para así poder gestionar el motor, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# influx
Connected to http://localhost:8086 version 1.8.4
InfluxDB shell version: 1.8.4
{% endhighlight %}

Como se puede apreciar, la _influx shell_ se ha abierto correctamente y está lista para su uso.

Lo primero que haremos estando dentro del servidor de bases de datos será crear una nueva base de datos en la que almacenar las métricas, valga la redundancia. En este caso, de nombre **telegraf**. Para ello, haremos uso de la instrucción:

{% highlight shell %}
> CREATE DATABASE telegraf
{% endhighlight %}

La base de datos en cuestión habrá sido generada, pero para verificarlo, vamos a listar todas las bases de datos existentes, ejecutando para ello la instrucción:

{% highlight shell %}
> SHOW DATABASES
name: databases
name
----
_internal
telegraf
{% endhighlight %}

Efectivamente, la base de datos **telegraf** ha sido correctamente generada, sin embargo, de nada nos sirve tener una base de datos si no tenemos un usuario que pueda escribir en la misma.

Únicamente necesitamos un usuario, ya que la distinción entre las máquinas que envían sus métricas se hace mediante la columna _host_ del registro almacenado, que como se puede suponer, contendrá el nombre de la máquina que ha enviado el registro.

En este caso, vamos a crear un usuario de nombre "**telegraf**" cuya contraseña, como es lógico, censuraré por seguridad. Para ello, haremos uso de la instrucción:

{% highlight shell %}
> CREATE USER telegraf WITH PASSWORD 'XXXXXXXXXX'
{% endhighlight %}

El usuario en cuestión habrá sido generado, pero para verificarlo, vamos a listar todos los usuarios existentes, ejecutando para ello la instrucción:

{% highlight shell %}
> SHOW USERS
user     admin
----     -----
telegraf false
{% endhighlight %}

Efectivamente, el usuario **telegraf** ha sido correctamente generado.

Una vez realizadas las oportunas modificaciones, podremos salir de _influx_ haciendo uso de la instrucción `exit`, con la finalidad de continuar ahora con la instalación de **Telegraf**.

## Telegraf

### VPS

El segundo paso consistirá en instalar la paquetería necesaria en todas las máquinas para poder recolectar nuestras métricas gracias a **Telegraf** y enviarlas a la base de datos previamente generada.

Dado que dicho paquete no se encuentra actualmente disponible en los repositorios de Debian, procederemos a descargarlo de los repositorios oficiales de _InfluxData_.

Como podemos apreciar, dicho _software_ es de la misma organización que **InfluxDB**, de manera que nos servirá la anterior clave pública GPG y repositorio instalados para realizar la descarga e instalación del paquete, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# apt install telegraf
{% endhighlight %}

El paquete **telegraf** ha sido instalado en la máquina, de manera que vamos a verificar el estado de dicho servicio para así comprobar que se encuentra actualmente en ejecución, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2021-02-05 18:35:08 CET; 3s ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 5480 (telegraf)
    Tasks: 6 (limit: 2318)
   Memory: 19.6M
   CGroup: /system.slice/telegraf.service
           └─5480 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
{% endhighlight %}

Efectivamente, el servicio se encuentra activo y listo para su uso, sin embargo, no se encuentra configurado para actuar de la manera que nosotros necesitamos.

Para solucionarlo, procederemos a modificar el fichero **/etc/telegraf/telegraf.conf**, en el que estableceremos los parámetros deseados, como por ejemplo la base de datos remota, el nombre de la máquina... ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/telegraf/telegraf.conf
{% endhighlight %}

Dentro del mismo, podremos encontrar las siguientes secciones:

* **Output Plugins**: Envía las métricas recolectadas a diferentes destinos, en mi caso a _InfluxDB_.
* **Processor Plugins**: Transforma, decora y filtra las métricas recolectadas.
* **Aggregator Plugins**: Crea conjuntos de métricas, por ejemplo, permitiendo hacer una media, mínimo, máximo...
* **Input Plugins**: Recolecta las métricas especificadas del sistema o servicios.

En este caso, tendremos que buscar la sección de nombre **[[outputs.influxdb]]** dentro de **Output Plugins** para indicar dentro de la misma los parámetros necesarios para llevar a cabo la conexión con InfluxDB.

* El primer parámetro que modificaremos dentro de dicha sección será el que por defecto tiene el siguiente aspecto:

{% highlight shell %}
# urls = ["http://127.0.0.1:8086"]
{% endhighlight %}

Como se puede suponer, tendremos que descomentarlo e indicar la dirección y el puerto en la que se aloja la base de datos InfluxDB, que en este caso es correcta, pues se está ejecutando en **_localhost_** en el puerto **8086**.

En el resto de máquinas, tendremos que introducir la dirección IP pública de dicha VPS, es decir, **51.210.109.246**. Por tanto, el resultado final de dicha directiva sería:

{% highlight shell %}
urls = ["http://127.0.0.1:8086"]
{% endhighlight %}

* El segundo parámetro que modificaremos dentro de dicha sección será el que por defecto tiene el siguiente aspecto:

{% highlight shell %}
# database = "telegraf"
{% endhighlight %}

Como se puede suponer, tendremos que descomentarlo e indicar el nombre de la base de datos InfluxDB que previamente hemos generado, que una vez más, es correcto. Por tanto, el resultado final de dicha directiva sería:

{% highlight shell %}
database = "telegraf"
{% endhighlight %}

* El tercer parámetro que modificaremos dentro de dicha sección será el que por defecto tiene el siguiente aspecto:

{% highlight shell %}
# skip_database_creation = false
{% endhighlight %}

Como se puede suponer, tendremos que descomentarlo y asignarle el valor **true** para que así no trate de generar la base de datos InfluxDB, ya que previamente la hemos generado nosotros manualmente. Por tanto, el resultado final de dicha directiva sería:

{% highlight shell %}
skip_database_creation = true
{% endhighlight %}

* El cuarto parámetro que modificaremos dentro de dicha sección será el que por defecto tiene el siguiente aspecto:

{% highlight shell %}
# timeout = "5s"
{% endhighlight %}

Como se puede suponer, tendremos que descomentarlo e indicar el valor de tiempo que queremos que transcurra antes de que la petición HTTP a la base de datos destino expire en caso de no haber sido alcanzada, que en este caso, 5 segundos es un valor que me parece correcto. Por tanto, el resultado final de dicha directiva sería:

{% highlight shell %}
timeout = "5s"
{% endhighlight %}

* Por último, el quinto y sexto parámetro que modificaremos dentro de dicha sección serán los que por defecto tienen el siguiente aspecto:

{% highlight shell %}
# username = "telegraf"
# password = "metricsmetricsmetricsmetrics"
{% endhighlight %}

Como se puede suponer, tendremos que descomentarlos e indicar el nombre del usuario que previamente hemos generado en **InfluxDB**, así como la contraseña del mismo, que censuraré por seguridad. Por tanto, el resultado final de dichas directivas sería:

{% highlight shell %}
username = "telegraf"
password = "XXXXXXXXXX"
{% endhighlight %}

Listo, la configuración básica del cliente **Telegraf** ha finalizado.

Es importante mencionar que **Telegraf** es un _software_ realmente grande y nos ofrece una cantidad descomunal de posibilidades de recolección de métricas (_plugins_). Tratar todas ellas en un artículo sería totalmente inviable, de manera que aquellas que hemos modificado son las justas y necesarias para el correcto funcionamiento básico del mismo, ya que se incluyen las más esenciales habilitadas por defecto (uso de disco, carga del sistema, uso de la memoria RAM, carga de la CPU...).

Otros dos parámetros muy importantes se encuentran en la sección **[agent]**, siendo los siguientes:

* **interval**: Frecuencia con la que queremos que se recolecten las métricas, que por defecto son 10 segundos. Mientras menor sea dicho valor, más exacto será nuestro sistema de recolección de métricas, pero a su vez, mayor será el flujo de datos que viajará entre las máquinas.
* **hostname**: En caso de no especificar un _hostname_ en concreto, se utilizará aquel que la máquina tiene asignado, que como se puede suponer, tiene la finalidad de identificar de forma única a cada máquina dentro del sistema de recolección de métricas.

Tras ello, guardaremos los cambios y dado que hemos modificado el fichero de configuración de un servicio, tendremos que reiniciarlo para así cargar la nueva configuración en memoria, haciendo para ello uso del comando:

{% highlight shell %}
root@sancho:~# systemctl restart telegraf
{% endhighlight %}

Dado que el proceso de instalación de **Telegraf** es prácticamente igual en todas las máquinas, vamos a hacer un pequeño resumen para no repetir la misma explicación para cada una de ellas:

* Añadimos la clave pública GPG y el correspondiente repositorio de _InfluxData_.
* Descargamos e instalamos el paquete _telegraf_, actualizando previamente la lista de la paquetería disponible.
* Comprobamos que el servicio se encuentra activo y habilitado durante el arranque.
* Llevamos a cabo las modificaciones previamente mencionadas en el fichero **/etc/telegraf/telegraf.conf**.
* Reiniciamos el servicio para cargar la nueva configuración en memoria.

### Dulcinea

{% highlight shell %}
root@dulcinea:~# wget -qO- https://repos.influxdata.com/influxdb.key | apt-key add -
OK

root@dulcinea:~# echo "deb https://repos.influxdata.com/debian buster stable" > /etc/apt/sources.list.d/influxdb.list

root@dulcinea:~# apt update && apt install telegraf

root@dulcinea:~# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2021-02-06 11:06:39 CET; 4s ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 16138 (telegraf)
    Tasks: 7 (limit: 562)
   Memory: 13.7M
   CGroup: /system.slice/telegraf.service
           └─16138 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d

root@dulcinea:~# nano /etc/telegraf/telegraf.conf

# urls = ["http://127.0.0.1:8086"] -> urls = ["http://51.210.109.246:8086"]
# database = "telegraf" -> database = "telegraf"
# skip_database_creation = false -> skip_database_creation = true
# timeout = "5s" -> timeout = "5s"
# username = "telegraf" -> username = "telegraf"
# password = "metricsmetricsmetricsmetrics" -> password = "XXXXXXXXXX"

root@dulcinea:~# systemctl restart telegraf
{% endhighlight %}

### Sancho

{% highlight shell %}
root@sancho:~# wget -qO- https://repos.influxdata.com/influxdb.key | apt-key add -
OK

root@sancho:~# echo "deb https://repos.influxdata.com/ubuntu focal stable" > /etc/apt/sources.list.d/influxdb.list

root@sancho:~# apt update && apt install telegraf

root@sancho:~# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
     Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2021-02-06 11:14:38 CET; 24s ago
       Docs: https://github.com/influxdata/telegraf
   Main PID: 18572 (telegraf)
      Tasks: 7 (limit: 533)
     Memory: 42.9M
     CGroup: /system.slice/telegraf.service
             └─18572 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d

root@sancho:~# nano /etc/telegraf/telegraf.conf

# urls = ["http://127.0.0.1:8086"] -> urls = ["http://51.210.109.246:8086"]
# database = "telegraf" -> database = "telegraf"
# skip_database_creation = false -> skip_database_creation = true
# timeout = "5s" -> timeout = "5s"
# username = "telegraf" -> username = "telegraf"
# password = "metricsmetricsmetricsmetrics" -> password = "XXXXXXXXXX"

root@sancho:~# systemctl restart telegraf
{% endhighlight %}

### Quijote

{% highlight shell %}
[root@quijote ~]# vi /etc/yum.repos.d/influxdb.repo

[influxdb]
name = InfluxDB Repository - RHEL 
baseurl = https://repos.influxdata.com/rhel/7/x86_64/stable/
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key

[root@quijote ~]# dnf install telegraf

[root@quijote ~]# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/usr/lib/systemd/system/telegraf.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: https://github.com/influxdata/telegraf

[root@quijote ~]# systemctl start telegraf

[root@quijote ~]# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/usr/lib/systemd/system/telegraf.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2021-02-06 11:24:15 CET; 1s ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 5932 (telegraf)
    Tasks: 4 (limit: 2635)
   Memory: 52.4M
   CGroup: /system.slice/telegraf.service
           └─5932 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d

[root@quijote ~]# vi /etc/telegraf/telegraf.conf

# urls = ["http://127.0.0.1:8086"] -> urls = ["http://51.210.109.246:8086"]
# database = "telegraf" -> database = "telegraf"
# skip_database_creation = false -> skip_database_creation = true
# timeout = "5s" -> timeout = "5s"
# username = "telegraf" -> username = "telegraf"
# password = "metricsmetricsmetricsmetrics" -> password = "XXXXXXXXXX"

[root@quijote ~]# systemctl restart telegraf
{% endhighlight %}

### Freston

{% highlight shell %}
root@freston:~# wget -qO- https://repos.influxdata.com/influxdb.key | apt-key add -
OK

root@freston:~# echo "deb https://repos.influxdata.com/debian buster stable" > /etc/apt/sources.list.d/influxdb.list

root@freston:~# apt update && apt install telegraf

root@freston:~# systemctl status telegraf
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2021-02-06 11:27:47 CET; 5s ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 15462 (telegraf)
    Tasks: 7 (limit: 562)
   Memory: 17.0M
   CGroup: /system.slice/telegraf.service
           └─15462 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d

root@freston:~# nano /etc/telegraf/telegraf.conf

# urls = ["http://127.0.0.1:8086"] -> urls = ["http://51.210.109.246:8086"]
# database = "telegraf" -> database = "telegraf"
# skip_database_creation = false -> skip_database_creation = true
# timeout = "5s" -> timeout = "5s"
# username = "telegraf" -> username = "telegraf"
# password = "metricsmetricsmetricsmetrics" -> password = "XXXXXXXXXX"

root@freston:~# systemctl restart telegraf
{% endhighlight %}

## Grafana

El tercer y último paso consistirá en instalar la paquetería necesaria en la VPS para poder visualizar de forma gráfica las métricas recolectadas en la base de datos gracias a **Grafana** y tratar las mismas como nos sea necesario.

Dado que dicho paquete no se encuentra actualmente disponible en los repositorios de Debian, procederemos a descargarlo de los repositorios oficiales de _Grafana_.

Para añadir dicho repositorio a nuestra máquina, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# echo "deb https://packages.grafana.com/enterprise/deb stable main" > /etc/apt/sources.list.d/grafana.list
{% endhighlight %}

De otro lado, tendremos que añadir también la clave pública GPG de _Grafana_ a nuestro anillo de claves para así poder verificar la integridad e instalar el paquete de Grafana que vamos a descargar, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
OK
{% endhighlight %}

Tras ello, podremos dar paso a la instalación de Grafana, no sin antes actualizar la lista de la paquetería disponible, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# apt update && apt install grafana
{% endhighlight %}

El paquete **grafana** ha sido instalado en la máquina, de manera que vamos a verificar el estado de dicho servicio para así comprobar que se encuentra actualmente en ejecución, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl status grafana-server
● grafana-server.service - Grafana instance
   Loaded: loaded (/lib/systemd/system/grafana-server.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: http://docs.grafana.org
{% endhighlight %}

Como se puede apreciar, el servicio no se encuentra habilitado para arrancar junto a la máquina (_disabled_), además, tampoco se encuentra activo (_inactive_), de manera que procederemos a habilitarlo y levantarlo manualmente, ejecutando para ello las siguientes instrucciones:

{% highlight shell %}
root@vps:~# systemctl start grafana-server
root@vps:~# systemctl enable grafana-server
{% endhighlight %}

Si verificamos una vez más el estado del servicio, haciendo para ello uso del comando utilizado con anterioridad, se nos mostrará lo siguiente:

{% highlight shell %}
root@vps:~# systemctl status grafana-server
● grafana-server.service - Grafana instance
   Loaded: loaded (/lib/systemd/system/grafana-server.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2021-02-05 18:08:19 CET; 22s ago
     Docs: http://docs.grafana.org
 Main PID: 3994 (grafana-server)
    Tasks: 7 (limit: 2318)
   Memory: 27.2M
   CGroup: /system.slice/grafana-server.service
           └─3994 /usr/sbin/grafana-server --config=/etc/grafana/grafana.ini --pidfile=/var/run/grafana/grafana-server.pid --packaging=deb cfg:default.paths.logs=/var/log/grafana cfg:defaul
{% endhighlight %}

Efectivamente, el servicio se encuentra ahora habilitado, activo y listo para su uso.

Como consecuencia del proceso que está actualmente ejecutándose, se habrá abierto un _socket TCP/IP_ en el puerto por defecto de Grafana (**3000**) que estará escuchando peticiones provenientes de todas las interfaces de la máquina, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@vps:~# netstat -tlnp | egrep 'grafana'
tcp6       0      0 :::3000                 :::*                    LISTEN      3994/grafana-server
{% endhighlight %}

Efectivamente, el proceso **grafana-server** está escuchando peticiones en todas las interfaces de la máquina en el puerto **3000**, que es aquel que se expone al exterior para poder acceder al panel de administración web acondicionado para ello.

Antes de continuar, es importante mencionar que la configuración por defecto de Grafana es poco restrictiva, pues permite a los usuarios registrarse de forma manual, que en mi caso no es lo que busco. Para evitarlo, tendremos que modificar el fichero de nombre **/etc/grafana/grafana.ini**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# nano /etc/grafana/grafana.ini
{% endhighlight %}

Dentro del mismo, tendremos que modificar los siguientes parámetros para deshabilitar el registro de usuarios y la creación de organizaciones:

{% highlight shell %}
;allow_sign_up = true -> allow_sign_up = false
;allow_org_create = true -> allow_org_create = false
{% endhighlight %}

Tras ello, guardaremos los cambios y dado que hemos modificado el fichero de configuración de un servicio, tendremos que reiniciarlo para así cargar la nueva configuración en memoria, haciendo para ello uso del comando:

{% highlight shell %}
root@vps:~# systemctl restart grafana-server
{% endhighlight %}

Como anteriormente hemos mencionado, el servicio **grafana-server** se encuentra en ejecución en el puerto 3000, lo que supone que cada vez que queramos acceder al panel de administración web tendremos que conectarnos a dicho puerto de forma explícita ([http://51.210.109.246:3000](http://51.210.109.246:3000)), que es algo que puede resultar un tanto incómodo.

Para solventar este "problema", he decidido hacer uso del servidor web **nginx** alojado en dicha máquina que se encuentra actualmente escuchando peticiones en los puertos **80** (HTTP) y **443** (HTTPS), de manera que crearemos un VirtualHost accesible desde un determinado nombre de dominio y reenviaremos la petición entrante a través de un _proxy_ inverso al puerto **3000**, consiguiendo por tanto utilizar los puertos HTTP y HTTPS para hacer uso del panel de administración web de Grafana.

El primer paso ha consistido en la generación de un nuevo nombre dentro de la zona DNS **iesgn19.es**, concretamente nombrando un nuevo servicio **metricas** mediante un registro _CNAME_ al registro _A_ que apunta a la dirección **51.210.109.246**, que será a través del cuál accederemos al VirtualHost en cuestión, tal y como podemos apreciar a continuación:

![dns1](https://i.ibb.co/zS0ZfXV/Captura-18.jpg "Zona DNS")

Como bien sabemos, si queremos hacer uso del protocolo HTTPS para dicho nombre de dominio, necesitaremos generar un certificado firmado por una autoridad certificadora de considerable reputación, así que en mi caso, he recurrido una vez más a **Let's Encrypt**, siguiendo para ello los pasos indicados en el artículo [Configuración de HTTPS en Nginx](https://www.alvarovf.com/seguridad/vps/2020/11/30/configuracion-https-nginx.html), pues mencionar aquí el proceso se saldría completamente del objetivo del _post_.

Cuando nuestro certificado haya sido correctamente generado, todo estará listo para configurar el nuevo VirtualHost dentro del directorio **/etc/nginx/sites-available/**, cuyo nombre he decidido que en este caso sea **metricas**, de manera que ejecutaremos el comando:

{% highlight shell %}
root@vps:~# nano /etc/nginx/sites-available/metricas
{% endhighlight %}

Dentro del mismo, tendremos que crear dos directivas **server**, una para el VirtualHost accesible en el puerto 80 HTTP, dentro del cuál únicamente asignaremos el ServerName correspondiente y una redirección permanente para así forzar HTTPS. En la otra, para el VirtualHost accesible en el puerto 443 HTTPS, tendremos que configurar las siguientes directivas:

* **server_name**: Indicaremos el nombre de dominio a través del cuál accederemos al servidor.
* **ssl**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **ssl_certificate**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/letsencrypt/live/metricas.iesgn19.es/fullchain.pem**.
* **ssl_certificate_key**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/letsencrypt/live/metricas.iesgn19.es/privkey.pem**.
* **location**: Especificamos las instrucciones para resolver una petición a la ruta introducida (en este caso, cualquier ruta, ya que hemos indicado la raíz y se aplica de forma recursiva a partir de la misma). Dentro de dicho bloque, definimos mediante una directiva **proxy_pass** a qué dirección se debe reenviar la petición, que en este caso, será al puerto **3000** de la máquina local.

El resultado final sería:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        server_name metricas.iesgn19.es;

        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        server_name metricas.iesgn19.es;

        ssl    on;
        ssl_certificate    /etc/letsencrypt/live/metricas.iesgn19.es/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/metricas.iesgn19.es/privkey.pem;

        location / {
                proxy_pass http://localhost:3000;
        }
}
{% endhighlight %}

Listo, toda la configuración necesaria ya ha sido realizada, así que únicamente queda habilitar dicho sitio y probar que realmente funciona. Para activar el sitio, a diferencia de _apache2_ que contaba con una utilidad para ello, tendremos crear el enlace simbólico al fichero de configuración ubicado en **/etc/nginx/sites-available/** dentro de **/etc/nginx/sites-enabled/** de forma manual. Para ello, haremos uso del comando:

{% highlight shell %}
root@vps:~# ln -s /etc/nginx/sites-available/metricas /etc/nginx/sites-enabled/
{% endhighlight %}

Al parecer, el sitio ha sido correctamente habilitado, pero para activar la nueva configuración, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl reload nginx
{% endhighlight %}

Una vez que la configuración del servicio se ha vuelto a cargar, vamos a listar el contenido de **/etc/nginx/sites-enabled/** para verificar que el correspondiente enlace simbólico ha sido correctamente creado. Para ello, haremos uso del comando:

{% highlight shell %}
root@vps:~# ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 34 Nov  3 16:11 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 33 Nov 11 11:21 drupal -> /etc/nginx/sites-available/drupal
lrwxrwxrwx 1 root root 34 Nov  3 17:14 iesgn19 -> /etc/nginx/sites-available/iesgn19
lrwxrwxrwx 1 root root 35 Feb  5 19:37 metricas -> /etc/nginx/sites-available/metricas
{% endhighlight %}

Efectivamente, el nuevo sitio se encuentra actualmente activo, así que es hora de realizar la correspondiente prueba de acceso. Para ello, abriremos un navegador web y trataremos de acceder a [https://metricas.iesgn19.es](https://metricas.iesgn19.es):

![grafana1](https://i.ibb.co/HFLzqbt/Captura-0.jpg "Grafana")

Como era de esperar, el sitio web ha cargado correctamente, por lo que podemos concluir que el _proxy_ inverso al puerto 3000 ha funcionado como debería.

De otro lado, vamos a comenzar con la configuración de Grafana, de manera que vamos a hacer _login_ con el usuario **admin** y la contraseña **admin**, que son las credenciales asignadas por defecto, pudiendo apreciar tras ello lo siguiente:

![grafana2](https://i.ibb.co/zZbntQ8/Captura-1.jpg "Grafana")

En el siguiente paso, se nos ha ofrecido la posibilidad de cambiar la contraseña del usuario **admin**, que es algo realmente recomendable, pues la contraseña asignada por defecto carece totalmente de seguridad. Tras realizar dicho cambio, accederemos al menú principal, que tendrá la siguiente forma:

![grafana3](https://i.ibb.co/RYpmcNB/Captura-2.jpg "Grafana")

Actualmente contamos con un panel de administración web totalmente virgen, pues no tiene configurada ninguna fuente de datos ni mucho menos, gráficas para visualizar las métricas recolectadas. Es por ello que tendremos que pulsar en **Add your first data source**, para así vincular nuestra base de datos con Grafana:

![grafana4](https://i.ibb.co/zXSqLr0/Captura-4.jpg "Grafana")

Tras seleccionar **InfluxDB** como fuente de datos, tendremos que realizar la configuración básica para la conexión con dicho gestor:

* **URL**: Introducimos la dirección y el puerto en el que se encuentra escuchando peticiones. En este caso, al estar ejecutando toda la pila en la misma máquina, será **http://localhost:8086**.
* **Database**: Introducimos el nombre de la base de datos. En este caso, **telegraf**.
* **User**: Introducimos el nombre del usuario creado. En este caso, **telegraf**.
* **Password**: Introducimos la contraseña asociada al usuario especificado.

Por último, comprobaremos la conexión con la base de datos y añadiremos dicha fuente de datos pulsando en **Save & Test**:

![grafana5](https://i.ibb.co/Xtgbg0Q/Captura-6.jpg "Grafana")

Como se puede apreciar, la conexión con la fuente de datos ha sido exitosa, de manera que ya habremos realizado el paso más importante en cuanto a la configuración de Grafana. Tras ello, tendremos que regresar una vez más al menú principal, pudiendo apreciar lo siguiente:

![grafana6](https://i.ibb.co/FJyGTv1/Captura-8.jpg "Grafana")

Una vez que tengamos la posibilidad de acceder a las métricas recolectadas, podremos proceder con la creación de un nuevo _dashboard_ en el que vamos a crear las gráficas con aquellos datos de métricas que consideremos oportunos, pulsando para ello en **Create your first dashboard**:

![grafana7](https://i.ibb.co/0cMHBDy/Captura-9.jpg "Grafana")

Los _dashboards_ de Grafana se componen de paneles, representando idealmente cada uno de ellos los datos de una máquina distinta, de manera que vamos a comenzar por configurar el correspondiente a la **VPS**, pulsando para ello en **Add new panel**:

![grafana8](https://i.ibb.co/T23WzJM/Captura-10.jpg "Grafana")

Dentro del panel tendremos que definir consultas a la base de datos para así mostrar la información que consideremos oportuna. En este caso, he especificado las siguientes directivas en la consulta:

* **FROM**: Indicamos la tabla origen de los datos. En este caso, la tabla **mem**, que como se puede suponer, almacena los datos de métricas de la memoria RAM.
* **WHERE**: Indicamos una condición para mostrar los datos. En este caso, filtramos los resultados únicamente para el _host_ de nombre **vps**.
* **SELECT**: Indicamos la columna de la tabla que deseamos mostrar. En este caso, el porcentaje de uso, que se encuentra referenciado en la columna **used_percent**.

Como se puede apreciar, la gráfica se genera automáticamente a partir de la consulta realizada, de manera que tras añadir varias consultas para mostrar información sobre determinados aspectos de dicha máquina, así como configurar los paneles para el resto de máquinas, guardaremos el _dashboard_, asignándole como es lógico un nombre identificativo. Este sería el resultado final:

![grafana9](https://i.ibb.co/DbJ7Yjk/Captura-20.jpg "Grafana")

Tras apreciar la apariencia gráfica de las métricas recolectadas en todas las máquinas existentes en mi infraestructura, podemos concluir que la pila **InfluxDB + Telegraf + Grafana** es realmente completa y potente, pues permite centralizar todas las métricas que deseemos y tenerlo todo controlado de una manera realmente sencilla.