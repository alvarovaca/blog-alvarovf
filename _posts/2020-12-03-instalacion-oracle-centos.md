---
layout: post
title:  "Instalación de Oracle 19c en CentOS 8"
banner: "/assets/images/banners/oracle.jpg"
date:   2020-12-03 12:41:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una instalación básica de **Oracle 19c**, un gestor de bases de datos relacionales (_DBMS_) sobre una máquina **CentOS 8**, así como la configuración necesaria para admitir peticiones desde máquinas remotas dentro de una red local. Personalmente, hasta el día de hoy, únicamente he instalado _Oracle Database_ en máquinas _Windows_, cuyo proceso de instalación es bastante trivial, pues consiste en pulsar "**Siguiente**" en reiteradas ocasiones, por lo que además de ayudar a cualquier persona que pueda necesitarlo, este artículo lo escribo por si me hiciese falta en algún momento de mi vida laboral.

Para esta ocasión, ya he traído los deberes hechos y he instalado previamente una máquina virtual con **CentOS 8**, pero no he llevado a cabo ninguna configuración, para así partir desde un punto totalmente limpio. No considero necesario explicar el proceso de instalación de dicha distribución, ya que es bastante sencillo, además de salirse del objetivo del artículo.

El primer paso como es lógico, será descargar el paquete comprimido de **Oracle 19c** de la página oficial de descargas de [Oracle](https://www.oracle.com/es/downloads/) (concretamente la versión **19.3**, aunque intuyo que los pasos a seguir serían los mismos independientemente de la versión), en la que se nos solicitará crear una cuenta en caso de que no tengamos, para así poder llevar a cabo la descarga.

Al estar trabajando con una distribución CentOS (_Red Hat_), tenemos soporte para paquetes **.rpm** de forma nativa, por lo que descargaré el fichero en dicho formato (en caso de estar trabajando con _Debian_, tendríamos que convertir dicho paquete descargado a un **.deb**, haciendo uso de `alien`, pero Oracle no nos garantiza la compatibilidad con dicha distribución).

Dado que la descarga requiere autentificación del usuario, no podremos hacer uso de `wget` para descargarlo, por lo que en mi caso, al tener entorno gráfico la máquina virtual, lo descargaré manualmente. En caso de no tenerlo, siempre se puede descargar en la máquina anfitriona y transferirlo por `scp` haciendo uso por debajo del protocolo SSH. En conclusión, maneras hay miles.

Para verificar que la descarga se ha realizado correctamente, vamos a listar el contenido del directorio personal, pues es donde he almacenado el fichero **.rpm**, estableciendo además un filtro por nombre, ejecutando para ello el comando:

{% highlight shell %}
[alvaro@localhost ~]$ ls -l | egrep 'oracle'
-rw-rw-r--. 1 alvaro alvaro 2694664264 nov 30 19:27 oracle-database-ee-19c-1.0-1.x86_64.rpm
{% endhighlight %}

Efectivamente, el paquete ha sido correctamente descargado y se encuentra contenido en el directorio actual, pero todavía no vamos a proceder a instalarlo, ya que tenemos que llevar a cabo unas consideraciones previas para asegurar el correcto funcionamiento del gestor de bases de datos. Para ello, se nos proporciona un paquete que lleva a cabo dichas configuraciones de forma automática, entre las que se encuentran:

* Descargar e instalar todas aquellas dependencias necesarias.
* Crear el usuario **oracle** e incluirlo en los grupos necesarios.
* Establecer determinadas configuraciones en el fichero **sysctl.conf** según las recomendaciones.
* Establecer límites duros (_hard_) y blandos (_soft_) para el uso de recursos.
* Establecer otros parámetros recomendados, dependiendo de la versión del _kernel_.

Una vez que conocemos las modificaciones que va a llevar a cabo dicho paquete sobre nuestra máquina, podremos proceder con su instalación, ejecutando para ello el comando (con privilegios de administrador, haciendo uso de `su -`):

{% highlight shell %}
[root@localhost ~]# dnf install https://yum.oracle.com/repo/OracleLinux/OL8/baseos/latest/x86_64/getPackage/oracle-database-preinstall-19c-1.0-1.el8.x86_64.rpm
{% endhighlight %}

La correspondiente configuración previa se habrá llevado a cabo gracias a dicho paquete, de manera que ya está todo listo para instalar el fichero **.rpm** de _Oracle Database 19c_, haciendo para ello uso de `rpm`:

{% highlight shell %}
[root@localhost ~]# rpm -Uhv oracle-database-ee-19c-1.0-1.x86_64.rpm
advertencia:oracle-database-ee-19c-1.0-1.x86_64.rpm: EncabezadoV3 RSA/SHA256 Signature, ID de clave ec551f03: NOKEY
Verifying...                          ################################# [100%]
Preparando...                         ################################# [100%]
Actualizando / instalando...
   1:oracle-database-ee-19c-1.0-1     ################################# [100%]
[INFO] Executing post installation scripts...
[INFO] Oracle home installed successfully and ready to be configured.
To configure a sample Oracle Database you can execute the following service configuration script as root: /etc/init.d/oracledb_ORCLCDB-19c configure
{% endhighlight %}

Donde:

* **-U**: Indica que se eliminen todas las versiones anteriores del paquete en caso de existir. En este caso, se podría haber obviado ya que no existe ninguna otra versión, pero la dejo por si en vuestro caso, tuviéseis una instalación previa.
* **-h**: Indica que se muestre una barra de progreso en forma de **#** para asegurarnos que la instalación se está llevando a cabo.
* **-v**: Complementa a la opción anterior, mostrando más información respectiva a la instalación.

Tras mostrarnos una advertencia indicando que no se ha podido comprobar la integridad del fichero instalado (principalmente debido a que no tenemos la clave pública de Oracle instalada), advertencia que podremos dejar de lado, el proceso de extracción e instalación del gestor de base de datos habrá finalizado, indicando finalmente que en caso de querer crear una base de datos de ejemplo, podemos hacerlo ejecutando el comando mostrado.

Dicho comando hace uso de un fichero en el que se encuentra la configuración por defecto a utilizar para generar dicha base de datos, que debemos comprobar previamente para asegurarnos que todo esté correcto. Dicho fichero se encuentra dentro de **/etc/sysconfig/**, con el nombre **oracledb_ORCLCDB-19c.conf**, así que visualizaremos su contenido haciendo uso de `cat`:

{% highlight shell %}
[root@localhost ~]# cat /etc/sysconfig/oracledb_ORCLCDB-19c.conf
#This is a configuration file to setup the Oracle Database.
#It is used when running '/etc/init.d/oracledb_ORCLCDB configure'.
#Please use this file to modify the default listener port and the
#Oracle data location.
 
# LISTENER_PORT: Database listener
LISTENER_PORT=1521
 
# ORACLE_DATA_LOCATION: Database oradata location
ORACLE_DATA_LOCATION=/opt/oracle/oradata
 
# EM_EXPRESS_PORT: Oracle EM Express listener
EM_EXPRESS_PORT=5500
{% endhighlight %}

Los parámetros más importantes para nosotros son los dos primeros, en los que se indica el puerto en el que va a escuchar el _listener_ que posteriormente configuraremos (**LISTENER_PORT**), y la ubicación de la base de datos (**ORACLE_DATA_LOCATION**), respectivamente. En mi caso, considero adecuados los valores asignados por defecto, pero si os fuese necesario, podéis modificarlos a vuestro interés.

Una vez adaptados los parámetros del fichero anteriormente mencionado, es hora de ejecutar el _script_ para crear la base de datos de ejemplo, facilitándonos por tanto el procedimiento de creación. Dicho _script_ creará una base de datos de nombre **ORCLCDB**, así como una base de datos "enchufable" (_pluggable_) de nombre **ORCLPDB1**. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@localhost ~]# /etc/init.d/oracledb_ORCLCDB-19c configure
Configuring Oracle Database ORCLCDB.
Preparar para funcionamiento de base de datos
8% finalizado
Copiando archivos de base de datos
31% finalizado
Creando e iniciando instancia Oracle
32% finalizado
36% finalizado
40% finalizado
43% finalizado
46% finalizado
Terminando creación de base de datos
51% finalizado
54% finalizado
Creando Bases de Datos de Conexión
58% finalizado
77% finalizado
Ejecutando acciones posteriores a la configuración
100% finalizado
Creación de la base de datos terminada. Consulte los archivos log de /opt/oracle/cfgtoollogs/dbca/ORCLCDB
 para obtener más información.
Información de Base de Datos:
Nombre de la Base de Datos Global:ORCLCDB
Identificador del Sistema (SID):ORCLCDB
Para obtener información detallada, consulte el archivo log "/opt/oracle/cfgtoollogs/dbca/ORCLCDB/ORCLCDB.log".
 
Database configuration completed successfully. The passwords were auto generated, you must change them by connecting to the database using 'sqlplus / as sysdba' as the oracle user.
{% endhighlight %}

La base de datos de prueba ya ha sido generada y podremos empezar a utilizarla, así como todos los binarios ejecutables que incluye la instalación que previamente hemos realizado, que se encuentran contenidos en **/opt/oracle/product/19c/dbhome_1/bin/**, pero sería poco práctico tener que indicar la ruta completa cada vez que queramos hacer uso de `sqlplus`, por ejemplo.

Para solventar dicho problema, llevaremos a cabo las modificaciones necesarias en el fichero **.bash_profile**, fichero cuya configuración es cargada durante cada inicio de sesión, consiguiendo así definir las variables de entorno necesarias en el usuario **oracle**. Lo primero que tendremos que hacer, como es lógico, es cambiarnos a dicho usuario, ejecutando para ello el comando:

{% highlight shell %}
[root@localhost ~]# su - oracle
{% endhighlight %}

Cuando nos encontremos dentro del usuario **oracle**, haremos uso del editor `vi` para modificar el fichero previamente mencionado, ya que a pesar de poder haber instalado `nano` previamente, en mi caso, no me supone ningún inconveniente:

{% highlight shell %}
[oracle@localhost ~]$ vi ~/.bash_profile
{% endhighlight %}

Dentro del mismo, encontraremos por defecto el siguiente contenido:

{% highlight shell %}
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi
{% endhighlight %}

Tras introducir la configuración necesaria, entre la que se encuentra la definición de la ruta en la que se encuentra almacenada la base de datos, el nombre de la misma, la ruta de los binarios, una actualización de la variable **$PATH** para que mire dentro de dicho directorio a la hora de utilizar un comando... el fichero quedaría de la siguiente forma:

{% highlight shell %}
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi
 
umask 022
export ORACLE_SID=ORCLCDB
export ORACLE_BASE=/opt/oracle/oradata
export ORACLE_HOME=/opt/oracle/product/19c/dbhome_1
export PATH=$PATH:$ORACLE_HOME/bin
{% endhighlight %}

Cuando finalicemos la configuración en dicho fichero, procederemos a guardar los cambios en el mismo. Sin embargo, los cambios no habrán surtido efecto de manera inmediata, ya que como previamente he mencionado, el contenido se carga en memoria tras el inicio de sesión del usuario, así que tenemos varias opciones, como por ejemplo reiniciar la máquina o cerrar sesión y volver a abrirla, aunque si tenemos unas mínimas nociones sobre _scripts_, sabremos que haciendo uso de `source` podremos volver a leer el contenido de dicho fichero sin necesidad de cerrar sesión o reiniciar:

{% highlight shell %}
[oracle@localhost ~]$ source ~/.bash_profile
{% endhighlight %}

La nueva configuración entre la que se encuentran las variables de entorno definidas ya ha sido cargada en memoria, por lo que todo está listo para comprobar que la instalación se ha completado de forma satisfactoria. Para ello, haremos uso de `sqlplus` para acceder a la base de datos haciendo uso del usuario **sysdba**, para así tener los privilegios necesarios para llevar a cabo determinadas acciones:

{% highlight shell %}
[oracle@localhost ~]$ sqlplus / as sysdba 
 
SQL*Plus: Release 19.0.0.0.0 - Production on Tue Dec 1 08:31:29 2020
Version 19.3.0.0.0
 
Copyright (c) 1982, 2019, Oracle.  All rights reserved.
 
Connected to an idle instance.
 
SQL>
{% endhighlight %}

Como se puede apreciar, en un principio hemos podido acceder correctamente al cliente **sqlplus**, así que vamos a consultar determinada información sobre la base de datos para así verificar que su funcionamiento es el correcto, y que por tanto, nos devuelve la información solicitada:

{% highlight sql %}
SQL> SELECT instance_name, host_name, version, startup_time FROM v$instance;
SELECT instance_name, host_name, version, startup_time FROM v$instance
*
ERROR at line 1:
ORA-01034: ORACLE not available
Process ID: 0
Session ID: 0 Serial number: 0
{% endhighlight %}

En esta ocasión, nos ha devuelto un error, debido a que la base de datos a la que queremos consultar no se encuentra actualmente montada. Para montarla y poder hacer uso de la misma, tendremos que ejecutar la instrucción **STARTUP**, tal y como se muestra:

{% highlight sql %}
SQL> STARTUP     
ORACLE instance started.
 
Total System Global Area 1593832664 bytes
Fixed Size		    9135320 bytes
Variable Size		  922746880 bytes
Database Buffers	  654311424 bytes
Redo Buffers		    7639040 bytes
Base de datos montada.
Base de datos abierta.
{% endhighlight %}

Según muestra la salida de dicha instrucción, la base de datos ya ha sido correctamente montada y abierta, por lo que volveremos a ejecutar la consulta anteriormente utilizada para comprobar que en esta ocasión funciona:

{% highlight sql %}
SQL> SELECT instance_name, host_name, version, startup_time FROM v$instance;

INSTANCE_NAME
----------------
HOST_NAME
----------------------------------------------------------------
VERSION 	  STARTUP_
----------------- --------
ORCLCDB
localhost.localdomain
19.0.0.0.0	  01/12/20
{% endhighlight %}

Como era de esperar, la consulta ha devuelto en este caso la información solicitada (aunque un poco desordenada, pero no es algo relevante), como por ejemplo el nombre de la base de datos, el nombre de la máquina sobre la que se está ejecutando, la versión de Oracle, la fecha de arranque...

Bien, llegados a este punto, me gustaría hacer una aclaración. Anteriormente he hecho uso del término base de datos "**enchufable**" (_pluggable_), término que muy posiblemente, si hasta ahora has estado trabajando con versiones inferiores a **Oracle 12c**, no conozcas, pues a partir de ese entonces, se diferencian dos tipos de bases de datos:

* **Container DataBase (_CDB_)**
* **Pluggable DataBase (_PDB_)**

A su vez, podemos diferenciar dos tipos de usuarios:

* **Common user**: Son aquellos usuarios que pertenecen a las _CDBs_, así como a todas las futuras _PDBs_ que se creen. En otras palabras, son usuarios que pueden realizar operaciones tanto en _CDBs_ como en _PDBs_, dependiendo de los privilegios asignados. Se crean con la sintaxis comúnmente conocida, anteponiendo **c##** al nombre de usuario (puede forzarse su creación sin anteponer dicha cadena al nombre, pero no es algo recomendable, ya que habría que modificar una variable de sesión que idealmente únicamente debe ser modificada por el propio gestor de bases de datos, y no por un usuario, para no alterar el correcto funcionamiento de la misma).
* **Local user**: Son aquellos usuarios que pertenecen a una única _PDB_. En otras palabras, son aquellos usuarios a los que se les pueden asignar privilegios de administrador, pero únicamente para la _PDB_ en la que existen. Se crean estableciendo la _PDB_ como variable de sesión (`ALTER SESSION SET CONTAINER = NombrePDB`) y posteriormente creándolo con la sintaxis comúnmente conocida.

Una vez explicada la teoría, voy a proceder a crear un usuario común (_common user_) para así establecerlo de forma "global" y poder usarlo tanto en la _CDB_ actual como en futuras _PDBs_ que se creen. El nombre para dicho usuario será **alvaro** (**c##alvaro**), y la contraseña será también **alvaro** (como se puede suponer, en una situación real se utilizarían credenciales más seguras):

{% highlight sql %}
SQL> CREATE USER c##alvaro IDENTIFIED BY alvaro;
 
Usuario creado.
{% endhighlight %}

Cuando el usuario haya sido generado, lo dotaremos de todos los privilegios para poder trabajar con el mismo sin ningún tipo de restricción (una vez más, reitero que en una situación real habría que hacer una gestión de permisos de forma granular y específica para que no suponga un agujero de seguridad en la base de datos), ejecutando para ello el comando:

{% highlight sql %}
SQL> GRANT ALL PRIVILEGES TO c##alvaro;
 
Concesion terminada correctamente.
{% endhighlight %}

Ya hemos finalizado con la creación y configuración de privilegios para el nuevo usuario, así que el siguiente paso será desconectarnos del usuario actual (**sysdba**), haciendo uso de la instrucción:

{% highlight sql %}
SQL> DISCONNECT
Desconectado de Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
{% endhighlight %}

Tras ello, trataremos de acceder al nuevo usuario creado, indicando seguidamente su contraseña (o bien, indicando únicamente el nombre de usuario, de manera que la contraseña se pedirá por teclado de forma oculta):

{% highlight sql %}
SQL> CONNECT c##alvaro/alvaro
Conectado.
{% endhighlight %}

Como se puede apreciar, la conexión al nuevo usuario se ha realizado exitosamente, por lo que para simular una situación un poco más "real", he procedido a crear algunas tablas e insertar una serie de registros. Para no ensuciar el artículo con la inserción de tablas y registros, se podrá encontrar [aquí](https://pastebin.com/Q7b1xW07) las instrucciones ejecutadas para ello.

Una vez insertadas las correspondientes tablas y registros, ya habremos completado la primera parte del artículo, correspondiente a la instalación de **Oracle Database 19c** en **CentOS 8**, de manera que saldremos del cliente **sqlplus** para continuar con la parte referente al uso de dicha base de datos de forma remota. Para ello, ejecutaremos la instrucción:

{% highlight sql %}
SQL> exit 
Desconectado de Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
{% endhighlight %}

Tras ello, volveremos al usuario **root** con el que previamente estábamos trabajando para comenzar a configurar el **_listener_**, aunque también podríamos haberle asignado una contraseña al usuario **oracle** y otorgarle privilegios de administrador. Lo dejo a gusto del consumidor.

Lo primero que haremos será visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a1:7f:cd brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.150/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86167sec preferred_lft 86167sec
    inet6 fe80::cb97:f2fd:40b3:a170/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:6e:3d:4e brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:6e:3d:4e brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

De todas las interfaces mostradas, la única que nos interesa es aquella de nombre **enp0s3**, que tiene un direccionamiento **192.168.1.150/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_). Nos será necesario conocer dicha información para la siguiente configuración que realizaremos.

Antes de ponernos manos a la obra, me gustaría mostrar las actuales direcciones y puertos en los que mi máquina está escuchando peticiones en el protocolo TCP, ejecutando para ello el comando:

{% highlight shell %}
[root@localhost ~]# netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:5355            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
tcp6       0      0 :::34987                :::*                    LISTEN     
tcp6       0      0 :::5355                 :::*                    LISTEN     
tcp6       0      0 :::111                  :::*                    LISTEN
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.

Como se puede apreciar, de todos los puertos en los que se está escuchando, ninguno de ellos corresponde al que utiliza por defecto el _listener_ de Oracle, es decir, el puerto **1521**. Eso se debe a que dicho proceso no se encuentra activo ni mucho menos, configurado.

El primer paso que tendremos que llevar a cabo, y como requisito por parte de Oracle, será configurar el fichero **/etc/hosts** para establecer una resolución estática de nombres para la máquina actual, que actúa como servidora de base de datos. Para ello, haremos uso del comando:

{% highlight shell %}
[root@localhost ~]# vi /etc/hosts
{% endhighlight %}

Dentro del mismo, encontraremos por defecto el siguiente contenido:

{% highlight shell %}
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
{% endhighlight %}

Tendremos que introducir manualmente una línea de la forma **[IP] [FQDN] [ALIAS]**, donde **[IP]** es la dirección IP de la máquina servidora alcanzable desde el exterior (es decir, **192.168.1.150**), donde **[FQDN]** es un nombre de dominio totalmente cualificado que le asignaremos a la máquina, que actuará como _hostname_ (por ejemplo, **oracle.alvarovf.com**) y donde **[ALIAS]** será el nombre del servicio o máquina asignado dentro del _FQDN_ (es decir, **oracle**). El resultado final sería:

{% highlight shell %}
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.150   oracle.alvarovf.com     oracle
{% endhighlight %}

Tras ello, guardaremos los cambios y reiniciaremos la máquina para que cargue en memoria el nuevo **_hostname_** asignado, ejecutando para ello el comando `reboot`.

Una vez reiniciada, haremos uso del comando `hostname` para verificar que el cambio se ha realizado correctamente, aunque a simple vista ya podremos apreciar que el nombre de la máquina en el _prompt_ ha variado:

{% highlight shell %}
[alvaro@oracle ~]$ hostname
oracle.alvarovf.com
{% endhighlight %}

Genial, la máquina ya conoce su _FQDN_ y es capaz de resolverlo a la dirección IP desde la que es alcanzable en la red local, así que todo está listo para comenzar a configurar el **_listener_**.

Para quién no conozca lo que es el _listener_, es el proceso encargado de escuchar las peticiones entrantes en el lado del servidor, cuyo fichero en el que se define su configuración se encuentra almacenado en **$ORACLE_HOME/network/admin/**, con el nombre **listener.ora**, estableciéndose en el mismo configuración entre la que se encuentran las bases de datos para las que va a escuchar peticiones y los puertos en los que va a hacerlo.

Dado que nos encontramos actualmente en un usuario que no tiene definida la variable de entorno **$ORACLE_HOME**, me he tomado la molestia de consultarla previamente, de manera que sé que dicho directorio se encuentra ubicado en **/opt/oracle/product/19c/dbhome_1/**, por lo que procederemos a modificar dicho fichero ejecutando para ello el comando:

{% highlight shell %}
[alvaro@oracle ~]$ sudo vi /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
{% endhighlight %}

Dentro del mismo, encontraremos por defecto el siguiente contenido:

{% highlight shell %}
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
{% endhighlight %}

En mi caso, me encuentro conforme con que las peticiones se escuchen en el puerto (**PORT**) 1521, por lo que no me será necesario llevar a cabo ninguna modificación en lo que a ello respecta. Sin embargo, en el parámetro **HOST** podemos apreciar que únicamente se van a escuchar peticiones provenientes del propio servidor local, al estar configurado como **localhost**.

Para solucionarlo, modificaremos el valor de dicho parámetro al _FQDN_ previamente configurado (**oracle.alvarovf.com**), para así escuchar peticiones en cualquier interfaz de red (**IN_ADDRANY**), aunque en realidad, podríamos especificar la dirección IP en su lugar, pero es más cómodo hacerlo así, pues tendremos centralizada la resolución de nombres en el fichero **/etc/hosts**, por lo que ante un posible cambio de dirección IP de la máquina, bastaría con modificarlo allí.

En este caso, el hecho de estar escuchando peticiones en todas las interfaces de red existentes es irrelevante ya que únicamente tenemos una interfaz conectada al exterior. El resultado final del fichero sería:

{% highlight shell %}
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = oracle.alvarovf.com)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
{% endhighlight %}

Tras ello, simplemente guardaremos los cambios realizados y todo estará listo para arrancar el proceso del _listener_. Para ello, nos cambiaremos al usuario **oracle** con la instrucción previamente utilizada y haremos uso del siguiente comando:

{% highlight shell %}
[oracle@oracle ~]$ lsnrctl start

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 03-DEC-2020 09:28:26

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Starting /opt/oracle/product/19c/dbhome_1/bin/tnslsnr: please wait...

TNSLSNR for Linux: Version 19.0.0.0.0 - Production
System parameter file is /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
Log messages written to /opt/oracle/diag/tnslsnr/oracle/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle.alvarovf.com)(PORT=1521)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracle.alvarovf.com)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                03-DEC-2020 09:28:26
Uptime                    0 days 0 hr. 0 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/oracle/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle.alvarovf.com)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
The listener supports no services
The command completed successfully
{% endhighlight %}

Como se puede apreciar, tras algunos algunos mensajes informativos de la configuración que está siendo utilizada, se ha notificado que el comando se ha completado satisfactoriamente, pero a pesar de ello, nos ha informado que el _listener_ no está soportando ningún servicio. Esto es debido a que hemos arrancado el _listener_ después de la base de datos, por lo que tendremos que esperar un periodo que ronda los 60 segundos hasta que la base de datos notifique al _listener_ que se encuentra activa y lista para procesar peticiones.

Cuando haya transcurrido aproximadamente el periodo anteriormente mencionado, procederemos a visualizar una vez más la información referente al proceso de escucha, para así verificar que tiene ya conocimiento sobre la base de datos que se encuentra activa y lista para escuchar peticiones, ejecutando para ello el comando:

{% highlight shell %}
[oracle@oracle ~]$ lsnrctl status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 03-DEC-2020 09:29:23

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracle.alvarovf.com)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                03-DEC-2020 09:28:26
Uptime                    0 days 0 hr. 0 min. 56 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/oracle/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle.alvarovf.com)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=oracle.alvarovf.com)(PORT=5500))(Security=(my_wallet_directory=/opt/oracle/oradata/admin/ORCLCDB/xdb_wallet))(Presentation=HTTP)(Session=RAW))
Services Summary...
Service "ORCLCDB" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "ORCLCDBXDB" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "b5589a33e48c35e1e055000000000001" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "orclpdb1" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
The command completed successfully
{% endhighlight %}

Efectivamente, tras 56 segundos de espera, el _listener_ ya ha levantado los servicios correspondientes a la base de datos que tenemos creada. Para verificar que nuestra máquina ahora sí está escuchando peticiones en el puerto **1521**, volveremos a hacer uso de `netstat`:

{% highlight shell %}
[oracle@oracle ~]$ netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:5355            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
tcp6       0      0 :::34987                :::*                    LISTEN     
tcp6       0      0 :::5355                 :::*                    LISTEN     
tcp6       0      0 :::111                  :::*                    LISTEN     
tcp6       0      0 :::1521                 :::*                    LISTEN
{% endhighlight %}

Como se puede apreciar, nuestra máquina está escuchando ahora las peticiones entrantes en el puerto 1521 desde cualquier interfaz de red, dado que hemos especificado el _FQDN_ de nuestra máquina como valor del parámetro **HOST** en el fichero **listener.ora**.

Aquí llega la parte que más dolores de cabeza me ha causado, y es que al estar acostumbrado al uso de **Debian**, no recordaba que en **CentOS 8** se utilizaba por defecto un _firewall_ que de primeras es bastante restrictivo, de nombre **firewalld**. En el mismo, viene permitido poco más que el acceso por SSH, de manera que el puerto 1521 se encuentra cerrado, bloqueando todas las peticiones entrantes al mismo. Para verificar el estado de dicho cortafuegos, ejecutaremos el comando:

{% highlight shell %}
[oracle@oracle ~]$ firewall-cmd --state
running
{% endhighlight %}

Donde:

* **--state**: Indicamos que compruebe si el _daemon_ **firewalld** se encuentra o no activo.

Efectivamente, el servicio se encuentra activo, por lo que vamos a proceder a listar todas aquellas reglas que se encuentran actualmente configuradas y habilitadas en el cortafuegos, haciendo uso del comando:

{% highlight shell %}
[oracle@oracle ~]$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
{% endhighlight %}

Donde:

* **--list-all**: Indicamos que muestre todas las reglas activas para la zona por defecto.

En esta ocasión, únicamente se encuentran permitidas las conexiones entrantes de 3 servicios, entre los que lógicamente, no se encuentra el _listener_ de Oracle, por lo que procederemos a añadir dicha regla de forma manual, ejecutando para ello el comando:

{% highlight shell %}
[oracle@oracle ~]$ firewall-cmd --permanent --add-port=1521/tcp
success
{% endhighlight %}

Donde:

* **--permanent**: Indicamos que la regla perdure incluso tras un reinicio. El cambio no surtirá efecto inmediatamente.
* **--add-port**: Indicamos el puerto y el protocolo permitido en dicha regla.

Al parecer, la regla se ha añadido correctamente al cortafuegos de forma permanente, por lo que tal y como he mencionado, los cambios no surtirán efecto inmediatamente, de manera que tendremos que reiniciar el servicio. Para ello podemos hacer uso de `systemctl` o bien de la propia opción que trae incluida `firewall-cmd`:

{% highlight shell %}
[oracle@oracle ~]$ firewall-cmd --reload
success
{% endhighlight %}

Donde:

* **--reload**: Indicamos que reinicie el servicio de cortafuegos, consiguiendo así que las reglas permanentes se carguen en memoria.

Genial, en un principio la nueva regla ya ha sido añadida y se encuentra actualmente activa, por lo que volveremos a listar todas las reglas existentes para así verificarlo, ejecutando para ello el comando utilizado con anterioridad:

{% highlight shell %}
[oracle@oracle ~]$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 1521/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
{% endhighlight %}

Como era de esperar, la regla se ha añadido correctamente y el puerto **1521/TCP** se encuentra actualmente abierto y con posibilidad de recibir peticiones, de manera que podemos confirmar que ya hemos finalizado la configuración en el lado del servidor.

De otro lado, he configurado otra máquina virtual CentOS 8 que actuará como cliente, conectada a su vez en modo puente a mi red doméstica, por lo que podrá alcanzar sin problemas a la máquina servidora. De nuevo, me encuentro haciendo uso del usuario **oracle**, para así contar con todas las utilidades necesarias para hacer las correspondientes pruebas.

Lo primero que haremos será comprobar la conectividad con la máquina servidora, pero no una conectividad normal y corriente como la que nos podría ofrecer el comando `ping`, sino conectividad con el _listener_ de Oracle, para así confirmar que tenemos acceso al puerto 1521. Para ello, haremos uso de `tnsping`, indicando a su vez la dirección IP de la máquina a la que nos queremos conectar (en este caso, **192.168.1.150**):

{% highlight shell %}
[oracle@cliente ~]$ tnsping 192.168.1.150

TNS Ping Utility for Linux: Version 19.0.0.0.0 - Production on 01-DEC-2020 18:34:20

Copyright (c) 1997, 2019, Oracle.  All rights reserved.

Used parameter files:
/opt/oracle/product/19c/dbhome_1/network/admin/sqlnet.ora

Used HOSTNAME adapter to resolve the alias
Attempting to contact (DESCRIPTION=(CONNECT_DATA=(SERVICE_NAME=))(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.150)(PORT=1521)))
OK (0 msec)
{% endhighlight %}

Como se puede apreciar en la última línea de la salida del comando, el intento de conexión con el _listener_ ha sido exitoso en un tiempo total de 0 ms, pues ambas máquinas virtuales están ubicadas en la misma máquina física, por lo que el tiempo de transferencia es ridículamente minúsculo.

Esto es un buen indicio de que la configuración está funcionando tal y como debería, así que ahora, vamos a dar paso a _la prueba de fuego_, tratando de llevar a cabo una conexión real a la base de datos, como si de una situación normal y corriente se tratase, haciendo para ello uso de la siguiente sintaxis:

{% highlight shell %}
sqlplus [usuario]/[contraseña]@[servidor]/[bd]
{% endhighlight %}

Donde:

* **usuario**: Indicamos el nombre del usuario del que queremos hacer uso, el cuál debe contar con los privilegios suficientes para ello. En este caso, **c##alvaro**.
* **contraseña**: Indicamos la contraseña asociada al usuario del que queremos hacer uso. Podríamos obviarla y nos la pedirá por teclado de forma oculta. En este caso, **alvaro**.
* **servidor**: Indicamos la dirección IP de la máquina servidora alcanzable desde la máquina cliente. En este caso, **192.168.1.150**.
* **bd**: Indicamos el nombre de la base de datos a la que nos queremos conectar. En este caso, la que hemos generado de ejemplo, **ORCLCDB**.

El resultado final sería:

{% highlight shell %}
[oracle@cliente ~]$ sqlplus c##alvaro/alvaro@192.168.1.150/ORCLCDB

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Dec 3 09:32:40 2020
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Hora de Ultima Conexion Correcta: Jue Dic 03 2020 09:27:07 +01:00

Conectado a:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL>
{% endhighlight %}

Tal y como se puede apreciar en la salida del comando, nos hemos logrado conectar correctamente a la base de datos ubicada en la máquina servidora, por lo que vamos a proceder a listar todas las tablas que el usuario **alvaro** tiene privilegios para ver, es decir, aquellas que hemos creado manualmente. Para ello, ejecutaremos la instrucción:

{% highlight sql %}
SQL> SELECT * FROM cat;

TABLE_NAME
--------------------------------------------------------------------------------
TABLE_TYPE
-----------
VIAJES
TABLE

EMPLEADOS
TABLE

VIAJESPOREMPLEADO
TABLE
{% endhighlight %}

Efectivamente, las tres tablas que anteriormente hemos creado de forma manual son visibles y accesibles por el usuario **alvaro**, al cuál estamos conectado de forma remota desde una máquina que actúa como cliente.

Antes de finalizar, me gustaría puntualizar una cosa, y es que en el lado del cliente existe un fichero de nombre **tnsnames.ora**, ubicado también en **$ORACLE_HOME/network/admin/**, que permite facilitar la tarea de acceso a servidores remotos, indicando en el mismo una entrada por cada uno de los servidores a los que se pretende acceder.

Dichas entradas contienen a su vez el protocolo, la dirección IP y el puerto, así como un alias que será el que se utilice a la hora de la conexión a un determinado servidor. Su configuración no es para nada complicada, pero considero que se sale un poco del objetivo de este artículo, así que quizás, más adelante me anime y haga un _post_ al respecto.