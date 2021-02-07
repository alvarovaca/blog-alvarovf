---
layout: post
title:  "Instalación e interconexión de PostgreSQL en Debian 10"
banner: "/assets/images/banners/postgresql.png"
date:   2021-02-05 14:44:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una instalación básica de **PostgreSQL**, un gestor de bases de datos relacionales (_SQL_) sobre una máquina **Debian Buster**, así como la configuración necesaria para admitir peticiones desde máquinas remotas.

Además, se instalará el gestor sobre una segunda máquina para así llevar a cabo una interconexión entre las mismas, siendo su finalidad la de permitir el acceso desde un mismo cliente a dos bases de datos de forma simultánea pero indirecta, por ejemplo, para hacer un JOIN con tablas que se encuentren en distintos servidores mediante un enlace entre ellas, de manera que uno de los servidores actuará como cliente del otro servidor, de manera unidireccional.

Al fin y al cabo, el cliente únicamente abre una conexión, pues es el primer servidor al que se conecta el que posteriormente abre una conexión al segundo de ellos.

Para esta ocasión, he traído los deberes hechos y he instalado previamente dos máquinas virtuales con **Debian Buster**, pero no he llevado a cabo ninguna configuración, para así partir desde un punto totalmente limpio. No considero necesario explicar el proceso de instalación de dicha distribución, ya que es bastante sencillo, además de salirse del objetivo del artículo. Las máquinas actualmente existentes en el escenario son las siguientes:

* **servidor1**: Conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.160**.
* **servidor2**: Conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.161**.

El paquete del gestor de bases de datos que instalaremos será el proporcionado por los repositorios oficiales de Debian, una versión mantenida por usuarios de la distribución, de manera que no es necesario importar ningún repositorio externo.

Dado que la instalación de PostgreSQL no requiere ninguna configuración previa, podremos dar paso a la misma sobre la primera de las máquinas, no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando:

{% highlight shell %}
root@servidor1:~# apt update && apt upgrade && apt install postgresql
{% endhighlight %}

El paquete **postgresql** ha sido instalado en la máquina, existiendo en consecuencia un proceso que está actualmente en ejecución, que habrá abierto un _socket TCP/IP_ en el puerto por defecto de PostgreSQL (**5432**) que estará escuchando peticiones provenientes de **_localhost_**, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@servidor1:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
tcp6       0      0 ::1:5432                :::*                    LISTEN
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.

Efectivamente, el proceso está escuchando peticiones tal y como debería, de manera que procederemos a abrir una _shell_ de _psql_ para así poder gestionar el motor, cambiándonos previamente al usuario **postgres**, pues es el que cuenta con los privilegios necesarios para ello, ejecutando por tanto el comando:

{% highlight shell %}
root@servidor1:~# su - postgres
{% endhighlight %}

Cuando nos encontremos haciendo uso del usuario **postgres**, todo estará listo para abrir la _shell_ de _psql_, haciendo para ello uso del comando:

{% highlight shell %}
postgres@servidor1:~$ psql
psql (11.9 (Debian 11.9-0+deb10u1))
Type "help" for help.
{% endhighlight %}

Como se puede apreciar, la _psql shell_ se ha abierto correctamente y está lista para su uso.

Para hacer un artículo un poco más completo, vamos a proceder a simular una situación real, en la que se tuviese que definir una nueva base de datos con un rol con permisos para administrarla (un rol es lo equivalente a un usuario). En este caso, voy a crear una base de datos de nombre **prueba1** en la que va a existir un rol **alvaro1** que será el administrador de la misma, pero totalmente aislado del resto de bases de datos que se pudiesen crear en un futuro.

Lo primero que haremos estando dentro del servidor de bases de datos, será crear una nueva base de datos, valga la redundancia. En este caso, de nombre **prueba1**. Para ello, haremos uso de la instrucción:

{% highlight sql %}
postgres=# CREATE DATABASE prueba1;
CREATE DATABASE
{% endhighlight %}

La base de datos en cuestión habrá sido generada, pero para verificarlo, vamos a listar todas las bases de datos existentes, ejecutando para ello la instrucción:

{% highlight sql %}
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 prueba1   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
{% endhighlight %}

Efectivamente, la base de datos **prueba1** ha sido correctamente generada, sin embargo, de nada nos sirve tener una base de datos si no tenemos un rol que pueda acceder a la misma. En este caso, vamos a crear un rol de nombre "**alvaro1**" y cuya contraseña sea también "**alvaro1**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso de la instrucción:

{% highlight sql %}
postgres=# CREATE USER alvaro1 WITH PASSWORD 'alvaro1';
CREATE ROLE
{% endhighlight %}

El rol ha sido generado, pero todavía no tiene permisos sobre la base de datos que acabamos de crear, así que debemos otorgarle dichos privilegios. Para ello, ejecutaremos la instrucción:

{% highlight sql %}
postgres=# GRANT ALL PRIVILEGES ON DATABASE prueba1 TO alvaro1;
GRANT
{% endhighlight %}

Una vez realizadas las oportunas modificaciones, podremos salir de _psql_, con la finalidad de conectarnos al nuevo rol para así verificar que todo funciona correctamente, haciendo para ello uso del comando:

{% highlight sql %}
postgres=# exit
{% endhighlight %}

Para comprobar que todo ha funcionado correctamente, vamos a realizar un intento de conexión utilizando para ello el nuevo rol **alvaro1** que acabamos de generar, haciendo para ello uso del comando:

{% highlight shell %}
postgres@servidor1:~$ psql -h localhost -U alvaro1 -d prueba1
Password for user alvaro1: 
psql (11.9 (Debian 11.9-0+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
{% endhighlight %}

Donde:

* **-h**: Especificamos el nombre de la máquina a la que nos queremos conectar, en este caso, **_localhost_**.
* **-U**: Especificamos el rol con el que nos queremos conectar, en este caso, **alvaro1**.
* **-d**: Especificamos la base de datos a la que nos queremos conectar, en este caso, **prueba1**.

Como se puede apreciar, la autenticación ha funcionado correctamente y estamos actualmente conectados a una _psql shell_ haciendo uso del rol que acabamos de definir en la base de datos que acabamos de crear. El gestor de bases de datos se encuentra ya totalmente funcional.

Tras ello, procederé a crear algunas tablas e insertar una serie de registros. Para no ensuciar el artículo con la inserción de tablas y registros, se podrá encontrar [aquí](https://pastebin.com/U2ziGtU5) las instrucciones ejecutadas para ello.

Una vez insertadas las correspondientes tablas y registros, saldremos del cliente ejecutando la instrucción `exit` para así continuar con la parte referente al uso de dicha base de datos de forma remota.

Lo primero que haremos será visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
root@servidor1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:7e:a4:98 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.160/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86346sec preferred_lft 86346sec
    inet6 fe80::a00:27ff:fe7e:a498/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
{% endhighlight %}

De las dos interfaces mostradas, la única que nos interesa es aquella de nombre **enp0s3**, que tiene un direccionamiento **192.168.1.160/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_). Nos será necesario conocer dicha información para la correspondiente conexión remota.

Como anteriormente hemos mostrado, el servicio **postgresql** únicamente está escuchando peticiones provenientes de **_localhost_** en el puerto **5432**, de manera que no aceptará conexiones remotas hasta que no hagamos que escuche también en la interfaz **enp0s3**. Para llevar a cabo dicha adaptación, tendremos que modificar el fichero **/etc/postgresql/11/main/postgresql.conf**, haciendo para ello uso del comando:

{% highlight shell %}
root@servidor1:~# nano /etc/postgresql/11/main/postgresql.conf
{% endhighlight %}

Dentro del mismo, buscaremos la directiva **listen_addresses**, que tendrá la siguiente forma:

{% highlight shell %}
#listen_addresses = 'localhost'
{% endhighlight %}

En nuestro caso, nos interesa modificar el valor para que en lugar de escuchar en la dirección **_localhost_ (127.0.0.1)**, escuche en __*__, significando por tanto que aceptará conexiones en todas las interfaces de red existentes en la máquina. El resultado final sería:

{% highlight shell %}
listen_addresses = '*'
{% endhighlight %}

Personalmente, no tengo inconveniente en que el puerto que dicho servicio utilice para las conexiones entrantes sea el **5432**, pero en caso contrario, se podría modificar el valor de la directiva **port** en ese mismo fichero, estableciendo así el que más convenga.

Una vez finalizada la configuración, saldremos del editor guardando los cambios y procederemos a modificar ahora un segundo fichero de nombre **/etc/postgresql/11/main/pg_hba.conf** en el que tendremos que indicar una vez más, quién puede utilizar nuestro _cluster_ PostgreSQL, ejecutando para ello el comando:

{% highlight shell %}
root@servidor1:~# nano /etc/postgresql/11/main/pg_hba.conf
{% endhighlight %}

Dentro del mismo, buscaremos la directiva **host**, que tendrá la siguiente forma:

{% highlight shell %}
host    all             all             127.0.0.1/32            md5
{% endhighlight %}

Una vez más, nos interesa modificar el valor para que en lugar de escuchar en la dirección **127.0.0.1**, escuche en __all__, significando por tanto que aceptará conexiones en todas las interfaces de red existentes en la máquina. El resultado final sería:

{% highlight shell %}
host    all             all             all            md5
{% endhighlight %}

Tras ello, guardaremos los cambios realizados y dado que hemos modificado dos ficheros de configuración, tendremos que reiniciar el servicio para que así cargue la nueva configuración en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@servidor1:~# systemctl restart postgresql
{% endhighlight %}

Una vez reiniciado el servicio, el _socket TCP/IP_ que anteriormente estaba únicamente configurado para escuchar peticiones provenientes de _localhost_ (**127.0.0.1**), debe estar ahora escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**), por lo que procederemos a verificarlo haciendo uso una vez más de `netstat`:

{% highlight shell %}
root@servidor1:~# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
tcp6       0      0 :::5432                 :::*                    LISTEN
{% endhighlight %}

Efectivamente, el puerto 5432 está ahora abierto para escuchar peticiones desde cualquier interfaz.

Resumiendo, tenemos un servicio que está actualmente escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**) en el puerto **5432**. Además, hemos configurado un rol **alvaro1** que cuenta con los permisos necesarios para hacer uso de la base de datos **prueba1**.

De otro lado, tendremos que repetir en la máquina servidora **servidor2** el mismo [procedimiento](https://pastebin.com/XbdqVH0p) seguido en la máquina **servidor1**, con la única diferencia de que el rol a generar será ahora **alvaro2** y el nombre de la base de datos será **prueba2**, para así poder diferenciarlos de los anteriores.

Resumiendo una vez más, tenemos un servicio que está actualmente escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**) en el puerto **5432**. Además, hemos configurado un rol **alvaro2** que cuenta con los permisos necesarios para hacer uso de la base de datos **prueba2**.

Ambas máquinas servidoras se encuentran ya configuradas y listas para recibir conexiones remotas, de manera que todo está listo para interconectarlas. Para ello, realizaremos el procedimiento de forma ordenada, configurando primero el enlace de la máquina **servidor1** a la máquina **servidor2** y posteriormente, a la inversa, ya que dichas conexiones son unidireccionales.

Un requisito indispensable para poder crear dicho enlace es tener instalado en la máquina origen el paquete **postgresql-contrib**, de manera que lo instalaremos ejecutando para ello el comando:

{% highlight shell %}
root@servidor1:~# apt install postgresql-contrib
{% endhighlight %}

Una vez instalado, tendremos que cambiarnos una vez más al usuario **postgres**, pues es el que cuenta con los privilegios necesarios para crear dicho enlace, haciendo para ello uso del comando:

{% highlight shell %}
root@servidor1:~# su - postgres
{% endhighlight %}

Cuando nos encontremos utilizando el usuario **postgres**, todo estará listo para abrir la _shell_ de _psql_ haciendo uso de la base de datos **prueba1**, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor1:~$ psql -d prueba1
psql (11.9 (Debian 11.9-0+deb10u1))
Type "help" for help.
{% endhighlight %}

Es muy importante que la conexión se realice a la base de datos **prueba1**, ya que es donde el rol **alvaro1** tiene privilegios, pues de lo contrario, no podría utilizar dicho enlace. La creación del mismo es muy sencilla, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
prueba1=# CREATE EXTENSION dblink;
CREATE EXTENSION
{% endhighlight %}

La extensión que hará la función de enlace ya ha sido creada, pero para verificarlo, vamos a listar todas las extensiones existentes, haciendo para ello uso de la instrucción:

{% highlight sql %}
prueba1=# \dx
                                 List of installed extensions
  Name   | Version |   Schema   |                         Description                          
---------+---------+------------+--------------------------------------------------------------
 dblink  | 1.2     | public     | connect to other PostgreSQL databases from within a database
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)
{% endhighlight %}

Efectivamente, la extensión **dblink** ha sido correctamente generada, de manera que podremos salir del cliente ejecutando la instrucción `exit` para así realizar una nueva conexión a la base de datos **prueba1** pero haciendo uso esta vez del rol **alvaro1**, y así verificar que puede utilizar dicho enlace, ejecutando para ello el comando visto con anterioridad:

{% highlight shell %}
postgres@servidor1:~$ psql -h localhost -U alvaro1 -d prueba1
Password for user alvaro1: 
psql (11.9 (Debian 11.9-0+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
{% endhighlight %}

Una vez más, la autenticación ha funcionado como debería, de manera que vamos a proceder a llevar a cabo una consulta cuya información mostrada provenga de ambas bases de datos, ubicadas como ya hemos visto, en servidores distintos.

La intención es mostrar los datos de los **Empleados** (tabla que se encuentra en **servidor1**) junto al nombre del departamento al que pertenecen, información que podremos obtener de **Departamentos** (tabla que se encuentra en **servidor2**), utilizando para ello la clave que tienen en común ambas tablas. La consulta a ejecutar sería:

{% highlight sql %}
prueba1=> SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos.Nombre AS Departamento
prueba1-> FROM dblink('dbname=prueba2 host=192.168.1.161 user=alvaro2 password=alvaro2', 'SELECT Identificador, Nombre FROM Departamentos') AS Departamentos (Identificador NUMERIC, Nombre VARCHAR)
prueba1-> LEFT OUTER JOIN Empleados
prueba1-> ON (Empleados.Departamento = Departamentos.Identificador);
    dni    |          nombre           |       direccion        | telefono  | fechanacimiento | salario |   departamento   
-----------+---------------------------+------------------------+-----------+-----------------+---------+------------------
 85145590G | Anabel Lerma Dominguez    | Crta. Cadiz, 1         | 675014823 | 1981-01-07      |    5919 | Administración
 67227129S | Ian Esquivel Laboy        | C/ Inglaterra, 64      | 728005136 | 1992-07-21      |    3519 | Administración
 90389058R | Joaquin Marrero Covas     | C/ Hijuela de Lojo, 22 | 618385118 | 1997-01-28      |    1446 | Administración
 52315160G | Heinz Collado Caraballo   | Escuadro, 60           | 600173822 | 1984-02-13      |    1672 | Recursos Humanos
 12777631G | Merlino Rosado Cordero    | C/ Henan Cortes, 58    | 609841755 | 1993-11-27      |    7961 | Recursos Humanos
 56228957Y | Dinorah Viera Tello       | Ctra. Villena, 22      | 642852778 | 1987-05-28      |    2567 | Seguridad
 94106513N | Marian Fonseca Betancourt | C/ Manuel Iradier, 37  | 638415823 | 1979-07-17      |    2561 | Seguridad
 18232747A | Aristarco Caban Meraz     | Puerta Nueva, 67       | 691204722 | 1994-05-07      |    4789 | Seguridad
 61242562W | Manases Castillo Camacho  | Ctra. Hornos, 91       | 607853354 | 1991-10-29      |    4270 | Informática
 68219319P | Tabare Chapa Alcantar     | C/ Arana, 12           | 682227206 | 1992-03-09      |    8568 | Informática
(10 rows)
{% endhighlight %}

Donde:

* **SELECT**: Indicamos las columnas que queremos mostrar de la información obtenida, de la forma **[tabla].[columna]**.
* **FROM**: Hacemos uso del enlace que acabamos de generar, indicando en el mismo los parámetros necesarios para la conexión (base de datos, dirección IP, usuario, contraseña y consulta que realizar a la base de datos remota). Tras ello, indicamos el tipo de datos de cada una de las columnas que la consulta remota nos ha devuelto, para que el gestor pueda interpretarlas correctamente.
* **LEFT OUTER JOIN**: Hacemos un JOIN de la consulta remota con la tabla Empleados que se encuentra en la base de datos local, para así poder mostrar información de ambas tablas en la misma consulta.
* **ON**: Establecemos la condición del JOIN, que deberá ser aquella columna mediante la cual vamos a unir los registros devueltos. Como es lógico, será el código del departamento, pues es la columna que se repite en ambas tablas.

Al parecer, la consulta se ha realizado sin ningún problema y ha devuelto la información que debería, pues tal y como he mencionado con anterioridad, ambas máquinas servidoras cuentan con un direccionamiento dentro de la red local, siendo ambas totalmente alcanzables entre sí, además de estar correctamente configuradas para aceptar dichas conexiones.

El enlace ha funcionado del extremo **servidor1** al extremo **servidor2**, pero como ya sabemos, dichos enlaces son unidireccionales, de manera que si quisiésemos realizar la conexión a la inversa, tendríamos que repetir el mismo procedimiento en la segunda máquina, así que vamos a proceder a ello.

Una vez más, tendremos que tener instalado el paquete **postgresql-contrib**, el cuál instalaremos ejecutando para ello el comando:

{% highlight shell %}
root@servidor2:~# apt install postgresql-contrib
{% endhighlight %}

Una vez instalado, tendremos que cambiarnos una vez más al usuario **postgres**, pues es el que cuenta con los privilegios necesarios para crear dicho enlace, haciendo para ello uso del comando:

{% highlight shell %}
root@servidor2:~# su - postgres
{% endhighlight %}

Cuando nos encontremos utilizando el usuario **postgres**, todo estará listo para abrir la _shell_ de _psql_ haciendo uso de la base de datos **prueba2**, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor2:~$ psql -d prueba2
psql (11.9 (Debian 11.9-0+deb10u1))
Type "help" for help.
{% endhighlight %}

Es muy importante que la conexión se realice a la base de datos **prueba2**, ya que es donde el rol **alvaro2** tiene privilegios, pues de lo contrario, no podría utilizar dicho enlace. La creación del mismo es muy sencilla, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
prueba2=# CREATE EXTENSION dblink;
CREATE EXTENSION
{% endhighlight %}

La extensión que hará la función de enlace ya ha sido creada, pero para verificarlo, vamos a listar todas las extensiones existentes, haciendo para ello uso de la instrucción:

{% highlight sql %}
prueba2=# \dx
                                 List of installed extensions
  Name   | Version |   Schema   |                         Description                          
---------+---------+------------+--------------------------------------------------------------
 dblink  | 1.2     | public     | connect to other PostgreSQL databases from within a database
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)
{% endhighlight %}

Efectivamente, la extensión **dblink** ha sido correctamente generada, de manera que podremos salir del cliente ejecutando la instrucción `exit` para así realizar una nueva conexión a la base de datos **prueba2** pero haciendo uso esta vez del rol **alvaro2**, y así verificar que puede utilizar dicho enlace, ejecutando para ello el comando visto con anterioridad:

{% highlight shell %}
postgres@servidor2:~$ psql -h localhost -U alvaro2 -d prueba2
Password for user alvaro2: 
psql (11.9 (Debian 11.9-0+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
{% endhighlight %}

Una vez más, la autenticación ha funcionado como debería, de manera que vamos a proceder a llevar a cabo una consulta cuya información mostrada provenga de ambas bases de datos, ubicadas como ya hemos visto, en servidores distintos.

La intención es realizar la misma consulta que en el caso anterior, pero adaptándola para consultar al primero de los servidores sobre la información de la tabla **Empleados**, ya que la tabla **Departamentos** se encuentra ahora en la máquina local. La consulta a ejecutar sería:

{% highlight sql %}
prueba2=> SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos.Nombre AS Departamento
prueba2-> FROM dblink('dbname=prueba1 host=192.168.1.160 user=alvaro1 password=alvaro1', 'SELECT * FROM Empleados') AS Empleados (DNI VARCHAR, Nombre VARCHAR, Direccion VARCHAR, Telefono VARCHAR, FechaNacimiento DATE, Salario NUMERIC, Departamento NUMERIC)
prueba2-> LEFT OUTER JOIN Departamentos
prueba2-> ON (Empleados.Departamento = Departamentos.Identificador);
    dni    |          nombre           |       direccion        | telefono  | fechanacimiento | salario |   departamento   
-----------+---------------------------+------------------------+-----------+-----------------+---------+------------------
 90389058R | Joaquin Marrero Covas     | C/ Hijuela de Lojo, 22 | 618385118 | 1997-01-28      |    1446 | Administración
 18232747A | Aristarco Caban Meraz     | Puerta Nueva, 67       | 691204722 | 1994-05-07      |    4789 | Seguridad
 94106513N | Marian Fonseca Betancourt | C/ Manuel Iradier, 37  | 638415823 | 1979-07-17      |    2561 | Seguridad
 12777631G | Merlino Rosado Cordero    | C/ Henan Cortes, 58    | 609841755 | 1993-11-27      |    7961 | Recursos Humanos
 68219319P | Tabare Chapa Alcantar     | C/ Arana, 12           | 682227206 | 1992-03-09      |    8568 | Informática
 67227129S | Ian Esquivel Laboy        | C/ Inglaterra, 64      | 728005136 | 1992-07-21      |    3519 | Administración
 52315160G | Heinz Collado Caraballo   | Escuadro, 60           | 600173822 | 1984-02-13      |    1672 | Recursos Humanos
 85145590G | Anabel Lerma Dominguez    | Crta. Cadiz, 1         | 675014823 | 1981-01-07      |    5919 | Administración
 56228957Y | Dinorah Viera Tello       | Ctra. Villena, 22      | 642852778 | 1987-05-28      |    2567 | Seguridad
 61242562W | Manases Castillo Camacho  | Ctra. Hornos, 91       | 607853354 | 1991-10-29      |    4270 | Informática
(10 rows)
{% endhighlight %}

Como era de esperar, la consulta ha vuelto a realizarse sin ningún problema y ha devuelto la información que debería, de manera que podemos corroborar que los servidores tienen conectividad entre sí mediante los enlaces creados, sea cual sea el sentido utilizado.