---
layout: post
title:  "Configuración LDAP en alta disponibilidad"
banner: "/assets/images/banners/ldap.jpg"
date:   2020-12-21 09:38:00 +0200
categories: sistemas openstack
---
Se recomienda realizar una previa lectura de los _posts_ [Instalación y configuración inicial de OpenLDAP](https://www.alvarovf.com/sistemas/openstack/2020/12/14/instalacion-configuracion-openldap.html) y [Configuración LDAPs](https://www.alvarovf.com/sistemas/openstack/2020/12/17/configuracion-ldaps.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, añadiendo un servidor LDAP secundario en la máquina **Sancho** para así ofrecer alta disponibilidad al servidor LDAP principal, ubicado en la máquina **Freston**.

Hasta ahora hemos hecho uso de **Freston** como único servidor LDAP, pero en una situación real, depender de un único servidor que ofrezca un servicio tan importante como es LDAP es una práctica muy arriesgada, ya que de dicho servicio dependen muchos otros y por tanto, en caso de que ocurriese un fallo en dicha máquina, las consecuencias serían bastante considerables.

Por ello, se suele practicar la redundancia o alta disponibilidad, consistente en configurar al menos un servidor LDAP secundario que será capaz de dar servicio a los clientes mientras se solucionan los hipotéticos problemas en el servidor principal, haciendo para ello uso del motor **_LDAP Sync Replication_**, más comúnmente conocido como **_syncrepl_**. Además, la implementación **OpenLDAP** es extremadamente flexible y soporta por tanto una variedad de modos de funcionamiento de dicho motor para diversos escenarios, que pasaré a explicar a continuación:

- **syncrepl**: Es el modo de funcionamiento básico, el típico _maestro-esclavo_, llamado en LDAP **_provider-consumer_**. Consiste en copiar un fragmento del árbol de información del directorio (**_DIT_**) en el servidor secundario (**_consumer_**) mediante una lectura en el servidor principal (**_provider_**), seguido de periódicas actualizaciones de dicha información, haciendo uso de uno de los [métodos](https://www.openldap.org/doc/admin24/replication.html) que se mencionarán a continuación. El motor de replicación, por tanto, se ejecuta en el _consumer_ como un _thread_ de **slapd**.

  - **pull-based**: El _consumer_ pregunta periódicamente al _provider_ por actualizaciones en dicha información.
  - **push-based**: El _consumer_ escucha para recibir dichas actualizaciones por parte del _provider_ en tiempo real.

  - **Desventajas**:
    - Es un mecanismo basado en objetos, es decir, cuando se modifica un único atributo de un objeto replicado, el _consumer_ procesa el objeto en su totalidad, incluyendo los atributos modificados y los no modificados, causando un gasto de recursos innecesario.
    - Únicamente se permiten escrituras en el _provider_, ya que los servidores secundarios consultan al mismo para obtener dichas actualizaciones.
  - **Ventajas**:
    - En caso de producirse varios cambios en un solo objeto, no es necesario conservar una secuencia precisa de dichos cambios, pues únicamente el estado final de la entrada es significativo.
    - Únicamente debe llevarse a cabo la configuración en el lado del _consumer_.

- **Delta-syncrepl**: Es una variante de **syncrepl** que pretende solventar la primera de las desventajas anteriormente mencionadas, almacenando el _provider_ para ello un registro de cambios (_changelog_) en una base de datos separada. El _consumer_ consultará dicho registro y en caso de encontrar diferencias, aplicará los cambios a su base de datos. Si el _consumer_ se encontrase muy separado del estado actual del _provider_ o sin sincronización alguna, se hará uso de **syncrepl** para la primera sincronización, y a partir de ahí, se hará uso del método **Delta**.

  - **Desventajas**:
    - Dado que el estado de la base de datos en el lado del _provider_ se almacena tanto en el registro de cambios como en la propia base de datos en sí, será necesario copiar o restaurar ambas partes en caso de una corrupción o migración.
    - Únicamente se permiten escrituras en el _provider_, ya que los servidores secundarios consultan al mismo para obtener dichas actualizaciones.
    - La configuración debe llevarse a cabo tanto en el lado del _consumer_ como en el lado del _provider_.
  - **Ventajas**:
    - Cuando se modifican los atributos de un objeto replicado, el _consumer_ procesa únicamente los cambios realizados, obteniendo una mayor eficiencia en las transferencias y evitando causar un gasto de recursos innecesario.

- **N-Way Multi-Provider**: Es una variante que hace uso del método **syncrepl** para replicar la información entre múltiples _providers_ (_multi-master_).

  - **Desventajas**:
    - Puede llegar a romper la consistencia de los datos existentes en el directorio, ya que si por problemas de red unos clientes tienen conectividad con un _provider_ y otros clientes con otro diferente, posteriormente puede ser complicado unificar la información de ambos.
  - **Ventajas**:
    - Si un _provider_ falla, otros podrán seguir atendiendo peticiones, evitando por tanto el punto único de fallo.
    - Los proveedores se podrán situar en lugares distintos físicamente hablando, combatiendo gracias a ello otros agentes que pudiesen ocasionar problemas en el sistema (terremotos, inundaciones...).

- **MirrorMode**: Es un método híbrido que aporta todas las garantías de consistencia de los datos de las réplicas en las que actúa un único proveedor, además de proveer la alta disponibilidad de las réplicas en las que intervienen varios. Consiste en configurar dos proveedores que se repliquen entre sí (como en el método **N-Way Multi-Provider**) pero utilizando una interfaz externa (_frontend_) que dirija todas las escrituras a un único proveedor, de manera que el secundario únicamente se utilizaría en caso de que el principal dejase de ofrecer servicio. Posteriormente, cuando se restaurase el proveedor principal, todos los cambios se sincronizarían con el secundario, al haber sido este último el que ha estado recibiendo escrituras durante ese periodo.

  - **Desventajas**:
    - No consiste exactamente en una solución **_Multi-Provider_**, ya que las escrituras únicamente se llevan a cabo en uno de los nodos (aunque posteriormente son replicadas).
    - Necesita un servidor externo (**slapd** en modo **_proxy_**) o dispositivo (balanceador de carga físico) para comprobar qué proveedor se encuentra actualmente activo.
  - **Ventajas**:
    - Proporciona alta disponibilidad para las escrituras en el directorio, de manera que mientras exista un proveedor operativo, las escrituras van a ser aceptadas.
    - Los proveedores se replican entre sí, de manera que la información está siempre actualizada en todos los nodos.
    - En caso de que uno de los proveedores deje de prestar servicio, se volverá a sincronizar automáticamente cuando vuelva a estar operativo.

En mi opinión, el método de funcionamiento que más se adapta a mi escenario es **MirrorMode**, pues es un modo híbrido que unifica las ventajas de los anteriores métodos a cambio de unas desventajas que prácticamente carecen de importancia.

Como anteriormente hemos mencionado, el único servidor LDAP actualmente activo en nuestro escenario es **Freston**, que cuenta además con el protocolo **ldaps://** configurado. Sin embargo, no ocurre lo mismo con **Sancho**, por lo que tendremos que llevar a cabo en dicha máquina las [modificaciones](https://pastebin.com/DJbJPb74) mencionadas en los artículos [Instalación y configuración inicial de OpenLDAP](https://www.alvarovf.com/sistemas/openstack/2020/12/14/instalacion-configuracion-openldap.html) y [Configuración LDAPs](https://www.alvarovf.com/sistemas/openstack/2020/12/17/configuracion-ldaps.html), para que así cuente con la misma configuración inicial. No es necesario generar también los objetos de prueba en el servidor secundario, ya que posteriormente se replicarán de forma automática del servidor principal.

Vamos a comenzar por llevar a cabo la configuración en el servidor principal, es decir, en **Freston**, generando en el directorio un usuario que cuente con los privilegios suficientes para leer todas las entradas existentes en el mismo, con la finalidad de poder replicarlas en el servidor secundario sin necesidad de hacer uso del usuario administrador, además de poder añadir nuevas entradas para así mantener en todo momento una réplica exacta. Para ello, haremos uso de los **objectClass** de nombre **account** y **simpleSecurityObject**, debiendo indicar por tanto, de forma obligatoria, el valor de los atributos **uid** y **userPassword**.

Antes de proceder con la creación, tendremos que encriptar una contraseña que posteriormente asignaremos a dicho usuario, haciendo para ello uso de `slappasswd`. En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@freston:~# slappasswd
New password:
Re-enter new password:
{SSHA}TlTAeN7S3B6vYx9JWPv/oSx0uYO2vmt9
{% endhighlight %}

Tras introducir en dos ocasiones la contraseña por teclado, se nos habrá mostrado la cadena encriptada haciendo uso del algoritmo **SSHA**, que debemos asignar al atributo **userPassword**.

Como ya bien sabemos, para interactuar con los objetos del directorio debemos utilizar ficheros **.ldif**, de manera que en este caso, generamos un fichero de nombre **usuariocopia.ldif**, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# nano usuariocopia.ldif
{% endhighlight %}

Dentro del mismo, introduciré el valor de los atributos necesarios para la creación de un usuario de nombre **mirrormode**, que no considero necesario explicar con detalle ya que en anteriores artículos lo hemos tratado con mayor detenimiento, quedando de la siguiente manera:

{% highlight shell %}
dn: uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: account
objectClass: simpleSecurityObject
uid: mirrormode
description: Usuario para MirrorMode
userPassword: {SSHA}TlTAeN7S3B6vYx9JWPv/oSx0uYO2vmt9
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así añadir el nuevo objeto a la estructura del directorio, haciendo para ello uso de uno de los binarios que nos ofrecen las **ldap-utils**, de nombre `ldapadd`:

{% highlight shell %}
root@freston:~# ldapadd -x -D "cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org" -f usuariocopia.ldif -W
Enter LDAP Password:
adding new entry "uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org"
{% endhighlight %}

Donde:

* **-x**: Especificamos que se haga uso de la autenticación simple en lugar de SASL.
* **-D**: Indicamos el nombre distintivo del usuario del que queremos hacer uso para realizar la inserción.
* **-f**: Indicamos el fichero en el que se encuentra definido el objeto a insertar.
* **-W**: Indicamos que se solicite la contraseña del usuario de manera oculta, en lugar de introducirla en el propio comando.

Como se puede apreciar en la salida del comando ejecutado, el usuario ha sido correctamente añadido. Sin embargo, de nada nos sirve dicho usuario si no cuenta con los privilegios necesarios para poder leer y escribir en el directorio, de manera que tendremos que crear un nuevo fichero **.ldif** de nombre **permisoscopia.ldif** en el que asignaremos dichos privilegios haciendo uso de las listas de control de acceso (_ACL_), ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano permisoscopia.ldif
{% endhighlight %}

Explicar el funcionamiento de las listas de control de acceso sería bastante extenso, además de salirse del objetivo del artículo, pues en Internet puede encontrarse bastante información al respecto. El resultado final del fichero sería:

{% highlight shell %}
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: to attrs=userPassword
  by self =xw
  by dn.exact="cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org" =xw #Modificar por el nombre distintivo del administrador del directorio.
  by dn.exact="uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org" read #Modificar por el nombre distintivo del nuevo usuario generado.
  by anonymous auth
  by * none
olcAccess: to *
  by anonymous auth
  by self write
  by dn.exact="uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org" read #Modificar por el nombre distintivo del nuevo usuario generado.
  by users read
  by * none
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así modificar los atributos del objeto correspondiente a la configuración, haciendo para ello uso de `ldapmodify`, pero utilizando en este caso unos parámetros concretos, haciendo para ello uso del comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f permisoscopia.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}mdb,cn=config"
{% endhighlight %}

Donde:

* **-Y**: Especificamos un mecanismo SASL a utilizar para la autenticación, en este caso, **EXTERNAL**.
* **-H**: Indicamos la URI de conexión al servidor LDAP, haciendo uso en este caso del _socket UNIX_ (**ldapi:///**) para así contar con los privilegios necesarios para llevar a cabo modificaciones en la configuración.
* **-f**: Indicamos el fichero en el que se encuentran definidos los cambios a realizar.

Como se puede apreciar, la entrada **olcDatabase={1}mdb,cn=config** ha sido correctamente modificada dado que contamos con los privilegios necesarios para ello al haber hecho uso del _socket UNIX_, de manera que el usuario ya tiene asignados los permisos que necesita.

Tras ello, tendremos que cargar en memoria el módulo encargado de llevar a cabo la sincronización entre los servidores, de nombre **syncprov**, de manera que tendremos que crear un nuevo fichero **.ldif** de nombre **modulocopia.ldif** en el que modificaremos la configuración del directorio para así añadir dicho módulo, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano modulocopia.ldif
{% endhighlight %}

El resultado final del fichero sería:

{% highlight shell %}
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así modificar los atributos del objeto correspondiente a la configuración, haciendo uso una vez más del comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f modulocopia.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=module{0},cn=config"
{% endhighlight %}

Como se puede apreciar, la entrada **cn=module{0},cn=config** ha sido correctamente modificada, de manera que el módulo necesario ya ha sido cargado en memoria.

El módulo ya está habilitado, pero no cuenta con configuración alguna asignada, de manera que tendremos que crear un nuevo fichero **.ldif** de nombre **modulocopia2.ldif** en el que añadiremos la configuración dicho módulo, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano modulocopia2.ldif
{% endhighlight %}

Dentro del mismo indicaremos la nueva configuración, concretamente añadiendo un atributo **olcSpCheckpoint** en el que estableceremos cada cuántas operaciones o minutos se va a llevar a cabo un _checkpoint_, pensado para minimizar la actividad de sincronización después de que un proveedor deje de ofrecer servicio y tenga que recuperarse. En este caso, he considerado que **5** es un buen valor para ambas posibilidades:

{% highlight shell %}
dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 5 5
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así añadir el nuevo objeto a la estructura del directorio, haciendo uso una vez más del comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f modulocopia2.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=syncprov,olcDatabase={1}mdb,cn=config"
{% endhighlight %}

Como se puede apreciar, la entrada **olcOverlay=syncprov,olcDatabase={1}mdb,cn=config** ha sido correctamente añadida al directorio, de manera que el módulo cuenta ahora con la primera parte de la configuración necesaria para su correcto funcionamiento.

Nuestra configuración de **MirrorMode** está cada vez más cerca de finalizar. El siguiente paso consiste en añadir un número identificativo al servidor, de manera que tendremos que crear un nuevo fichero **.ldif** de nombre **servidorcopia.ldif** en el que especifiquemos dicha información, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano servidorcopia.ldif
{% endhighlight %}

Dentro del mismo estableceremos un atributo **olcServerId** en el que indicaremos el identificador **1**, siendo muy importante que posteriormente, cuando tengamos que llevar a cabo los mismos pasos en el servidor secundario, dicho número sea diferente al que vamos a introducir ahora. El resultado final del fichero sería:

{% highlight shell %}
dn: cn=config
changetype: modify
add: olcServerId
olcServerId: 1
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así modificar los atributos del objeto correspondiente a la configuración, haciendo uso una vez más del comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f servidorcopia.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"
{% endhighlight %}

Como se puede apreciar, la entrada **cn=config** ha sido correctamente modificada, de manera que el servidor se encuentra ahora identificado por el número **1**.

Hemos llegado al último paso de la configuración del servidor principal, consistente en indicar algunos parámetros que se utilizarán para la sincronización, así como habilitarla propiamente, de manera que tendremos que crear un nuevo fichero **.ldif** de nombre **habilitarsinc.ldif** en el que especifiquemos dicha configuración, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# nano habilitarsinc.ldif
{% endhighlight %}

El resultado final del fichero sería:

{% highlight shell %}
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncrepl
olcsyncrepl: rid=000
  provider=ldaps://sancho.alvaro.gonzalonazareno.org #Modificar por el servidor contrario al que estamos utilizando, es decir, en Freston será Sancho y viceversa. Podemos utilizar la IP en lugar del nombre DNS. Dado que tengo LDAPs correctamente configurado, utilizaré dicho protocolo para así aumentar la seguridad en las transferencias.
  type=refreshAndPersist
  retry="5 5 300 +" 
  searchbase="dc=alvaro,dc=gonzalonazareno,dc=org" #Modificar por la base del directorio que se pretende replicar, que generalmente se encuentra definida a partir del FQDN de la máquina.
  attrs="*,+" 
  bindmethod=simple
  binddn="uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org" #Modificar por el nombre distintivo del nuevo usuario generado.
  credentials=[contraseñaenclaro] #Modificar por la contraseña del nuevo usuario generado.
-
add: olcDbIndex
olcDbIndex: entryUUID eq
olcDbIndex: entryCSN eq
-
replace: olcMirrorMode
olcMirrorMode: TRUE
{% endhighlight %}

Una vez definido el fichero **.ldif**, tendremos que importarlo para así modificar los atributos del objeto correspondiente a la configuración, haciendo uso una vez más del comando:

{% highlight shell %}
root@freston:~# ldapmodify -Y EXTERNAL -H ldapi:/// -f habilitarsinc.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}mdb,cn=config"
{% endhighlight %}

Como se puede apreciar, la entrada **olcDatabase={1}mdb,cn=config** ha sido correctamente modificada, de manera que el servidor principal ya se encuentra configurado en su totalidad para hacer uso de _MirrorMode_.

De otro lado, tendremos que llevar a cabo las mismas [modificaciones](https://pastebin.com/6W8QQaCL) en la máquina **Sancho** para que así la conexión sea recíproca, con la única diferencia de que el identificador del servidor (**olcServerId**) será en esta ocasión **2** y su servidor proveedor (**provider**) será en esta ocasión **ldaps://freston.alvaro.gonzalonazareno.org**.

Tras ello, haremos uso de `ldapsearch` para así ejecutar una búsqueda anónima sobre el directorio del servidor **Sancho**. En este caso, no quiero establecer ningún tipo de filtro, sino listar todos los objetos existentes en la estructura actualmente definida, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org"

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

# mirrormode, alvaro.gonzalonazareno.org
dn: uid=mirrormode,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: account
objectClass: simpleSecurityObject
uid: mirrormode
description: Usuario para MirrorMode

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

Donde:

* **-x**: Especificamos que se haga uso de la autenticación simple en lugar de SASL.
* **-b**: Indicamos la base a partir de la cuál queremos mostrar la información.

Como se puede apreciar, todos los objetos previamente existentes en el directorio de **Freston** existen ahora también en el directorio de **Sancho**, por lo que podemos concluir que la sincronización funciona correctamente para aquellos objetos que se inserten en la primera de las máquinas.

De otro lado, vamos a llevar a cabo una pequeña prueba para verificar que funciona también en el sentido inverso, pues he preparado un fichero **.ldif** en el que he definido un usuario de nombre **Pablo**, fichero que tendremos que importar para así añadir el nuevo objeto a la estructura del directorio, haciendo uso una vez más del comando:

{% highlight shell %}
root@sancho:~# ldapadd -x -D "cn=admin,dc=alvaro,dc=gonzalonazareno,dc=org" -f usuarioprueba.ldif -W
Enter LDAP Password:
adding new entry "uid=pablo,ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org"
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el usuario ha sido correctamente añadido, así que vamos a volver a la máquina **Freston** para así ejecutar una búsqueda anónima sobre el directorio de dicho servidor. En este caso, no quiero establecer ningún tipo de filtro, sino listar todos los objetos existentes en la estructura actualmente definida, ejecutando para ello el comando:

{% highlight shell %}
root@freston:~# ldapsearch -x -b "dc=alvaro,dc=gonzalonazareno,dc=org"

# alvaro.gonzalonazareno.org
dn: dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: alvaro.gonzalonazareno.org
dc: alvaro

...

# pablo, Personas, alvaro.gonzalonazareno.org
dn: uid=pablo,ou=Personas,dc=alvaro,dc=gonzalonazareno,dc=org
objectClass: posixAccount
objectClass: inetOrgPerson
uid: pablo
cn: Pablo Ferreras
givenName: Pablo
sn: Ferreras
uidNumber: 2000
gidNumber: 2000
homeDirectory: /home/pablo
loginShell: /bin/bash

# search result
search: 2
result: 0 Success
{% endhighlight %}

Efectivamente, podemos concluir que la sincronización ha funcionado también y el usuario añadido existe ahora en ambas máquinas, gracias a la interfaz externa (_frontend_) que decide de qué manera se escribe la información, de una forma totalmente abstracta y ajena a nosotros.