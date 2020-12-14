---
layout: post
title:  "Instalación y configuración inicial de OpenLDAP"
banner: "/assets/images/banners/ldap.jpg"
date:   2020-12-14 19:43:00 +0200
categories: sistemas openstack
---
Se recomienda realizar una previa lectura del _post_ [Modificación del escenario OpenStack](https://www.alvarovf.com/hlc/openstack/2020/12/12/modificacion-escenario-openstack.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, llevando a cabo una instalación de un servidor LDAP sobre la máquina **Freston** anteriormente creada, la cuál se encuentra conectada a la red interna (**10.0.1.0/24**).

Hasta ahora hemos hecho uso de un mecanismo de autenticación de usuarios basado en ficheros (**passwd**, **shadow**, **group** y **gshadow**), mecanismo que es realmente útil para usuarios locales, pero que pierde efectividad y se vuelve más complejo de gestionar cuando queremos aumentar la infraestructura a diferentes equipos. Es por ello que acudimos al protocolo **LDAP**.

**LDAP** son las siglas de **_Lightweight Directory Access Protocol_**, o en castellano, **Protocolo Ligero de Acceso a Directorios**. Se trata de un conjunto de protocolos de licencia abierta que son utilizados a nivel de aplicación para acceder a los servicios del directorio remoto, una base de datos no relacional que se encuentra centralizada (replicada en la mayoría de ocasiones para ofrecer alta disponibilidad) que almacena entre otras cosas información de los usuarios. El uso de ese tipo de base de datos es debido a las mejoras de rendimiento que ofrece con respecto a las relacionales para nuestro propósito concreto, ya que estas últimas están pensadas para ejecutar operaciones como consultas complejas.

Como hemos mencionado, su uso no se limita al almacenamiento de la información relativa a las cuentas de usuario existentes en la máquina, sino que también permite almacenar otra información que no está contemplada en los ficheros anteriormente mencionados, como por ejemplo una foto del usuario, sus claves públicas, certificados X.509...

A pesar de que el objetivo de este artículo no es el de aprender a utilizar LDAP, sino instalarlo y configurarlo, considero necesario tener unas nociones mínimas para conocer de forma superficial su funcionamiento.

La información se almacena haciendo uso de una estructura jerárquica o arborescente, es decir, dentro de dicho directorio se definen **entradas** u **objetos** que cuelgan a su vez de otros. En la parte más superior encontramos la entrada raíz (también conocida como _root_ o base), que como su propio nombre indica, es el primer objeto que se define y a partir del cuál se establecerá nuestra estructura.

Cada una de las entradas u objetos que definimos pertenecen a una o más de una **clase** diferente, las cuales especifican los posibles **atributos** de los que se pueden hacer uso para añadir información sobre dicho objeto, indicando a su vez cuáles de ellos son obligatorios u opcionales, así como permitir la inserción de un único valor o de varios.

Las clases vienen ya predefinidas, es decir, no las creamos nosotros, sino que cuando necesitamos almacenar una determinada información, buscamos el atributo que más se adapte a nuestras necesidades, y dado que dicho atributo pertenece a una clase, haremos uso de la misma. Esto puede ser considerado una limitación o una ventaja, ya que evitamos tener que definir los tipos de datos de los que haremos uso.

Dentro de la jerarquía se hace uso de un atributo **_Relative Distinguished Name_** (**RDN**) o en castellano, **Nombre Distintivo Relativo**, cuya finalidad es la de caracterizar a un objeto de manera única dentro de la jerarquía.

Muy posiblemente estéis pensando... si las clases contienen un número finito de posibles atributos de los que podemos hacer uso, ¿qué puedo hacer si la información que deseo almacenar no se encuentra contemplada en ninguno de ellos? Bien, anteriormente no he mencionado que las clases que podemos utilizar están contenidas en **esquemas**, es decir, para que podamos hacer uso de una determinada clase, debe existir un esquema en el que dicha clase esté contenida. Gracias a esta característica, podremos importar manualmente nuevos esquemas que obtengamos de Internet y por tanto, utilizar nuevas clases. Por defecto, la implementación de LDAP de la que haremos uso (**_OpenLDAP_**) contiene un total de 4 esquemas predefinidos.

Antes de comenzar con la instalación, es recomendable configurar correctamente el _FQDN_ de la máquina, ya que por defecto se hará uso del mismo para generar la entrada base, aunque en caso de ser necesario, se podría modificar tras la instalación. En mi caso, comprobaré si la configuración es correcta haciendo para ello uso del comando:

{% highlight shell %}
debian@freston:~$ hostname -f
freston.alvaro.gonzalonazareno.org
{% endhighlight %}

Efectivamente, la máquina se encuentra correctamente nombrada dentro del subdominio **alvaro.gonzalonazareno.org**, del que hará uso LDAP para generar dicho objeto raíz.

Ya está todo listo para proceder con la instalación de **OpenLDAP** (**_slapd_**), la implementación más extendida de LDAP que se encuentra actualmente en la versión **2.4**, no sin antes actualizar la lista de la paquetería disponible, así como asegurar que toda la paquetería instalada en la máquina se encuentra en su última versión, por lo que ejecutaremos el comando (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
root@freston:~# apt update && apt upgrade && apt install slapd
{% endhighlight %}

Durante la instalación (y dependiendo del nivel de prioridad de preguntas configurado), se nos preguntará la contraseña para el usuario administrador (**admin**) que será automáticamente generado durante la misma, tal y como se puede apreciar a continuación:

![instalacion1](https://i.ibb.co/HF2DZLn/Captura-de-pantalla-de-2020-12-14-10-31-27.png "Instalación slapd")

Cuando hayamos introducido una contraseña lo suficientemente segura, tendremos que volver a escribirla para confirmarla y la instalación finalizará.

Como consecuencia del proceso que está actualmente ejecutándose, se habrá abierto un _socket TCP/IP_ en el puerto por defecto de LDAP (**389**) que estará escuchando peticiones provenientes de todas las interfaces de la máquina (**0.0.0.0**), así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@freston:~# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      453/sshd            
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      9422/slapd          
tcp6       0      0 :::22                   :::*                    LISTEN      453/sshd            
tcp6       0      0 :::389                  :::*                    LISTEN      9422/slapd
{% endhighlight %}

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho _socket_.

Efectivamente, el proceso **slapd** está escuchando peticiones tal y como debería, de manera que vamos a proceder a instalar un paquete de utilidades de línea de comandos totalmente esencial, de nombre **ldap-utils**, que nos permitirán interactuar con el servidor ejecutando búsquedas, modificaciones, inserciones de nuevos objetos... Para ello, haremos uso del comando:

{% highlight shell %}
root@freston:~# apt install ldap-utils
{% endhighlight %}

Una vez instalado el paquete, utilizaremos uno de los binarios que se incluyen, de nombre `ldapsearch`, que nos permitirá ejecutar una búsqueda sobre el directorio. En este caso, no quiero establecer ningún tipo de filtro, sino listar todos los objetos existentes en la estructura actualmente definida. Para ello, haremos uso del comando:

{% highlight shell %}
root@freston:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org"

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

# admin, alvaro.gonzalonazareno.org
dn: cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# search result
search: 2
result: 0 Success
{% endhighlight %}

Donde:

* **-x**: Especificamos que se haga uso de la autenticación simple en lugar de SASL.
* **-b**: Indicamos la base a partir de la cuál queremos mostrar la información. En este caso, **dc=alvaro,dc=gonzalonazareno,dc=org**, resultado de haber hecho uso del _FQDN_ para generarla.

Como se puede apreciar, se han mostrado un total de dos objetos, siendo el primero de ellos el objeto base a partir del cuál cuelgan el resto, y el segundo, el objeto correspondiente al usuario administrador que se generó automáticamente durante la instalación, que cuelga del anterior. Además, se nos han mostrado las clases a las que dichos objetos pertenecen (**objectClass**), así como algunos atributos y sus valores, entre los que se encuentra el nombre distintivo (**dn**), utilizado para identificar dicho objeto de manera única en la jerarquía.

Sin embargo, tal y como he mencionado, únicamente se han mostrado algunos atributos, no todos los existentes. Eso es debido a que estamos llevando a cabo una **búsqueda anónima**, que ofrece una gran comodidad ya que no es necesario autenticarse, pero que por consecuencia, evita mostrar determinados atributos por motivos de seguridad, como por ejemplo, las contraseñas encriptadas de los usuarios. Si quisiésemos llevar a cabo una búsqueda mostrando la totalidad de los atributos existentes, tendríamos que hacer uso del usuario **admin**, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ldapsearch -x -D "cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org" -b "dc=alvaro,dc=gonzalonazareno,dc=org" -W
Enter LDAP Password:

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

# admin, alvaro.gonzalonazareno.org
dn: cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e1NTSEF9V0s1cG42SGdiTFo5UkVQRWgxUVlIMm1vQ0dEY0JZSkc=

# search result
search: 2
result: 0 Success
{% endhighlight %}

Donde:

* **-D**: Indicamos el nombre distintivo del usuario del que queremos hacer uso para realizar la búsqueda. En este caso, **cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org**.
* **-W**: Indicamos que se solicite la contraseña del usuario de manera oculta, en lugar de introducirla en el propio comando.

Como se puede apreciar, el atributo **userPassword** ha sido ahora mostrado, ya que hemos ejecutado la búsqueda haciendo uso de un usuario con los privilegios suficientes para ello. Es importante mencionar que la contraseña encriptada se encuentra codificada en **Base64** (por eso se ha indicado **::** en lugar de **:** entre el atributo y su valor).

La instalación y configuración necesaria para el uso del servidor LDAP ha concluido, y a pesar de que ya podríamos comenzar a incluir nuevos objetos que colgasen de la base del árbol, no es lo recomendable, ya que la información tendría una estructura bastante simple y desorganizada.

Para lograr una mejor estructura, la información suele organizarse en forma de ramas de las que cuelgan objetos similares (por ejemplo, una rama para **usuarios** y otra para **grupos**). Organizar de esta manera la estructura nos aporta también una mayor agilidad en las búsquedas, así como una gestión más eficiente sobre los permisos.

Dichas ramas son comúnmente denominadas **unidades organizativas**, que al fin y al cabo son un tipo de objeto especial, que no tiene en sí más características que la anteriormente mencionada, el cuál pertenecerá a la clase **organizationalUnit**, cuyo único atributo obligatorio será **ou**, en el que se indicará el nombre de la unidad organizativa.

Para definir dichos objetos, haremos uso de un fichero con extensión **.ldif**, que crearemos y modificaremos haciendo uso del comando:

{% highlight shell %}
root@freston:~# nano unidades.ldif
{% endhighlight %}

Dentro del mismo, definiremos en este caso un total de dos objetos, separados entre sí por una línea vacía, que es el delimitador utilizado para designar que se trata de objetos diferentes. Vamos a proceder a crear una unidad organizativa para almacenar personas (de nombre **Personas**, por ejemplo) y otra para almacenar grupos (de nombre **Grupos**, por ejemplo), quedando de la siguiente manera:

{% highlight shell %}
dn: ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Personas 

dn: ou=Grupos,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Grupos
{% endhighlight %}

* En la primera línea (atributo **dn**) hemos indicado el nombre distintivo que tendrá dentro de la jerarquía, haciendo para ello uso del único atributo obligatorio (**ou**) que utiliza la clase en cuestión, teniendo por tanto la certeza de que nunca se va a repetir.
* En la segunda línea (atributo **objectClass**) hemos importado la clase **_organizationalUnit_** necesaria para definir una unidad organizativa.
* En la tercera línea (atributo **ou**) hemos asignado un valor al atributo obligatorio que utiliza la clase en cuestión, correspondiente al nombre de dicha unidad organizativa.

Una vez definido el fichero **.ldif**, tendremos que importarlo para así añadir los nuevos objetos a la estructura del directorio, haciendo para ello uso de otro de los binarios que nos ofrecen las **ldap-utils**, de nombre `ldapadd`:

{% highlight shell %}
root@freston:~# ldapadd -x -D "cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org" -f unidades.ldif -W
Enter LDAP Password: 
adding new entry "ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org"

adding new entry "ou=Grupos,dc=alvaro,dc=gonzalonazareno,dc=org"
{% endhighlight %}

Donde:

* **-x**: Especificamos que se haga uso de la autenticación simple en lugar de SASL.
* **-D**: Indicamos el nombre distintivo del usuario del que queremos hacer uso para realizar la inserción. En este caso, **cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org**.
* **-f**: Indicamos el fichero en el que se encuentran definidos los objetos a insertar.
* **-W**: Indicamos que se solicite la contraseña del usuario de manera oculta, en lugar de introducirla en el propio comando.

Como se puede apreciar en la salida del comando ejecutado, ambos objetos han sido correctamente añadidos, por lo que vamos a proceder a llevar a cabo una nueva búsqueda anónima para así verificar que los nuevos objetos cuelgan correctamente de la base del árbol, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org"

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

# admin, alvaro.gonzalonazareno.org
dn: cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# Personas, alvaro.gonzalonazareno.org
dn: ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Personas

# Grupos, alvaro.gonzalonazareno.org
dn: ou=Grupos,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: organizationalUnit
ou: Grupos

# search result
search: 2
result: 0 Success
{% endhighlight %}

Como era de esperar, de la base del árbol (**dc=alvaro,dc=gonzalonazareno,dc=org**) cuelgan un total de tres objetos, siendo el primero de ellos el usuario administrador y los otros dos, las unidades organizativas que acabamos de crear.

De dichas unidades organizativas podrán colgar posteriormente nuevos objetos que creemos, clasificándolos gracias a las mismas. Para quién se lo esté preguntando, el usuario administrador es un tanto peculiar, por lo que es normal que cuelgue de la propia base del árbol y no de la nueva unidad organizativa pensada para albergar información de las personas.