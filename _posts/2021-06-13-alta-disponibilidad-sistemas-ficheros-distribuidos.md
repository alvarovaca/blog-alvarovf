---
layout: post
title:  "Alta disponibilidad con sistemas de ficheros distribuidos"
banner: "/assets/images/banners/altadisponibilidad.jpg"
date:   2021-06-13 18:02:00 +0200
categories: seguridad
---
Esta entrada corresponde al desarrollo del **T**rabajo de **F**in de **G**rado (**TFG**) del ciclo **A**dministración de **S**istemas **I**nformáticos en **R**ed (**ASIR**) durante la promoción 2019-2021 en el IES Gonzalo Nazareno, Dos Hermanas, Sevilla.

Mi idea a desarrollar para el TFG de forma paralela a la formación en centros de trabajo consiste en profundizar en la **alta disponibilidad** (_HA_), así como en los **sistemas de ficheros distribuidos**, necesarios para el correcto funcionamiento de por ejemplo, una base de datos replicada.

Dada la velocidad con la que se ha tratado el tema de la alta disponibilidad en la asignatura **Seguridad y Alta Disponibilidad** durante el curso, añadido además mi interés sobre el tema, considero que los conocimientos adquiridos no se adaptan a los que esperaba, siendo por tanto una ocasión perfecta para hacerlo durante el desarrollo del TFG.

El objetivo principal de este artículo es el de mostrar de una forma muy similar a lo que nos podríamos encontrar tradicionalmente en un entorno real de producción, cómo funciona la alta disponibilidad de una determinada aplicación desplegada en un conjunto de servidores, desde el momento de la configuración inicial de los mismos hasta el despliegue descentralizado de la aplicación, para así seguir ofreciendo el servicio en todo momento. En este caso desplegaremos un _PrestaShop_.

Para el desarollo del mismo, he montado un escenario en OpenStack constituido por un total de 10 máquinas con sistema operativo **Debian 10.6**, tal y como se puede apreciar a continuación:

![escenario](https://i.ibb.co/SQTnnmg/escenario.jpg "Escenario")

Sin embargo, antes de comenzar a desarrollar la tarea de manera práctica, es necesario conocer una serie de conceptos que nos ayudarán a entender mucho mejor qué es lo que pretendemos conseguir finalmente.

La **Alta Disponibilidad** (_HA_) consiste en tener un sistema redundante que permita que en caso de ocurrir algún tipo de fallo, el servicio se siga ofreciendo con normalidad. Para ello, nos serviremos de clústeres, un conjunto de equipos independientes que realizan alguna tarea común en la que se comportan como un único equipo.

Los clústeres de alta disponibilidad han ido perdiendo importancia en los últimos años dada la aparición de nuevas tecnologías como **Kubernetes**, que nos permiten hacerlo de una forma más abstracta y sencilla, aunque tradicionalmente se han configurado como vamos a mostrar aquí.

Existen varios tipos de clústeres:

* **Alto rendimiento** (_HPC_): Aumentamos la potencia computacional, por ejemplo, consiguiendo hacer un mayor número de cálculos en menos tiempo. Se suelen utilizar equipos físicos. Utilizados en centros de investigación, ingeniería y otras actividades.

* **Alta disponibilidad** (_HA_): Redundancia entre equipos ofreciendo tolerancia a fallos, de manera que el servicio siempre se encuentre levantado aunque ocurran fallos. Se pretende eliminar todos los puntos únicos de fallo (_SPOF_) mediante redundancia a todos los niveles: hardware, almacenamiento, redes... Se suelen implementar con balanceo de carga para repartir las peticiones entre los nodos. Debe ser lo suficientemente inteligente como para detectar fallos, reiniciar la aplicación en otro nodo y mantener el servicio (garantizando en todo momento su integridad) sin ningún tipo de interacción por parte del usuario. Utilizados en servicios de Internet. Es el tipo de clúster que utilizaremos en este caso, haciendo uso de herramientas como **Pacemaker** y **Corosync**.

* **Balanceo de carga**: Evitamos sobrecargar un equipo, repartiendo el tráfico entre las máquinas. Debe ir acompañado de un algoritmo para el reparto de dicha carga: aleatorio, Round Robin, carga de nodos, tiempo de respuesta... Se puede implementar un clúster de balanceo de carga sin alta disponibilidad, aunque no es lo común.

* **Escalabilidad**: Propiedad que permite modificar el tamaño del clúster en función de las necesidades, siendo comúnmente utilizado con máquinas virtuales e instancias de _cloud_, dada la dificultad que supondría hacerlo con máquinas físicas. Con la escalabilidad asumimos que el número de nodos siempre crece, mientras que con la elasticidad se asume la adaptación de cada momento, creciendo o decreciendo el número de nodos. Por ejemplo, _Amazon EC2_ es una solución _IaaS_ en la que se paga por horas de uso y configuramos mediante plantillas los despliegues que se van a realizar.

Originalmente, los clústeres surgieron como una alternativa al escalado vertical de las máquinas, pues para aumentar la potencia computacional, en lugar de sustituir el equipo por uno nuevo más potente, añadíamos otra máquina en paralelo (nodo) que trabajaría de forma síncrona con el anterior.

Gracias al uso de tecnologías de virtualización y _cloud computing_, dicho escalado horizontal se puede llevar a cabo con máquinas virtuales o instancias de _cloud_, en lugar de máquinas físicas, con las correspondientes ventajas que ello supone: versatilidad, evitar nuevo cableado, evitar nuevas instalaciones... aunque es posible utilizar combinaciones de las anteriores opciones.

Dentro del mundo de la alta disponibilidad existe una serie de términos que encontraremos con una gran frecuencia y que son necesarios conocer:

* **Recurso**: Normalmente asociado a un servicio que queremos poner a prueba de fallos, que pertenece y es gestionado por el clúster como tal, mediante el software correspondiente. Por ejemplo, un servidor web.

* **Heartbeat**: Elemento utilizado en el clúster para conocer de forma constante y mediante una comunicación generalmente dedicada y cifrada, cuáles de los nodos se encuentran activos y funcionales. Es por ello que se hace la analogía de las pulsaciones o latidos de un corazón (_heartbeat_).

* **Split brain**: Mal funcionamiento que se produce cuando se pierde la comunicación entre los nodos y estos empiezan a tomar decisiones por su cuenta.

* **Quorum**: Mecanismo que pretende prevenir el _split brain_, basándose en una decisión compartida entre los nodos, que decidirán por votación si un determinado nodo se encuentra o no activo (ya que la conexión puede haberse perdido entre dos nodos concretos pero no entre todos). Es necesario hacerlo así ya que no podemos delegar toda la gestión del clúster en un único equipo, pues supondría la existencia de un _SPOF_.

* **Stonith**: También conocido como _Shoot The Other Node In The Head_, se utiliza cuando el _quorum_ ha decidido que el nodo no se encuentra activo y por tanto, se asegura que dicho nodo no accede a los datos, evitando una corrupción en los mismos.

Sin embargo, existe un problema que todavía no hemos tratado, y es que los sistemas de ficheros tradicionales sólo pueden montarse en un equipo de forma concurrente. En muchos casos, los clústeres requieren algún sistema de almacenamiento compartido con más propiedades, como por ejemplo los de bases de datos o servidores web de sitios dinámicos como WordPress, ya que necesitamos que el contenido que se sirva sea el mismo en todos ellos. Para ello, hacemos uso de los sistemas de ficheros distribuidos, de manera que cuando un nodo escriba, dicho cambio se refleje en todos los nodos.

Los sistemas de ficheros distribuidos utilizan su propio protocolo para comunicar el cliente y el servidor, proporcionando almacenamiento remoto a sus clientes de forma transparente, ya que ellos lo verán como si se tratase de almacenamiento local. Incluyen tolerancia a fallos, gran escalabilidad, control de concurrencia... Entre los más conocidos se encuentran **Lustre**, **Google FileSystem**, **GlusterFS**... En este caso haremos uso de **Ceph**.

Una vez comprendidos todos los términos indispensables, todo está listo para comenzar con la creación y configuración del escenario sobre el que trabajaremos.

En mi caso, he llevado a cabo la creación de las máquinas de forma previa, pero se encuentran sin ningún tipo de configuración, la cuál procederemos a realizarla con el cliente de comandos de OpenStack. La instalación de dicho cliente puede encontrarse detallada en [anteriores artículos](https://www.alvarovf.com/hlc/openstack/2020/11/15/instalacion-escenario-openstack.html), por lo que procederé a obviarla.

Una vez dentro del mismo, podremos proceder a listar las instancias existentes en nuestro proyecto para así verificar que funciona correctamente. El comando a ejecutar sería:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack server list
+--------------------------------------+--------------+--------+----------------------------------------------+--------------------+-----------+
| ID                                   | Name         | Status | Networks                                     | Image              | Flavor    |
+--------------------------------------+--------------+--------+----------------------------------------------+--------------------+-----------+
| 30bdc36b-2fa6-4017-8ef4-2925122db4d8 | ceph-dns     | ACTIVE | red de alvaro.vaca=10.0.0.17, 172.22.201.131 | Debian Buster 10.6 | m1.mini   |
| 468703b5-091d-4250-af14-ba40b6d1379b | ceph-client2 | ACTIVE | red de alvaro.vaca=10.0.0.19, 172.22.201.128 | Debian Buster 10.6 | m1.medium |
| 6f47afc2-77ad-41a1-ab73-dc94f9141663 | ceph-client1 | ACTIVE | red de alvaro.vaca=10.0.0.9, 172.22.201.127  | Debian Buster 10.6 | m1.medium |
| 7924631c-b9e6-4177-b894-88c5806d8735 | ceph-mon3    | ACTIVE | red de alvaro.vaca=10.0.0.14, 172.22.201.117 | Debian Buster 10.6 | m1.medium |
| adca60ae-b729-4e6d-88c3-e47d7eca94d0 | ceph-mon2    | ACTIVE | red de alvaro.vaca=10.0.0.5, 172.22.201.115  | Debian Buster 10.6 | m1.medium |
| c898f19d-482c-44b7-875e-380973697236 | ceph-mon1    | ACTIVE | red de alvaro.vaca=10.0.0.11, 172.22.201.59  | Debian Buster 10.6 | m1.medium |
| 2bae6ab5-1820-4350-9555-e8ba3971dbde | ceph-osd3    | ACTIVE | red de alvaro.vaca=10.0.0.10, 172.22.200.184 | Debian Buster 10.6 | m1.medium |
| 5d806b64-ee19-406e-a668-6c84110bbc42 | ceph-osd2    | ACTIVE | red de alvaro.vaca=10.0.0.15, 172.22.200.108 | Debian Buster 10.6 | m1.medium |
| 9c14fb13-14e7-42d7-93e8-650cb2ebec1d | ceph-osd1    | ACTIVE | red de alvaro.vaca=10.0.0.13, 172.22.200.106 | Debian Buster 10.6 | m1.medium |
| e6e150bc-ecc7-4af6-b2cf-bba60e9ff5b4 | ceph-admin   | ACTIVE | red de alvaro.vaca=10.0.0.3, 172.22.200.103  | Debian Buster 10.6 | m1.normal |
+--------------------------------------+--------------+--------+----------------------------------------------+--------------------+-----------+
{% endhighlight %}

Efectivamente, el cliente de OpenStack se encuentra actualmente operativo y nos ha mostrado la información referente a las 10 instancias creadas en mi proyecto, que como se puede apreciar los sabores (_flavors_) se encuentran comprendidos entre los siguientes:

- **m1.mini**:
    - **vCPUs**: 1
    - **RAM**: 512 MB
- **m1.normal**:
    - **vCPUs**: 2
    - **RAM**: 1 GB
- **m1.medium**:
    - **vCPUs**: 2
    - **RAM**: 2 GB

Es importante asignar los suficientes recursos a las máquinas, ya que tendrán que alojar varios servicios medianamente exigentes en cuanto a prestaciones.

De forma adicional, he asociado 3 volúmenes de una capacidad de 5 GiB cada uno a las máquinas **OSD**, que tal y como posteriormente veremos, son las encargadas del almacenamiento en el clúster que montaremos. Vamos a verificar que dichos volúmenes han sido correctamente creados y asociados haciendo uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack volume list
+--------------------------------------+---------+----------------+------+------------------------------------+
| ID                                   | Name    | Status         | Size | Attached to                        |
+--------------------------------------+---------+----------------+------+------------------------------------+
| 6c59d1cd-54a8-4402-9aec-65819cf217c4 | ceph-3  | in-use         |    5 | Attached to ceph-osd3 on /dev/vdb  |
| 48aa0185-f772-4e7d-9181-0b48ac6e061e | ceph-2  | in-use         |    5 | Attached to ceph-osd2 on /dev/vdb  |
| cd84226f-ca74-4be8-8538-1f706fab7b2b | ceph-1  | in-use         |    5 | Attached to ceph-osd1 on /dev/vdb  |
+--------------------------------------+---------+----------------+------+------------------------------------+
{% endhighlight %}

De forma predeterminada, al crear una máquina en OpenStack se le asigna un grupo de seguridad con unas reglas de cortafuegos que en este caso no necesitamos, pues únicamente van a causarnos conflictos al asignarles direcciones IP virtuales a algunas interfaces de red para el correcto funcionamiento de la alta disponibilidad, tal y como veremos durante el desarrollo del artículo.

Para solventar dicho problema, haremos uso de una instrucción iterativa que procederá a deshabilitar dicho grupo de seguridad en todas las máquinas existentes en el clúster, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ for i in admin osd{1..3} mon{1..3} client{1..2} dns
> do
> openstack server remove security group ceph-$i default
> done
{% endhighlight %}

Esto no es todo, ya que también tenemos que deshabilitar la seguridad en los correspondientes puertos de las máquinas, ya que cuando una máquina no tiene ningún grupo de seguridad asignado, el _modus operandi_ por defecto es el de bloquear la conexión en todos los puertos asociados a la misma.

Una vez más, utilizaremos una instrucción iterativa que deshabilitará la seguridad en los puertos existentes en las máquinas del clúster, obteniendo en un primer paso la dirección IP de la máquina para posteriormente buscar el identificador del puerto asociado a dicha dirección, para finalmente, en una tercera instrucción, deshabilitar dicha seguridad. Para ello, haremos uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ for i in admin osd{1..3} mon{1..3} client{1..2} dns
> do
> ip=`openstack server list | egrep $i | egrep -o "10.0.0.[0-9]{1,3}"`
> port=`openstack port list | egrep $ip | awk '{ print $2 }'`
> openstack port set --disable-port-security $port
> done
{% endhighlight %}

Listo, ya hemos deshabilitado la seguridad en los mismos, de manera que ya tenemos accesible todo el rango de puertos sin limitación alguna y podemos asociar nuevas direcciones IP a las interfaces de red en caso de así necesitarlo. Es importante mencionar que deshabilitar el cortafuegos es una acción un tanto precipitada, por lo que únicamente debe llevarse a cabo en un entorno controlado.

Ya hemos realizado todas las acciones necesarias en el cliente de OpenStack, así que llega el momento de conectarnos a las 10 instancias y configurar una por una... o quizás no, ya que existen **herramientas de orquestación** como **Ansible** que automatiza el aprovisionamiento de software, la gestión de configuraciones y el despliegue de aplicaciones. Su ámbito se puede limitar a un único servidor o a una granja de ellos.

En otras palabras, **Ansible** permite a los _DevOps_ gestionar sus servidores, configuraciones y aplicaciones de forma sencilla, robusta y paralela, ahorrando tiempo y esfuerzo. En caso de que la ejecución falle en uno de los servidores, se seguirá ejecutando de forma paralela sobre el resto.

No debe importar las veces que ejecutemos **Ansible** ya que el resultado de una ejecución repetida debe dejar nuestro escenario en el mismo punto que una ejecución única (idempotencia).

La gestión de los diferentes nodos se lleva a cabo utilizando **SSH** y únicamente requiere **Python** en el servidor remoto en el que se vaya a ejecutar para poder utilizarlo, paquete que generalmente suele venir instalado por defecto.

A pesar de que este proyecto no está enfocado a enseñar a utilizar **Ansible**, sí considero que ha sido una herramienta de gran utilidad y que por tanto, es necesario mencionar aquellas características indispensables para poder aplicarlo a nuestro escenario, empezando por la instalación del mismo en la máquina controladora, la cuál es recomendable llevar a cabo sobre un entorno virtual Python.

En esta ocasión, el nombre que le voy a asignar al nuevo entorno virtual es **ansible**, así que lo generaré dentro de mi directorio donde almaceno todos los entornos virtuales, ejecutando para ello sobre una nueva terminal el comando:

{% highlight shell %}
alvaro@debian:~$ python3 -m venv virtualenv/ansible
{% endhighlight %}

Una vez creado, tendremos que iniciarlo haciendo uso de `source` con el binario **activate** que se encuentra contenido en el directorio **bin/**:

{% highlight shell %}
alvaro@debian:~$ source virtualenv/ansible/bin/activate
{% endhighlight %}

Una vez activado, ya podremos proceder a instalar dicha herramienta haciendo uso de `pip`, no sin antes actualizar dicho paquete en sí mismo, haciendo para ello uso del comando:

{% highlight shell %}
(ansible) alvaro@debian:~$ pip install --upgrade pip
{% endhighlight %}

Una vez actualizado el gestor de paquetes, podremos instalar el paquete en cuestión de nombre **ansible**, ejecutando para ello el comando:

{% highlight shell %}
(ansible) alvaro@debian:~$ pip install ansible
{% endhighlight %}

Nuestro nuevo entorno virtual se encuentra ahora equipado para hacer uso de la herramienta **Ansible**, sin embargo, dadas las características del proyecto que pretendemos llevar a cabo, no nos será suficiente con las funcionalidades predeterminadas que incluye, de manera que necesitaremos instalar nuevas **colecciones** para así ampliar sus funciones. En este caso, las colecciones a instalar son **ansible.posix** y **community.mysql**, de manera que haremos uso de los comandos:

{% highlight shell %}
(ansible) alvaro@debian:~$ ansible-galaxy collection install ansible.posix
(ansible) alvaro@debian:~$ ansible-galaxy collection install community.mysql
{% endhighlight %}

Todo está listo para llevar a cabo la clonación del repositorio de **GitHub** que contiene el proyecto **Ansible** que utilizaremos para configurar de forma automática nuestro escenario compuesto por las 10 máquinas previamente mostradas, de manera que en mi caso me he movido al directorio **GitHub/** (haciendo uso de `cd`), pues es donde almaceno todos los repositorios de GitHub, y por consecuencia, el que voy a [clonar](https://github.com/alvarovaca/ansible-tfg), ejecutando para ello el comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub$ git clone git@github.com:alvarovaca/ansible-tfg.git
Clonando en 'ansible-tfg'...
remote: Enumerating objects: 40, done.
remote: Counting objects: 100% (40/40), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 40 (delta 0), reused 37 (delta 0), pack-reused 0
Recibiendo objetos: 100% (40/40), 9.32 KiB | 9.32 MiB/s, listo.
{% endhighlight %}

Una vez completada la clonación, me he movido al directorio **ansible-tfg/** resultante de dicha clonación (haciendo uso de `cd`), de manera que procederé ahora a listar el contenido de dicho directorio de forma recursiva y arborescente, para así poder apreciar de una forma mucho más gráfica de lo que se compone nuestro proyecto **Ansible**, haciendo para ello uso del comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ tree
.
├── ansible.cfg
├── group_vars
│   └── all
├── hosts
├── post.yml
├── pre.yml
├── README.md
└── roles
    ├── ceph
    │   ├── files
    │   │   ├── ansible_rsa
    │   │   └── ansible_rsa.pub
    │   └── tasks
    │       └── main.yml
    ├── common
    │   ├── handlers
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── dns
    │   ├── files
    │   │   ├── named.conf.local
    │   │   └── named.conf.options
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── db.example.com.j2
    └── pacemaker
        ├── files
        │   └── prestashop.conf
        ├── handlers
        │   └── main.yml
        └── tasks
            └── main.yml

17 directories, 19 files
{% endhighlight %}

Como se puede apreciar, existen varios directorios que contienen a su vez ficheros, pero no hay de qué preocuparse, ya que los cambios a realizar sobre dicho proyecto para poder adaptarlos a cada caso son prácticamente mínimos y no requieren ningún tipo de conocimientos sobre **Ansible**.

Existen diferentes maneras de decirle a **Ansible** qué servidores debe gestionar. La más fácil es añadir nuestras máquinas a un inventario de nombre **hosts** separadas por grupos, dependiendo del ámbito a la que estén dirigidas (servidores web, bases de datos...).

La separación en grupos no es obligatoria pero sí recomendable, pudiendo pertenecer un mismo nodo a varios grupos, para así poder disponer de una mayor granularidad. En mi caso, el resultado final sería el siguiente (sustituir las direcciones IP por las correspondientes alcanzables):

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ cat hosts

[admin]
admin1 ansible_host=172.22.200.103

[osd]
osd1 ansible_host=172.22.200.106
osd2 ansible_host=172.22.200.108
osd3 ansible_host=172.22.200.184

[mon]
mon1 ansible_host=172.22.201.59
mon2 ansible_host=172.22.201.115
mon3 ansible_host=172.22.201.117

[client]
client1 ansible_host=172.22.201.127
client2 ansible_host=172.22.201.128

[dns]
dns1 ansible_host=172.22.201.131
{% endhighlight %}

Una buena práctica en **Ansible** es la de no almacenar las variables en el fichero principal de inventario, ya que puede llegar a afectar notablemente en la legibilidad del mismo.

Por dicha razón, es recomendable generar un directorio de nombre **group_vars** en el que se incluyan todas las variables que apliquen a determinados grupos pertenecientes al inventario, que posteriormente podrán utilizarse dentro de la ejecución de **plays** (conjunto ordenado de tareas que se ejecutan).

En este caso, he generado un fichero de nombre **all** dentro del directorio previamente mencionado para que así dichas variables sean visibles para todos los nodos del proyecto, existiendo las siguientes variables:

__*_ip__: Indica la dirección IP privada de cada uno de los nodos pertenecientes al clúster, ya que ellos no utilizarán la dirección IP "pública" para comunicarse entre ellos. Es necesario cambiarlas para adaptarlas a cada caso.
__*_virt_ip__: Indica la dirección IP virtual que será asignada a cada uno de los recursos que se configurán en alta disponibilidad. Puede modificarse pero no es necesario.
**hacluster_crypt_pass**: Indica la contraseña encriptada para el usuario **hacluster** que posteriormente tendremos que configurar. Puede modificarse pero no es necesario.
**hacluster_plain_pass**: Indica la contraseña en texto plano para el usuario **hacluster** que posteriormente tendremos que configurar. Puede modificarse pero no es necesario.

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ cat group_vars/all

admin_ip: 10.0.0.3
osd1_ip: 10.0.0.13
osd2_ip: 10.0.0.15
osd3_ip: 10.0.0.10
mon1_ip: 10.0.0.11
mon2_ip: 10.0.0.5
mon3_ip: 10.0.0.14
client1_ip: 10.0.0.9
client2_ip: 10.0.0.19
dns_ip: 10.0.0.17

apache_virt_ip: 10.0.0.200
mariadb_virt_ip: 10.0.0.201

hacluster_crypt_pass: $6$vs272OD3toORiva2$SNDSrdhEDPfWo28ZoTmrC21NWVxoueRfuYbatNGprsv.PY6KpOQEV/zYYsfiwGTWkCCVogEp3WSgFnA0I3b5J/
hacluster_plain_pass: hacluster
{% endhighlight %}

En el fichero de nombre **ansible.cfg** podemos incluir parámetros de configuración comunes a todo el proyecto, como por ejemplo indicar el nombre del inventario, el usuario remoto que se utilizará en las máquinas a las que vamos a conectarnos, la ruta a la clave privada para la conexión SSH (es necesario modificarla)...

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ cat ansible.cfg

[defaults]
inventory = hosts
remote_user = debian
host_key_checking = False
deprecation_warnings = False
private_key_file = /home/alvaro/.ssh/linux.pem
ansible_python_interpreter = /usr/bin/python3
{% endhighlight %}

Los proyectos **Ansible** suelen organizarse en forma de **roles** en los que se definen una serie de tareas ordenadas (**play**) a las que se le asigna posteriormente un nombre identificativo al conjunto en sí. Por último, se genera un **playbook** en el que se indicará la correspondencia entre los **roles** y las máquinas o grupos de máquinas del inventario a los que hay que aplicar dichas tareas.

En este caso, disponemos de un **playbook** inicial de nombre **pre.yml** que hace uso de un total de 3 roles previamente definidos, aplicando las siguientes tareas sobre los siguientes grupos de máquinas:

- **all**:
    - **Ensure apt does not use debian replica**: Elimina cualquier repositorio de **debian.org** del fichero **/etc/apt/sources.list**.
    - **Ensure apt uses cica replica**: Añade los repositorios necesarios de **cica.es** al fichero **/etc/apt/sources.list** para evitar problemas con el proxy.
    - **Ensure system is updated**: Actualiza la paquetería instalada.
    - **Set timezone to Europe/Madrid**: Establece la zona horaria a Europa/Madrid.
    - **Ensure hosts file is not managed by cloud_init**: Evita que _cloud init_ gestione el fichero **/etc/hosts**, para evitar cambios tras un reinicio.
    - **Ensure hosts file does not resolve hostname**: Elimina la línea referente a la resolución de su _hostname_ en el fichero **/etc/hosts**, para así delegar dicha tarea al servidor DNS.
    - **Add DNS server to resolvconf configuration**: Añade la dirección IP del servidor DNS existente en el clúster al fichero **/etc/resolvconf/resolv.conf.d/head**.
    - **Add search pattern to resolvconf configuration**: Añade la cadena **search example.com** al fichero **/etc/resolvconf/resolv.conf.d/base**.
    - **restart resolvconf**: Reinicia el servicio **resolvconf**, generando así un fichero **/etc/resolv.conf** acorde a las configuraciones previamente realizadas. Las peticiones DNS irán a partir de ahora al servidor DNS existente en el clúster, pudiendo obviar el nombre de dominio **example.com**.
- **dns**:
    - **Ensure bind9 is installed**: Instala el paquete **bind9**.
    - **Copy named.conf.options file**: Copia el fichero **named.conf.options** a **/etc/bind/**.
    - **Copy named.conf.local file**: Copia el fichero **named.conf.local** a **/etc/bind/**.
    - **Copy db.example.com zone using template**: Utiliza la plantilla **db.example.com.j2** para generar un fichero de nombre **/var/cache/bind/db.example.com**, utilizando para ello las variables de las direcciones IP privadas.
    - **restart bind9**: Reinicia el servicio **bind9**, surtiendo así efecto las configuraciones previamente realizadas.
- **admin, osd, mon, client**:
    - **Create cephuser user**: Crea un usuario **cephuser** con un directorio en **/home** asociado, que necesitaremos posteriormente para la creación del clúster.
    - **Allow cephuser to execute sudo commands without password**: Añade una línea al fichero **/etc/sudoers** para permitir al usuario **cephuser** ejecutar comandos con sudo sin contraseña.
    - **Create .ssh structure**: Crea el directorio **.ssh** en el directorio del usuario **cephuser**.
    - **Copy ansible_rsa private key**: Copia una clave privada previamente generada a la ruta **.ssh/id_rsa**.
    - **Copy ansible_rsa public key**: Copia una clave pública previamente generada a la ruta **.ssh/id_rsa.pub**.
    - **Copy authorized_keys file**: Copia la clave pública previamente copiada a un fichero de nombre **.ssh/authorized_keys**, de manera que todas las máquinas podrán conectarse entre sí.
    - **Check if known_hosts file exists**: Comprueba si el fichero **.ssh/known_hosts** existe, para que en caso de que no, realizar la siguiente tarea.
    - **Scan and add SSH keys of the machines to known_hosts file**: Genera el fichero **.ssh/known_hosts** y escanea y añade las claves SSH del resto de máquinas al mismo, para así evitar tener que introducir confirmación la primera vez.
    - **Change known_hosts file owner**: Establece correctamente el propietario y grupo del fichero **.ssh/known_hosts**.
    - **Add Ceph apt key**: Añade la clave apt de **Ceph**, necesaria para descargar paquetería de sus repositorios y verificar su integridad.
    - **Add Ceph repository**: Añade el repositorio de **Ceph**, del que posteriormente descargaremos la paquetería necesaria.
    - **Ensure needed packages are installed**: Instala la paquetería necesaria para el correcto funcionamiento.

Por último, procederemos a ejecutar dicho **playbook** para así llevar a cabo las tareas previamente mencionadas sobre las máquinas, ejecutando para ello el comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ ansible-playbook pre.yml 

PLAY [all] ***************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************
ok: [mon1]
ok: [admin1]
ok: [osd1]
ok: [osd3]
ok: [osd2]
ok: [mon2]
ok: [mon3]
ok: [dns1]
ok: [client2]
ok: [client1]

...

PLAY RECAP ***************************************************************************************************************************************************************************
admin1                     : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
client1                    : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
client2                    : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dns1                       : ok=16   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
mon1                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
mon2                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
mon3                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
osd1                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
osd2                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
osd3                       : ok=23   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
{% endhighlight %}

Tras alrededor de 30 minutos, todas las tareas se han completado correctamente, tal y como podemos apreciar en el resumen mostrado al final de la ejecución del mismo.

Dado el tamaño de la salida por pantalla resultante de dicha ejecución, he decidido recortarla para no ensuciar demasiado. No obstante, es posible encontrar [aquí](https://pastebin.com/8nz4Xk9w) la salida de dicho comando al completo.

Tras ello, todo estaría listo para continuar con la configuración del escenario, sin embargo, entre las tareas realizadas en todas las máquinas se encuentra una referente a la actualización de la paquetería instalada, encontrándose en dicha paquetería la versión del último núcleo disponible, que no se cargará en memoria hasta que ocurra un reinicio.

Para producir dicho reinicio, volveremos a nuestra terminal con el entorno virtual del cliente OpenStack y ejecutaremos la siguiente instrucción iterativa, que reiniciará todas las máquinas existentes:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ for i in admin osd{1..3} mon{1..3} client{1..2} dns
> do
> openstack server reboot ceph-$i
> done
{% endhighlight %}

Una vez ejecutada la instrucción, tendremos que esperar unos segundos a que todas las máquinas terminen de arrancar para seguidamente realizar un par de gestiones que nos serán necesarias durante el desarrollo del artículo. La primera de ellas consiste en listar las redes existentes en nuestro proyecto, haciendo para ello uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack network list
+--------------------------------------+--------------------+----------------------------------------------------------------------------+
| ID                                   | Name               | Subnets                                                                    |
+--------------------------------------+--------------------+----------------------------------------------------------------------------+
| 2526bd3a-3c01-40b3-b37d-605d99adb980 | red de alvaro.vaca | 747952ea-a0a0-4bd5-8ad6-8dec0666a299                                       |
| 49812d85-8e7a-4c31-baa2-d427692f6568 | ext-net            | 158bbe3e-3c98-485e-8042-ba6402111ea6, 6218710b-aa05-46f7-b198-7639efe3da95 |
+--------------------------------------+--------------------+----------------------------------------------------------------------------+
{% endhighlight %}

Como se puede apreciar, en mi proyecto existen un total de 2 redes, sin embargo, la primera de ellas debe contener a su vez una subred con el direccionamiento 10.0.0.0/24, así que procederemos a listar las subredes creadas ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack subnet list
+--------------------------------------+------+--------------------------------------+-------------+
| ID                                   | Name | Network                              | Subnet      |
+--------------------------------------+------+--------------------------------------+-------------+
| 747952ea-a0a0-4bd5-8ad6-8dec0666a299 |      | 2526bd3a-3c01-40b3-b37d-605d99adb980 | 10.0.0.0/24 |
+--------------------------------------+------+--------------------------------------+-------------+
{% endhighlight %}

Efectivamente, la subred se encuentra generada. Es normal que todavía no se entienda qué es lo que pretendo con todo esto, sin embargo, lo iremos viendo con mayor detenimiento durante el desarrollo de este _post_.

Por último, vamos a generar una IP flotante dentro del rango enrutable de forma directa desde nuestra máquina. Dicha dirección IP deberá generarse en la red externa, que como podemos apreciar en el comando superior, tiene asociado el identificador **49812d85-8e7a-4c31-baa2-d427692f6568**, de manera que haremos uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack floating ip create 49812d85-8e7a-4c31-baa2-d427692f6568
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2021-06-10T09:26:30Z                 |
| description         |                                      |
| dns_domain          | None                                 |
| dns_name            | None                                 |
| fixed_ip_address    | None                                 |
| floating_ip_address | 172.22.200.28                        |
| floating_network_id | 49812d85-8e7a-4c31-baa2-d427692f6568 |
| id                  | aeffdcb1-1c4d-4471-b21e-fb606ceb30de |
| name                | 172.22.200.28                        |
| port_details        | None                                 |
| port_id             | None                                 |
| project_id          | dab5156c32654875b6c54ce23c6712a2     |
| qos_policy_id       | None                                 |
| revision_number     | 1                                    |
| router_id           | None                                 |
| status              | DOWN                                 |
| subnet_id           | None                                 |
| tags                | []                                   |
| updated_at          | 2021-06-10T09:26:30Z                 |
+---------------------+--------------------------------------+
{% endhighlight %}

Como se puede apreciar, la ejecución del comando ha resultado en la efectiva creación de una IP flotante, concretamente la **172.22.200.28**, sin embargo, vamos a proceder a verificarlo listando para ello todas las direcciones IP flotantes asociadas a nuestro proyecto, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ openstack floating ip list
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port                                 | Floating Network                     | Project                          |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| 0805e916-61e4-4c1c-bd82-276c2f463b6e | 172.22.201.128      | 10.0.0.19        | ad391091-7078-49c9-8032-9f06059c3534 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 360c3f8c-3677-410d-b994-992d1e1c7edd | 172.22.200.106      | 10.0.0.13        | f9d30785-3b39-439d-a906-1a33ba8b6965 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 3d54a154-9bbe-4ea3-bdf5-d447cbf9fa20 | 172.22.201.115      | 10.0.0.5         | 92a26bd0-85a3-4bec-9a23-b1bfa04b7dee | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 622afc06-92ed-4862-bfbb-531b1a0b9984 | 172.22.201.131      | 10.0.0.17        | 95eace1b-8732-416c-8c7b-f4902fb140f6 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 63fc1fc7-c73d-4eb7-905e-e017fe1daa49 | 172.22.201.127      | 10.0.0.9         | 69b86694-a9cd-47d6-bf4f-250dcee41838 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 6d48bab1-5d45-49ac-95c8-1475d2f00e79 | 172.22.201.117      | 10.0.0.14        | d308a511-871f-496d-9944-107ecdab57fb | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 6f45b92d-4130-4090-969d-f29eb761eb04 | 172.22.200.103      | 10.0.0.3         | b9db108e-c418-416a-826d-309510ddeb4f | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| 83ba8bca-f18c-4f9a-8d0d-eb710ca07c7e | 172.22.200.184      | 10.0.0.10        | c93827ca-a219-4dcd-898a-7b5065c27c63 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| aeffdcb1-1c4d-4471-b21e-fb606ceb30de | 172.22.200.28       | None             | None                                 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| f51c0df9-75ec-4d54-8af1-49f8972f49fb | 172.22.201.59       | 10.0.0.11        | 54f10e76-4470-4909-acc1-02376a3b09f2 | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
| ffcfab43-9023-41bb-be96-63be788bfd51 | 172.22.200.108      | 10.0.0.15        | d967ec3f-6c62-491d-9f1a-0d6c94921c4c | 49812d85-8e7a-4c31-baa2-d427692f6568 | dab5156c32654875b6c54ce23c6712a2 |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
{% endhighlight %}

Como era de esperar, la IP flotante ha sido correctamente asociada a nuestro proyecto y actualmente se encuentra disponible para asociar a una dirección IP del rango de la red interna.

Tras ello, estableceremos una conexión SSH con la máquina de nombre **ceph-admin** que utilizaremos para realizar la mayor parte de las gestiones restantes. Para ello, haremos uso del comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ ssh debian@172.22.200.103 -i /home/alvaro/.ssh/linux.pem
Linux ceph-admin 4.19.0-16-cloud-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

La conexión SSH se ha establecido correctamente, sin embargo, antes de proceder, considero necesario explicar qué es lo que viene a continuación.

Tal y como previamente hemos mencionado, necesitamos un sistema de ficheros distribuido para desplegar en alta disponibilidad cualquier aplicación que requiera compartir el almacenamiento entre los diferentes nodos sobre los que se ejecutará la misma, como es este caso.

La solución _open source_ de la que haremos uso es conocida como **Ceph**, que es basada en objetos binarios y evita en todo momento las rígidas jerarquías de los sistemas convencionales. Para el almacenamiento utilizamos discos duros, como es lógico, sin embargo, un algoritmo se encarga de gestionar los objetos binarios, que están divididos en numerosas partes y repartidos entre muchos servidores (nunca almacenando más de una copia en el mismo nodo), pero vuelven luego a unificarse.

Al tener una aplicación de creciente tamaño, no sabemos cuál será la cantidad de datos a gestionar en un futuro, por tanto, los sistemas deben poder ampliarse fácilmente, sin dejar de funcionar, con servidores adicionales que puedan integrarse sin obstáculos en el sistema de almacenamiento dado, que en todo momento se mostrará al usuario como un sencillo directorio de un sistema de archivos tradicional, sin necesidad de que el mismo conozca absolutamente nada sobre la distribución de los mismos.

Otra de las características indispensables, como se puede suponer, en la redundancia de dichos datos, ya que de nada nos sirve tener los datos distribuidos incluso en diferentes áreas geográficas del planeta si un simple fallo en uno de los discos puede llegar a causarnos la corrupción total o parcial de los datos almacenados. Por ello, el sistema de autogestión y autorrecuperación de Ceph reduce dramáticamente las interrupciones, convirtiéndolo en una excelente elección para las empresas. **Facebook** y **Dropbox** son de las empresas más grandes que utilizan Ceph.

![replicacion](https://i.ibb.co/sJLdH7L/cephreplication.png "Replicación Ceph")

Aunque en este caso no vamos a hacer uso de estas características, es importante mencionar que las aplicaciones pueden acceder a _Ceph Object Storage_ a través de una interfaz **RESTful** que admite las API de **Amazon S3** y **Openstack Swift**.

Entrando un poco más en profundidad, el sistema se mantiene mediante una red de demonios que se ejecutan entre los diferentes nodos constituyentes del clúster:

* **Monitor (MONs)**: Gestionan el estado de cada uno de los nodos en el clúster y vigilan en especial los componentes _manager service_, _object storage service_ y _metadata server_. Para poder asegurar cierta seguridad, se recomienda disponer de al menos tres _monitor nodes_.
* **Manager (MGRs)**: Gestionan el estado de la utilización del espacio, de la carga del sistema y del nivel de utilización de los nodos.
* **OSDs**: También conocidos como _Object Storage Devices_, son los servicios que realmente se encargan de gestionar los archivos: son responsables del almacenamiento, el duplicado y la restauración de los datos. Se recomienda tener al menos tres _OSDs_ en el cluster. Se ejecuta un OSD por cada disco.
* **MDs**: Se encargan de almacenar metadatos como por ejemplo rutas de almacenamiento, sellos de tiempo y nombres de los archivos guardados en CephFS, por motivos de rendimiento.

Lo dicho anteriormente hace que **Ceph** sea capaz de lo mejor, pero también que sea más complejo que otras opciones como **GlusterFS**, su principal competidor.

El componente clave del almacenamiento de datos es un algoritmo llamado **CRUSH** (_Controlled Replication Under Scalable Hashing_). Este algoritmo es capaz de encontrar un OSD con el archivo solicitado gracias a una tabla de asignaciones almacenada en los **MDs**.

La distribución de los archivos en **Ceph** se realiza de forma pseudoaleatoria, calculándose el lugar más adecuado para almacenarlos basándose en criterios definidos por el administrador de la red. Al hacerlo, además, se duplican los archivos y se almacenan en soportes físicos separados.

Por si no era suficiente, con la finalidad de garantizar la seguridad de los datos, al nivel del OSD se utiliza un **_journaling_** (registro de cambios con fecha y hora, dificultando por tanto la corrupción de los datos). Allí se guardan temporalmente los archivos que se pretenden almacenar mientras se espera a que se ubiquen correctamente en todos los OSD previstos.

No todo podía ser bonito, y es que como podemos apreciar, para utilizar este software necesitamos una red relativamente "grande" para así poder albergar de forma redundante los componentes del clúster. De otro lado, Ceph únicamente es compatible con sistemas Linux, aunque estoy seguro que esto último no es un problema para nosotros.

Una vez comprendido de forma superficial el funcionamiento de **Ceph**, vamos a proceder a crear un clúster, así que vamos a aprovechar nuestra conexión SSH existente con la máquina **ceph-admin** que será la que utilizaremos para ello.

El primero de los pasos será cambiarnos al usuario **cephuser**, pues tiene los privilegios necesarios para ejecutar comandos sin contraseña, además de poder realizar conexiones SSH al resto de máquina, mediante el mismo usuario. Para ello, haremos uso del comando:

{% highlight shell %}
debian@ceph-admin:~$ sudo su - cephuser
{% endhighlight %}

Tras ello, nos encontraremos haciendo uso del nuevo usuario generado, así que el siguiente paso consistirá en generar un directorio en el que se almacenarán los ficheros resultantes de dicho clúster. Su nombre no es relevante aunque es recomendable que sea significativo para evitar eliminarlo por error, de manera que en mi caso lo generaré y accederé al mismo ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~$ mkdir cephcluster && cd cephcluster/
{% endhighlight %}

Para el tercero de los pasos haremos uso del binario **ceph-deploy** previamente instalado durante la ejecución del **playbook** de **Ansible**, que nos permitirá llevar a cabo de forma automatizada dicho proceso. Para comenzar, tendremos que indicar el nombre de las máquinas que actuarán como _monitor nodes_, siendo en mi caso **ceph-mon1**, **ceph-mon2** y **ceph-mon3**, pudiendo indicarlo de la forma **ceph-mon{1..3}** para evitar indicar los tres nombres. El comando final del que haré uso sería:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy new ceph-mon{1..3}
{% endhighlight %}

Como consecuencia de la ejecución de dicho comando se habrán generado un total de 3 ficheros en el directorio actual, de manera que procederemos a verificarlo listando para ello el contenido existente gracias a a la ejecución del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ls -l
total 16
-rw-r--r-- 1 cephuser cephuser  237 Jun  9 14:47 ceph.conf
-rw-r--r-- 1 cephuser cephuser 5476 Jun  9 14:47 ceph-deploy-ceph.log
-rw------- 1 cephuser cephuser   73 Jun  9 14:47 ceph.mon.keyring
{% endhighlight %}

De todos los ficheros únicamente nos importa el primero de ellos, de nombre **ceph.conf**, cuya finalidad es la de albergar parámetros de configuración del clúster, pues el segundo (**ceph-deploy-ceph.log**) almacena _logs_ que por ahora no son relevantes y el tercero (**ceph.mon.keyring**) una clave que utilizarán los _monitor nodes_ pertenecientes al clúster para autenticarse, que será distribuida a los mismos de forma totalmente automatizada.

Por defecto, **Ceph** crea un total de 3 réplicas de los datos almacenados (una en cada uno de los OSD), lo que a _grosso modo_ podemos llamar un **RAID-1**. Como consecuencia, el tamaño total disponible y útil sería aproximadamente el de uno de los discos, ya que disponemos de 3 discos de 5 GiB, pero los otros 2 se limitarían a almacenar una copia (es decir, 1/3 del total). Con dicho valor predeterminado, podrían dejar de funcionar un máximo de 2 de 3 OSD, ya que la información seguiría ubicada en aquel nodo funcional.

Por ello, vamos a modificar dicho valor por defecto y vamos a establecer que el número de copias sea 2, pudiendo aprovechar por tanto una mayor cantidad de espacio, pero sacrificando como es lógico, seguridad, pues únicamente podría morir 1 de 3 OSD.

En realidad, no es necesario llevar a cabo ninguna modificación sobre dicho fichero, aunque en mi caso he decidido hacerlo para que así las posteriores pruebas se vean de una forma mucho más clara, de manera que vamos a hacer uso del siguiente comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ echo "osd pool default size = 2" >> ceph.conf
{% endhighlight %}

Para verificar que la línea indicada se ha introducido correctamente dentro del fichero en cuestión, vamos a proceder a visualizar el contenido del mismo, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ cat ceph.conf 
[global]
fsid = 073817c1-bd48-451e-ad7a-2fe769245057
mon_initial_members = ceph-mon1, ceph-mon2, ceph-mon3
mon_host = 10.0.0.11,10.0.0.5,10.0.0.14
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default size = 2
{% endhighlight %}

Efectivamente, la línea especificada ha sido correctamente introducida en el mismo, además de los _monitor nodes_ previamente especificados junto con sus correspondientes direcciones IP.

En un entorno más real, se utilizarían además dos redes diferentes, una **pública** que sería indicada mediante la directiva **public network** que es aquella a través de la cuál los clientes van a hacer uso del sistema de ficheros y una **privada** que sería indicada mediante la directiva **private network** que es aquella a través de la cuál los OSD realizarían las replicaciones de datos, que debe tener un ancho de banda considerable, para evitar cuellos de botella durante las mismas. En este caso, al únicamente existir una red, no es necesario realizar distinción entre las mismas.

El quinto paso para la configuración de nuestro clúster **Ceph** consiste en llevar a cabo la instalación del _software_ en todos los nodos pertenecientes al mismo. En este caso, la instalación la llevaremos a cabo sobre 9 de las 10 máquinas (ya que el DNS no pertenece al clúster como tal), haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy install ceph-admin ceph-mon{1..3} ceph-osd{1..3} ceph-client{1..2}
{% endhighlight %}

La instalación puede llevar unos minutos en completarse, consistiendo el siguiente paso en llevar a cabo todas las gestiones necesarias en los 3 _monitor nodes_ indicados en el momento de la creación del clúster, como por ejemplo establecer el **quorum**, que será el encargado de determinar por mayoría el estado de los nodos (por ello siempre es necesario que exista un número impar de nodos, para evitar empates), ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy mon create-initial
{% endhighlight %}

El séptimo paso refiere a la copia del fichero de configuración y la clave de administración en todos y cada uno de los nodos pertenecientes al clúster, para así poder utilizar el intérprete de comandos de Ceph desde cualquiera de las máquinas y sin necesidad de indicar ningún parámetro adicional. Para ello, haremos uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy admin ceph-admin ceph-mon{1..3} ceph-osd{1..3} ceph-client{1..2}
{% endhighlight %}

Una vez finalizada la copia en todos los nodos, tendremos que ajustar los permisos del fichero **/etc/ceph/ceph.client.admin.keyring** para que así pueda hacer uso del mismo el usuario **cephuser** previamente generado, al cuál estableceremos como propietario del mismo. Comenzaremos por realizar dicha modificación en la máquina actual, **ceph-admin**, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ cephuser@ceph-admin:~/cephcluster$ sudo chown cephuser:cephuser /etc/ceph/ceph.client.admin.keyring
{% endhighlight %}

Tras ello, tendremos que repetir el mismo procedimiento en el resto de máquinas, sin embargo, podemos automatizarlo con una pequeña estructura iterativa, que quedará de la siguiente forma:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ for i in mon{1..3} osd{1..3} client{1..2}
> do
> ssh cephuser@ceph-$i "sudo chown cephuser:cephuser /etc/ceph/ceph.client.admin.keyring"
> done
{% endhighlight %}

Posteriormente llevaremos a cabo el establecimiento del **_manager_**, que en este caso y por recomendación por parte de la documentación de Ceph, serán las mismas máquinas que albergan el demonio _monitor_. Para ello, haremos uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy mgr create ceph-mon{1..3}
{% endhighlight %}

Una vez desplegados los demonios _manager_, es hora de configurar los **OSD** con su correspondiente almacenamiento, así que de forma previa a ello, vamos a verificar que los 3 nodos destinados a dicha finalidad tengan correctamente asociado dicho volumen secundario de 5 GiB, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy disk list ceph-osd{1..3}
...
[ceph-osd1][INFO  ] Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
[ceph-osd1][INFO  ] Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
...
[ceph-osd2][INFO  ] Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
[ceph-osd2][INFO  ] Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
...
[ceph-osd3][INFO  ] Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
[ceph-osd3][INFO  ] Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
{% endhighlight %}

Como era de esperar, en las tres máquinas existe un volumen en **/dev/vdb** de 5 GiB, así que procederemos a implementar la funcionalidad de OSD en dichas máquinas, utilizando para ello la ruta al disco vacío, en los que se crearán las particiones de datos y _journal_, haciendo para ello uso de los comandos:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy osd create --data /dev/vdb ceph-osd1
cephuser@ceph-admin:~/cephcluster$ ceph-deploy osd create --data /dev/vdb ceph-osd2
cephuser@ceph-admin:~/cephcluster$ ceph-deploy osd create --data /dev/vdb ceph-osd3
{% endhighlight %}

Los tres volúmenes han sido correctamente particionados y listos para su uso, sin embargo, todavía nos falta un elemento esencial en nuestro clúster, los **MDs**, que debido a que "únicamente" contamos con 10 máquinas, tendremos que alojar dichos demonios en las mismas máquinas que ejecutan los **OSDs**, aunque en un caso práctico real, sería mucho más óptimo tenerlo lo más distribuido posible. Para ello, ejecutaremos el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy mds create ceph-osd{1..3}
{% endhighlight %}

Una vez finalizada la ejecución del comando, vamos a verificar que los demonios se han generado correctamente en dichas 3 máquinas, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph mds stat
 1 up:standby
{% endhighlight %}

Como se puede apreciar, los 3 demonios han sido correctamente ubicados (aunque todavía se encuentran en estado **standby**, dada la carencia de un sistema de ficheros), teniendo por tanto, todos los servicios necesarios para el correcto funcionamiento del clúster activos y configurados.

Estamos llegando a la recta final, y es que a pesar de tener nuestro clúster creado, todavía no existe ninguna _pool_ en su interior, y por consecuencia, ningún sistema de ficheros.

Para hacer uso de CephFS necesitamos un mínimo de 2 _pools_ para los **MDs**, uno para **datos** y otro para **metadatos**. Dado el frecuente acceso a los mismos, es recomendable que se utilicen discos de baja latencia para aumentar así la eficiencia de lectura y escritura. Para la generación de dichos _pools_, ejecutaremos los siguientes comandos:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph osd pool create cephfs_data 64
cephuser@ceph-admin:~/cephcluster$ ceph osd pool create cephfs_metadata 64
{% endhighlight %}

Una vez finalizada la ejecución de los comandos, vamos a verificar que los _pools_ se han generado correctamente en las correspondientes máquinas, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph osd lspools
1 device_health_metrics
2 cephfs_data
3 cephfs_metadata
{% endhighlight %}

Efectivamente, se han generado 2 nuevos pools con los nombres **cephfs_data** y **cephfs_metadata**, habiendo sido asignados los identificadores **2** y **3**, respectivamente.

El último paso consiste en crear un sistema de ficheros en el que podremos empezar a alojar objetos, que hará uso de los 2 _pools_ creados con anterioridad, por ejemplo, con nombre **cephfs**. El comando a ejecutar sería:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph fs new cephfs cephfs_metadata cephfs_data
new fs with metadata pool 3 and data pool 2
{% endhighlight %}

Una vez finalizada la ejecución del comando, vamos a verificar que el sistema de ficheros se ha generado correctamente en las correspondientes máquinas, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
{% endhighlight %}

Como era de esperar, se ha generado un nuevo sistema de ficheros de nombre **cephfs** que hace uso del _pool_ de datos **cephfs_data** y del _pool_ de metadatos **cephfs_metadata**. A partir de este momento, los **MDs** habrán pasado a estado **active**.

Por último, vamos a verificar el estado general del clúster, para así comprobar que todos los pasos llevados a cabo han sido efectivos y no ha ocurrido ningún problema durante el mismo, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph health
HEALTH_OK
{% endhighlight %}

Efectivamente, el estado general del clúster es correcto (**HEALTH_OK**), sin embargo, la información mostrada es muy limitada, de manera que vamos a profundizar un poco más haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 2m)
    mgr: ceph-admin(active, since 102s)
    osd: 3 osds: 3 up (since 36s), 3 in (since 49s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 15 GiB / 15 GiB avail
    pgs:     1 active+clean
{% endhighlight %}

Como se puede apreciar de una forma más concreta en la salida del comando, contamos con un total de 3 _pools_ y los servicios actualmente activos se encuentran en las siguientes máquinas:

* **mon**: **ceph-mon1**, **ceph-mon2** y **ceph-mon3**
* **mgr**: **ceph-mon1**, **ceph-mon2** y **ceph-mon3**
* **osd**: **ceph-osd1**, **ceph-osd2** y **ceph-osd3**
* **mds**: **ceph-osd1**, **ceph-osd2** y **ceph-osd3**

Las labores en nuestro clúster Ceph han finalizado, encontrándose a partir de ahora totalmente disponible para albergar ficheros en su interior, así que es hora de explicar todo lo referente al despliegue de la aplicación sobre dicho clúster de almacenamiento distribuido, sobre el que utilizaremos también _software_ para asegurar la alta disponibilidad de los servicios relacionados.

Tal y como se mencionó al principio del artículo, vamos a utilizar **Pacemaker** cuya finalidad es la de permitir controlar y coordinar las máquinas del clúster y **Corosync**, cuya finalidad es la de permitir la comunicación de las máquinas pertenecientes al mismo y enviar órdenes a _Pacemaker_.

Para interconectar las máquinas del clúster de Pacemaker, que serán aquellas sobre las que se ejecuten los servicios necesarios para el despliegue de la aplicación y tengan acceso al sistema de ficheros previamente generado, es decir, **ceph-client1** y **ceph-client2**, necesitaremos un usuario de nombre **hacluster** cuya contraseña sea la misma en ambas máquinas, para posibilitar la comunicación entre las mismas.

En total vamos a desplegar un total de 4 recursos:

* **WebVirtualIP**: Dirección IP virtual (**10.0.0.200**) que se asignará de forma dinámica a cualquiera de los dos nodos, dependiendo de su disponibilidad.
* **WebSite**: Encargado de gestionar el demonio de **apache2** y que irá de la mano al recurso anterior, ya que deben estar siempre ejecutándose en la misma máquina.
* **DBVirtualIP**: Dirección IP virtual (**10.0.0.201**) que se asignará de forma dinámica a cualquiera de los dos nodos, dependiendo de su disponibilidad.
* **Database**: Encargado de gestionar el demonio de **mariadb** y que irá de la mano al recurso anterior, ya que deben estar siempre ejecutándose en la misma máquina.

**Pacemaker** intenta gestionar de la mejor manera posible la distribución de los recursos, teniendo en cuenta valores como el uso de recursos de cada uno de los nodos, aunque podemos forzar las colocaciones de los recursos en caso de así necesitarlo, por ejemplo, cuando un nodo tiene más recursos y que otro y preferimos que la ejecución se lleve a cabo sobre el mismo siempre que sea posible.

Quizás puede llegar a ser algo confuso, así que para aclarar un poco las ideas, vamos a ir haciéndolo y entendiéndolo sobre la marcha. Para ello, vamos a volver a nuestra terminal con el entorno virtual **ansible** y vamos a ejecutar el **playbook** de nombre **post.yml**, que hace uso de un rol previamente definido, aplicando las siguientes tareas sobre el siguiente grupo de máquinas:

- **client**:
    - **Ensure needed packages are installed**: Instala la paquetería necesaria para el correcto funcionamiento.
    - **disable apache2**: Para y deshabilita el servicio **apache2**, ya que a partir de ahora será gestionado por _Pacemaker_ y _Corosync_.
    - **disable mariadb**: Para y deshabilita el servicio **mariadb**, ya que a partir de ahora será gestionado por _Pacemaker_ y _Corosync_.
    - **Obtain secret key and save as a variable**: Busca la clave secreta en el fichero **/etc/ceph/ceph.client.admin.keyring** y la almacena en una variable.
    - **Store the secret key in admin.secret file**: Almacena la clave secreta en un fichero de nombre **/home/cephuser/admin.secret** con los permisos apropiados.
    - **Mount CephFS on /ceph and add to fstab**: Monta el sistema de ficheros previamente generado en **/ceph** (generará dicho directorio) y añade una entrada al **/etc/fstab** para su montaje automático durante el arranque, haciendo uso de la clave privada que acabamos de almacenar para la autenticación.
    - **Create the necessary structure on /ceph**: Crea los subdirectorios **/ceph/sql** y **/ceph/prestashop** para almacenar datos de la base de datos y de la aplicación, respectivamente.
    - **Check if /ceph/prestashop folder is empty before proceeding**: Comprueba si el directorio **/ceph/prestashop** está vacío, para en ese caso, realizar las 3 siguientes tareas.
    - **Download and unzip prestashop_1.7.7.0.zip package**: Descarga y descomprime el paquete de PrestaShop 1.7.7.0 en el directorio **/ceph/prestashop**.
    - **Unzip prestashop.zip package**: Descomprime el paquete resultante de la previa descompresión en el mismo directorio, estableciendo correctamente los permisos y propietarios.
    - **Delete unnecessary PrestaShop files**: Elimina los ficheros **Install_PrestaShop.html** y **prestashop.zip** de **/ceph** dada su nula utilidad, y así liberar espacio.
    - **Copy prestashop.conf VirtualHost file**: Copia el fichero **prestashop.conf** a **/etc/apache2/sites-available/**.
    - **Create prestashop.conf VirtualHost symlink**: Crea un enlace simbólico al fichero anterior en **/etc/apache2/sites-enabled/** para así habilitar el VirtualHost.
    - **Delete 000-default.conf VirtualHost symlink**: Elimina el enlace simbólico existente en **/etc/apache2/sites-enabled/000-default.conf** para así deshabilitar el VirtualHost por defecto.
    - **Create rewrite module symlink**: Crea un enlace simbólico al fichero **/etc/apache2/mods-available/rewrite.load** en **/etc/apache2/mods-enabled/** para así habilitar el módulo.
    - **Change MariaDB datadir to /ceph/sql**: Modifica el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf** y cambia el _datadir_ de **/var/lib/mysql** a **/ceph/sql**.
    - **Change MariaDB bind-address to 0.0.0.0**: Modifica el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf** y cambia la _bind-address_ de **127.0.0.1** a **0.0.0.0**.
    - **Check if /ceph/sql folder is empty before proceeding**: Comprueba si el directorio **/ceph/sql** está vacío, para en ese caso, realizar la siguiente tarea.
    - **Install DB prerequisites**: Instala los ficheros necesarios en el directorio **/ceph/sql** para que **mariadb** pueda utilizar dicho directorio como _datadir_.
    - **Set hacluster user password**: Establece la contraseña común para el usuario **hacluster**, utilizando para ello la variable encriptada previamente definida.
    - **Destroy default cluster**: Elimina el clúster de _Pacemaker_ que se genera por defecto.
    - **Add hosts to the new cluster**: Añade a ambos nodos al nuevo clúster de _Pacemaker_, autenticándose mediante el uso de la variable en texto plano previamente definida.
    - **Create, start and enable the new cluster**: Crea, inicia y habilita el nuevo clúster de _Pacemaker_ que acabamos de definir.
    - **Disable STONITH property**: Deshabilita la propiedad STONITH, para evitar conflictos en el quorum al únicamente disponer de dos nodos.
    - **Create VirtualIP resource for Apache2**: Crea el recurso de nombre **WebVirtualIP** previamente mencionado.
    - **Create Apache2 resource**: Crea el recurso de nombre **WebSite** previamente mencionado.
    - **Create Apache2 and VirtualIP colocation**: Crea una colocación para que los recursos **WebVirtualIP** y **WebSite** siempre se ejecuten en el mismo nodo.
    - **Create VirtualIP resource for MariaDB**: Crea el recurso de nombre **DBVirtualIP** previamente mencionado.
    - **Create MariaDB resource**: Crea el recurso de nombre **Database** previamente mencionado.
    - **Create MariaDB and VirtualIP colocation**: Crea una colocación para que los recursos **DBVirtualIP** y **Database** siempre se ejecuten en el mismo nodo.
    - **Create constraint to preferably run all resources on the first node**: Crea una restricción para que preferiblemente se ejecuten todos los recursos en el nodo **ceph-client1**, para así posteriormente poder apreciar los resultados de las pruebas de una forma más clara. No es necesario ni recomendable en un entorno de producción real.
    - **Create a new MariaDB database**: Crea una nueva base de datos en MariaDB de nombre **prestashop**.
    - **Create a new user with privileges in the previous database**: Crea un nuevo usuario de nombre **usuario** y contraseña **usuario**, cuyo acceso estará permitido desde todas las direcciones (**%**) y cuenta con los privilegios suficientes sobre la base de datos previamente generada.

Por último, procederemos a ejecutar dicho **playbook** para así llevar a cabo las tareas previamente mencionadas sobre las máquinas, ejecutando para ello el comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ ansible-playbook post.yml 

PLAY [client] ************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************
ok: [client1]
ok: [client2]

...

PLAY RECAP ***************************************************************************************************************************************************************************
client1                    : ok=34   changed=31   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
client2                    : ok=13   changed=12   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
{% endhighlight %}

Tras alrededor de 15 minutos, todas las tareas se han completado correctamente, tal y como podemos apreciar en el resumen mostrado al final de la ejecución del mismo.

Dado el tamaño de la salida por pantalla resultante de dicha ejecución, he decidido recortarla para no ensuciar demasiado. No obstante, es posible encontrar [aquí](https://pastebin.com/D5KUyugH) la salida de dicho comando al completo.

Nuestra aplicación ya ha sido desplegada correctamente, sin embargo, existe un problema que posiblemente no habíais planteado hasta ahora. La dirección IP virtual a través de la cuál accederemos al servicio web es una IP dentro de un rango interno de mi proyecto de OpenStack, lo que imposibilita su acceso de forma directa, ya que no es una red enrutable.

Sin embargo, existe una posibilidad un tanto "retorcida" que consiste en crear un balanceador de carga en OpenStack con una IP flotante dentro del rango alcanzable y asociarlo a dicha IP interna, de manera que a través de una especie de DNAT, conseguiríamos acceder al servidor web desde el exterior. El inconveniente es que el cliente de línea de comandos de OpenStack que hemos estado utilizando hasta ahora no nos sirve, ya que la versión que se está utilizando no soporta dicha característica, de manera que tendremos que instalar en un nuevo entorno virtual el cliente del [elemento](https://docs.openstack.org/ocata/cli-reference/neutron.html) de OpenStack responsable de la gestión de las redes, **neutron**.

En esta ocasión, el nombre que le voy a asignar al nuevo entorno virtual es **neutronclient**, así que lo generaré dentro de mi directorio donde almaceno todos los entornos virtuales, ejecutando para ello sobre una nueva terminal el comando:

{% highlight shell %}
alvaro@debian:~$ python3 -m venv /home/alvaro/virtualenv/neutronclient
{% endhighlight %}

Una vez creado, tendremos que iniciarlo haciendo uso de `source` con el binario **activate** que se encuentra contenido en el directorio **bin/**:

{% highlight shell %}
alvaro@debian:~$ source virtualenv/neutronclient/bin/activate
{% endhighlight %}

Una vez activado, ya podremos proceder a instalar dicha herramienta haciendo uso de `pip`, no sin antes actualizar dicho paquete en sí mismo, haciendo para ello uso del comando:

{% highlight shell %}
(neutronclient) alvaro@debian:~$ pip install --upgrade pip
{% endhighlight %}

Una vez actualizado el gestor de paquetes, podremos instalar el paquete en cuestión de nombre **neutron**, ejecutando para ello el comando:

{% highlight shell %}
(neutronclient) alvaro@debian:~$ pip install neutron
{% endhighlight %}

Nuestro nuevo entorno virtual se encuentra ahora equipado para hacer uso de la herramienta **neutron**, así que vamos a generar un balanceador de carga asociado a la dirección IP virtual **10.0.0.200** y que como es lógico, pertenezca a la única subnet existente, con identificador **747952ea-a0a0-4bd5-8ad6-8dec0666a299**. Para ello, haremos uso del comando:

{% highlight shell %}
(neutronclient) alvaro@debian:~$ neutron lbaas-loadbalancer-create --vip-address 10.0.0.200 747952ea-a0a0-4bd5-8ad6-8dec0666a299
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Created a new loadbalancer:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| admin_state_up      | True                                 |
| description         |                                      |
| id                  | 155b3587-e06f-4909-b5a0-7f63d3040cd6 |
| listeners           |                                      |
| name                |                                      |
| operating_status    | OFFLINE                              |
| pools               |                                      |
| provider            | haproxy                              |
| provisioning_status | PENDING_CREATE                       |
| tenant_id           | dab5156c32654875b6c54ce23c6712a2     |
| vip_address         | 10.0.0.200                           |
| vip_port_id         | 4581b50a-f2d4-4136-880a-bb2a34ecd730 |
| vip_subnet_id       | 747952ea-a0a0-4bd5-8ad6-8dec0666a299 |
+---------------------+--------------------------------------+
{% endhighlight %}

La creación del balanceador de carga se ha iniciado, sin embargo, todavía no hemos asociado la IP flotante **172.22.200.28** con identificador **aeffdcb1-1c4d-4471-b21e-fb606ceb30de** a dicho balanceador con identificador **4581b50a-f2d4-4136-880a-bb2a34ecd730**, así que procederemos a realizar dicha asociación ejecutando para ello el comando:

{% highlight shell %}
(neutronclient) alvaro@debian:~$ neutron floatingip-associate aeffdcb1-1c4d-4471-b21e-fb606ceb30de 4581b50a-f2d4-4136-880a-bb2a34ecd730
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Associated floating IP aeffdcb1-1c4d-4471-b21e-fb606ceb30de
{% endhighlight %}

La asociación de la IP flotante con el balanceador se ha completado correctamente, pudiendo acceder a partir de ahora desde nuestro navegador a la dirección **172.22.200.28** y pudiendo visualizar por tanto el contenido del servidor web alojado en la dirección **10.0.0.200**.

A pesar de que no es necesario ya que el único VirtualHost habilitado en el servidor web es el de PrestaShop, es recomendable crear una resolución estática en la máquina anfitriona para que así al acceder a **www.example.com**, resuelva dicha dirección a la IP flotante previamente generada. Para ello, haremos uso del comando:

{% highlight shell %}
(ansible) alvaro@debian:~/GitHub/ansible-tfg$ sudo nano /etc/hosts

127.0.0.1       localhost
127.0.1.1       debian
172.22.200.28   www.example.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Ha llegado el momento de la verdad, vamos a abrir el navegador de nuestra máquina anfitriona y vamos a tratar de acceder a **www.example.com**, obteniendo en mi caso el siguiente resultado:

![prestashop1](https://i.ibb.co/hff1BXj/Captura-de-pantalla-de-2021-06-08-13-26-38.png "Instalación PrestaShop")

¡Genial! El hecho de que el instalador de la aplicación se esté mostrando ya es una buena señal, aunque todavía queda una fase crítica: el proceso de instalación. Dicho proceso es totalmente trivial, así que vamos a obviarlo hasta llegar al punto de introducir la información de la base de datos.

En mi caso, la información a introducir es la siguiente:

* **Dirección del servidor de la base de datos**: bd.example.com (también puede introducirse la dirección IP, pero ya que tenemos un registro DNS para ello, vamos a aprovecharlo)
* **Nombre de la base de datos**: prestashop
* **Usuario de la base de datos**: usuario
* **Contraseña de la base de datos**: usuario

Tras ello, presionaremos el botón de **¡Comprobar la conexión con tu base de datos!** para verificar que el acceso a la mismo se produce correctamente, obteniendo el siguiente resultado:

![prestashop2](https://i.ibb.co/8NLc95J/Captura-de-pantalla-de-2021-06-08-13-30-26.png "Instalación PrestaShop")

Finalmente, pulsaremos en **Siguiente** y la instalación de nuestro CMS comenzará, mostrándonos en todo momento una barra de progreso de la siguiente forma:

![prestashop3](https://i.ibb.co/kHS4vvS/Captura-de-pantalla-de-2021-06-08-13-35-52.png "Instalación PrestaShop")

Transcurridos unos minutos, la instalación concluirá, en mi caso, sin ningún tipo de error. Además, se nos mostrarán las credenciales de acceso que previamente hemos introducido en la segunda fase de la instalación:

![prestashop4](https://i.ibb.co/dkmtycb/Captura-de-pantalla-de-2021-06-08-13-46-43.png "Instalación PrestaShop")

Tal y como se muestra en la advertencia, por razones de seguridad es necesario eliminar el directorio **install/**, de manera que volveremos a nuestra terminal para proceder a ello.

Actualmente nos encontramos con una sesión SSH abierta a la máquina **ceph-admin**, sin embargo, dicha máquina no cuenta con acceso al sistema de ficheros CephFS y por tanto, no podríamos llevar a cabo ninguna modificación. La única solución consiste en abrir una segunda conexión SSH a uno de los clientes desde dicha máquina, concretamente al usuario **cephuser**, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ssh cephuser@ceph-client1
Linux ceph-client1 4.19.0-16-cloud-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Posteriormente, ya podremos llevar a cabo la eliminación de dicho directorio, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-client1:~$ sudo rm -r /ceph/prestashop/install/
{% endhighlight %}

El subdirectorio **install/** ha sido correctamente eliminado. De otro lado, existe una segunda práctica que es recomendable durante la instalación de un PrestaShop, consistente en modificar el nombre del directorio que contiene los ficheros de la página de administración, en este caso, del directorio **admin/**, para así evitar posibles ataques.

En este caso, voy a renombrar el subdirectorio **admin/** a **admin966qmwvy7/**, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-client1:~$ sudo mv /ceph/prestashop/admin/ /ceph/prestashop/admin966qmwvy7
{% endhighlight %}

Antes de volver a nuestro navegador web, vamos a listar el contenido del directorio **/ceph/prestashop/** para así verificar que los dos últimos comandos ejecutados han surtido efecto correctamente, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-client1:~$ ls -l /ceph/prestashop/
total 557
drwxr-xr-x  9 www-data www-data     28 Dec  2  2020 admin966qmwvy7
drwxr-xr-x  5 www-data www-data      6 Dec  2  2020 app
-rw-r--r--  1 www-data www-data   1316 Dec  2  2020 autoload.php
drwxr-xr-x  2 www-data www-data      3 Dec  2  2020 bin
drwxr-xr-x  8 www-data www-data      9 Dec  2  2020 cache
drwxr-xr-x 25 www-data www-data    133 Dec  2  2020 classes
-rw-r--r--  1 www-data www-data 365681 Dec  2  2020 composer.lock
drwxr-xr-x  5 www-data www-data     16 Jun  9 15:26 config
drwxr-xr-x  4 www-data www-data      4 Dec  2  2020 controllers
drwxr-xr-x  6 www-data www-data      7 Dec  2  2020 docs
drwxr-xr-x  2 www-data www-data      2 Jun  9 15:26 download
-rw-r--r--  1 www-data www-data   2466 Dec  2  2020 error500.html
-rw-r--r--  1 www-data www-data   4830 Dec  2  2020 images.inc.php
drwxr-xr-x 19 www-data www-data     35 Jun  9 15:26 img
-rw-r--r--  1 www-data www-data   1169 Dec  2  2020 index.php
-rw-r--r--  1 www-data www-data   1256 Dec  2  2020 init.php
-rw-r--r--  1 www-data www-data   5072 Dec  2  2020 INSTALL.txt
drwxr-xr-x  7 www-data www-data     19 Dec  2  2020 js
-rw-r--r--  1 www-data www-data 186018 Dec  2  2020 LICENSES
drwxr-xr-x  3 www-data www-data     97 Dec  2  2020 localization
drwxr-xr-x  5 www-data www-data      5 Jun  9 15:32 mails
drwxr-xr-x 74 www-data www-data     74 Jun  9 15:37 modules
drwxr-xr-x  5 www-data www-data      5 Dec  2  2020 override
drwxr-xr-x  2 www-data www-data     39 Dec  2  2020 pdf
drwxr-xr-x  5 www-data www-data      4 Dec  2  2020 src
drwxr-xr-x  4 www-data www-data      8 Dec  2  2020 themes
drwxr-xr-x  3 www-data www-data      3 Dec  2  2020 tools
drwxr-xr-x  4 www-data www-data      4 Jun  9 15:26 translations
drwxr-xr-x  2 www-data www-data      2 Jun  9 15:26 upload
drwxr-xr-x  5 www-data www-data      6 Dec  2  2020 var
drwxr-xr-x 49 www-data www-data     49 Dec  2  2020 vendor
drwxr-xr-x  2 www-data www-data      2 Dec  2  2020 webservice
{% endhighlight %}

Como se puede apreciar, el directorio **install/** ha sido correctamente eliminado y el directorio **admin/** ha sido renombrado a **admin966qmwvy7/**, tal y como queríamos.

La instalación de nuestro CMS ha finalizado, así que vamos a añadir un artículo de prueba que nos servirá para los _tests_ que aplicaremos a continuación. Para ello, accederemos a la página de administración, ubicada en **www.example.com/admin966qmwvy7**, que nos debería mostrar lo siguiente:

![prestashop5](https://i.ibb.co/gz721x1/Captura-de-pantalla-de-2021-06-09-15-41-42.png "Acceso PrestaShop")

Una vez solicitadas e introducidas las credenciales indicadas durante la instalación de PrestaShop, accederemos a la página de administración, dentro de la cuál accederemos al apartado **Catálogo** en el menú izquierdo y seguidamente pulsaremos en **Productos**.

Como es lógico, nos dirá que no tenemos ningún producto creado (a no ser que hayamos instalado los productos DEMO), así que seguidamente pulsaremos en **Añade tu primer producto** y rellenaremos los campos con la información que consideremos oportuna. En mi caso, el resultado final ha sido:

![prestashop6](https://i.ibb.co/mGCYcPJ/Captura-de-pantalla-de-2021-06-09-15-43-23.png "Creación producto PrestaShop")

Una vez introducida toda la información necesaria, pulsaremos en **Guardar** y volveremos una vez más al apartado **Productos** para verificar que la creación del mismo se ha completado correctamente, pudiendo apreciar lo siguiente:

![prestashop7](https://i.ibb.co/q91G42Q/Captura-de-pantalla-de-2021-06-09-15-43-34.png "Creación producto PrestaShop")

Efectivamente, mi tienda de PrestaShop ahora cuenta con un producto creado y cuya información se habrá guardado en la base de datos y en el sistema de ficheros distribuido.

Por último, pulsaremos en **Ver mi tienda** en la parte superior derecha para así visualizar la tienda junto a su nuevo producto, quedando de la siguiente forma:

![prestashop8](https://i.ibb.co/vHFNnnx/Captura-de-pantalla-de-2021-06-09-15-43-55.png "Creación producto PrestaShop")

Bien, una vez finalizada la creación del artículo en nuestra tienda, es hora de volver a la terminal para proseguir con una serie de pruebas cuya finalidad es demostrar el potencial de la alta disponibilidad, tanto por parte del clúster **Ceph** como del clúster **Pacemaker**.

Lo primero que haremos será visualizar el estado del clúster **Ceph**, para así ver como se ha incrementado el número de objetos almacenados, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-client1:~$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 55m)
    mgr: ceph-admin(active, since 55m)
    mds: 1/1 daemons up
    osd: 3 osds: 3 up (since 54m), 3 in (since 54m)
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 81 pgs
    objects: 38.43k objects, 707 MiB
    usage:   3.7 GiB used, 11 GiB / 15 GiB avail
    pgs:     81 active+clean
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, hay un total de más de 38000 objetos en el sistema de ficheros distribuido, suponiendo un tamaño total de más de **700 MiB**. Sin embargo, al existir redundancia, el espacio ocupado es superior, llegando a más de **3.7 GiB**.

Hasta ahora hemos visualizado en varias ocasiones el estado del clúster Ceph pero no hemos visto ninguna información relacionado al clúster Pacemaker, así que vamos a proceder a ello, haciendo uso del comando:

{% highlight shell %}
cephuser@ceph-client1:~$ sudo pcs status
Cluster name: mycluster
Stack: corosync
Current DC: ceph-client1 (version 2.0.1-9e909a5bdd) - partition with quorum
Last updated: Wed Jun  9 15:45:39 2021
Last change: Wed Jun  9 15:23:26 2021 by root via cibadmin on ceph-client1

2 nodes configured
4 resources configured

Online: [ ceph-client1 ceph-client2 ]

Full list of resources:

 WebVirtualIP	(ocf::heartbeat:IPaddr2):	Started ceph-client1
 WebSite	(ocf::heartbeat:apache):	Started ceph-client1
 DBVirtualIP	(ocf::heartbeat:IPaddr2):	Started ceph-client1
 Database	(ocf::heartbeat:mysql):	Started ceph-client1

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
{% endhighlight %}

Como era de esperar, los 2 nodos pertenecientes al clúster se encuentran actualmente activos (**online**) y el primero de ellos se encuentra ejecutando además los 4 recursos creados, es decir, en el momento que hemos accedido a **www.example.com** o **bd.example.com**, la máquina que ha procesado dichas peticiones ha sido **ceph-client1**.

¿Pero qué ocurriría si el primero de los nodos sufriese un fallo de _hardware_ o incluso un apagón eléctrico? Vamos a comprobarlo, apagando para ello dicho nodo, ejecutando el comando:

{% highlight shell %}
cephuser@ceph-client1:~$ sudo poweroff
Connection to ceph-client1 closed by remote host.
Connection to ceph-client1 closed.
{% endhighlight %}

Una vez realizada la orden de apagado de la máquina, se cerrará la sesión SSH y volveremos a la máquina **ceph-admin**, estableciendo ahora una conexión SSH con el segundo de los nodos, para así realizar las comprobaciones oportunas. Para ello, haremos uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ssh cephuser@ceph-client2
Linux ceph-client2 4.19.0-16-cloud-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Acto seguido vamos a proceder a visualizar de nuevo el estado del clúster de Pacemaker, ya que dicho clúster es el afectado en caso de fallo del nodo **ceph-client1**, pues para el clúster Ceph, dicho nodo es un simple cliente y no supone ninguna variación en su funcionamiento. Para verificar el estado del mismo, ejecutaremos el comando:

{% highlight shell %}
cephuser@ceph-client2:~$ sudo pcs status
Cluster name: mycluster
Stack: corosync
Current DC: ceph-client2 (version 2.0.1-9e909a5bdd) - partition with quorum
Last updated: Wed Jun  9 15:46:37 2021
Last change: Wed Jun  9 15:23:26 2021 by root via cibadmin on ceph-client1

2 nodes configured
4 resources configured

Online: [ ceph-client2 ]
OFFLINE: [ ceph-client1 ]

Full list of resources:

 WebVirtualIP	(ocf::heartbeat:IPaddr2):	Started ceph-client2
 WebSite	(ocf::heartbeat:apache):	Started ceph-client2
 DBVirtualIP	(ocf::heartbeat:IPaddr2):	Started ceph-client2
 Database	(ocf::heartbeat:mysql):	Started ceph-client2

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
{% endhighlight %}

Como se puede apreciar, ahora únicamente hay un nodo activo (**ceph-client2**), pues el otro ha pasado a estado **offline** como consecuencia del apagado.

Además, podemos observar que los 4 recursos han sido migrados al segundo de los nodos, ya que el primero no se encuentra disponible para ello. Como consecuencia, las dos direcciones IP virtuales ahora están ubicadas en el nodo **ceph-client2**, así como el servidor web y el servidor de bases de datos.

**Nota**: En raras ocasiones, _Pacemaker_ no es capaz de levantar alguno de los recursos en el nuevo nodo tras un apagón dado que el nodo que ejecutaba dicho recurso previamente se ha quedado "colgado" y no ha terminado de morir, así que el recurso expira (_timeout_) y se para. Para solventarlo basta con ejecutar el comando `pcs resource cleanup [RECURSO]` y el recurso volverá a intentar levantarse.

La teoría es muy bonita, pero todavía no he visto de forma práctica si nuestra aplicación sigue funcionando tal y como debería, así que vamos a volver a **www.example.com** y vamos a realizar una nueva petición, por ejemplo, intentando acceder al producto previamente generado:

![prestashop9](https://i.ibb.co/hWKw5L8/Captura-de-pantalla-de-2021-06-09-15-48-31.png "Prueba Pacemaker")

Efectivamente, la aplicación sigue funcionando y atendiendo las peticiones de los clientes, sin que estos hayan notado ningún tipo de diferencia, ya que lo único que ha ocurrido es que los recursos se han movido de nodo, sin embargo, la IP desde la que se accedía a los mismos no ha variado.

Como es lógico, al únicamente existir dos nodos en el nodo **Pacemaker**, la caída del segundo de ellos sí provocaría una problema en el correcto funcionamiento de la aplicación, aunque siempre podemos aumentar el tamaño del clúster con nuevos nodos para así curarnos en salud y tener un mayor margen de error.

Ya hemos comprobado el correcto funcionamiento de unos de los clúster, concretamente el que opera a un mayor nivel, pero todavía nos falta por verificar el funcionamiento del clúster **Ceph**, para verificar si nuestro almacenamiento realmente se encuentra replicado y tiene tolerancia a fallos.

Para ello, vamos a salir de la máquina **ceph-client2** y vamos a volver a **ceph-admin** (comando `exit`), desde la cuál vamos a apagar el primero de los OSDs, concretamente el nodo **ceph-osd1**, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ssh cephuser@ceph-osd1 "sudo poweroff"
{% endhighlight %}

Una vez apagado el nodo, podremos verificar el estado del clúster **Ceph** para comprobar si ha ocurrido algún tipo de modificación en el mismo, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_WARN
            1 osds down
            1 host (1 osds) down
            Degraded data redundancy: 23343/77528 objects degraded (30.109%), 49 pgs degraded
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 60m)
    mgr: ceph-admin(active, since 59m)
    mds: 1/1 daemons up
    osd: 3 osds: 2 up (since 23s), 3 in (since 58m)
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 81 pgs
    objects: 38.76k objects, 707 MiB
    usage:   3.0 GiB used, 12 GiB / 15 GiB avail
    pgs:     23343/77528 objects degraded (30.109%)
             49 active+undersized+degraded
             32 active+clean
{% endhighlight %}

Como era de esperar, los _monitor nodes_ han detectado el problema ocurrido en el primero de los OSD, de manera que han decidido por _quorum_ marcarlo como inactivo.

En consecuencia a lo ocurrido, la redundancia de datos se encuentra degradada, aunque la información todavía es accesible, ya que si lo pensamos, tenemos un total de 2 copias de la información distribuidas entre 3 volúmenes, de manera que los otros 2 volúmenes todavía funcionales contienen la suficiente información como para poder acceder a los objetos almacenados en su totalidad.

Para verificarlo, he vuelto al navegador web y he realizado una nueva petición en **www.example.com**, por ejemplo, intentando acceder al gestor de módulos de PrestaShop:

![prestashop10](https://i.ibb.co/wwbVwBf/Captura-de-pantalla-de-2021-06-09-15-57-13.png "Prueba Ceph")

Efectivamente, la aplicación sigue funcionando tal y como debería, aunque un fallo en uno de los dos OSDs restantes supondría una parada de la misma. Vamos a hacer la prueba apagando para ello el segundo OSD, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ssh cephuser@ceph-osd2 "sudo poweroff"
{% endhighlight %}

Una vez apagado el nodo, podremos verificar el estado del clúster **Ceph** para comprobar si ha ocurrido algún tipo de modificación en el mismo, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_WARN
            2 osds down
            2 hosts (2 osds) down
            Reduced data availability: 25 pgs stale
            Degraded data redundancy: 38764/77528 objects degraded (50.000%), 80 pgs degraded, 81 pgs undersized
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 61m)
    mgr: ceph-admin(active, since 61m)
    mds: 1/1 daemons up
    osd: 3 osds: 1 up (since 75s), 3 in (since 60m)
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 81 pgs
    objects: 38.76k objects, 707 MiB
    usage:   3.0 GiB used, 12 GiB / 15 GiB avail
    pgs:     38764/77528 objects degraded (50.000%)
             55 active+undersized+degraded
             25 stale+active+undersized+degraded
             1  active+undersized
{% endhighlight %}

Como era de esperar, los _monitor nodes_ han vuelto a detectar el problema ocurrido en el segundo de los OSD, de manera que han decidido por _quorum_ marcarlo como inactivo, habiendo ahora un total de 2 OSD inactivos.

En consecuencia a lo ocurrido, la redundancia de datos se encuentra degradada, además de ser inaccesible la información, ya que si lo pensamos, tenemos un total de 2 copias de la información distribuidas entre 3 volúmenes, de manera que el único volumen funcional no contiene una copia al completo, imposibilitando por tanto el acceso a dicha información. Si en lugar de configurar el clúster para tener 2 copias lo hubiésemos configurado para que tuviese 3, la información todavía sería accesible.

Antes de proceder con la última de las pruebas, quizás una de las más útiles, vamos a recuperar el estado funcional del clúster Ceph, levantando para ello las máquinas **ceph-osd1** y **ceph-osd2**. Para ello, volveremos a nuestra terminal con el cliente de comandos de OpenStack y haremos uso de los siguientes comandos:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack server start ceph-osd1
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack server start ceph-osd2
{% endhighlight %}

Para mayor tranquilidad, vamos a verificar que el clúster ha vuelto a estar totalmente funcional ejecutando para ello el siguiente comando desde el nodo **ceph-admin**:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph health
HEALTH_OK
{% endhighlight %}

Efectivamente, el clúster vuelve a estar ahora totalmente operativo y por tanto, nuestra aplicación vuelve a estar disponible.

La última prueba consiste en simular un fallo en uno de los volúmenes empleados para nuestro clúster Ceph, algo que podría pasar en una situación cotidiana, y llevar a cabo un reemplazo del mismo, asegurando que la información es correctamente recuperada en el mismo.

Una vez más, volveremos a nuestra terminal con el cliente de comandos de OpenStack y desasociaremos en caliente el disco del nodo **ceph-osd3**, simulando un fallo en el disco duro de uno de los nodos como si de una situación real se tratase, haciendo para ello uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack server remove volume ceph-osd3 ceph-3
{% endhighlight %}

Una vez desasociado, vamos a proceder a generar un nuevo volumen de 5 GiB totalmente vacío de nombre **ceph-recovery**, que extrapolando al caso real, sería el nuevo disco duro que acabamos de comprar y hemos conectado a la máquina. Para ello, ejecutaremos el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack volume create --size 5 ceph-recovery
{% endhighlight %}

Cuando el volumen haya sido creado, únicamente faltará asociarlo a la máquina cuyo volumen previamente asociado ha fallado, haciendo para ello uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack server add volume ceph-osd3 ceph-recovery
{% endhighlight %}

Supuestamente, el volumen creado ya ha sido asociado, pero para verificarlo, vamos a listar todos los volúmenes existentes y sus asociaciones, para así comprobar además que el anterior volumen también ha sido desasociado. Para ello, ejecutaremos el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/GitHub/ansible-tfg$ openstack volume list
+--------------------------------------+---------------+----------------+------+------------------------------------+
| ID                                   | Name          | Status         | Size | Attached to                        |
+--------------------------------------+---------------+----------------+------+------------------------------------+
| 9223fe20-d624-410c-82f0-5a5f838b78c2 | ceph-recovery | in-use         |    5 | Attached to ceph-osd3 on /dev/vdb  |
| 6c59d1cd-54a8-4402-9aec-65819cf217c4 | ceph-3        | available      |    5 |                                    |
| 48aa0185-f772-4e7d-9181-0b48ac6e061e | ceph-2        | in-use         |    5 | Attached to ceph-osd2 on /dev/vdb  |
| cd84226f-ca74-4be8-8538-1f706fab7b2b | ceph-1        | in-use         |    5 | Attached to ceph-osd1 on /dev/vdb  |
+--------------------------------------+---------------+----------------+------+------------------------------------+
{% endhighlight %}

Como se puede apreciar, el volumen de nombre **ceph-3** ha sido correctamente desasociado y se encuentra disponible para su uso, mientras que de otro lado, el volumen **ceph-recovery** ha sido creado y asociado a la máquina **ceph-osd3** sin ningún tipo de problema.

Una vez cambiado el volumen, podremos verificar el estado del clúster **Ceph** para comprobar si ha ocurrido algún tipo de modificación en el mismo, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_WARN
            1 osds down
            1 host (1 osds) down
            Degraded data redundancy: 27264/78152 objects degraded (34.886%), 55 pgs degraded, 56 pgs undersized
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 69m)
    mgr: ceph-admin(active, since 68m)
    mds: 1/1 daemons up
    osd: 3 osds: 2 up (since 68s), 3 in (since 68m)
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 81 pgs
    objects: 39.08k objects, 710 MiB
    usage:   2.1 GiB used, 13 GiB / 15 GiB avail
    pgs:     27264/78152 objects degraded (34.886%)
             55 active+undersized+degraded
             25 active+clean
             1  active+undersized
{% endhighlight %}

Como era de esperar, la redundancia de datos se encuentra degradada, ya que uno de los volúmenes ha "fallado", y por consecuencia, el OSD se marca como inactivo, ya que si recordamos la explicación inicial, el demonio del OSD se encuentra asociado al volumen, no a la máquina en sí.

Entrando un poco más en detalle, otra de las acciones que se han llevado a cabo en un segundo plano ha sido una redistribución gracias al algoritmo CRUSH para que así los clientes tengan una nueva ruta para encontrar los objetos, sin importar la pérdida de dicho volumen.

Para llevar a cabo la sustitución del volumen en el clúster, lo primero que tendremos que hacer será sacar al anterior del mismo, necesitando para ello el identificador, que podremos obtener mediante la ejecución del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph osd tree | grep -i down
 2    hdd  0.00490          osd.2         down   1.00000  1.00000
{% endhighlight %}

En este caso, el identificador del OSD fallido es **osd.2**, de manera que lo eliminaremos del clúster haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph osd destroy osd.2 --yes-i-really-mean-it
destroyed osd.2
{% endhighlight %}

Tal y como podemos apreciar en la salida del comando ejecutado, el OSD ha sido destruido, de manera que todo está listo para insertar el nuevo, no sin antes listar los discos asociados al nodo **ceph-osd3**, para así verificar que reconoce el nuevo volumen de 5 GiB. Para ello, ejecutaremos el comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy disk list ceph-osd3
...
[ceph-osd3][INFO  ] Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
[ceph-osd3][INFO  ] Disk /dev/vdc: 5 GiB, 5368709120 bytes, 10485760 sectors
{% endhighlight %}

Efectivamente, así ha sido. La única diferencia es que en lugar de estar ubicado en **/dev/vdb**, lo está en **/dev/vdc**, pero no tiene mayor importancia, pues bastaría con cambiar dicha letra a la hora de darle formato al disco, haciendo para ello uso del comando:

{% highlight shell %}
cephuser@ceph-admin:~/cephcluster$ ceph-deploy osd create --data /dev/vdc ceph-osd3
{% endhighlight %}

El volumen ha sido correctamente particionado y se debería encontrar listo para su uso, así que vamos a verificar el estado del clúster **Ceph** para comprobar si ha ocurrido algún tipo de modificación en el mismo, ejecutando para ello el comando:

{% highlight shell %}
cephuser@ceph-admin:~$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_WARN
            Degraded data redundancy: 35183/79334 objects degraded (44.348%), 120 pgs degraded, 120 pgs undersized
            1 daemons have recently crashed
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 2h)
    mgr: ceph-admin(active, since 2h)
    mds: 1/1 daemons up
    osd: 4 osds: 3 up (since 80m), 3 in (since 79m); 120 remapped pgs
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 273 pgs
    objects: 39.67k objects, 701 MiB
    usage:   2.4 GiB used, 13 GiB / 15 GiB avail
    pgs:     35183/79334 objects degraded (44.348%)
             3415/79334 objects misplaced (4.305%)
             152 active+clean
             107 active+recovery_wait+undersized+degraded+remapped
             12  active+undersized+degraded+remapped+backfill_wait
             1   active+recovering+undersized+degraded+remapped
             1   active+remapped+backfill_wait
 
  io:
    recovery: 17 KiB/s, 5 objects/s
 
  progress:
    Global Recovery Event (70m)
      [===============.............] (remaining: 56m)
{% endhighlight %}

Como se puede apreciar en la parte inferior de la salida del comando ejecutado, el clúster ha comenzado un proceso de auto-reparación, pues se estará rellenando el nuevo volumen con la parte de información correspondiente, que en este caso duró hasta 2 horas.

Una vez transcurrido un periodo de tiempo considerable, podremos volver a hacer uso del comando anterior para verificar si el proceso ha finalizado:

{% highlight shell %}
debian@ceph-admin:~$ ceph status
  cluster:
    id:     073817c1-bd48-451e-ad7a-2fe769245057
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon1,ceph-mon3 (age 18h)
    mgr: ceph-admin(active, since 18h)
    mds: 1/1 daemons up
    osd: 4 osds: 3 up (since 17h), 3 in (since 17h)
 
  data:
    volumes: 1/1 healthy
    pools:   3 pools, 81 pgs
    objects: 39.67k objects, 701 MiB
    usage:   3.3 GiB used, 12 GiB / 15 GiB avail
    pgs:     81 active+clean
{% endhighlight %}

Efectivamente, tras un buen rato de espera, el proceso de auto-reparación ha finalizado y el disco se encuentra ahora totalmente funcional, tal y como se encontraba antes del cambio.

Una vez más, vamos a hacer una prueba práctica para así demostrar que la teoría es correcta, realizando para ello una nueva petición en **www.example.com**, por ejemplo, intentando acceder al formulario de contacto de PrestaShop:

![prestashop11](https://i.ibb.co/kyGh055/Captura-de-pantalla-de-2021-06-10-09-38-17.png "Prueba Ceph")

Nuestra aplicación se encuentra actualmente operativa y preparada para tolerar prácticamente la mayoría de fallos tanto de _hardware_ como de _software_, proporcionando una gran seguridad y margen de error.

Espero que este artículo haya sido de gran ayuda además de interesante para todo el mundo. Si os habéis entretenido al menos la mitad de lo que lo he hecho yo, ¡me doy por satisfecho!