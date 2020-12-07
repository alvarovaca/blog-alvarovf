---
layout: post
title:  "Instalación de MongoDB 4.4 en Debian 10"
banner: "/assets/images/banners/mongodb.png"
date:   2020-12-05 11:12:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una instalación básica de **MongoDB 4.4**, un gestor de bases de datos no-relacionales (_NoSQL_) sobre una máquina **Debian Buster**, así como la configuración necesaria para admitir peticiones desde máquinas remotas dentro de una red local.

Para esta ocasión, he traído los deberes hechos y he instalado previamente una máquina virtual con **Debian Buster**, pero no he llevado a cabo ninguna configuración, para así partir desde un punto totalmente limpio. No considero necesario explicar el proceso de instalación de dicha distribución, ya que es bastante sencillo, además de salirse del objetivo del artículo.

El paquete del gestor de bases de datos que instalaremos será el proporcionado por la propia empresa desarrolladora (_MongoDB Inc._), que se encuentra disponible en su propio repositorio y siempre contiene la última versión existente. Aclaro este punto ya que en los repositorios oficiales de Debian existe una versión de MongoDB que no es mantenida por la empresa desarrolladora, sino por usuarios de la distribución, de manera que no contendrá la última versión existente.

El primer paso se encuentra relacionado con lo que acabo de explicar. Al tratarse de un paquete que no vamos a descargar de los repositorios oficiales de la distribución, tendremos que importar la clave pública de MongoDB, para que así nuestro sistema pueda verificar la firma realizada por parte de la empresa desarrolladora sobre el paquete que estamos descargando, asegurando así que el paquete no ha sido manipulado. Para ello, ejecutaremos el comando (con privilegios de administrador, haciendo uso de `su -`):

{% highlight shell %}
root@mongo:~# wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
OK
{% endhighlight %}

Como se puede apreciar, la operación ha devuelto un **OK**, asegurando por tanto que la clave pública ha sido añadida a nuestro anillo de claves de `apt-key`. En caso de que nos haya mostrado un error, muy posiblemente se deba a que el paquete **gnupg** no se encuentra instalado, aunque en la mayoría de las situaciones no será necesario instalarlo manualmente.

Ya hemos añadido la clave pública, pero nuestro gestor de paquetes todavía no sabe de qué repositorio debe descargar el paquete para su posterior instalación, pues todavía no hemos añadido el correspondiente repositorio. En mi caso, voy a hacer uso del gestor de paquetes `apt`, así que el repositorio puedo añadirlo directamente en el fichero **/etc/apt/sources.list** o bien, crear un fichero en el directorio **/etc/apt/sources.list.d/** cuyo contenido sea el repositorio en cuestión.

Para esta ocasión, he optado por la última opción, pues en mi opinión, permite mantener una organización mucho mayor. Para crear el fichero cuyo contenido sea el repositorio, haremos uso del comando:

{% highlight shell %}
root@mongo:~# echo "deb http://repo.mongodb.org/apt/debian buster/mongodb-org/4.4 main" > /etc/apt/sources.list.d/mongodb-org-4.4.list
{% endhighlight %}

En este caso, he añadido el repositorio en cuestión dentro de un fichero de nombre **mongodb-org-4.4.list** en el directorio **/etc/apt/sources.list.d/**, pues tiene la misma efectividad que si lo hubiésemos hecho directamente el fichero **/etc/apt/sources.list**. Para verificar que su contenido es el correcto, lo visualizaremos haciendo uso de `cat`:

{% highlight shell %}
root@mongo:~# cat /etc/apt/sources.list.d/mongodb-org-4.4.list
deb http://repo.mongodb.org/apt/debian buster/mongodb-org/4.4 main
{% endhighlight %}

Efectivamente, el contenido es el correcto. Ya está todo listo para proceder con la instalación de MongoDB, no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando:

{% highlight shell %}
root@mongo:~# apt update && apt upgrade && apt install mongodb-org
{% endhighlight %}

El paquete **mongodb-org** ha sido instalado en la máquina, pero antes de continuar, me gustaría mencionar que por defecto, cuando en un futuro realicemos una actualización de la paquetería instalada en la máquina, va a actualizar también aquellos paquetes referentes a MongoDB, que en mi caso no es algo relevante, pero si ponemos el servidor en producción, puede ser aconsejable monitorizar las actualizaciones y los cambios que suponen antes de llevarlas a cabo. Para evitar lo que acabo de mencionar, podemos ejecutar los comandos:

{% highlight shell %}
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-org-shell hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
{% endhighlight %}

De otro lado, la mayoría de sistemas UNIX limitan los recursos de la máquina de los que un proceso puede hacer uso (**ulimit**), por lo que si te has encontrado con algún problema referente a ello, [aquí](https://docs.mongodb.com/manual/reference/ulimit/) puedes encontrar información al respecto, ya que es un tema bastante amplio para explicar en este artículo. Para la tranquilidad de la mayoría de los lectores, al haber instalado MongoDB a través de un gestor de paquetes, el fichero del servicio utilizado en la instalación contiene dichos límites, por lo que los habrá configurado y no debe haber ningún problema.

A su vez, como consecuencia de la instalación haciendo uso de un gestor de paquetes, se habrá generado un directorio de nombre **mongodb/** dentro de **/var/lib/** y otro en **/var/log/**, por lo que vamos a asegurarnos de que han sido correctamente generados, listando para ello el contenido de los directorios padre y estableciendo un filtro por nombre, haciendo para ello uso de los comandos:

{% highlight shell %}
root@mongo:~# ls -l /var/lib/ | egrep 'mongodb'
drwxr-xr-x 2 mongodb       mongodb       4096 Dec  3 06:55 mongodb

root@mongo:~# ls -l /var/log/ | egrep 'mongodb'
drwxr-xr-x 2 mongodb           mongodb      4096 Dec  3 06:55 mongodb
{% endhighlight %}

Efectivamente, dichos directorios se encuentran generados en las correspondientes rutas, siendo su usuario y grupo propietario, **mongodb**. En caso de que no existiesen, habría que crearlos manualmente haciendo uso de `mkdir` y establecer los correspondientes propietarios haciendo uso de `chown`, aunque como he mencionado anteriormente, si se ha hecho uso de un gestor de paquetes, no debe haber inconveniente.

Ya está todo listo para arrancar el proceso correspondiente a MongoDB. En esta ocasión, estamos haciendo uso de **Debian Buster**, tratándose de una versión lo suficientemente reciente como para hacer uso de **systemd** (que es controlado mediante el comando `systemctl`), así que haremos uso del comando mencionado para levantar el proceso **mongod**:

{% highlight shell %}
root@mongo:~# systemctl start mongod
{% endhighlight %}

En un principio, el proceso ya está ejecutándose, así que para verificar que MongoDB se encuentra ya operativo, ejecutaremos el comando:

{% highlight shell %}
root@mongo:~# systemctl status mongod
● mongod.service - MongoDB Database Server
   Loaded: loaded (/lib/systemd/system/mongod.service; disabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-12-03 06:58:49 EST; 8s ago
     Docs: https://docs.mongodb.org/manual
 Main PID: 3581 (mongod)
   Memory: 59.9M
   CGroup: /system.slice/mongod.service
           └─3581 /usr/bin/mongod --config /etc/mongod.conf

Dec 03 06:58:49 mongo systemd[1]: Started MongoDB Database Server.
{% endhighlight %}

Como era de esperar, el proceso se encuentra ya cargado y ejecutándose, por lo que ya podemos empezar a hacer uso del mismo. Antes de ponernos manos a la obra, me gustaría mencionar que el proceso no se levantará automáticamente tras un reinicio, ya que se encuentra **_disabled_**. En mi caso, lo prefiero así, pero si necesitáseis que se cargue automáticamente, podéis ejecutar el comando:

{% highlight shell %}
systemctl enable mongod
{% endhighlight %}

Como consecuencia del proceso que está actualmente ejecutándose, se habrá abierto un _socket TCP/IP_ en el puerto por defecto de MongoDB (**27017**) que estará escuchando peticiones provenientes de **_localhost_**, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@mongo:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.

Efectivamente, el proceso está escuchando peticiones tal y como debería, de manera que procederemos a abrir una _shell_ de MongoDB (conocida como **_mongo shell_**) para así poder gestionar el motor, sin especificar en esta ocasión ningún parámetro, pues por defecto se conectará a **_localhost_**, haciendo uso del puerto **27017**:

{% highlight shell %}
root@mongo:~# mongo
MongoDB shell version v4.4.2
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("53aaffc7-bd51-4629-96d4-d28accea2c30") }
MongoDB server version: 4.4.2
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	https://docs.mongodb.com/
Questions? Try the MongoDB Developer Community Forums
	https://community.mongodb.com
---
The server generated these startup warnings when booting: 
        2020-12-03T06:58:49.481-05:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
        2020-12-03T06:58:50.113-05:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
        2020-12-03T06:58:50.113-05:00: /sys/kernel/mm/transparent_hugepage/enabled is 'always'. We suggest setting it to 'never'
---
---
        Enable MongoDB's free cloud-based monitoring service, which will then receive and display
        metrics about your deployment (disk utilization, CPU, operation statistics, etc).

        The monitoring data will be available on a MongoDB website with a unique URL accessible to you
        and anyone you share the URL with. MongoDB may use this information to make product
        improvements and to suggest MongoDB products and deployment options to you.

        To enable free monitoring, run the following command: db.enableFreeMonitoring()
        To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---
>
{% endhighlight %}

Como se puede apreciar, la _mongo shell_ se ha abierto correctamente y está lista para su uso, sin embargo, nos ha mostrado una serie de advertencias que generó el servidor durante su arranque, existiendo entre ellas una que nos indica que el control de acceso no se encuentra habilitado para la base de datos, de manera que la lectura y la escritura de la información no presenta ningún tipo de restricción.

Para solucionarlo, tendremos que aumentar la seguridad del servidor, creando para ello un usuario con privilegios y habilitando posteriormente la autenticación. Para crear el usuario, tendremos que hacer uso de la base de datos **admin**, pues es donde se encuentran definidos aquellos usuarios con privilegios superiores, ejecutando para ello la instrucción:

{% highlight shell %}
> use admin
switched to db admin
{% endhighlight %}

Tal y como se nos ha mostrado en la salida de la instrucción, ya estamos ubicados en dicha base de datos, de manera que ahora tendremos que hacer uso del método **db.createUser()** para crear el nuevo usuario (ya que no existe un comando `CREATE` como en _Oracle Database_, por ejemplo), utilizando la sintaxis propia del lenguaje JSON, pues es el lenguaje soportado por MongoDB.

En este caso, tendremos que asignar además los roles correspondientes para que goce de los privilegios necesarios para ser administrador de cualquier base de datos, siendo el nombre del mismo **alvaro**, y la contraseña, **alvaro** (como se puede suponer, en una situación real se utilizarían credenciales más seguras):

{% highlight json %}
> db.createUser(
...   {
...     user: "alvaro",
...     pwd: "alvaro",
...     roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
...   }
... )
Successfully added user: {
	"user" : "alvaro",
	"roles" : [
		{
			"role" : "userAdminAnyDatabase",
			"db" : "admin"
		},
		"readWriteAnyDatabase"
	]
}
{% endhighlight %}

Una vez más, la ejecución del método ha sido exitosa, informándonos de ello en la salida del mismo. Ya hemos finalizado con la configuración necesaria dentro del motor de MongoDB, por lo que saldremos de la _mongo shell_ ejecutando para ello la instrucción:

{% highlight shell %}
> exit
bye
{% endhighlight %}

Cuando nos encontremos en la _shell_ de Linux, tendremos que proceder a modificar el fichero **/etc/mongod.conf** para así requerir una autenticación basada en usuario/contraseña antes de hacer uso de cualquier base de datos, aumentando por tanto la seguridad y evitando que se nos muestre la correspondiente advertencia. Para modificar el fichero, haremos uso del comando:

{% highlight shell %}
root@mongo:~# nano /etc/mongod.conf
{% endhighlight %}

Dentro del mismo, tendremos que descomentar la sección **security** y añadir la directiva **authorization**, cuyo valor, como es lógico, será **enabled**. El resultado final tendrá la siguiente forma:

{% highlight shell %}
security:
  authorization: enabled
{% endhighlight %}

Tras ello, guardaremos los cambios realizados y dado que hemos modificado un fichero de configuración de un servicio, tendremos que reiniciarlo para que así cargue la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@mongo:~# systemctl restart mongod
{% endhighlight %}

Una vez reiniciado el servicio, la autenticación habrá sido habilitada y a partir de este momento, necesitaremos hacer uso de las credenciales previamente establecidas durante la creación del usuario administrador para conectarnos a la _mongo shell_. Para comprobar que todo ha funcionado correctamente, vamos a realizar un intento de conexión, haciendo para ello uso del comando:

{% highlight shell %}
root@mongo:~# mongo --authenticationDatabase "admin" -u "alvaro" -p "alvaro"
MongoDB shell version v4.4.2
connecting to: mongodb://127.0.0.1:27017/?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("e762a418-de1e-423e-b1db-7e6102ded38e") }
MongoDB server version: 4.4.2
>
{% endhighlight %}

Donde:

* **--authenticationDatabase**: Especificamos la base de datos en la que el usuario al que nos vamos a tratar de conectar ha sido previamente definido. En este caso, **admin**.
* **-u**: Indicamos el nombre del usuario del que queremos hacer uso. En este caso, **alvaro**.
* **-p**: Indicamos la contraseña asociada al usuario del que queremos hacer uso. Podríamos obviarla y nos la pedirá por teclado de forma oculta. En este caso, **alvaro**.

Como se puede apreciar, la autenticación ha funcionado correctamente y estamos actualmente conectados a una _mongo shell_ haciendo uso del usuario administrador previamente definido. El gestor de bases de datos se encuentra ya totalmente funcional y sin mostrar ningún tipo de advertencia, de manera que ya hemos finalizado la primera mitad del artículo, referente a la instalación del mismo.

Para hacer un artículo un poco más completo, vamos a proceder a simular una situación real, en la que se tuviese que definir una nueva base de datos con un usuario con permisos para administrarla. En este caso, voy a crear una base de datos de nombre **empresa** en la que va a existir un usuario **empresario** que será el administrador de la misma, pero totalmente aislado del resto de bases de datos que se pudiesen crear en un futuro.

El primer paso será crear dicha base de datos, que a diferencia de _Oracle Database_, por ejemplo, no existe una instrucción `CREATE`, sino que utilizaremos `use` seguido del nombre de la base de datos, de manera que en caso de no existir, la creará automáticamente. La instrucción a ejecutar sería:

{% highlight shell %}
> use empresa
switched to db empresa
{% endhighlight %}

Ya estamos haciendo uso de la nueva base de datos, por lo que vamos a listar todas aquellas bases de datos existentes para comprobar si ha sido correctamente generada:

{% highlight shell %}
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
{% endhighlight %}

Muy posiblemente os estaréis preguntando por qué no aparece la base de datos que acabamos de generar cuando hemos listado todas las existentes. Bien, esto tiene una sencilla razón, y es que las bases de datos vacías no se muestran en el listado, de manera que hasta que no introduzcamos el primer **documento** (unidad básica de datos en MongoDB, es decir, lo equivalente a un registro en una tabla de una base de datos relacional), no aparecerá listada.

Antes de proceder con la inserción de documentos, vamos a crear el usuario **empresario** cuya contraseña será **password**, que contará con los privilegios suficientes para administrar la base de datos actual, consiguiendo así simular una situación más real, pues generalmente el administrador de sistemas no es el encargado de introducir dichos documentos. La creación la llevaremos a cabo haciendo uso del método **db.createUser()** previamente utilizado, pero adaptando en esta ocasión los roles para que únicamente tenga permisos de creador sobre la base de datos actual:

{% highlight json %}
> db.createUser(
...   {
...     user: "empresario",
...     pwd: "password",
...     roles: ["dbOwner"]
...   }
... )
Successfully added user: { "user" : "empresario", "roles" : [ "dbOwner" ] }
{% endhighlight %}

A pesar de que la salida del comando ha notificado que ha finalizado exitosamente, vamos a verificar que la creación de dicho usuario se ha realizado tal y como debería, listando para ello los usuarios definidos en la base de datos actual, haciendo uso de la instrucción:

{% highlight json %}
> show users
{
	"_id" : "empresa.empresario",
	"userId" : UUID("553a04a1-e647-4800-8271-6fe03113af20"),
	"user" : "empresario",
	"db" : "empresa",
	"roles" : [
		{
			"role" : "dbOwner",
			"db" : "empresa"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}
{% endhighlight %}

Efectivamente, el usuario **empresario** ha sido correctamente generado y está listo para ser utilizado, de manera que saldremos del usuario actual (**alvaro**), ejecutando para ello la instrucción `exit`, para así hacer uso del nuevo usuario, poniéndonos por tanto en la piel del dueño de una empresa que desee introducir información sobre determinados aspectos de la misma.

Una vez más, nos encontraremos en la _shell_ de Linux, así que realizaremos una nueva conexión a la _mongo shell_, pero haciendo uso esta vez, del nuevo usuario creado en la base de datos **empresa**, ejecutando para ello el comando:

{% highlight shell %}
root@mongo:~# mongo --authenticationDatabase "empresa" -u "empresario" -p "password"
MongoDB shell version v4.4.2
connecting to: mongodb://127.0.0.1:27017/?authSource=empresa&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("7077ae37-d036-44a0-82c9-5e3d9cf473ba") }
MongoDB server version: 4.4.2
>
{% endhighlight %}

La conexión se ha realizado correctamente, así que antes de comenzar a insertar documentos, vamos a comprobar de qué base de datos estamos haciendo uso. Para ello, haremos uso de la instrucción:

{% highlight shell %}
> db
test
{% endhighlight %}

Al no haber especificado explícitamente a qué base de datos nos queremos conectar, nos ha conectado a una base de datos de prueba, de nombre **test**, por lo que volveremos a hacer uso de la instrucción `use` para cambiarnos a la base de datos **empresa**:

{% highlight shell %}
> use empresa
switched to db empresa
{% endhighlight %}

Cuando estemos haciendo uso del usuario y la base de datos correcta, procederé a insertar algunos documentos en **colecciones** (están compuestas de documentos que almacenan información sobre la misma finalidad, es decir, lo equivalente a una tabla de una base de datos relacional). Para no ensuciar el artículo con la inserción de documentos en las colecciones, se podrá encontrar [aquí](https://pastebin.com/zaUWcy9n) las instrucciones ejecutadas para ello.

Una vez insertados los correspondientes documentos en las colecciones, saldremos del cliente ejecutando la instrucción `exit` para así continuar con la parte referente al uso de dicha base de datos de forma remota.

Lo primero que haremos será visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
root@mongo:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:7d:53:6e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.151/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86334sec preferred_lft 86334sec
    inet6 fe80::a00:27ff:fe7d:536e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
{% endhighlight %}

De las dos interfaces mostradas, la única que nos interesa es aquella de nombre **enp0s3**, que tiene un direccionamiento **192.168.1.151/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_). Nos será necesario conocer dicha información para la correspondiente conexión desde el cliente.

Como anteriormente hemos mostrado, el servicio **mongod** únicamente está escuchando peticiones provenientes de **_localhost_** en el puerto **27017**, de manera que no aceptará conexiones remotas hasta que no hagamos que escuche también en la interfaz **enp0s3**. Para llevar a cabo dicha adaptación, tendremos que modificar una vez más el fichero **/etc/mongod.conf**, haciendo para ello uso del comando:

{% highlight shell %}
root@mongo:~# nano /etc/mongod.conf
{% endhighlight %}

Dentro del mismo, buscaremos la sección **net**, que tendrá la siguiente forma:

{% highlight shell %}
net:
  port: 27017
  bindIp: 127.0.0.1
{% endhighlight %}

En nuestro caso, nos interesa modificar el valor de la directiva **bindIp** para que en lugar de escuchar en la dirección **127.0.0.1**, escuche en **0.0.0.0**, significando por tanto que aceptará conexiones en todas las interfaces de red existentes en la máquina (ya que el puerto asignado por defecto me parece correcto). A pesar de ello, podríamos indicar que únicamente escuche en **192.168.1.151**, pero no tendría sentido, ya que en ese caso, bloquearía las conexiones entrantes desde _localhost_, cosa que no nos interesa. El resultado final sería:

{% highlight shell %}
net:
  port: 27017
  bindIp: 0.0.0.0
{% endhighlight %}

Tras ello, guardaremos los cambios realizados y dado que hemos vuelto a modificar un fichero de configuración, tendremos que reiniciar el servicio para que así cargue la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@mongo:~# systemctl restart mongod
{% endhighlight %}

Una vez reiniciado el servicio, el _socket TCP/IP_ que anteriormente estaba únicamente configurado para escuchar peticiones provenientes de _localhost_ (**127.0.0.1**), debe estar ahora escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), por lo que procederemos a verificarlo haciendo uso una vez más de `netstat`:

{% highlight shell %}
root@mongo:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:27017           0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN
{% endhighlight %}

Efectivamente, el puerto 27017 está ahora abierto para escuchar peticiones desde cualquier interfaz, de manera que nos queda un último paso para poder conectarnos de forma remota a la base de datos.

Si lo pensamos, queremos limitar las conexiones a nuestra red local, en este caso, al direccionamiento **192.168.1.0/24**, por lo que en lugar de hacerlo a nivel de configuración, vamos a hacerlo a nivel de usuario, consiguiendo así una gestión más minuciosa y granular. Para ello, vamos a volver a conectarnos a la _mongo shell_, pero esta vez, haciendo uso del usuario administrador (**alvaro**):

{% highlight shell %}
root@mongo:~# mongo --authenticationDatabase "admin" -u "alvaro" -p "alvaro"
MongoDB shell version v4.4.2
connecting to: mongodb://127.0.0.1:27017/?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("da51e0b8-93b9-4007-8822-0a09c1223737") }
MongoDB server version: 4.4.2
>
{% endhighlight %}

Hago un pequeño inciso aquí para listar las bases de datos existentes, para así verificar que la base de datos **empresa** es ahora visible, ejecutando para ello la instrucción:

{% highlight shell %}
> show dbs
admin    0.000GB
config   0.000GB
empresa  0.000GB
local    0.000GB
{% endhighlight %}

Como era de esperar, dado que la base de datos ya tiene documentos en su interior, es ahora totalmente visible a la hora de listarlas.

Volviendo la configuración, tendremos que ubicarnos de nuevo en la base de datos **empresa**, pues el usuario que queremos configurar se encuentra definido en la misma. Para ello, volveremos a hacer uso de la instrucción `use`:

{% highlight shell %}
> use empresa
switched to db empresa
{% endhighlight %}

Tras ello, acudiremos al método **db.updateUser()** para modificar un usuario ya existente, en este caso, el usuario **empresario**, añadiéndole al mismo una restricción de autenticación para así limitar su acceso a aquellas IPs que provengan de la red local, es decir, de **192.168.1.0/24** (en caso de ser necesario, podríamos definir una IP de _host_ en lugar de una red completa). La instrucción a ejecutar sería:

{% highlight json %}
> db.updateUser(
...     "empresario",
...   {
...     authenticationRestrictions: [ { clientSource: ["192.168.1.0/24"] } ]
...   }
... )
{% endhighlight %}

Me gustaría puntualizar el hecho de que únicamente hemos permitido el acceso desde la red local, lo que supone que no podríamos conectarnos a dicho usuario desde la propia máquina servidora, ya que no hemos permitido el acceso desde **127.0.0.1**. Lo he hecho a propósito, para así posteriormente verificar que funciona, aunque dado que la directiva **clientSource** recibe una lista de parámetros, podríamos haber permitido también el acceso desde _localhost_ sin ningún tipo de problema.

En esta ocasión, la instrucción ejecutada no ha devuelto ninguna salida, generándome cierta inquietud ya que no tengo certeza de que la actualización se haya llevado a cabo, por lo que haré uso del método **db.runCommand()** para ejecutar el comando **usersInfo**, que recibe como parámetros el nombre de un usuario y la base de datos en la que se encuentra definido, mostrando por tanto la información que le solicitemos, en este caso, las restricciones de autenticación:

{% highlight json %}
> db.runCommand(
...    {
...      usersInfo:  { user: "empresario", db: "empresa" },
...      showAuthenticationRestrictions: true
...    }
... )
{
	"users" : [
		{
			"_id" : "empresa.empresario",
			"userId" : UUID("553a04a1-e647-4800-8271-6fe03113af20"),
			"user" : "empresario",
			"db" : "empresa",
			"mechanisms" : [
				"SCRAM-SHA-1",
				"SCRAM-SHA-256"
			],
			"roles" : [
				{
					"role" : "dbOwner",
					"db" : "empresa"
				}
			],
			"authenticationRestrictions" : [
				{
					"clientSource" : [
						"192.168.1.0/24"
					]
				}
			],
      ...
			"inheritedAuthenticationRestrictions" : [ ]
		}
	],
	"ok" : 1
}
{% endhighlight %}

Como se puede apreciar en la salida que manualmente he recortado para no ensuciar mucho, el usuario introducido cuenta con una restricción de autenticación que únicamente le permite acceder desde la red **192.168.1.0/24**, tal y como hemos establecido.

Ha llegado la hora de comprobar que la configuración que hemos realizado realmente funciona, así que lo primero que haremos será salir del usuario **alvaro** con la instrucción `exit` para posteriormente tratar de acceder al usuario **empresario** desde la máquina servidora:

{% highlight shell %}
root@mongo:~# mongo --authenticationDatabase "empresa" -u "empresario" -p "password"
MongoDB shell version v4.4.2
connecting to: mongodb://127.0.0.1:27017/?authSource=empresa&compressors=disabled&gssapiServiceName=mongodb
Error: Authentication failed. :
connect@src/mongo/shell/mongo.js:374:17
@(connect):2:6
exception: connect failed
exiting with code 1
{% endhighlight %}

Como era de esperar, la autenticación ha fallado, al no estar permitida para dicho usuario la dirección IP desde la que se está tratando de conectar (**127.0.0.1**).

De otro lado, he creado otra máquina virtual conectada a su vez en modo puente (_bridge_) a mi red doméstica, de manera que cuenta con direccionamiento dentro de la red permitida, con la que he seguido el mismo proceso de instalación que con la máquina servidora, pero sin llegar a levantar el proceso **mongod**, ya que no será necesario para realizar conexiones remotas.

Para llevar a cabo la conexión remota haremos uso de las opciones **-u** y **-p** anteriormente utilizadas para indicar el usuario y contraseña a utilizar, con la única diferencia que tendremos que indicar al final la dirección IP del servidor al que nos vamos a tratar de conectar, en este caso, **192.168.1.151**, así como la base de datos, en este caso, **empresa**. La instrucción a ejecutar sería:

{% highlight shell %}
debian@cliente:~$ mongo -u empresario -p password 192.168.1.151/empresa
MongoDB shell version v4.4.2
connecting to: mongodb://192.168.1.151:27017/empresa?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("66d021ea-efae-4a17-9472-b96f3b9effca") }
MongoDB server version: 4.4.2
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	https://docs.mongodb.com/
Questions? Try the MongoDB Developer Community Forums
	https://community.mongodb.com
>
{% endhighlight %}

Al parecer, la conexión se ha realizado sin ningún problema, pues tal y como he mencionado con anterioridad, la máquina cliente cuenta con un direccionamiento dentro de la red local, que se encuentra actualmente permitido.

De nada nos serviría tener conexión si no podemos mostrar la información contenida en la base de datos, así que antes de nada, vamos a asegurarnos que nos encontramos haciendo uso de la base de datos **empresa**, utilizando para ello la instrucción:

{% highlight shell %}
> db
empresa
{% endhighlight %}

Efectivamente, al haber indicado la base de datos de la que queremos hacer uso de forma explícita a la hora de la conexión, nos ha conectado automáticamente a la misma, por lo que vamos a proceder a listar las colecciones existentes, para así asegurarnos de que tenemos acceso a la información, ejecutando para ello la instrucción:

{% highlight shell %}
> show collections
empleados
productos
reuniones
{% endhighlight %}

Como era de esperar, las 3 colecciones que previamente he creado son visibles desde la máquina cliente, así que vamos a ir un paso mas allá y vamos a tratar de mostrar los documentos definidos en la colección **reuniones**, por ejemplo, haciendo para ello uso del método **db.[colección].find()**, que permitirá mostrar todos los documentos de una determinada colección, combinado con el método **pretty()**, que hará que la información se muestre de una forma más "visual" y fácil de entender. La instrucción de la que haremos uso sería:

{% highlight json %}
> db.reuniones.find().pretty()
{
	"_id" : ObjectId("5fc913951ff28cf908abed20"),
	"empresa" : "Intel",
	"fecha" : ISODate("2018-10-29T22:00:00Z"),
	"proposito" : "Discutir un posible acuerdo de venta de sus procesadores."
}
...
{
	"_id" : ObjectId("5fc913d41ff28cf908abed26"),
	"empresa" : "Stadium Goods",
	"fecha" : ISODate("2018-06-26T21:30:00Z"),
	"proposito" : "Discutir un posible acuerdo para programar su nueva página web."
}
{% endhighlight %}

Como se puede apreciar en la salida que manualmente he recortado para no ensuciar, se han mostrado todos los documentos pertenecientes a la colección indicada, por lo que podemos corroborar que la conexión remota a la base de datos está totalmente operativa y funcional para cualquier cliente ubicado en la red local.