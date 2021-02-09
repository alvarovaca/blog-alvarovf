---
layout: post
title:  "Interconexión de PostgreSQL y Oracle 19c"
banner: "/assets/images/banners/oraclepostgres.jpg"
date:   2021-02-06 20:03:00 +0200
categories: bbdd
---
El objetivo de este _post_ es el de mostrar el procedimiento a seguir para llevar a cabo una interconexión entre dos servidores **Oracle 19c** y **PostgreSQL** alojados en máquinas **CentOS 8** y **Debian Buster**, respectivamente, siendo su finalidad la de permitir el acceso desde un mismo cliente a dos bases de datos ubicadas en gestores completamente distintos, de una forma simultánea pero indirecta, de manera que uno de los servidores actuará como cliente del otro servidor, de manera unidireccional.

Al fin y al cabo, el cliente únicamente abre una conexión, pues es el primer servidor al que se conecta el que posteriormente abre una conexión al segundo de ellos.

Para esta ocasión, voy a reutilizar dos de las máquinas virtuales creadas en los artículos [Interconexión de Oracle 19c en CentOS 8](https://www.alvarovf.com/bbdd/2021/02/04/interconexion-oracle-centos.html) e [Instalación e interconexión de PostgreSQL en Debian 10](https://www.alvarovf.com/bbdd/2021/02/05/instalacion-interconexion-postgresql-debian.html), por lo que se recomienda la previa lectura de dichos _posts_, pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad. Las máquinas actualmente existentes en el escenario son las siguientes:

* **oracle1**: Máquina **CentOS 8** con **Oracle 19c**, conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.150**.
* **servidor2**: Máquina **Debian Buster** con **PostgreSQL**, conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.161**.

## Oracle 19c a PostgreSQL

Partimos del punto en el que tanto el servidor **Oracle 19c** como el **PostgreSQL** están totalmente operativos y listos para escuchar peticiones remotas. Como en anteriores artículos hemos visto, los servidores Oracle cuentan con soporte nativo para realizar enlaces con otros servidores Oracle, sin embargo, no ocurre lo mismo cuando deseamos conectarnos a otro gestor totalmente diferente, como puede ser PostgreSQL.

En estos casos, podemos acudir a **ODBC** (_Open Database Connectivity_), un estándar de acceso a las bases de datos cuyo objetivo es hacer posible el acceso a cualquier dato desde cualquier aplicación, sin importar qué sistema de gestión de bases de datos almacene los datos. En este caso, vamos a configurar un enlace a una base de datos **PostgreSQL**, sin embargo, es posible realizar una integración con muchos otros gestores como **MySQL** e incluso gestores no relacionales como **Excel**.

![diagrama1](https://i.ibb.co/7WDGktg/dblink-1.png "Diagrama enlace")

El primer paso consistirá en instalar la paquetería necesaria para crear dicho enlace, concretamente el paquete **unixODBC**, que es el proyecto de código abierto que implementa la API **ODBC** de la que previamente hemos hablado.

De otro lado, dado que pretendemos crear un enlace con un servidor **PostgreSQL**, tendremos que instalar el _driver_ específico para ello, de nombre **postgresql-odbc**, no sin antes asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando:

{% highlight shell %}
[root@oracle1 ~]# dnf update && dnf install unixODBC postgresql-odbc
{% endhighlight %}

Cuando la instalación de los paquetes haya finalizado, procederemos a modificar el fichero **/etc/odbcinst.ini**, en el que encontraremos la configuración de todos los _drivers_ de **ODBC** existentes, haciendo para ello uso del comando:

{% highlight shell %}
[root@oracle1 ~]# nano /etc/odbcinst.ini
{% endhighlight %}

De todos los _drivers_ existentes, el único que nos interesa es el referente a **PostgreSQL**, que tendrá la siguiente forma:

{% highlight shell %}
[PostgreSQL]
Description     = ODBC for PostgreSQL
Driver          = /usr/lib/psqlodbcw.so
Setup           = /usr/lib/libodbcpsqlS.so
Driver64        = /usr/lib64/psqlodbcw.so
Setup64         = /usr/lib64/libodbcpsqlS.so
FileUsage	= 1
{% endhighlight %}

Como se puede apreciar, la definición del mismo se encuentra compuesta por una breve descripción y las rutas a las diferentes librerías que lo constituyen. En mi caso, he comentado la definición del resto de _drivers_, ya que para este caso no me van a ser necesarios.

El siguiente paso consistirá en crear un **DSN** (_Data Source Name_) en el fichero **/etc/odbc.ini**, pues dicho fichero será posteriormente utilizado para determinar la manera de conectarse al gestor especificado. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@oracle1 ~]# nano /etc/odbc.ini
{% endhighlight %}

Dentro del mismo, introduciremos el siguiente contenido:

{% highlight shell %}
[PSQLU]
Debug           = 0
CommLog         = 0
ReadOnly        = 0
Driver          = PostgreSQL
Servername      = 192.168.1.161
Username        = alvaro2
Password        = alvaro2
Port            = 5432
Database        = prueba2
Trace           = 0
TraceFile       = /tmp/sql.log
{% endhighlight %}

Donde:

* **Driver**: Indicamos el nombre del _driver_ previamente configurado en el fichero **/etc/odbcinst.ini**. En este caso, **PostgreSQL**.
* **Servername**: Indicamos la dirección IP del servidor PostgreSQL al que deseamos crear el enlace. En este caso, **192.168.1.161**.
* **Username**: Indicamos el nombre del usuario para el acceso a la base de datos remota. En este caso, **alvaro2**.
* **Password**: Indicamos la contraseña del usuario para el acceso a la base de datos remota. En este caso, **alvaro2**.
* **Port**: Indicamos el puerto en el que está escuchando peticiones el servidor PostgreSQL. En este caso, **5432**.
* **Database**: Indicamos el nombre de la base de datos remota a la que pretendemos conectarnos. En este caso, **prueba2**.

La configuración del _driver_ de ODBC ha finalizado, de manera que es aconsejable hacer una pequeña prueba llegado este punto para verificar su correcto funcionamiento, haciendo para ello uso del binario `isql` incluido en el paquete previamente instalado, seguido del **DSN** en cuestión.

Gracias al mismo, podremos realizar una conexión ODBC para así comprobar que es posible acceder a la base de datos PostgreSQL remota haciendo uso de los parámetros que hemos especificado, ejecutando para ello el comando:

{% highlight shell %}
[root@oracle1 ~]# isql PSQLU
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, la conexión ha sido exitosa y actualmente nos encontramos haciendo uso de un cliente en el que podríamos ejecutar órdenes SQL, como por ejemplo listar el contenido de la tabla **Departamentos**, haciendo para ello uso de la instrucción:

{% highlight shell %}
SQL> SELECT * FROM Departamentos;
+--------------+---------------------+----------------+
| identificador| nombre              | localizacion   |
+--------------+---------------------+----------------+
| 10           | Administración     | Sevilla        |
| 20           | Recursos Humanos    | Barcelona      |
| 30           | Seguridad           | Madrid         |
| 40           | Informática        | Valencia       |
+--------------+---------------------+----------------+
SQLRowCount returns 4
4 rows fetched
{% endhighlight %}

Efectivamente, la información devuelta concuerda con la almacenada en dicha tabla remota, de manera que podremos salir del cliente ejecutando la instrucción `exit` para así continuar con la configuración del enlace.

Como anteriormente hemos mencionado, la configuración del _driver_ ha finalizado, sin embargo, Oracle no está todavía configurado para poder utilizar dicho _driver_, por lo que el siguiente paso consistirá en generar un fichero en el que se especifiquen determinados parámetros necesarios para ello (**_Heterogeneous Services_**).

La ubicación de dicho fichero será **$ORACLE_HOME/hs/admin/** y su nombre, **init[DSN].ora**, de manera que en este caso, al ser **PSQLU** el **DSN**, el nombre del mismo sería **initPSQLU.ora**. Para generar y editar dicho fichero, haremos uso del comando:

{% highlight shell %}
[root@oracle1 ~]# nano /opt/oracle/product/19c/dbhome_1/hs/admin/initPSQLU.ora
{% endhighlight %}

Dentro del mismo, introduciremos el siguiente contenido:

{% highlight shell %}
HS_FDS_CONNECT_INFO = PSQLU
HS_FDS_TRACE_LEVEL = DEBUG
HS_FDS_SHAREABLE_NAME = /usr/lib64/psqlodbcw.so
HS_LANGUAGE = AMERICAN_AMERICA.WE8ISO8859P1
set ODBCINI=/etc/odbc.ini
{% endhighlight %}

Como se puede apreciar, dentro del mismo hemos definido determinados parámetros como el nombre del DSN, el _driver_ que debe utilizar que habrá sido previamente especificado en el fichero **/etc/odbcinst.ini**, el fichero en el que se encuentran definidos los DSN...

Tras ello, todo estará listo para comenzar a configurar el **_listener_**, que para quién no conozca lo que es, es el proceso encargado de escuchar las peticiones entrantes en el lado del servidor, cuyo fichero en el que se define su configuración se encuentra almacenado en **$ORACLE_HOME/network/admin/**, con el nombre **listener.ora**, estableciéndose en el mismo configuración entre la que se encuentran las bases de datos para las que va a escuchar peticiones y los puertos en los que va a hacerlo, por lo que procederemos a modificar dicho fichero ejecutando para ello el comando:

{% highlight shell %}
[root@oracle1 ~]# nano /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
{% endhighlight %}

En este caso, tendremos que añadir una nueva entrada que habilitará la escucha hacia el driver ODBC, especificando para ello el DSN, la ruta del directorio principal de Oracle y el programa en cuestión, quedando en mi caso de la siguiente forma:

{% highlight shell %}
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = oracle1.alvarovf.com)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER=
  (SID_LIST=
      (SID_DESC=
         (SID_NAME=PSQLU)
         (ORACLE_HOME=/opt/oracle/product/19c/dbhome_1)
         (PROGRAM=dg4odbc)
      )
  )
{% endhighlight %}

El siguiente paso será modificar el fichero de nombre **tnsnames.ora**, ubicado en **$ORACLE_HOME/network/admin/**, que permite facilitar la tarea de acceso a servidores remotos, indicando en el mismo una entrada por cada uno de los servidores a los que se pretende acceder, que en este caso nos servirá para "mapear" la conexión hacia el driver ODBC. Su configuración no es para nada complicada, así que vamos a llevarla a cabo ejecutando para ello el comando:

{% highlight shell %}
[root@oracle1 ~]# nano /opt/oracle/product/19c/dbhome_1/network/admin/tnsnames.ora
{% endhighlight %}

En dicha entrada tendremos que incluir el protocolo, la dirección IP y el puerto (que como es lógico corresponden a la propia máquina local), así como un alias que será el que se utilice a la hora de la conexión, que en este caso coincidirá con el DSN asignado, quedando en mi caso de la siguiente forma:

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

PSQLU  =
  (DESCRIPTION=
    (ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521))
    (CONNECT_DATA=(SID=PSQLU))
    (HS=OK)
  )
{% endhighlight %}

Tras ello, simplemente guardaremos los cambios realizados y todo estará listo para reiniciar el proceso del _listener_, con la intención de que cargue en memoria la nueva configuración asignada, cambiándonos previamente al usuario **oracle**, pues es el único que tiene actualmente definida en sus variables de entorno la ruta a los binarios de Oracle, ejecutando por tanto el comando:

{% highlight shell %}
[root@oracle1 ~]# su - oracle
{% endhighlight %}

Cuando nos encontremos haciendo uso del usuario **oracle**, podremos llevar a cabo dicho reinicio, haciendo para ello uso de los comandos:

{% highlight shell %}
[oracle@oracle1 ~]$ lsnrctl stop
[oracle@oracle1 ~]$ lsnrctl start
{% endhighlight %}

Una vez que el _listener_ haya sido correctamente reiniciado, es momento de abrir una _shell_ de _sqlplus_ haciendo uso del usuario **c##alvaro1** actualmente existente en el gestor Oracle, ejecutando para ello el comando:

{% highlight shell %}
[oracle@oracle1 ~]$ sqlplus c##alvaro1/alvaro1

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Feb 4 12:05:36 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.
{% endhighlight %}

Por último, tendremos que llevar a cabo la creación del enlace, que será muy sencilla, ya que previamente hemos definido los parámetros de la conexión en el fichero **tnsnames.ora**, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
SQL> CREATE DATABASE LINK postgreslink
CONNECT TO "alvaro2" IDENTIFIED BY "alvaro2"
USING 'PSQLU';

Enlace con la base de datos creado.
{% endhighlight %}

Donde:

* **CREATE DATABASE LINK**: Especificamos un nombre identificativo para el enlace.
* **CONNECT TO**: Indicamos las credenciales de acceso a la base de datos remota.
* **USING**: Indicamos el nombre del alias de la conexión que previamente hemos definido en el fichero **tnsnames.ora**.

**Nota**: Es importante utilizar comillas dobles (**" "**) para el nombre de usuario y la contraseña, pero comillas simples (**' '**) para el nombre del alias.

Efectivamente, el enlace **postgreslink** ha sido correctamente generado, de manera que vamos a proceder a verificar el funcionamiento de dicho enlace, ejecutando para ello una consulta cuya información mostrada provenga de ambas bases de datos, ubicadas como ya hemos visto, en servidores distintos.

La intención es mostrar los datos de los **Empleados** (tabla que se encuentra en **oracle1**) junto al nombre del departamento al que pertenecen, información que podremos obtener de **Departamentos** (tabla que se encuentra en **servidor2**), utilizando para ello la clave que tienen en común ambas tablas. La consulta a ejecutar sería:

{% highlight sql %}
SQL> SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos."nombre" AS Departamento
FROM Empleados, "departamentos"@postgreslink Departamentos
WHERE Empleados.Departamento = Departamentos."identificador";

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------
90389058R Joaquin Marrero Covas 	 C/ Hijuela de Lojo, 22    618385118
28/01/97       1446
Administraci??n

67227129S Ian Esquivel Laboy		 C/ Inglaterra, 64	   728005136
21/07/92       3519
Administraci??n

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------

85145590G Anabel Lerma Dominguez	 Crta. Cadiz, 1 	   675014823
07/01/81       5919
Administraci??n

12777631G Merlino Rosado Cordero	 C/ Henan Cortes, 58	   609841755
27/11/93       7961

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------
Recursos Humanos

52315160G Heinz Collado Caraballo	 Escuadro, 60		   600173822
13/02/84       1672
Recursos Humanos

18232747A Aristarco Caban Meraz 	 Puerta Nueva, 67	   691204722

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------
07/05/94       4789
Seguridad

94106513N Marian Fonseca Betancourt	 C/ Manuel Iradier, 37	   638415823
17/07/79       2561
Seguridad


DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------
56228957Y Dinorah Viera Tello		 Ctra. Villena, 22	   642852778
28/05/87       2567
Seguridad

68219319P Tabare Chapa Alcantar 	 C/ Arana, 12		   682227206
09/03/92       8568
Inform?!tica

DNI	  NOMBRE			 DIRECCION		   TELEFONO
--------- ------------------------------ ------------------------- ---------
FECHANAC    SALARIO
-------- ----------
DEPARTAMENTO
--------------------------------------------------------------------------------

61242562W Manases Castillo Camacho	 Ctra. Hornos, 91	   607853354
29/10/91       4270
Inform?!tica


10 filas seleccionadas.
{% endhighlight %}

Donde:

* **SELECT**: Indicamos las columnas que queremos mostrar de la información obtenida, de la forma **[tabla].[columna]**.
* **FROM**: Hacemos un JOIN de la tabla Empleados que se encuentra en la base de datos local junto a la consulta remota, haciendo uso del enlace que acabamos de generar, para así poder mostrar información de ambas tablas en la misma consulta.
* **WHERE**: Establecemos la condición del JOIN, que deberá ser aquella columna mediante la cual vamos a unir los registros devueltos. Como es lógico, será el código del departamento, pues es la columna que se repite en ambas tablas.

**Nota**: Es importante utilizar comillas dobles (**" "**) para el nombre de las tablas y las columnas siempre que se haga referencia al gestor PostgreSQL, indicando el nombre de las mismas en minúsculas.

Al parecer, la consulta se ha realizado sin ningún problema y ha devuelto la información que debería, pues tal y como he mencionado con anterioridad, ambas máquinas servidoras cuentan con un direccionamiento dentro de la red local, siendo ambas totalmente alcanzables entre sí, además de estar correctamente configuradas para aceptar dichas conexiones.

Otra utilidad que estos enlaces nos aportan es la capacidad de copiar las tablas de un gestor a otro, utilizando el resultado de una consulta simple para crear una tabla a partir de la misma. Por ejemplo, podríamos copiar la tabla **Departamentos** haciendo uso de la siguiente instrucción:

{% highlight sql %}
SQL> CREATE TABLE Departamentos
AS (SELECT *
    FROM "departamentos"@postgreslink);

Tabla creada.
{% endhighlight %}

En dicha instrucción, hemos realizado una consulta a la tabla **Departamentos** ubicada en la base de datos del servidor **servidor2**, utilizando la respuesta obtenida para crear una nueva tabla con el mismo nombre, que se almacenará ahora de forma local en el servidor **oracle1**, y que podremos empezar a utilizar sin necesidad de recurrir al enlace con el segundo servidor. Si consultamos la nueva tabla generada, obtendremos el siguiente resultado:

{% highlight sql %}
SQL> SELECT *
  2  FROM Departamentos;

identificador
-------------
nombre
--------------------------------------------------------------------------------
localizacion
--------------------------------------------------------------------------------
	   10
Administraci??n
Sevilla

	   20
Recursos Humanos
Barcelona

identificador
-------------
nombre
--------------------------------------------------------------------------------
localizacion
--------------------------------------------------------------------------------

	   30
Seguridad
Madrid

	   40
Inform?!tica

identificador
-------------
nombre
--------------------------------------------------------------------------------
localizacion
--------------------------------------------------------------------------------
Valencia
{% endhighlight %}

Como se puede apreciar, el contenido es exactamente el mismo que el existente en la tabla ubicada en el gestor remoto, por lo que podemos concluir que su clonación ha sido efectiva.

El enlace ha funcionado del extremo **oracle1** al extremo **servidor2**, pero como ya sabemos, dichos enlaces son unidireccionales, de manera que si quisiésemos realizar la conexión a la inversa, de **PostgreSQL** a **Oracle**, tendríamos que llevar a cabo un proceso un tanto distinto, así que vamos a proceder a ello.

## PostgreSQL a Oracle 19c

Una vez más, partimos del punto en el que tanto el servidor **PostgreSQL** como el **Oracle 19c** están totalmente operativos y listos para escuchar peticiones remotas. Como en anteriores artículos hemos visto, los servidores PostgreSQL cuentan con soporte nativo para realizar enlaces con otros servidores PostgreSQL, sin embargo, no ocurre lo mismo cuando deseamos conectarnos a otro gestor totalmente diferente, como puede ser Oracle.

En estos casos, podemos acudir a **oracle_fdw** (_Foreign Data Wrapper for Oracle_), un concepto que introdujo PostgreSQL hace varias versiones cuyo objetivo es hacer posible el acceso de una forma muy sencilla y eficiente a las bases de datos Oracle, tal y como mostraremos a continuación. Existen otros _Data Wrappers_, que posibilitan una integración con muchos otros gestores como **MySQL** e incluso gestores no relacionales como **MongoDB**.

![diagrama2](https://i.ibb.co/YhtZR1G/0-0nv5okg-JCc-PPXUEk.png "Diagrama enlace")

El paquete **oracle_fdw** no se encuentra actualmente disponible en los repositorios oficiales de CentOS 8, sino que tendremos que compilarlo a mano. Por ese mismo motivo, necesitaremos instalar una serie de paquetes que nos proporcionarán las herramientas necesarias para llevar a cabo dicha compilación, así como los paquetes oficiales de **_Oracle Instant Client_**.

Comenzaremos por la instalación de aquellos paquetes esenciales para la compilación, no sin antes actualizar la lista de la paquetería disponible, por lo que haremos uso del comando:

{% highlight shell %}
root@servidor2:/home/postgres# apt update && apt install libaio1 postgresql-server-dev-all build-essential git
{% endhighlight %}

Donde:

* **libaio1**: Biblioteca activa en el espacio de usuario importante para el rendimiento de las bases de datos y otras aplicaciones avanzadas.
* **postgresql-server-dev-all**: Proporciona las herramientas necesarias para la compilación de extensiones para PostgreSQL.
* **build-essential**: Como su propio nombre indica, contiene las utilidades esenciales para compilar un paquete en Debian (compiladores y bibliotecas).
* **git**: Nos permitirá clonar el repositorio GitHub en el que está contenido el código fuente de la aplicación.

Una vez instalados los paquetes esenciales, tendremos que descargar también los paquetes oficiales de **_Oracle Instant Client_**, que nos permitirán hacer uso del cliente Oracle para llevar a cabo las conexiones a las bases de datos remotas. Antes de ello, vamos a cambiarnos al usuario **postgres**, pues será aquel que utilizará principalmente dichos binarios, ejecutando para ello el comando:

{% highlight shell %}
root@servidor2:/home/postgres# su postgres
{% endhighlight %}

Los paquetes que debemos descargar del [sitio](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html) oficial de Oracle son los siguientes:

* [instantclient-basic-linux.x64-21.1.0.0.0.zip](https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-basic-linux.x64-21.1.0.0.0.zip)
* [instantclient-sdk-linux.x64-21.1.0.0.0.zip](https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sdk-linux.x64-21.1.0.0.0.zip)
* [instantclient-sqlplus-linux.x64-21.1.0.0.0.zip](https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sqlplus-linux.x64-21.1.0.0.0.zip)

Para llevar a cabo la descarga de dichos paquetes, haremos uso de los comandos:

{% highlight shell %}
postgres@servidor2:/home/postgres$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-basic-linux.x64-21.1.0.0.0.zip
postgres@servidor2:/home/postgres$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sdk-linux.x64-21.1.0.0.0.zip
postgres@servidor2:/home/postgres$ wget https://download.oracle.com/otn_software/linux/instantclient/211000/instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
{% endhighlight %}

Tras ello, vamos a listar el contenido del directorio actual para verificar que las descargas se han completado satisfactoriamente:

{% highlight shell %}
postgres@servidor2:/home/postgres$ ls -l
total 79288
-rw-r--r-- 1 postgres postgres 79250994 Dec  1 12:07 instantclient-basic-linux.x64-21.1.0.0.0.zip
-rw-r--r-- 1 postgres postgres   998327 Dec  1 12:07 instantclient-sdk-linux.x64-21.1.0.0.0.zip
-rw-r--r-- 1 postgres postgres   936169 Dec  1 12:07 instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
{% endhighlight %}

Efectivamente, se han descargado los siguientes ficheros:

* **instantclient-basic-linux.x64-21.1.0.0.0.zip** de peso total **75.58M** (79250994 bytes).
* **instantclient-sdk-linux.x64-21.1.0.0.0.zip** de peso total **974.93K** (998327 bytes).
* **instantclient-sqlplus-linux.x64-21.1.0.0.0.zip** de peso total **914.23K** (936169 bytes).

Si nos fijamos, los paquetes están comprimidos, por lo que no podremos utilizar los binarios incluidos hasta que no hagamos una extracción de los mismos. Además, para liberar espacio, borraremos tras ello los ficheros comprimidos ya que no nos harán falta. Para descomprimir un paquete **.zip** haremos uso de `unzip`, ejecutando los comandos:

{% highlight shell %}
postgres@servidor2:/home/postgres$ unzip instantclient-basic-linux.x64-21.1.0.0.0.zip
postgres@servidor2:/home/postgres$ unzip instantclient-sqlplus-linux.x64-21.1.0.0.0.zip
postgres@servidor2:/home/postgres$ unzip instantclient-sdk-linux.x64-21.1.0.0.0.zip
postgres@servidor2:/home/postgres$ rm *.zip
{% endhighlight %}

Para verificar que los ficheros se han descomprimido y eliminado correctamente, haremos uso del comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ ls -l
total 4
drwxr-xr-x 4 postgres postgres 4096 Feb  4 11:28 instantclient_21_1
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos en un directorio de nombre **instantclient_21_1**, que contendrá todos los binarios que nos ofrece un cliente Oracle, entre los que se encuentra nuestro preciado `sqlplus`.

Para verificarlo, podemos listar los ejecutables de dicho directorio, estableciendo un filtro por nombre para no mostrar las librerías compartidas, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ find instantclient_21_1/ -executable -type f | egrep -v '.so.*$'
instantclient_21_1/sqlplus
instantclient_21_1/sdk/ott
instantclient_21_1/uidrvci
instantclient_21_1/adrci
instantclient_21_1/genezi
{% endhighlight %}

Sin embargo, si ahora mismo quisiésemos hacer uso de cualquiera de esos binarios, tendríamos que indicar la ruta completa para que el sistema sea capaz de encontrarlo. Esto tiene una razón, y es que en Linux existe una variable de entorno de nombre **PATH** ($PATH) que indica las rutas en las que buscar los binarios (y en qué orden), en la que no se encuentra actualmente especificada la ruta **/home/postgres/instantclient_21_1**.

Para solucionar esto entre otras cosas, vamos a definir determinadas variables de entorno necesarias para el correcto funcionamiento de dichos binarios, entre las que se encuentra el **$ORACLE_HOME**, que será aquella ruta en la que se encuentran ubicados los binarios en cuestión. Además, anexaremos el valor de dicha variable a **$LD_LIBRARY_PATH** y **$PATH**, haciendo para ello uso de los comandos:

{% highlight shell %}
postgres@servidor2:~$ export ORACLE_HOME=/home/postgres/instantclient_21_1
postgres@servidor2:~$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
postgres@servidor2:~$ export PATH=$PATH:$ORACLE_HOME
{% endhighlight %}

Al haber añadido la nueva ruta a la variable **$PATH**, ya será posible encontrar dicho binario sin indicar su ruta completa, así que vamos a comprobarlo haciendo uso de `which`, que nos devolverá la ruta en la que se ubica un binario en cuestión, por ejemplo `sqlplus`, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ which sqlplus
/home/postgres/instantclient_21_1/sqlplus
{% endhighlight %}

Como era de esperar, gracias a haber añadido la nueva ruta a dicha variable de entorno, el binario es ahora totalmente accesible.

Para verificar el correcto funcionamiento del mismo, vamos a llevar a cabo una conexión remota real a la base de datos, como si de una situación normal y corriente se tratase, haciendo para ello uso del comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ sqlplus c##alvaro1/alvaro1@192.168.1.150/ORCLCDB

SQL*Plus: Release 21.0.0.0.0 - Production on Thu Feb 4 11:34:53 2021
Version 21.1.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.

Hora de Ultima Conexion Correcta: Jue Feb 04 2021 11:12:43 -05:00

Conectado a:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
{% endhighlight %}

Tal y como se puede apreciar en la salida del comando, nos hemos logrado conectar correctamente a la base de datos ubicada en la primera máquina servidora, por lo que vamos a proceder a listar todas las tablas que el usuario **c##alvaro1** tiene privilegios para ver, es decir, aquellas que hemos creado manualmente. Para ello, ejecutaremos la instrucción:

{% highlight sql %}
SQL> SELECT * FROM cat;

TABLE_NAME
--------------------------------------------------------------------------------
TABLE_TYPE
-----------
EMPLEADOS
TABLE
{% endhighlight %}

Efectivamente, la tabla que anteriormente hemos creado de forma manual es visible y accesible de forma remota desde una máquina PostgreSQL que actualmente actúa como cliente, de manera que podremos salir de dicho cliente ejecutando la instrucción `exit` para así continuar con la configuración del enlace.

Tras comprobar que los binarios del cliente Oracle funcionan correctamente, es hora de llevar a cabo la compilación de **oracle_fdw**, de manera que tendremos que clonar el [repositorio](https://github.com/laurenz/oracle_fdw) GitHub en el que está contenido el código fuente necesario para ello, haciendo por tanto uso del comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ git clone https://github.com/laurenz/oracle_fdw.git
Cloning into 'oracle_fdw'...
remote: Enumerating objects: 77, done.
remote: Counting objects: 100% (77/77), done.
remote: Compressing objects: 100% (54/54), done.
remote: Total 2223 (delta 39), reused 56 (delta 23), pack-reused 2146
Receiving objects: 100% (2223/2223), 1.35 MiB | 2.58 MiB/s, done.
Resolving deltas: 100% (1513/1513), done.
{% endhighlight %}

Una vez finalizada la clonación, vamos a listar una vez más el contenido del directorio actual para así verificar que dicha clonación ha sido efectiva:

{% highlight shell %}
postgres@servidor2:/home/postgres$ ls -l
total 8
drwxr-xr-x 4 postgres postgres 4096 Feb  4 11:28 instantclient_21_1
drwxr-xr-x 6 postgres postgres 4096 Feb  4 11:39 oracle_fdw
{% endhighlight %}

Efectivamente, el repositorio de nombre **oracle_fdw** ha sido correctamente clonado en nuestra máquina local y podremos empezar a hacer uso del mismo, de manera que nos moveremos dentro dicho directorio para visualizar su contenido, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor2:/home/postgres$ cd oracle_fdw/
{% endhighlight %}

Tras ello, listaremos el contenido existente una vez más para verificar que el repositorio se ha clonado correctamente:

{% highlight shell %}
postgres@servidor2:/home/postgres/oracle_fdw$ ls -l
total 460
-rw-r--r-- 1 postgres postgres  20254 Feb  4 11:39 CHANGELOG
drwxr-xr-x 2 postgres postgres   4096 Feb  4 11:39 expected
-rw-r--r-- 1 postgres postgres   1086 Feb  4 11:39 LICENSE
-rw-r--r-- 1 postgres postgres   2932 Feb  4 11:39 Makefile
drwxr-xr-x 2 postgres postgres   4096 Feb  4 11:39 msvc
-rw-r--r-- 1 postgres postgres    231 Feb  4 11:39 oracle_fdw--1.0--1.1.sql
-rw-r--r-- 1 postgres postgres    240 Feb  4 11:39 oracle_fdw--1.1--1.2.sql
-rw-r--r-- 1 postgres postgres   1244 Feb  4 11:39 oracle_fdw--1.2.sql
-rw-r--r-- 1 postgres postgres 207606 Feb  4 11:39 oracle_fdw.c
-rw-r--r-- 1 postgres postgres    133 Feb  4 11:39 oracle_fdw.control
-rw-r--r-- 1 postgres postgres   8667 Feb  4 11:39 oracle_fdw.h
-rw-r--r-- 1 postgres postgres  44511 Feb  4 11:39 oracle_gis.c
-rw-r--r-- 1 postgres postgres  98942 Feb  4 11:39 oracle_utils.c
lrwxrwxrwx 1 postgres postgres     17 Feb  4 11:39 README.md -> README.oracle_fdw
-rw-r--r-- 1 postgres postgres  39227 Feb  4 11:39 README.oracle_fdw
drwxr-xr-x 2 postgres postgres   4096 Feb  4 11:39 sql
-rw-r--r-- 1 postgres postgres    313 Feb  4 11:39 TODO
{% endhighlight %}

Efectivamente, todos los componentes necesarios para la compilación se encuentran ubicados dentro del directorio en cuestión.

Explicar con detenimiento cómo funciona una compilación sería bastante extenso, por lo que [aquí](https://www.alvarovf.com/sistemas/2020/10/31/compilacion-usando-makefile.html) se puede encontrar otro artículo de mi blog en el que se explica y muestra un ejemplo práctico, aunque no es un requisito conocer cómo funciona para hacerlo.

En resumen, cuando queremos compilar programas "grandes", en los que hay gran cantidad de ficheros fuente (**.c**), _headers_ (**.h**) y librerías, sería totalmente inviable compilarlos todos a mano para posteriormente _linkar_ todos los **.o** en un único ejecutable. Es por ello que recurrimos al comando **make**, que nos permite hacerlo de una manera mucho más eficiente e inteligente gracias a un fichero de nombre **Makefile**, en el que se define mediante instrucciones, el orden en el que se compilarán los ficheros **.c** y _linkarán_ los ficheros **.o**.

Por tanto, para llevar a cabo dicha compilación e instalar el resultado en los correspondientes directorios de la máquina, ejecutaremos los comandos:

{% highlight shell %}
postgres@servidor2:/home/postgres/oracle_fdw$ make
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I/home/postgres/instantclient_21_1/sdk/include -I/home/postgres/instantclient_21_1/oci/include -I/home/postgres/instantclient_21_1/rdbms/public -I/home/postgres/instantclient_21_1 -I/usr/include/oracle/19.9/client -I/usr/include/oracle/19.9/client64 -I/usr/include/oracle/19.8/client -I/usr/include/oracle/19.8/client64 -I/usr/include/oracle/19.6/client -I/usr/include/oracle/19.6/client64 -I/usr/include/oracle/19.3/client -I/usr/include/oracle/19.3/client64 -I/usr/include/oracle/18.5/client -I/usr/include/oracle/18.5/client64 -I/usr/include/oracle/18.3/client -I/usr/include/oracle/18.3/client64 -I/usr/include/oracle/12.2/client -I/usr/include/oracle/12.2/client64 -I/usr/include/oracle/12.1/client -I/usr/include/oracle/12.1/client64 -I/usr/include/oracle/11.2/client -I/usr/include/oracle/11.2/client64 -I/usr/include/oracle/11.1/client -I/usr/include/oracle/11.1/client64 -I/usr/include/oracle/10.2.0.5/client -I/usr/include/oracle/10.2.0.5/client64 -I/usr/include/oracle/10.2.0.4/client -I/usr/include/oracle/10.2.0.4/client64 -I/usr/include/oracle/10.2.0.3/client -I/usr/include/oracle/10.2.0.3/client64 -I. -I./ -I/usr/include/postgresql/11/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2  -I/usr/include/mit-krb5  -c -o oracle_fdw.o oracle_fdw.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I/home/postgres/instantclient_21_1/sdk/include -I/home/postgres/instantclient_21_1/oci/include -I/home/postgres/instantclient_21_1/rdbms/public -I/home/postgres/instantclient_21_1 -I/usr/include/oracle/19.9/client -I/usr/include/oracle/19.9/client64 -I/usr/include/oracle/19.8/client -I/usr/include/oracle/19.8/client64 -I/usr/include/oracle/19.6/client -I/usr/include/oracle/19.6/client64 -I/usr/include/oracle/19.3/client -I/usr/include/oracle/19.3/client64 -I/usr/include/oracle/18.5/client -I/usr/include/oracle/18.5/client64 -I/usr/include/oracle/18.3/client -I/usr/include/oracle/18.3/client64 -I/usr/include/oracle/12.2/client -I/usr/include/oracle/12.2/client64 -I/usr/include/oracle/12.1/client -I/usr/include/oracle/12.1/client64 -I/usr/include/oracle/11.2/client -I/usr/include/oracle/11.2/client64 -I/usr/include/oracle/11.1/client -I/usr/include/oracle/11.1/client64 -I/usr/include/oracle/10.2.0.5/client -I/usr/include/oracle/10.2.0.5/client64 -I/usr/include/oracle/10.2.0.4/client -I/usr/include/oracle/10.2.0.4/client64 -I/usr/include/oracle/10.2.0.3/client -I/usr/include/oracle/10.2.0.3/client64 -I. -I./ -I/usr/include/postgresql/11/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2  -I/usr/include/mit-krb5  -c -o oracle_utils.o oracle_utils.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -I/home/postgres/instantclient_21_1/sdk/include -I/home/postgres/instantclient_21_1/oci/include -I/home/postgres/instantclient_21_1/rdbms/public -I/home/postgres/instantclient_21_1 -I/usr/include/oracle/19.9/client -I/usr/include/oracle/19.9/client64 -I/usr/include/oracle/19.8/client -I/usr/include/oracle/19.8/client64 -I/usr/include/oracle/19.6/client -I/usr/include/oracle/19.6/client64 -I/usr/include/oracle/19.3/client -I/usr/include/oracle/19.3/client64 -I/usr/include/oracle/18.5/client -I/usr/include/oracle/18.5/client64 -I/usr/include/oracle/18.3/client -I/usr/include/oracle/18.3/client64 -I/usr/include/oracle/12.2/client -I/usr/include/oracle/12.2/client64 -I/usr/include/oracle/12.1/client -I/usr/include/oracle/12.1/client64 -I/usr/include/oracle/11.2/client -I/usr/include/oracle/11.2/client64 -I/usr/include/oracle/11.1/client -I/usr/include/oracle/11.1/client64 -I/usr/include/oracle/10.2.0.5/client -I/usr/include/oracle/10.2.0.5/client64 -I/usr/include/oracle/10.2.0.4/client -I/usr/include/oracle/10.2.0.4/client64 -I/usr/include/oracle/10.2.0.3/client -I/usr/include/oracle/10.2.0.3/client64 -I. -I./ -I/usr/include/postgresql/11/server -I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2  -I/usr/include/mit-krb5  -c -o oracle_gis.o oracle_gis.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer -fPIC -shared -o oracle_fdw.so oracle_fdw.o oracle_utils.o oracle_gis.o -L/usr/lib/x86_64-linux-gnu  -Wl,-z,relro -Wl,-z,now -L/usr/lib/llvm-7/lib  -L/usr/lib/x86_64-linux-gnu/mit-krb5 -Wl,--as-needed  -L/home/postgres/instantclient_21_1 -L/home/postgres/instantclient_21_1/bin -L/home/postgres/instantclient_21_1/lib -L/home/postgres/instantclient_21_1/lib/amd64 -lclntsh -L/usr/lib/oracle/19.9/client/lib -L/usr/lib/oracle/19.9/client64/lib -L/usr/lib/oracle/19.8/client/lib -L/usr/lib/oracle/19.8/client64/lib -L/usr/lib/oracle/19.6/client/lib -L/usr/lib/oracle/19.6/client64/lib -L/usr/lib/oracle/19.3/client/lib -L/usr/lib/oracle/19.3/client64/lib -L/usr/lib/oracle/18.5/client/lib -L/usr/lib/oracle/18.5/client64/lib -L/usr/lib/oracle/18.3/client/lib -L/usr/lib/oracle/18.3/client64/lib -L/usr/lib/oracle/12.2/client/lib -L/usr/lib/oracle/12.2/client64/lib -L/usr/lib/oracle/12.1/client/lib -L/usr/lib/oracle/12.1/client64/lib -L/usr/lib/oracle/11.2/client/lib -L/usr/lib/oracle/11.2/client64/lib -L/usr/lib/oracle/11.1/client/lib -L/usr/lib/oracle/11.1/client64/lib -L/usr/lib/oracle/10.2.0.5/client/lib -L/usr/lib/oracle/10.2.0.5/client64/lib -L/usr/lib/oracle/10.2.0.4/client/lib -L/usr/lib/oracle/10.2.0.4/client64/lib -L/usr/lib/oracle/10.2.0.3/client/lib -L/usr/lib/oracle/10.2.0.3/client64/lib

postgres@servidor2:/home/postgres/oracle_fdw$ sudo make install
/bin/mkdir -p '/usr/lib/postgresql/11/lib'
/bin/mkdir -p '/usr/share/postgresql/11/extension'
/bin/mkdir -p '/usr/share/postgresql/11/extension'
/bin/mkdir -p '/usr/share/doc/postgresql-doc-11/extension'
/usr/bin/install -c -m 755  oracle_fdw.so '/usr/lib/postgresql/11/lib/oracle_fdw.so'
/usr/bin/install -c -m 644 .//oracle_fdw.control '/usr/share/postgresql/11/extension/'
/usr/bin/install -c -m 644 .//oracle_fdw--1.2.sql .//oracle_fdw--1.0--1.1.sql .//oracle_fdw--1.1--1.2.sql  '/usr/share/postgresql/11/extension/'
/usr/bin/install -c -m 644 .//README.oracle_fdw '/usr/share/doc/postgresql-doc-11/extension/'
{% endhighlight %}

Una vez que el resultado de la compilación ha sido correctamente instalado en los directorios locales de PostgreSQL, ya podremos hacer uso de dicha extensión. Sin embargo, en mi caso, a pesar de tener correctamente definidas las variables de entorno, el gestor no era capaz de encontrar las librerías compartidas de Oracle, de manera que ha sido necesario indicar de forma explícita la ruta a las mismas en un fichero de nombre **oracle.conf** dentro de **/etc/ld.so.conf.d/**, haciendo para ello uso del comando:

{% highlight shell %}
postgres@servidor2:/home/postgres/oracle_fdw$ echo '/home/postgres/instantclient_21_1' | sudo tee /etc/ld.so.conf.d/oracle.conf
{% endhighlight %}

Por último, tendremos que generar los enlaces necesarios y cargar en memoria las nuevas librerías compartidas que hemos introducido, de manera que ejecutaremos el comando:

{% highlight shell %}
postgres@servidor2:/home/postgres/oracle_fdw$ sudo ldconfig
{% endhighlight %}

Toda la configuración necesaria ha finalizado, pues únicamente faltaría generar el correspondiente enlace y empezar a hacer uso del mismo. Para ello, abriremos una _shell_ de _psql_ haciendo uso de la base de datos **prueba2** desde el usuario actual, pues es el que cuenta con los privilegios necesarios para crear dicho enlace, haciendo para ello uso del comando:

{% highlight shell %}
postgres@servidor2:/home/postgres/oracle_fdw$ psql -d prueba2
psql (11.9 (Debian 11.9-0+deb10u1))
Type "help" for help.
{% endhighlight %}

Es muy importante que la conexión se realice a la base de datos **prueba2**, ya que es donde el rol **alvaro2** tiene privilegios, pues de lo contrario, no podría utilizar dicho enlace. La creación del mismo es muy sencilla, llevándose a cabo mediante la ejecución del comando:

{% highlight sql %}
prueba2=# CREATE EXTENSION oracle_fdw;
CREATE EXTENSION
{% endhighlight %}

La extensión que hará la función de enlace ya ha sido creada, pero para verificarlo, vamos a listar todas las extensiones existentes, haciendo para ello uso de la instrucción:

{% highlight sql %}
prueba2=# \dx
                                   List of installed extensions
    Name    | Version |   Schema   |                         Description                          
------------+---------+------------+--------------------------------------------------------------
 dblink     | 1.2     | public     | connect to other PostgreSQL databases from within a database
 oracle_fdw | 1.2     | public     | foreign data wrapper for Oracle access
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
(3 rows)
{% endhighlight %}

Efectivamente, la extensión **oracle_fdw** ha sido correctamente generada, por lo que el siguiente paso consistirá en crear un nuevo esquema (_schema_) al que posteriormente importaremos las tablas de la base de datos Oracle remota. Para ello, utilizaremos la instrucción:

{% highlight sql %}
prueba2=# CREATE SCHEMA oracle;
CREATE SCHEMA
{% endhighlight %}

Nuestro esquema de nombre **oracle** ya ha sido generado, sin embargo, se encuentra actualmente vacío. Para llevar a cabo la importación de dichas tablas, tendremos que definir un nuevo servidor remoto que utilice la extensión que acabamos de generar, especificando, como es lógico, la dirección IP y la base de datos a la que pretendemos acceder, de la siguiente forma:

{% highlight sql %}
prueba2=# CREATE SERVER oracle FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//192.168.1.150/ORCLCDB');
CREATE SERVER
{% endhighlight %}

Sin embargo, definir un servidor remoto no es suficiente para acceder al mismo, ya que necesitamos "mapear" nuestro usuario local a un usuario existente en dicho gestor remoto que cuente con los privilegios necesarios para visualizar las tablas.

En este caso, vamos a "mapear" el usuario local **alvaro2** al usuario remoto **c##alvaro1**, indicando a su vez las credenciales del mismo, ejecutando para ello la instrucción:

{% highlight sql %}
prueba2=# CREATE USER MAPPING FOR alvaro2 SERVER oracle OPTIONS (user 'c##alvaro1', password 'alvaro1');
CREATE USER MAPPING
{% endhighlight %}

Dado que estas configuraciones las hemos llevado a cabo como un usuario administrador de la base de datos, tendremos que otorgar los privilegios necesarios al usuario **alvaro2** para utilizar tanto el nuevo esquema generado como el servidor remoto definido, haciendo para ello uso de las instrucciones:

{% highlight sql %}
prueba2=# GRANT ALL PRIVILEGES ON SCHEMA oracle TO alvaro2;
GRANT

prueba2=# GRANT ALL PRIVILEGES ON FOREIGN SERVER oracle TO alvaro2;
GRANT
{% endhighlight %}

La configuración como administrador ha finalizado, de manera que podremos salir del cliente ejecutando la instrucción `exit` para así realizar una nueva conexión a la base de datos **prueba2** pero haciendo uso esta vez del rol **alvaro2**, y así verificar que puede utilizar dicho enlace, ejecutando para ello el comando:

{% highlight shell %}
postgres@servidor1:~$ psql -h localhost -U alvaro2 -d prueba2
Password for user alvaro2: 
psql (11.9 (Debian 11.9-0+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
{% endhighlight %}

Para verificar que el enlace funciona correctamente, vamos a proceder a importar las tablas existentes en el esquema remoto **c##alvaro1** a nuestro nuevo esquema local de nombre **oracle**, haciendo para ello uso del comando:

{% highlight sql %}
prueba2=# IMPORT FOREIGN SCHEMA "C##ALVARO1" FROM SERVER oracle INTO oracle;
IMPORT FOREIGN SCHEMA
{% endhighlight %}

Listo, según la salida que dicha instrucción nos ha devuelto, el esquema ha sido correctamente importado, de manera que ya podremos hacer uso de las tablas existentes en nuestro esquema **public** por defecto y del nuevo que hemos generado, de nombre **oracle**, que contiene las tablas del gestor remoto Oracle ubicado en el servidor **oracle1**.

Es importante mencionar que en caso de ejecutar la instrucción `\d`, las tablas remotas no se mostrarían, ya que por defecto, PostgreSQL busca en el esquema por defecto **public**. Si quisiésemos cambiar la ruta de búsqueda al nuevo esquema generado, tendríamos que ejecutar el comando `SET search_path='oracle';`, aunque no es necesario hacerlo para consultar dichas tablas.

La intención es mostrar los datos de los **Empleados** (tabla que se encuentra en el esquema **public**) junto al nombre del departamento al que pertenecen, información que podremos obtener de **Departamentos** (tabla que se encuentra en el esquema **oracle**), utilizando para ello la clave que tienen en común ambas tablas. La consulta a ejecutar sería:

{% highlight sql %}
prueba2=# SELECT Empleados.DNI AS DNI, Empleados.Nombre AS Nombre, Empleados.Direccion AS Direccion, Empleados.Telefono AS Telefono, Empleados.FechaNacimiento AS FechaNacimiento, Empleados.Salario AS Salario, Departamentos.Nombre AS Departamento
prueba2-# FROM oracle.Empleados Empleados, Departamentos
prueba2-# WHERE Empleados.Departamento = Departamentos.Identificador;
    dni    |          nombre           |       direccion        | telefono  |   fechanacimiento   | salario |   departamento   
-----------+---------------------------+------------------------+-----------+---------------------+---------+------------------
 90389058R | Joaquin Marrero Covas     | C/ Hijuela de Lojo, 22 | 618385118 | 1997-01-28 00:00:00 |    1446 | Administración
 18232747A | Aristarco Caban Meraz     | Puerta Nueva, 67       | 691204722 | 1994-05-07 00:00:00 |    4789 | Seguridad
 94106513N | Marian Fonseca Betancourt | C/ Manuel Iradier, 37  | 638415823 | 1979-07-17 00:00:00 |    2561 | Seguridad
 12777631G | Merlino Rosado Cordero    | C/ Henan Cortes, 58    | 609841755 | 1993-11-27 00:00:00 |    7961 | Recursos Humanos
 68219319P | Tabare Chapa Alcantar     | C/ Arana, 12           | 682227206 | 1992-03-09 00:00:00 |    8568 | Informática
 67227129S | Ian Esquivel Laboy        | C/ Inglaterra, 64      | 728005136 | 1992-07-21 00:00:00 |    3519 | Administración
 52315160G | Heinz Collado Caraballo   | Escuadro, 60           | 600173822 | 1984-02-13 00:00:00 |    1672 | Recursos Humanos
 85145590G | Anabel Lerma Dominguez    | Crta. Cadiz, 1         | 675014823 | 1981-01-07 00:00:00 |    5919 | Administración
 56228957Y | Dinorah Viera Tello       | Ctra. Villena, 22      | 642852778 | 1987-05-28 00:00:00 |    2567 | Seguridad
 61242562W | Manases Castillo Camacho  | Ctra. Hornos, 91       | 607853354 | 1991-10-29 00:00:00 |    4270 | Informática
(10 rows)
{% endhighlight %}

Donde:

* **SELECT**: Indicamos las columnas que queremos mostrar de la información obtenida, de la forma **[tabla].[columna]**.
* **FROM**: Hacemos un JOIN de la tabla Empleados que se encuentra en el esquema **public** junto con la tabla Departamentos que se encuentra en el esquema **oracle**, de la forma **[esquema].[tabla]**, para así poder mostrar información de ambas tablas en la misma consulta.
* **WHERE**: Establecemos la condición del JOIN, que deberá ser aquella columna mediante la cual vamos a unir los registros devueltos. Como es lógico, será el código del departamento, pues es la columna que se repite en ambas tablas.

Como era de esperar, la consulta ha vuelto a realizarse sin ningún problema y ha devuelto la información que debería.

Otra utilidad que podemos aprovechar es la capacidad de copiar las tablas de un esquema a otro, utilizando el resultado de una consulta simple para crear una tabla a partir de la misma. Por ejemplo, podríamos copiar la tabla **Empleados** desde el esquema **oracle** haciendo uso de la siguiente instrucción:

{% highlight sql %}
prueba2=# CREATE TABLE Empleados
prueba2-# AS (SELECT *
prueba2(#     FROM oracle.Empleados);
SELECT 10
{% endhighlight %}

En dicha instrucción, hemos realizado una consulta a la tabla **Empleados** ubicada en el esquema **oracle** generado a partir de la base de datos existente en el servidor **oracle1**, utilizando la respuesta obtenida para crear una nueva tabla con el mismo nombre, que se almacenará ahora en el esquema **public**, y que podremos empezar a utilizar sin necesidad de recurrir al enlace con el primer servidor. Si consultamos la nueva tabla generada, obtendremos el siguiente resultado:

{% highlight sql %}
prueba2=# SELECT *
prueba2-# FROM Empleados;
    dni    |          nombre           |       direccion        | telefono  |   fechanacimiento   | salario | departamento 
-----------+---------------------------+------------------------+-----------+---------------------+---------+--------------
 90389058R | Joaquin Marrero Covas     | C/ Hijuela de Lojo, 22 | 618385118 | 1997-01-28 00:00:00 |    1446 |           10
 18232747A | Aristarco Caban Meraz     | Puerta Nueva, 67       | 691204722 | 1994-05-07 00:00:00 |    4789 |           30
 94106513N | Marian Fonseca Betancourt | C/ Manuel Iradier, 37  | 638415823 | 1979-07-17 00:00:00 |    2561 |           30
 12777631G | Merlino Rosado Cordero    | C/ Henan Cortes, 58    | 609841755 | 1993-11-27 00:00:00 |    7961 |           20
 68219319P | Tabare Chapa Alcantar     | C/ Arana, 12           | 682227206 | 1992-03-09 00:00:00 |    8568 |           40
 67227129S | Ian Esquivel Laboy        | C/ Inglaterra, 64      | 728005136 | 1992-07-21 00:00:00 |    3519 |           10
 52315160G | Heinz Collado Caraballo   | Escuadro, 60           | 600173822 | 1984-02-13 00:00:00 |    1672 |           20
 85145590G | Anabel Lerma Dominguez    | Crta. Cadiz, 1         | 675014823 | 1981-01-07 00:00:00 |    5919 |           10
 56228957Y | Dinorah Viera Tello       | Ctra. Villena, 22      | 642852778 | 1987-05-28 00:00:00 |    2567 |           30
 61242562W | Manases Castillo Camacho  | Ctra. Hornos, 91       | 607853354 | 1991-10-29 00:00:00 |    4270 |           40
(10 rows)
{% endhighlight %}

Como se puede apreciar, el contenido es exactamente el mismo que el existente en la tabla ubicada en el gestor remoto, por lo que podemos concluir que su clonación ha sido efectiva y que los servidores tienen conectividad entre sí mediante los enlaces creados, sea cual sea el sentido utilizado.