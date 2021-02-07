---
layout: post
title:  "Interconexión de Oracle 19c en CentOS 8"
banner: "/assets/images/banners/oracle.jpg"
date:   2021-02-04 19:32:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una interconexión entre dos servidores **Oracle 19c** alojados en máquinas **CentOS 8**, siendo su finalidad la de permitir el acceso desde un mismo cliente a dos bases de datos de forma simultánea pero indirecta, por ejemplo, para hacer un JOIN con tablas que se encuentren en distintos servidores mediante un enlace entre ellas, de manera que uno de los servidores actuará como cliente del otro servidor, de manera unidireccional.

Al fin y al cabo, el cliente únicamente abre una conexión, pues es el primer servidor al que se conecta el que posteriormente abre una conexión al segundo de ellos.

Para esta ocasión, he traído los deberes hechos y he instalado previamente dos máquinas virtuales con **CentOS 8**, a las que he instalado a su vez el gestor de bases de datos **Oracle 19c**, siguiendo para ello los pasos indicados con mayor detenimiento en el artículo [Instalación de Oracle 19c en CentOS 8](https://www.alvarovf.com/bbdd/2020/12/03/instalacion-oracle-centos.html). Las máquinas actualmente existentes en el escenario son las siguientes:

* **oracle1**: Conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.150**.
* **oracle2**: Conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.151**.

Además de la instalación, he llevado a cabo sobre ambos servidores las configuraciones necesarias para montar las bases de datos, así como permitir conexiones remotas, estableciendo para ello el correspondiente _FQDN_ a la máquina y editando el **listener.ora**, además de levantar, como es lógico, el **_listener_**. Si los pasos seguidos no han quedado claros, se pueden encontrar de forma más detallada en el artículo anteriormente mencionado.

Cuando la configuración inicial haya sido correctamente realizada en ambas máquinas servidoras, procederemos a abrir una _shell_ de _sqlplus_ en la primera de ellas para así poder gestionar el motor, cambiándonos previamente al usuario **oracle**, pues es el único que tiene actualmente definida en sus variables de entorno la ruta a los binarios de Oracle, ejecutando por tanto el comando:

{% highlight shell %}
[root@oracle1 ~]# su - oracle
{% endhighlight %}

Cuando nos encontremos haciendo uso del usuario **oracle**, todo estará listo para abrir la _shell_ de _sqlplus_ utilizando el usuario **sysdba**, pues es el que cuenta con los privilegios necesarios para llevar a cabo determinadas acciones, haciendo para ello uso del comando:

{% highlight shell %}
[oracle@oracle1 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Feb 2 17:07:14 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.
{% endhighlight %}

Como se puede apreciar, la _sqlplus shell_ se ha abierto correctamente y está lista para su uso.

Para hacer un artículo un poco más completo, vamos a proceder a simular una situación real, en la que se tuviese que definir un nuevo usuario (_schema_) que pueda hacer uso tanto de la _CDB_ actual como de futuras _PDBs_ que se creen. En este caso, voy a crear un usuario común (_common user_) de nombre "**c##alvaro1**" y cuya contraseña sea también "**alvaro1**" (lógicamente, en un caso real se usarían credenciales más seguras). Para ello, haremos uso de la instrucción:

{% highlight sql %}
SQL> CREATE USER c##alvaro1 IDENTIFIED BY alvaro1;   

Usuario creado.
{% endhighlight %}

Cuando el usuario haya sido generado, lo dotaremos de todos los privilegios para poder trabajar con el mismo sin ningún tipo de restricción (es importante mencionar que en una situación real habría que hacer una gestión de permisos de forma granular y específica para que no suponga un agujero de seguridad en la base de datos), ejecutando para ello el comando:

{% highlight sql %}
SQL> GRANT ALL PRIVILEGES TO c##alvaro1;

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
SQL> CONNECT c##alvaro1/alvaro1
Conectado.
{% endhighlight %}

Como se puede apreciar, la conexión al nuevo usuario se ha realizado exitosamente, por lo que he procedido a crear algunas tablas e insertar una serie de registros. Para no ensuciar el artículo con la inserción de tablas y registros, se podrá encontrar [aquí](https://pastebin.com/M2FZPtiy) las instrucciones ejecutadas para ello.

Una vez insertadas las correspondientes tablas y registros, saldremos del cliente ejecutando la instrucción `exit` para así visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, pues nos será necesario conocer dicha información para la correspondiente conexión remota, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
[oracle@oracle1 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a1:7f:cd brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.150/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86359sec preferred_lft 86359sec
    inet6 fe80::6522:996d:56b3:45b0/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:6e:3d:4e brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:6e:3d:4e brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

De las cuatro interfaces mostradas, la única que nos interesa es aquella de nombre **enp0s3**, que tiene un direccionamiento **192.168.1.150/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_).

Resumiendo, tenemos un servicio que está actualmente escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**) en el puerto **1521**. Además, hemos configurado un usuario **c##alvaro1** que cuenta con los permisos necesarios para hacer uso de la base de datos.

De otro lado, tendremos que repetir en la máquina servidora **oracle2** el mismo [procedimiento](https://pastebin.com/4FbyvHrR) seguido en la máquina **oracle1**, con la única diferencia de que el usuario a generar será ahora **c##alvaro2**, para así poder diferenciarlo del anterior.

Resumiendo una vez más, tenemos un servicio que está actualmente escuchando peticiones en todas las interfaces de la máquina (**0.0.0.0**) en el puerto **1521**. Además, hemos configurado un usuario **c##alvaro2** que cuenta con los permisos necesarios para hacer uso de la base de datos.

Ambas máquinas servidoras se encuentran ya totalmente configuradas, de manera que todo está listo para interconectarlas. Para ello, realizaremos el procedimiento de forma ordenada, configurando primero el enlace de la máquina **oracle1** a la máquina **oracle2** y posteriormente, a la inversa, ya que dichas conexiones son unidireccionales.

Lo primero que haremos será comprobar la conectividad a la máquina **oracle2**, pero no una conectividad normal y corriente como la que nos podría ofrecer el comando `ping`, sino conectividad con el _listener_ de Oracle, para así confirmar que tenemos acceso al puerto 1521. Para ello, haremos uso de `tnsping`, indicando a su vez la dirección IP de la máquina a la que nos queremos conectar (en este caso, **192.168.1.151**):

{% highlight shell %}
[oracle@oracle1 ~]$ tnsping 192.168.1.151

TNS Ping Utility for Linux: Version 19.0.0.0.0 - Production on 02-FEB-2021 18:15:34

Copyright (c) 1997, 2019, Oracle.  All rights reserved.

Used parameter files:
/opt/oracle/product/19c/dbhome_1/network/admin/sqlnet.ora

Used HOSTNAME adapter to resolve the alias
Attempting to contact (DESCRIPTION=(CONNECT_DATA=(SERVICE_NAME=))(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.151)(PORT=1521)))
OK (0 msec)
{% endhighlight %}

Como se puede apreciar en la última línea de la salida del comando, el intento de conexión con el _listener_ ha sido exitoso en un tiempo total de 0 ms, pues ambas máquinas virtuales están ubicadas en la misma máquina física, por lo que el tiempo de transferencia es ridículamente minúsculo.

El siguiente paso será modificar el fichero de nombre **tnsnames.ora**, ubicado en **$ORACLE_HOME/network/admin/**, que permite facilitar la tarea de acceso a servidores remotos, indicando en el mismo una entrada por cada uno de los servidores a los que se pretende acceder.

Dichas entradas contienen a su vez el protocolo, la dirección IP y el puerto, así como un alias que será el que se utilice a la hora de la conexión a un determinado servidor. Su configuración no es para nada complicada, así que vamos a llevarla a cabo ejecutando para ello el comando:

{% highlight shell %}
[root@oracle1 ~]# nano /opt/oracle/product/19c/dbhome_1/network/admin/tnsnames.ora
{% endhighlight %}

El contenido que encontraremos por defecto dentro de dicho fichero es el siguiente:

{% highlight shell %}
ORCLCDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )

LISTENER_ORCLCDB =
  (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
{% endhighlight %}

Dentro del mismo, tendremos que definir un nuevo alias para la máquina a la que pretendemos conectarnos, en este caso, para **oracle2**. El nombre del mismo no influye, de manera que en mi caso lo llamaré **ORACLE2**. El protocolo de conexión será **TCP** a la dirección **192.168.1.151**, utilizando el puerto por defecto, **1521**. Por último, tendremos que indicar el nombre del servicio/base de datos al que queremos conectarnos, que por defecto será **ORCLCDB**. El resultado final sería:

{% highlight shell %}
ORCLCDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )

LISTENER_ORCLCDB =
  (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))

ORACLE2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.151)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )
{% endhighlight %}

Una vez modificado el fichero **tnsnames.ora**, tendremos que cambiarnos una vez más al usuario **oracle** para utilizar el binario **sqlplus**, haciendo para ello uso del comando:

{% highlight shell %}
[root@oracle1 ~]# su - oracle
{% endhighlight %}

Cuando nos encontremos utilizando el usuario **oracle**, todo estará listo para abrir la _shell_ de _sqlplus_ haciendo uso del usuario **c##alvaro1**, ejecutando para ello el comando:

{% highlight shell %}
[oracle@oracle1 ~]$ sqlplus c##alvaro1/alvaro1

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Feb 2 17:07:14 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.
{% endhighlight %}

La creación del enlace será muy sencilla, ya que previamente hemos definido los parámetros de la conexión en el fichero **tnsnames.ora**, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
SQL> CREATE DATABASE LINK oracle2link
CONNECT TO c##alvaro2 IDENTIFIED BY alvaro2
USING 'oracle2';

Enlace con la base de datos creado.
{% endhighlight %}

Donde:

* **CREATE DATABASE LINK**: Especificamos un nombre identificativo para el enlace.
* **CONNECT TO**: Indicamos las credenciales de acceso a la base de datos remota.
* **USING**: Indicamos el nombre del alias de la conexión que previamente hemos definido en el fichero **tnsnames.ora**.

Efectivamente, el enlace **oracle2link** ha sido correctamente generado, de manera que vamos a proceder a verificar el funcionamiento de dicho enlace, ejecutando para ello una consulta cuya información mostrada provenga de ambas bases de datos, ubicadas como ya hemos visto, en servidores distintos.

La intención es mostrar los datos de los **Empleados** (tabla que se encuentra en **oracle1**) junto al nombre del departamento al que pertenecen, información que podremos obtener de **Departamentos** (tabla que se encuentra en **oracle2**), utilizando para ello la clave que tienen en común ambas tablas. La consulta a ejecutar sería:

{% highlight sql %}
SQL> SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos.Nombre AS Departamento
FROM Empleados, Departamentos@oracle2link Departamentos
WHERE Empleados.Departamento = Departamentos.Identificador;

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
90389058R Joaquin Marrero Covas 	 C/ Hijuela de Lojo, 22    618385118
28/01/97       1446 Administraci??n

67227129S Ian Esquivel Laboy		 C/ Inglaterra, 64	   728005136
21/07/92       3519 Administraci??n

85145590G Anabel Lerma Dominguez	 Crta. Cadiz, 1 	   675014823
07/01/81       5919 Administraci??n


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
12777631G Merlino Rosado Cordero	 C/ Henan Cortes, 58	   609841755
27/11/93       7961 Recursos Humanos

52315160G Heinz Collado Caraballo	 Escuadro, 60		   600173822
13/02/84       1672 Recursos Humanos

18232747A Aristarco Caban Meraz 	 Puerta Nueva, 67	   691204722
07/05/94       4789 Seguridad


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
94106513N Marian Fonseca Betancourt	 C/ Manuel Iradier, 37	   638415823
17/07/79       2561 Seguridad

56228957Y Dinorah Viera Tello		 Ctra. Villena, 22	   642852778
28/05/87       2567 Seguridad

68219319P Tabare Chapa Alcantar 	 C/ Arana, 12		   682227206
09/03/92       8568 Inform??tica


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
61242562W Manases Castillo Camacho	 Ctra. Hornos, 91	   607853354
29/10/91       4270 Inform??tica


10 filas seleccionadas.
{% endhighlight %}

Donde:

* **SELECT**: Indicamos las columnas que queremos mostrar de la información obtenida, de la forma **[tabla].[columna]**.
* **FROM**: Hacemos un JOIN de la tabla Empleados que se encuentra en la base de datos local junto a la consulta remota, haciendo uso del enlace que acabamos de generar, para así poder mostrar información de ambas tablas en la misma consulta.
* **WHERE**: Establecemos la condición del JOIN, que deberá ser aquella columna mediante la cual vamos a unir los registros devueltos. Como es lógico, será el código del departamento, pues es la columna que se repite en ambas tablas.

Al parecer, la consulta se ha realizado sin ningún problema y ha devuelto la información que debería, pues tal y como he mencionado con anterioridad, ambas máquinas servidoras cuentan con un direccionamiento dentro de la red local, siendo ambas totalmente alcanzables entre sí, además de estar correctamente configuradas para aceptar dichas conexiones.

El enlace ha funcionado del extremo **oracle1** al extremo **oracle2**, pero como ya sabemos, dichos enlaces son unidireccionales, de manera que si quisiésemos realizar la conexión a la inversa, tendríamos que repetir el mismo procedimiento en la segunda máquina, así que vamos a proceder a ello.

Una vez más, tendremos que comprobar la conectividad con el _listener_ de la máquina **oracle1**, haciendo uso de `tnsping`, indicando a su vez la dirección IP de la máquina a la que nos queremos conectar (en este caso, **192.168.1.150**):

{% highlight shell %}
[oracle@oracle2 ~]$ tnsping 192.168.1.150

TNS Ping Utility for Linux: Version 19.0.0.0.0 - Production on 02-FEB-2021 18:44:48

Copyright (c) 1997, 2019, Oracle.  All rights reserved.

Used parameter files:
/opt/oracle/product/19c/dbhome_1/network/admin/sqlnet.ora

Used HOSTNAME adapter to resolve the alias
Attempting to contact (DESCRIPTION=(CONNECT_DATA=(SERVICE_NAME=))(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.150)(PORT=1521)))
OK (0 msec)
{% endhighlight %}

Como era de esperar, el intento de conexión con el _listener_ ha sido exitoso una vez más en un tiempo total de 0 ms.

El siguiente paso será modificar el fichero de nombre **tnsnames.ora**, ubicado en **$ORACLE_HOME/network/admin/**, indicando en el mismo una entrada para el nuevo servidor al que se pretende acceder, ejecutando para ello el comando:

{% highlight shell %}
[root@oracle2 ~]# nano /opt/oracle/product/19c/dbhome_1/network/admin/tnsnames.ora
{% endhighlight %}

Dentro del mismo, tendremos que definir un nuevo alias para la máquina a la que pretendemos conectarnos, en este caso, para **oracle1**. El nombre del mismo no influye, de manera que en mi caso lo llamaré **ORACLE1**. El protocolo de conexión será **TCP** a la dirección **192.168.1.150**, utilizando el puerto por defecto, **1521**. Por último, tendremos que indicar el nombre del servicio/base de datos al que queremos conectarnos, que por defecto será **ORCLCDB**. El resultado final sería:

{% highlight shell %}
ORCLCDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )

LISTENER_ORCLCDB =
  (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))

ORACLE1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.150)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLCDB)
    )
  )
{% endhighlight %}

Una vez modificado el fichero **tnsnames.ora**, tendremos que cambiarnos una vez más al usuario **oracle** para utilizar el binario **sqlplus**, haciendo para ello uso del comando:

{% highlight shell %}
[root@oracle2 ~]# su - oracle
{% endhighlight %}

Cuando nos encontremos utilizando el usuario **oracle**, todo estará listo para abrir la _shell_ de _sqlplus_ haciendo uso del usuario **c##alvaro2**, ejecutando para ello el comando:

{% highlight shell %}
[oracle@oracle2 ~]$ sqlplus c##alvaro2/alvaro2

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Feb 2 17:07:14 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.
{% endhighlight %}

La creación del enlace será muy sencilla, ya que previamente hemos definido los parámetros de la conexión en el fichero **tnsnames.ora**, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
SQL> CREATE DATABASE LINK oracle1link
CONNECT TO c##alvaro1 IDENTIFIED BY alvaro1
USING 'oracle1';

Enlace con la base de datos creado.
{% endhighlight %}

Efectivamente, el enlace **oracle1link** ha sido correctamente generado, de manera que vamos a proceder a verificar el funcionamiento de dicho enlace, ejecutando para ello una consulta cuya información mostrada provenga de ambas bases de datos, ubicadas como ya hemos visto, en servidores distintos.

La intención es realizar la misma consulta que en el caso anterior, pero adaptándola para consultar al primero de los servidores sobre la información de la tabla **Empleados**, ya que la tabla **Departamentos** se encuentra ahora en la máquina local. La consulta a ejecutar sería:

{% highlight sql %}
SQL> SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos.Nombre AS Departamento
FROM Empleados@oracle1link Empleados, Departamentos
WHERE Empleados.Departamento = Departamentos.Identificador;

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
90389058R Joaquin Marrero Covas 	 C/ Hijuela de Lojo, 22    618385118
28/01/97       1446 Administraci??n

18232747A Aristarco Caban Meraz 	 Puerta Nueva, 67	   691204722
07/05/94       4789 Seguridad

94106513N Marian Fonseca Betancourt	 C/ Manuel Iradier, 37	   638415823
17/07/79       2561 Seguridad


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
12777631G Merlino Rosado Cordero	 C/ Henan Cortes, 58	   609841755
27/11/93       7961 Recursos Humanos

68219319P Tabare Chapa Alcantar 	 C/ Arana, 12		   682227206
09/03/92       8568 Inform??tica

67227129S Ian Esquivel Laboy		 C/ Inglaterra, 64	   728005136
21/07/92       3519 Administraci??n


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
52315160G Heinz Collado Caraballo	 Escuadro, 60		   600173822
13/02/84       1672 Recursos Humanos

85145590G Anabel Lerma Dominguez	 Crta. Cadiz, 1 	   675014823
07/01/81       5919 Administraci??n

56228957Y Dinorah Viera Tello		 Ctra. Villena, 22	   642852778
28/05/87       2567 Seguridad


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO DEPARTAMENTO
-------- ---------- --------------------
61242562W Manases Castillo Camacho	 Ctra. Hornos, 91	   607853354
29/10/91       4270 Inform??tica


10 filas seleccionadas.
{% endhighlight %}

Como era de esperar, la consulta ha vuelto a realizarse sin ningún problema y ha devuelto la información que debería, de manera que podemos corroborar que los servidores tienen conectividad entre sí mediante los enlaces creados, sea cual sea el sentido utilizado.