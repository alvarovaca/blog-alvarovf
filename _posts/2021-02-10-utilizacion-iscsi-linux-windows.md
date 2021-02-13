---
layout: post
title:  "Utilización de iSCSI en Linux y Windows"
banner: "/assets/images/banners/iscsi.jpg"
date:   2021-02-10 10:28:00 +0200
categories: hlc
---
El protocolo **iSCSI** es un protocolo que nos proporciona acceso a dispositivos de bloques sobre la red (TCP/IP). A diferencia de otros protocolos como NFS o Samba, no proporciona ficheros o directorios, sino dispositivos de bloques de forma íntegra, es decir, se conecta un nuevo disco duro y es posible compartir el disco duro en bruto a través de la red, habilitando su uso de forma remota, sin necesidad de crear una tabla de particiones ni introducir un sistema de ficheros en el mismo, pues esa será labor del extremo que actúe como cliente, que tendrá control total sobre dicho dispositivo de bloques.

Otra de las grandes características es que nos permite montar el mismo dispositivo de bloques en varios equipos clientes de forma simultánea (puede llegar a provocar un problema de concurrencia que habría que solventar a otro nivel, pero como tal, está soportado), de una forma mucho más económica que _Fibre Channel_, teniendo en cuenta además que tiene soporte en la mayoría de sistemas operativos.

Dicho protocolo suele utilizarse en redes de almacenamiento, ya que nos proporcionan un aislamiento adecuado para compartir dichos dispositivos de una forma más segura, aunque tal y como veremos a continuación, dicho protocolo nos ofrece sus propios mecanismos de autenticación.

Antes de empezar a hacer uso de forma práctica de dicho protocolo, es necesario conocer una serie de términos:

* **Unidad lógica (LUN)**: Dispositivo de bloques a compartir por el servidor iSCSI (discos duros, particiones, volúmenes lógicos...).
* **Target**: Recurso a compartir desde el servidor, que incluye una o varias **LUN**. Por ejemplo, podemos conectar a la máquina servidora 3 discos duros e incorporarlos dentro de un mismo _target_, al que posteriormente se conectará un cliente mediante una única conexión y podrá hacer uso de dichos discos. Dependiendo del caso, también podrían haberse creado 3 _targets_, uno por cada disco, y distribuirlos de distinta manera.
* **Initiator**: Cliente iSCSI que realiza la conexión.
* **Multipath**: Permite garantizar la disponibilidad del dispositivo de bloques remoto en caso de haber más de una ruta posible entre el _target_ y el _initiator_, pues en caso de perder la conexión mediante una ruta, dicha conexión se mantendría a través de otra. Como es lógico, depende de la infraestructura que tengamos.
* **IQN**: Formato utilizado para la descripción única de los recursos que se comparten. Se suele utilizar una regla para nombrarlos de la forma _iqn.(año)-(mes).(nombre de dominio invertido):(nombre único)_. Por ejemplo, _iqn.2021-02.com.alvarovf:target1_.
* **iSNS**: Protocolo que permite gestionar recursos iSCSI como si fueran _Fibre Channel_.

Una vez comprendido qué es el protocolo **iSCSI** y su funcionamiento de forma superficial, vamos a proceder a aclarar dichos conceptos de una forma más práctica. Para ello, he generado un pequeño escenario compuesto por las siguientes máquinas:

* **iscsis1**: Máquina **Debian Buster** conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.136**. Actuará como servidor y tiene 3 discos duros de 1GB asociados.
* **iscsic1**: Máquina **Debian Buster** conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.137**. Actuará como cliente (_initiator_).
* **windows**: Máquina **Windows 7** conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.138**. Actuará como cliente (_initiator_).

La primera prueba que voy a hacer consistirá en crear un _target_ que contendrá una única unidad lógica (_LUN_), que posteriormente se compartirá con la máquina cliente **iscsic1**.

Me cuento actualmente haciendo uso de la máquina servidora **iscsis1**, de manera que lo primero que haremos será visualizar las interfaces de red existentes en la máquina junto con sus direcciones IP asignadas, haciendo para ello uso del comando `ip a`:

{% highlight shell %}
root@iscsis1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86373sec preferred_lft 86373sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:25:4d:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.136/24 brd 192.168.1.255 scope global dynamic eth1
       valid_lft 86378sec preferred_lft 86378sec
    inet6 fe80::a00:27ff:fe25:4dcc/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

De las tres interfaces mostradas, la única que nos interesa es aquella de nombre **eth1**, que tiene un direccionamiento **192.168.1.136/24**, resultante de estar conectada a mi red doméstica en modo puente (_bridge_). Nos será necesario conocer dicha información para la correspondiente conexión remota.

De otro lado, vamos a proceder a verificar que los 3 discos se han creado y anexado correctamente a la misma, listando los dispositivos de bloques existentes en la máquina gracias al comando:

{% highlight shell %}
root@iscsis1:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 19.8G  0 disk 
├─sda1   8:1    0 18.8G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0 1021M  0 part [SWAP]
sdb      8:16   0    1G  0 disk 
sdc      8:32   0    1G  0 disk 
sdd      8:48   0    1G  0 disk
{% endhighlight %}

Efectivamente, se han anexado correctamente 3 discos de 1GB (_sdb_, _sdc_ y _sdd_).

Sin embargo, como todos bien sabemos, el nombre asignado a los dispositivos de bloques no es algo totalmente identificativo a lo largo del tiempo, pues es posible que su orden o nomenclatura cambie tras un reinicio, cuando por ejemplo añadimos un nuevo dispositivo.

Es por ello que tendremos que identificarlos de forma inequívoca, acudiendo al directorio **/dev/disk/by-id/**, que nos mostrará un identificador único para cada dispositivo de bloques (no confundir con el UUID de los sistemas de fichos, ya que actualmente estamos trabajando a un nivel mucho más inferior). Para ello, vamos a listar el contenido de dicho directorio ejecutando el comando:

{% highlight shell %}
root@iscsis1:~# ls -l /dev/disk/by-id/
total 0
lrwxrwxrwx 1 root root  9 Feb 11 14:02 ata-VBOX_HARDDISK_VB23af71a2-40586852 -> ../../sdc
lrwxrwxrwx 1 root root  9 Feb 11 14:02 ata-VBOX_HARDDISK_VBa2232259-3b0dda66 -> ../../sdd
lrwxrwxrwx 1 root root  9 Feb 11 14:02 ata-VBOX_HARDDISK_VBc912936c-7d03844e -> ../../sdb
lrwxrwxrwx 1 root root  9 Feb 11 14:02 ata-VBOX_HARDDISK_VBce9c25fa-bef74688 -> ../../sda
lrwxrwxrwx 1 root root 10 Feb 11 14:02 ata-VBOX_HARDDISK_VBce9c25fa-bef74688-part1 -> ../../sda1
lrwxrwxrwx 1 root root 10 Feb 11 14:02 ata-VBOX_HARDDISK_VBce9c25fa-bef74688-part2 -> ../../sda2
lrwxrwxrwx 1 root root 10 Feb 11 14:02 ata-VBOX_HARDDISK_VBce9c25fa-bef74688-part5 -> ../../sda5
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, los identificadores únicos para los tres dispositivos de bloques son los siguientes:

* **sdb**: ata-VBOX_HARDDISK_VBc912936c-7d03844e
* **sdc**: ata-VBOX_HARDDISK_VB23af71a2-40586852
* **sdd**: ata-VBOX_HARDDISK_VBa2232259-3b0dda66

Ahora que somos capaces de identificar a cada uno de los discos de forma única, todo está listo para proceder con la creación del primer _target_, que contendrá al disco **sdb** como _LUN_.

Lo primero que haremos, como es lógico, es instalar el paquete que nos permitirá actuar como servidor iSCSI, de nombre **tgt**, no sin antes actualizar la lista de la paquetería disponible. Para ello, haremos uso del comando:

{% highlight shell %}
root@iscsis1:~# apt update && apt install tgt
{% endhighlight %}

Llegados a este punto, tenemos dos posibilidades para llevar a cabo la creación del _target_ en cuestión:

* Generarlo mediante la línea de comandos, de manera que no perdurará tras un reinicio.
* Generarlo mediante ficheros de configuración, de manera que perdurará tras un reinicio.

En mi caso, y conocida cuál es la finalidad del protocolo iSCSI, considero que es mucho más útil mostrar el procedimiento para generar un _target_ persistente, aunque en caso de que no sean esas tus necesidades, [aquí](https://gist.github.com/albertomolina/6c621aee3f80c5e7baf3c111df670cf0) puedes encontrar una hoja de referencia muy útil con los comandos a ejecutar para ello.

Para la generación persistente de dicho _target_, podemos o bien modificar el fichero de configuración principal ubicado en **/etc/tgt/targets.conf** o bien generar un nuevo fichero de extensión **.conf** dentro de **/etc/tgt/conf.d/**. Personalmente, recomiendo esta última opción, ya que permite llevar a cabo una gestión mucho más organizada de todos los _targets_ existentes en la máquina, así que en mi caso, ejecutaré para ello el comando:

{% highlight shell %}
root@iscsis1:~# nano /etc/tgt/conf.d/target1.conf
{% endhighlight %}

En mi caso, el contenido a introducir dentro de dicho fichero es el siguiente:

{% highlight shell %}
<target iqn.2021-02.com.alvarovf:target1>
    driver iscsi
    controller_tid 1
    backing-store /dev/disk/by-id/ata-VBOX_HARDDISK_VBc912936c-7d03844e
</target>
{% endhighlight %}

Donde:

* **target**: Definimos un bloque identificado de forma única gracias al _IQN_ asignado, en el que insertaremos la configuración del _target_ que estamos generando. En este caso, **iqn.2021-02.com.alvarovf:target1**.
* **driver**: Definimos el _driver_ a utilizar por este _target_. En este caso, **iscsi**.
* **controller_tid**: Asignamos un identificador numérico para el controlador. No es necesario hacerlo, ya que por defecto los asigna de manera secuencial. En este caso, **1**.
* **backing-store**: Definimos una nueva _LUN_ a exportar por el _target_, que como previamente hemos mencionado, es recomendable indicar el identificador único de dicho dispositivo de bloques (al fin y al cabo, dicho identificador es un enlace simbólico). En este caso, **/dev/disk/by-id/ata-VBOX_HARDDISK_VBc912936c-7d03844e**.

Nuestro primer _target_ ya ha sido definido, sin embargo, al haberlo hecho en un fichero de configuración, todavía no ha sido cargado en memoria y por tanto, no se encuentra activo. Para ello, tendremos que reiniciar el servicio para que así detecte los nuevos cambios, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsis1:~# systemctl restart tgt
{% endhighlight %}

Una vez que el reinicio haya finalizado, podremos listar todos los _targets_ actualmente existentes ejecutando para ello la instrucción:

{% highlight shell %}
root@iscsis1:~# tgtadm --op show --mode target
Target 1: iqn.2021-02.com.alvarovf:target1
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 1074 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/disk/by-id/ata-VBOX_HARDDISK_VBc912936c-7d03844e
            Backing store flags: 
    Account information:
    ACL information:
        ALL
{% endhighlight %}

Como se puede apreciar, sin entrar en demasiado detalle, se ha definido un _target_ **iqn.2021-02.com.alvarovf:target1** que se encuentra actualmente activo (_ready_) que cuenta con un total de 2 unidades lógicas (_LUN_):

* **LUN 0**: Definida automáticamente y conocida como unidad lógica de control, que contiene las características del _target_.
* **LUN 1**: Definida por nosotros, de tipo disco (_disk_) y con un tamaño total de 1074 MB, siendo la ruta del mismo la previamente indicada.

Al haber definido el _target_ mediante ficheros de configuración, pasa a estar disponible a través de todas las interfaces de red de forma automática (en caso de haberlo hecho desde línea de comandos, sería necesario _"bindearlo"_ de forma manual).

Nuestra labor en la máquina servidora **iscsis1** ha finalizado, de manera que vamos a dar paso a la máquina cliente **iscsic1**, desde la que realizaremos la conexión a dicho _target_.

Lo primero que haremos, como es lógico, es instalar el paquete que nos permitirá actuar como cliente iSCSI, de nombre **open-iscsi**, no sin antes actualizar la lista de la paquetería disponible. Para ello, haremos uso del comando:

{% highlight shell %}
root@iscsic1:~# apt update && apt install open-iscsi
{% endhighlight %}

Dicho proceso de instalación habrá dado lugar a la generación de un nombre para el _initiator_ que permitirá identificar de forma única desde el servidor a dicho cliente, así como utilizar determinados mecanismos de autenticación, como por ejemplo permitir el acceso a un determinado _target_ desde un único cliente (haciendo uso como es lógico de dicho nombre).

Para visualizar dicho identificador del _initiator_ tendremos que visualizar el contenido del fichero **/etc/iscsi/initiatorname.iscsi**, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1993-08.org.debian:01:6c92b3891490
{% endhighlight %}

Dicho identificador es modificable, sin embargo, no es una práctica recomendable, ya que está pensado para ser un nombre único.

El siguiente paso será hacer un _discovery_, que consistirá en conectarse a la máquina servidora en el puerto TCP que utiliza el protocolo por defecto (**3260**) y pedirle una lista de todos los _targets_ disponibles, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# iscsiadm --mode discovery --type sendtargets --portal 192.168.1.136
192.168.1.136:3260,1 iqn.2021-02.com.alvarovf:target1
{% endhighlight %}

**Nota**: Reemplácese la dirección IP del servidor por la correspondiente.

En este caso, como era de esperar, únicamente existe un _target_ disponible, que es el que acabamos de generar, con _IQN_ asociado **iqn.2021-02.com.alvarovf:target1**.

Sin embargo, lo que acabamos de hacer era simplemente para obtener información, pues todavía no nos hemos conectado a dicho _target_ (_login_) para poder hacer uso de forma remota del dispositivo de bloques asociado. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@iscsic1:~# iscsiadm --mode node -T iqn.2021-02.com.alvarovf:target1 --portal 192.168.1.136 --login
Logging in to [iface: default, target: iqn.2021-02.com.alvarovf:target1, portal: 192.168.1.136,3260] (multiple)
Login to [iface: default, target: iqn.2021-02.com.alvarovf:target1, portal: 192.168.1.136,3260] successful.
{% endhighlight %}

**Nota**: Reemplácese el _IQN_ y la dirección IP del servidor por la correspondiente.

En un principio, la conexión al _target_ ha sido exitosa, de manera que el dispositivo de bloques debe estar ahora disponible para su uso íntegro desde la máquina cliente, de manera que vamos a listar una vez más los dispositivos de bloques existentes, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 19.8G  0 disk 
├─sda1   8:1    0 18.8G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0 1021M  0 part [SWAP]
sdb      8:16   0    1G  0 disk
{% endhighlight %}

Efectivamente, un nuevo dispositivo de bloques **sdb** ha aparecido en la máquina, el cuál podremos utilizar según nuestras necesidades como si se tratase de un disco duro local, como por ejemplo, creando un nuevo sistema de ficheros _ext4_ en su interior, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# mkfs.ext4 /dev/sdb
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: 0fb4df4d-5990-414f-acac-5e5c8e9b4585
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
{% endhighlight %}

Una vez que el sistema de ficheros _ext4_ haya sido generado, ya podremos montarlo en un directorio para empezar a hacer uso del mismo, como por ejemplo en **/mnt**, de manera que haremos uso del comando:

{% highlight shell %}
root@iscsic1:~# mount /dev/sdb /mnt/
{% endhighlight %}

Una vez que el sistema de ficheros haya sido montado en **/mnt**, podremos volver a hacer uso de `lsblk` con una opción para que muestre ahora información sobre los sistemas de ficheros ubicados en los correspondientes dispositivos de bloques, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# lsblk -f
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda                                                                     
├─sda1 ext4         983742b1-65a8-49d1-a148-a3865ea09e24   16.2G     7% /
├─sda2                                                                  
└─sda5 swap         04559374-06db-46f1-aa31-e7a4e6ec3286                [SWAP]
sdb    ext4         0fb4df4d-5990-414f-acac-5e5c8e9b4585  906.2M     0% /mnt
{% endhighlight %}

Como se puede apreciar, el dispositivo de bloques **sdb** tiene un sistema de ficheros _ext4_ en su interior, cuyo UUID es **0fb4df4d-5990-414f-acac-5e5c8e9b4585** (nos será necesario más adelante), además de encontrarse actualmente montado en **/mnt**.

Para hacer una pequeña prueba, vamos a generar en su interior un pequeño fichero con un contenido cualquiera, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# echo "Hola." > /mnt/prueba.txt
{% endhighlight %}

Para verificar que el fichero ha sido correctamente generado dentro de **/mnt/**, listaremos el contenido de dicho directorio ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# ls -l /mnt/
total 20
drwx------ 2 root root 16384 Feb 11 14:07 lost+found
-rw-r--r-- 1 root root     6 Feb 11 14:08 prueba.txt
{% endhighlight %}

Efectivamente, así ha sido. Por último, vamos a realizar ahora la operación inversa, desconectándonos de un _target_, para así comprobar que el dispositivo de bloques deja de estar por tanto disponible. Para ello, desmontaremos antes de nada el sistema de ficheros, para no causar problemas en la integridad de los datos, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# umount /mnt
{% endhighlight %}

Una vez que el sistema de ficheros haya sido desmontado, podremos proceder a hacer el _logout_ del _target_ iSCSI, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# iscsiadm --mode node -T iqn.2021-02.com.alvarovf:target1 --portal 192.168.1.136 -u
Logging out of session [sid: 1, target: iqn.2021-02.com.alvarovf:target1, portal: 192.168.1.136,3260]
Logout of [sid: 1, target: iqn.2021-02.com.alvarovf:target1, portal: 192.168.1.136,3260] successful.
{% endhighlight %}

**Nota**: Reemplácese el _IQN_ y la dirección IP del servidor por la correspondiente.

En un principio, la desconexión del _target_ ha sido exitosa, de manera que el dispositivo de bloques debe haber desaparecido de la máquina cliente, de manera que vamos a listar una vez más los dispositivos de bloques existentes, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 19.8G  0 disk 
├─sda1   8:1    0 18.8G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0 1021M  0 part [SWAP]
{% endhighlight %}

Como era de esperar, el dispositivo de bloques **sdb** ha desaparecido de la máquina, ya que actualmente no estamos conectados al _target_ previamente generado.

Si lo pensamos bien, es un protocolo realmente útil y sus aplicaciones son prácticamente infinitas, sin embargo, tener que realizar manualmente el proceso de _login_  y montaje del sistema de ficheros cada vez que se inicie la máquina cliente puede llegar a ser un tanto tedioso. Es por ello que vamos a llevar a cabo la configuración necesaria para automatizar dicho proceso.

El primer paso consistirá en generar un directorio que actúe como punto de montaje para dicho dispositivo de bloques remoto, pues hacerlo de forma persistente dentro de **/mnt** no sigue las recomendaciones de la [Filesystem Hierarchy Standard](https://es.wikipedia.org/wiki/Filesystem_Hierarchy_Standard). En este caso, voy a generar un directorio de nombre **iscsi/** dentro de **/mnt**, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# mkdir /mnt/iscsi
{% endhighlight %}

Una vez generado el punto de montaje para dicho sistema de ficheros, daremos lugar al segundo paso, que consistirá en indicar a **open-iscsi** que realice el _login_ a dicho _target_ de forma automatizada durante el arranque de la máquina cliente, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# iscsiadm --mode node -T iqn.2021-02.com.alvarovf:target1 --portal 192.168.1.136 -o update -n node.startup -v automatic
{% endhighlight %}

**Nota**: Reemplácese el _IQN_ y la dirección IP del servidor por la correspondiente.

Por último, tendremos que crear una unidad **systemd** para gestionar el montaje automático de dicho sistema de ficheros (aunque también podríamos hacerlo en el **/etc/fstab**), que deberá ubicarse dentro de **/etc/systemd/system/** y su nombre deberá ser **[puntomontaje].mount**, sustituyendo las **/** de la ruta por **-**, quedando en mi caso de la siguiente forma:

{% highlight shell %}
root@iscsic1:~# nano /etc/systemd/system/mnt-iscsi.mount
{% endhighlight %}

En mi caso, el contenido a introducir dentro de dicho fichero es el siguiente:

{% highlight shell %}
[Unit]
Description=Prueba iSCSI    

[Mount]
What=/dev/disk/by-uuid/0fb4df4d-5990-414f-acac-5e5c8e9b4585
Where=/mnt/iscsi   
Type=ext4
Options=_netdev

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Donde:

* **Description**: Establecemos una descripción para la unidad **systemd**, que no es algo relevante.
* **What**: Indicamos qué es lo que queremos montar. En este caso, será el sistema de ficheros con el UUID previamente obtenido, pues hacer referencia al mismo mediante el nombre de dispositivo no es algo identificativo a largo plazo.
* **Where**: Indicamos el punto de montaje para el sistema de ficheros en cuestión, que en este caso será el directorio que acabamos de generar.
* **Type**: Indicamos el tipo de sistema de ficheros que deseamos montar.
* **Options**: Indicamos las opciones de montaje para dicho sistema de ficheros, en este caso, es muy importante poner **_netdev** para que no intente montar sistemas de ficheros compartidos por red hasta que ésta no esté disponible y por tanto no ralentice el proceso de arranque.
* **WantedBy**: Indicamos las dependencias de la unidad **systemd**, que en este caso, indica que será necesario que todos los servicios de red hayan arrancado y que el sistema acepte _logins_ por parte de los usuarios, pero no es necesario que la _GUI_ haya sido iniciada.

Una vez finalizada la creación de la unidad, guardaremos los cambios y saldremos para así habilitar la misma para que arranque de forma automática en el siguiente inicio de la máquina, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# systemctl enable mnt-iscsi.mount
{% endhighlight %}

Ha llegado el momento de _la prueba de fuego_, de manera que vamos a proceder a reiniciar la máquina ejecutando para ello el comando `reboot` para así poder verificar que el funcionamiento de dicha unidad es el esperado.

Lo primero que haremos será comprobar el estado de la unidad **systemd** de nombre **mnt-iscsi.mount**, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# systemctl status mnt-iscsi.mount
● mnt-iscsi.mount - Prueba iSCSI
   Loaded: loaded (/etc/systemd/system/mnt-iscsi.mount; enabled; vendor preset: enabled)
   Active: active (mounted) since Thu 2021-02-11 14:12:47 GMT; 20s ago
    Where: /mnt/iscsi
     What: /dev/sdb
    Tasks: 0 (limit: 544)
   Memory: 120.0K
   CGroup: /system.slice/mnt-iscsi.mount
{% endhighlight %}

Como era de esperar, la unidad se encuentra actualmente activa como consecuencia de haber habilitado su arranque durante el inicio de la máquina, de manera que el sistema de ficheros debería haber sido correctamente montado de forma automática en **/mnt/iscsi**, por lo que vamos a comprobarlo listando una vez más los dispositivos de bloques existentes en la máquina cliente, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsic1:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 19.8G  0 disk 
├─sda1   8:1    0 18.8G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0 1021M  0 part [SWAP]
sdb      8:16   0    1G  0 disk /mnt/iscsi
{% endhighlight %}

Efectivamente, se ha hecho un _login_ de forma automática en el _target_ previamente generado en la máquina servidora, además de haberse montado el sistema de ficheros del dispositivo de bloques compartido en el punto de montaje especificado.

De igual forma, vamos a listar el contenido de dicho directorio para verificar que el fichero previamente creado sigue existiendo en el mismo, ejecutando para ello el comando:

{% highlight shell %}
root@iscsic1:~# ls -l /mnt/iscsi/
total 20
drwx------ 2 root root 16384 Feb 11 14:07 lost+found
-rw-r--r-- 1 root root     6 Feb 11 14:08 prueba.txt
{% endhighlight %}

Como se puede apreciar, el contenido sigue siendo exactamente el mismo, por lo que podemos concluir que toda la configuración llevada a cabo hasta ahora ha funcionado tal y como debería.

La segunda prueba que voy a hacer consistirá en crear un nuevo _target_ que contendrá dos unidades lógicas (_LUN_), que posteriormente se compartirá con la máquina cliente **windows** pero utilizando la autenticación CHAP que nos proporciona el protocolo iSCSI.

Me cuento actualmente haciendo uso de la máquina servidora **iscsis1**, de manera que todo está listo para proceder con la creación del segundo _target_, que contendrá a los discos **sdc** y **sdd** como _LUN_.

Lo primero que haremos, como ya sabemos, es generar un nuevo fichero de extensión **.conf** dentro de **/etc/tgt/conf.d/**, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsis1:~# nano /etc/tgt/conf.d/target2.conf
{% endhighlight %}

En mi caso, el contenido a introducir dentro de dicho fichero es el siguiente:

{% highlight shell %}
<target iqn.2021-02.com.alvarovf:target2>
    driver iscsi
    controller_tid 2
    backing-store /dev/disk/by-id/ata-VBOX_HARDDISK_VB23af71a2-40586852
    backing-store /dev/disk/by-id/ata-VBOX_HARDDISK_VBa2232259-3b0dda66
    incominguser alvaro pruebaiscsi2021
</target>
{% endhighlight %}

Donde:

* **incominguser**: Definimos un usuario y una contraseña que tendrá permisos para hacer uso del _target_ (conocido como autenticación **CHAP**). En este caso, existirá un usuario **alvaro** con contraseña **pruebaiscsi2021** que podrá utilizar dicho _target_.

Nuestro segundo _target_ ya ha sido definido, sin embargo, al haberlo hecho en un fichero de configuración, todavía no ha sido cargado en memoria y por tanto, no se encuentra activo. Para ello, tendremos que reiniciar el servicio para que así detecte los nuevos cambios, haciendo para ello uso del comando:

{% highlight shell %}
root@iscsis1:~# systemctl restart tgt
{% endhighlight %}

Una vez que el reinicio haya finalizado, podremos listar todos los _targets_ actualmente existentes ejecutando para ello la instrucción:

{% highlight shell %}
root@iscsis1:~# tgtadm --lld iscsi --op show --mode target
Target 1: iqn.2021-02.com.alvarovf:target1
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
        I_T nexus: 1
            Initiator: iqn.1993-08.org.debian:01:6c92b3891490 alias: iscsic1
            Connection: 0
                IP Address: 192.168.1.137
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 1074 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/disk/by-id/ata-VBOX_HARDDISK_VBc912936c-7d03844e
            Backing store flags: 
    Account information:
    ACL information:
        ALL
Target 2: iqn.2021-02.com.alvarovf:target2
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00020000
            SCSI SN: beaf20
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00020001
            SCSI SN: beaf21
            Size: 1074 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/disk/by-id/ata-VBOX_HARDDISK_VB23af71a2-40586852
            Backing store flags: 
        LUN: 2
            Type: disk
            SCSI ID: IET     00020002
            SCSI SN: beaf22
            Size: 1074 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/disk/by-id/ata-VBOX_HARDDISK_VBa2232259-3b0dda66
            Backing store flags: 
    Account information:
        alvaro
    ACL information:
        ALL
{% endhighlight %}

Como se puede apreciar, sin entrar en demasiado detalle, se ha definido un segundo _target_ **iqn.2021-02.com.alvarovf:target2** que se encuentra actualmente activo (_ready_) que cuenta con un total de 3 unidades lógicas (_LUN_):

* **LUN 0**: Definida automáticamente y conocida como unidad lógica de control, que contiene las características del _target_.
* **LUN 1**: Definida por nosotros, de tipo disco (_disk_) y con un tamaño total de 1074 MB, siendo la ruta del mismo el identificador único del disco **sdc**.
* **LUN 2**: Definida por nosotros, de tipo disco (_disk_) y con un tamaño total de 1074 MB, siendo la ruta del mismo el identificador único del disco **sdd**.

Nuestra labor en la máquina servidora **iscsis1** ha finalizado, de manera que vamos a dar paso a la máquina cliente **windows**, desde la que realizaremos la conexión a dicho _target_.

Lo primero que haremos, como es lógico, es acceder a la aplicación que nos permitirá actuar como cliente iSCSI, de nombre **iSCSI Initiator**. Para ello, haremos una búsqueda de la misma mediante el explorador:

![windows1](https://i.ibb.co/m43rqmw/Captura-de-pantalla-de-2021-02-09-17-46-38.png "iSCSI Windows")

Cuando nos encontremos dentro de la misma, tendremos que hacer un _discovery_, que consistirá en conectarse a la máquina servidora en el puerto TCP que utiliza el protocolo por defecto (**3260**) y pedirle una lista de todos los _targets_ disponibles, accediendo para ello al apartado **Discovery** en el menú superior:

![windows2](https://i.ibb.co/dfQRKSp/Captura-de-pantalla-de-2021-02-09-18-00-15.png "iSCSI Windows")

Dentro de dicho apartado, tendremos que pulsar en **Discover Portal...** para así añadir un nuevo servidor (_portal_) destino, que nos abrirá una ventana de la siguiente forma:

![windows3](https://i.ibb.co/3rtHgjh/Captura-de-pantalla-de-2021-02-09-18-00-34.png "iSCSI Windows")

Como se puede suponer, tendremos que indicar la dirección IP de la máquina servidora **iscsis1**, así como el puerto en el que está escuchando dichas peticiones, que es el utilizado por defecto. Si tras ello volvemos al apartado **Targets**, podremos apreciar lo siguiente:

![windows4](https://i.ibb.co/ww5Hwd7/Captura-de-pantalla-de-2021-02-09-18-02-02.png "iSCSI Windows")

En este caso, como era de esperar, existen un total de dos _targets_ disponibles, que son aquellos que manualmente hemos generado, con _IQN_ asociados **iqn.2021-02.com.alvarovf:target1** y **iqn.2021-02.com.alvarovf:target2**, respectivamente.

Sin embargo, lo que acabamos de hacer era simplemente para obtener información, pues todavía no nos hemos conectado al segundo _target_ (_login_) para poder hacer uso de forma remota de los dispositivos de bloques asociados. Para ello, lo seleccionaremos y pulsaremos en **Connect**:

![windows5](https://i.ibb.co/mB9nZz8/Captura-de-pantalla-de-2021-02-09-18-03-45.png "iSCSI Windows")

Dado que estamos haciendo uso de una autenticación CHAP, no bastará con pulsar en **OK** para realizar la conexión, sino que tendremos que introducir las credenciales del usuario con permisos para ello.

Simplemente, tendremos que pulsar en **Advanced** y se nos abrirá una nueva ventana de la siguiente forma:

![windows6](https://i.ibb.co/VN97DJ8/Captura-de-pantalla-de-2021-02-09-18-08-31.png "iSCSI Windows")

Como se puede apreciar, hemos marcado la casilla **Enable CHAP log on** y hemos especificado las siguientes credenciales en el mismo, que tendrán que coincidir con las asignadas en el momento de la creación del _target_:

* **Name**: alvaro
* **Target secret**: pruebaiscsi2021

Tras ello, pulsaremos en **OK** para realizar la conexión y podremos apreciar lo siguiente:

![windows7](https://i.ibb.co/pZxqJ54/Captura-de-pantalla-de-2021-02-09-18-08-43.png "iSCSI Windows")

En un principio, la conexión al segundo _target_ ha sido exitosa, de manera que los dos dispositivos de bloques deben estar ahora disponibles para su uso íntegro desde la máquina cliente **windows**. Para verificarlo, accederemos a **Create and format hard disk partitions**, haciendo para ello una búsqueda mediante el explorador, en caso de ser necesario:

![windows8](https://i.ibb.co/R4FQMZy/Captura-de-pantalla-de-2021-02-09-18-08-59.png "iSCSI Windows")

Cuando hayamos accedido, nos aparecerá una ventana emergente indicando que se han detectado dos nuevos discos que no se encuentran actualmente inicializados ni tampoco cuentan con tabla de particiones, de manera que la crearemos en su interior pulsando para ello en **OK**:

![windows9](https://i.ibb.co/19p8CRj/Captura-de-pantalla-de-2021-02-09-18-09-28.png "iSCSI Windows")

Los discos ya habrán sido iniciados y tendrán una tabla de particiones en su interior, de manera que simplemente queda alojar un sistema de ficheros _NTFS_ en los mismos. Para ello, haremos click derecho sobre el primero de ellos y pulsaremos en **New Simple Volume...**:

![windows10](https://i.ibb.co/v1S0sWB/Captura-de-pantalla-de-2021-02-09-18-09-49.png "iSCSI Windows")

Tras ello se abrirá una nueva ventana emergente que nos permitirá continuar con el proceso, consistente como suele ser común en Windows, en pulsar en **Siguiente** en reiteradas ocasiones. Repetiremos el mismo procedimiento para el segundo disco, de manera que el resultado final debe ser similar al siguiente:

![windows11](https://i.ibb.co/QCthWPc/Captura-de-pantalla-de-2021-02-09-18-11-11.png "iSCSI Windows")

Efectivamente, los discos ya cuentan con un sistema de ficheros _NTFS_ en su interior, por lo que estarán totalmente operativos y listos para su uso. Para verificarlo de nuevo, accederemos al explorador de Windows y podremos apreciar que las dos nuevas unidades aparecen ahí y podríamos empezar a hacer uso de las mismas desde este momento:

![windows12](https://i.ibb.co/jrvnjCW/Captura-de-pantalla-de-2021-02-09-18-11-31.png "iSCSI Windows")

A mi parecer, el artículo ha sido bastante claro y conciso, mencionando en todo momento las numerosas ventajas que nos ofrece este simple pero útil protocolo para compartir dispositivos de bloques a través de la red, de manera que ya queda ser creativo y aplicarlo en aquellos escenarios en los que pueda ofrecernos ventajas. ¡Un saludo!