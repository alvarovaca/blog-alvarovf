---
layout: post
title:  "Configuración RAID 5"
banner: "/assets/images/banners/raid.jpg"
date:   2020-10-01 13:05:00 +0200
categories: raid seguridad
---
## Tarea 1: Crea un RAID llamado md5 con los discos que hemos conectado a la máquina. ¿Cuántos discos tienes que conectar? ¿Qué diferencia existe entre el RAID 5 y el RAID 1?

El primer paso será instalar la paquetería necesaria en nuestro equipo para poder trabajar con **Vagrant** junto a **Virtualbox**. Dado que Virtualbox no se encuentra actualmente en los repositorios de Debian, tendremos que descargarlo de los repositorios oficiales de Oracle. Para añadir dicho repositorio a nuestra máquina, ejecutaremos el siguiente comando (con permisos de administrador, ejecutando el comando `su -`):

{% highlight shell %}
echo "deb https://download.virtualbox.org/virtualbox/debian buster contrib" > /etc/apt/sources.list.d/oracle-virtualbox.list
{% endhighlight %}

También tendremos que añadir la clave pública GPG de Oracle a nuestro anillo de claves para poder descifrar e instalar el paquete de Virtualbox que vamos a descargar, ejecutando para ello el comando:

{% highlight shell %}
wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | apt-key add -
{% endhighlight %}

Listo. El siguiente paso consiste en realizar la instalación, pero hay una cosa muy importante a tener en cuenta. La versión actual de Vagrant en la rama stable (**2.2.3**) soporta hasta la versión **6.0** de Virtualbox, tal y como se puede ver en el [changelog](https://github.com/hashicorp/vagrant/blob/master/CHANGELOG.md), por lo que no podemos instalar la **6.1**, ya que no la soportaría. Procedemos a la instalación (no sin antes actualizar la paquetería disponible) ejecutando el comando:

{% highlight shell %}
apt-get -y update && apt-get -y install virtualbox-6.0 vagrant
{% endhighlight %}

Durante la instalación, nos preguntará el hipervisor que usaremos, por lo que en este caso, debemos elegir **Virtualbox** (opción 2).

Todo está correctamente configurado, así que lo primero que haremos será crear un directorio que únicamente usaremos para esta infraestructura, donde se alojará el Vagrantfile (contiene la configuración de nuestra estructura). Para ello, ejecutaremos el comando:

{% highlight shell %}
mkdir -p ~/vagrant/raid
{% endhighlight %}

Esto nos generará en nuestro directorio personal un directorio llamado **vagrant**, dentro del cuál almacenaremos todos los directorios de las infraestructuras de Vagrant (para mantener una organización en todo momento), en este caso, la alojaremos dentro del directorio generado dentro del mismo, llamado **raid**. Gracias a la opción **-p** habrá generado el directorio padre también.

El siguiente paso será generar un fichero **Vagrantfile** dentro del directorio **~/vagrant/raid** donde indicaremos la infraestructura que deseamos. Para generar dicho fichero, ejecutaremos el comando:

{% highlight shell %}
touch ~/vagrant/raid/Vagrantfile
{% endhighlight %}

Creo que es necesario aclarar lo referente a los discos antes de empezar a configurar el fichero. En el enunciado se pide hacer un RAID 5, es decir, se necesitan al menos 3 discos para ello, pues la información se almacena alternativamente en dichos discos en forma de bloques, almacenando en el restante la paridad, para poder restablecer la información en caso de pérdida, incrementando así la fiabilidad, permitiendo por tanto, que como máximo se estropease un disco de forma simultánea. Cuando dicho disco estropeado se cambie por uno nuevo, la información se recuperará en tiempo real, sin que el servidor deje de funcionar en ningún momento.

![raid5](https://www.norender.com/wp-content/uploads/raid_5.png "RAID 5")

Como se puede apreciar en este ejemplo, el primer bloque (**A1**) se ha situado en el disco 0, el segundo bloque (**A2**) en el disco 1 y la información de paridad (**Ap**), en el disco 2, de manera que si cualquiera de los discos fallase, gracias a la operación matemática de paridad, podríamos recuperar la información para cualquiera de ellos. Sin embargo, con el tercer y cuarto bloque cambia la localización, pues el tercero (**B1**) se ha situado en el disco 1 y el cuarto (**B2**), en el disco 2, almacenándose la paridad (**Bp**) en el disco 0. El hecho de tener que almacenar la paridad de los bloques hace que se pierda la capacidad de uno de los discos, es decir, si tenemos un RAID 5 de 3 discos de 1GB, en el RAID 5 aparecerá que tiene una capacidad de 2GB. Esto demuestra que se trabaja de forma alternativa con los discos.

A diferencia del RAID 1, que necesita al menos 2 discos y en lugar de almacenar la información en forma de bloques entre todos los discos, lo que hace es guardarla por duplicado (o triplicado, o cuadruplicado... dependiendo del número de discos), permitiendo por tanto, que como máximo se estropease uno de los dos discos (si tuviese 5, podrían estropearse 4, quedando siempre uno de ellos con la información en su interior). Cuando dicho disco estropeado se cambie por uno nuevo, la información se duplicará de nuevo en tiempo real, sin que el servidor deje de funcionar en ningún momento.

![raid1](https://www.doctoresdelpc.com/images/doctoresdelpc/noticias_software/raid1.png "RAID 1")

Como se puede apreciar en este ejemplo, el primer bloque (**A1**) se ha situado tanto en el disco 0 como en el disco 1, de manera que si cualquiera de los discos fallase, todavía quedaría uno funcionando con la información en su interior. El hecho de tener que almacenar la información por duplicado (o triplicado, o cuadruplicado...) hace que se pierda la capacidad de todos los discos excepto de uno, es decir, el espacio utilizable es únicamente el del más pequeño de ellos, por lo que si tenemos un RAID 1 de 3 discos de 1GB, en el RAID 1 aparecerá que tiene una capacidad de 1GB.

Ha quedado claro que el RAID 5 necesita al menos 3 discos para trabajar, pero si seguimos leyendo el enunciado, la _Tarea 8_ y la _Tarea 9_, nos pide usar un total de dos discos más para hacer determinadas tareas, así que vamos a matar _dos pájaros de un tiro_ y vamos a añadir los 5 directamente, para no tener que parar la máquina en ningún momento.

En este caso, voy a modificar el fichero usando **Visual Studio Code**, pero se puede utilizar cualquier otro. Para abrir el fichero con VSCode:

{% highlight shell %}
code ~/vagrant/raid/Vagrantfile
{% endhighlight %}

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

disco1 = '.vagrant/disco1.vdi' #Generamos el fichero .vdi para el primer disco.
disco2 = '.vagrant/disco2.vdi' #Generamos el fichero .vdi para el segundo disco.
disco3 = '.vagrant/disco3.vdi' #Generamos el fichero .vdi para el tercer disco.
disco4 = '.vagrant/disco4.vdi' #Generamos el fichero .vdi para el cuarto disco.
disco5 = '.vagrant/disco5.vdi' #Generamos el fichero .vdi para el quinto disco.

Vagrant.configure("2") do |config|
    config.vm.box = "debian/buster64" #Seleccionamos el box con el que vamos a trabajar, en este caso Debian stable.
    config.vm.hostname = "practicaraid" #Establecemos el nombre (hostname) de la máquina. Considero que "practicaraid" es bastante acertado.
    config.vm.provider :virtualbox do |v|
        v.customize ['createhd', '--filename', disco1, '--size', 1024] #Le damos un tamaño máximo al primer disco creado anteriormente, de un total de 1024 MiB. Esta instrucción es necesario borrarla a partir del segundo arranque, ya que el disco ya habrá sido creado con anterioridad.
        v.customize ['createhd', '--filename', disco2, '--size', 1024] #Le damos un tamaño máximo al segundo disco creado anteriormente, de un total de 1024 MiB. Esta instrucción es necesario borrarla a partir del segundo arranque, ya que el disco ya habrá sido creado con anterioridad.
        v.customize ['createhd', '--filename', disco3, '--size', 1024] #Le damos un tamaño máximo al tercer disco creado anteriormente, de un total de 1024 MiB. Esta instrucción es necesario borrarla a partir del segundo arranque, ya que el disco ya habrá sido creado con anterioridad.
        v.customize ['createhd', '--filename', disco4, '--size', 1024] #Le damos un tamaño máximo al cuarto disco creado anteriormente, de un total de 1024 MiB. Esta instrucción es necesario borrarla a partir del segundo arranque, ya que el disco ya habrá sido creado con anterioridad.
        v.customize ['createhd', '--filename', disco5, '--size', 1024] #Le damos un tamaño máximo al quinto disco creado anteriormente, de un total de 1024 MiB. Esta instrucción es necesario borrarla a partir del segundo arranque, ya que el disco ya habrá sido creado con anterioridad.
        v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disco1] #Anexamos el primer disco creado anteriormente a nuestra máquina virtual mediante SATA al puerto 1.
        v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', disco2] #Anexamos el segundo disco creado anteriormente a nuestra máquina virtual mediante SATA al puerto 2.
        v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 3, '--device', 0, '--type', 'hdd', '--medium', disco3] #Anexamos el tercer disco creado anteriormente a nuestra máquina virtual mediante SATA al puerto 3.
        v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 4, '--device', 0, '--type', 'hdd', '--medium', disco4] #Anexamos el cuarto disco creado anteriormente a nuestra máquina virtual mediante SATA al puerto 4.
        v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 5, '--device', 0, '--type', 'hdd', '--medium', disco5] #Anexamos el quinto disco creado anteriormente a nuestra máquina virtual mediante SATA al puerto 5.
    end
end
{% endhighlight %}

Tras haber guardado los cambios, debemos acceder al directorio donde tenemos el Vagrantfile almacenado (pues de lo contrario no funcionará) y arrancar el escenario/infraestructura. Para ello, ejecutaremos el comando:

{% highlight shell %}
cd ~/vagrant/raid/ && vagrant up
{% endhighlight %}

A continuación comenzará a descargar el box (en caso de que no lo tuviésemos previamente) y a generar la máquina virtual con los parámetros que le hemos establecido en el Vagrantfile. Una vez que haya finalizado el proceso, nos podremos conectar a la misma ejecutando la siguiente instrucción:

{% highlight shell %}
alvaro@debian:~/vagrant/raid$ vagrant ssh
Linux practicaraid 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2 (2020-04-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Ya nos encontramos dentro de la máquina. Para verificar que los 5 discos se han creado y anexado correctamente, ejecutaremos el siguiente comando para listar los dispositivos de bloques existentes en la máquina:

{% highlight shell %}
vagrant@practicaraid:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0  19.8G  0 disk 
├─sda1   8:1    0  18.8G  0 part /
├─sda2   8:2    0     1K  0 part
└─sda5   8:5    0  1021M  0 part [SWAP]
sdb      8:16   0     1G  0 disk
sdc      8:32   0     1G  0 disk
sdd      8:48   0     1G  0 disk
sde      8:64   0     1G  0 disk
sdf      8:80   0     1G  0 disk
{% endhighlight %}

Efectivamente, se han creado y anexado 5 discos de 1GB (_sdb_, _sdc_, _sdd_, _sde_ y _sdf_).

Lo siguiente será instalar el paquete necesario para crear RAID software (**mdadm**), no sin antes upgradear los paquetes instalados, ya que el box con el tiempo se va quedando desactualizado. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `sudo su -`):

{% highlight shell %}
apt-get -y update && apt-get -y upgrade && apt-get -y install mdadm
{% endhighlight %}

Ya tenemos todo lo necesario para crear un RAID 5, así que ahora seguiremos la siguiente estructura para ejecutar la instrucción:

{% highlight shell %}
mdadm --create /dev/[nombreraid] --level=[nivel] --raid-devices=[numdiscos] /dev/[disco1].../dev/[discoN]
{% endhighlight %}

En este caso, el nombre del raid tiene que ser **md5**, de nivel **5**, con un total de **3** discos (**sdb**, **sdc** y **sdd**, por ejemplo). La instrucción quedaría de la siguiente forma:

{% highlight shell %}
root@practicaraid:~# mdadm --create /dev/md5 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md5 started.
{% endhighlight %}

En un principio, el RAID 5 ya está creado e iniciado, gracias a la información que nos ha devuelto por pantalla. Es importante mencionar que podríamos haber creado una tabla de particiones dentro de dichos discos y en lugar de agregar al RAID el disco _sdb_, haber agregado la partición _sdb1_, pero no es algo necesario en este caso.

## Tarea 2: Comprueba las características del RAID. Comprueba el estado del RAID. ¿Qué capacidad tiene el RAID que hemos creado?

Para comprobar las características (detalles) de un RAID debemos hacer uso de la sintaxis:

{% highlight shell %}
mdadm --detail /dev/[nombreraid]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Tue Sep 29 18:20:51 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent
   
       Update Time : Tue Sep 29 18:21:01 2020
             State : clean 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0
   
            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : practicaraid:5  (local to host practicaraid)
              UUID : 9974942d:1a0a0bc2:538d513e:b3cb1c91
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
{% endhighlight %}

Como se puede apreciar en la salida del comando, el tamaño disponible del RAID (Array Size) es de 2044MiB, es decir, 2GiB, pues tal y como hemos explicado anteriormente, al tratarse de un RAID 5, se pierde la capacidad de uno de los discos (Used Dev Size), es decir, 1GiB. También podemos apreciar que se encuentra compuesto por los tres discos que hemos introducido (_sdb_, _sdc_ y _sdd_).

Para comprobar el estado de un RAID debemos ejecutar el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md5 : active raid5 sdd[3] sdc[1] sdb[0]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
{% endhighlight %}

Como se puede apreciar, el RAID 5 de nombre **md5** se encuentra actualmente activo (**active**), además, los **3** discos se encuentran en perfecto estado y en uso actualmente (no hay unused devices).

## Tarea 3: Crea un volumen lógico (LVM) de 500MB en el RAID 5.

Lo primero será instalar el paquete necesario para usar LVM (**lvm2**), ya que en el box no se encuentra instalado. Para ello, ejecutamos el comando:

{% highlight shell %}
apt-get -y install lvm2
{% endhighlight %}

En LVM tenemos que realizar una serie de pasos antes de crear un volumen lógico. El primero es añadir el dispositivo de bloques (en este caso el RAID 5 que acabamos de crear) como volumen físico (**PV**). Tras ello, dicho volumen físico tendremos que añadirlo a un "contenedor" conocido como grupo de volúmenes (**VG**), y de ahí es de donde posteriormente cogeremos la capacidad para crear un volumen lógico (**LV**).

![lvm](https://www.howtogeek.com/thumbcache/2/200/9e05607dd48079977abc73a87d635959/wp-content/uploads/2011/02/banner-1.png "LVM")

Esta estructura tiene una gran ventaja y es que al ser volúmenes dinámicos, podemos aumentar su tamaño si nos quedamos escasos, ya que simplemente bastaría con comprar un nuevo disco duro, añadirlo como volumen físico y al grupo de volúmenes y posteriormente, extender el volumen lógico. Finalmente, redimensionamos el sistema de ficheros y listo. Sin embargo, esto no es posible con las particiones físicas, ya que al estar delimitado por sectores, el hecho de aumentar la partición puede afectar a los datos que se encuentren contiguos.

Antes de nada, añadiremos el RAID 5 como volumen físico, haciendo uso de la sintaxis:

{% highlight shell %}
pvcreate /dev/[nombreraid]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# pvcreate /dev/md5
   Physical volume "/dev/md5" successfully created.
{% endhighlight %}

El dispositivo de bloques ya se ha añadido como volumen físico, así que el siguiente paso será crear un grupo de volúmenes (de nombre **raid5**, por ejemplo) y añadirlo al mismo, haciendo uso de la sintaxis:

{% highlight shell %}
vgcreate [nombrevg] /dev/[nombrepv]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# vgcreate raid5 /dev/md5
   Volume group "raid5" successfully created
{% endhighlight %}

Para verificar que hasta ahora lo hemos hecho todo correctamente, ejecutaremos los siguientes comandos para listar los volúmenes físicos y los grupos de volúmenes existentes en la máquina:

{% highlight shell %}
root@practicaraid:~# pvs
   PV           VG      Fmt     Attr    PSize   PFree
   /dev/md5     raid5   lvm2    a--     1.99g   1.99g
root@practicaraid:~# vgs
   VG       #PV #LV #SN Attr    VSize   VFree
   raid5    1   0   0   wz--n-  1.99g   1.99g
{% endhighlight %}

Efectivamente, un volumen físico de nombre **/dev/md5** se encuentra creado, perteneciente al grupo de volúmenes **raid5**, que tiene todo el espacio disponible.

Ha llegado el momento de crear el volumen lógico (de nombre **volumen1**, por ejemplo), así que haremos uso de la sintaxis:

{% highlight shell %}
lvcreate -L [tamaño] -n [nombrelv] [nombrevg]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# lvcreate -L 500M -n volumen1 raid5
   Logical volume "volumen1" created.
{% endhighlight %}

De nuevo, para verificar que el volumen lógico ha sido correctamente creado, ejecutaremos el comando:

{% highlight shell %}
root@practicaraid:~# lvs
   LV           VG      Attr        LSize   Pool    Origin  Data%   Meta%   Move    Log Cpy%Sync    Convert
   volumen1     raid5   -wi-a-----  500.00m
{% endhighlight %}

Efectivamente, así ha sido. Además, ejecutaremos el comando `lsblk` de nuevo para ver cómo va la estructura.

{% highlight shell %}
root@practicaraid:~# lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  19.8G  0 disk 
├─sda1                8:1    0  18.8G  0 part /
├─sda2                8:2    0     1K  0 part
└─sda5                8:5    0  1021M  0 part [SWAP]
sdb                   8:16   0     1G  0 disk
└─md5                 9:5    0     2G  0 raid5
  └─raid5-volumen1  253:0    0   500M  0 lvm
sdc                   8:32   0     1G  0 disk
└─md5                 9:5    0     2G  0 raid5
  └─raid5-volumen1  253:0    0   500M  0 lvm
sdd                   8:48   0     1G  0 disk
└─md5                 9:5    0     2G  0 raid5
  └─raid5-volumen1  253:0    0   500M  0 lvm
sde                   8:64   0     1G  0 disk
sdf                   8:80   0     1G  0 disk
{% endhighlight %}

De aquí podemos deducir que el RAID 5 de nombre **md5** se encuentra contenido entre los dispositivos **sdb**, **sdc** y **sdd**, de 1GB cada uno, teniendo un tamaño total de 2GB, además de un volumen lógico de nombre **volumen1**, que se encuentra dentro de un grupo de volúmenes de nombre **raid5**, teniendo un tamaño total de 500MB.

## Tarea 4: Formatea ese volumen con un sistema de archivos xfs.

Lo primero será instalar el paquete necesario para poder crear sistemas de ficheros XFS (**xfsprogs**), ya que en el box no se encuentra instalado. Para ello, ejecutamos el comando:

{% highlight shell %}
apt-get -y install xfsprogs
{% endhighlight %}

Ya podemos crear el sistema de ficheros XFS en el volumen lógico **volumen1**. Para ello, haremos uso de la sintaxis:

{% highlight shell %}
mkfs.xfs /dev/[nombrevg]/[nombrelv]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mkfs.xfs /dev/raid5/volumen1
meta-data=/dev/raid5/volumen1    isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
{% endhighlight %}

Para verificar que el sistema de ficheros se ha generado correctamente dentro del dispositivo de bloques **/dev/raid5/volumen1**, ejecutaremos el comando:

{% highlight shell %}
root@practicaraid:~# lsblk -f
NAME                 FSTYPE            LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1               ext4                             983742b1-65a8-49d1-a148-a3865ea09e24     15.9G     8% /
├─sda2
└─sda5               swap                             04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450
sdc                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450
sdd
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450
sde
sdf
{% endhighlight %}

Tal y como se puede apreciar en los tipos de sistemas de ficheros (**FSTYPE**), el volumen lógico **volumen1** tiene un sistema de ficheros **XFS**.

## Tarea 5: Monta el volumen en el directorio /mnt/raid5 y crea un fichero. ¿Qué tendríamos que hacer para que este punto de montaje sea permanente?

El primer paso será crear un directorio dentro de **/mnt/** de nombre **raid5/**. Para ello, ejecutaremos el comando:

{% highlight shell %}
mkdir /mnt/raid5
{% endhighlight %}

Una vez creado, ya podremos montar el sistema de ficheros. Para ello, haremos uso de la sintaxis:

{% highlight shell %}
mount /dev/[nombrevg]/[nombrelv] [puntomontaje]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
mount /dev/raid5/volumen1 /mnt/raid5/
{% endhighlight %}

En un principio, no ha devuelto ningún mensaje, así que volveremos a ejecutar el comando `lsblk -f` para verificarlo.

{% highlight shell %}
root@practicaraid:~# lsblk -f
NAME                 FSTYPE            LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1               ext4                             983742b1-65a8-49d1-a148-a3865ea09e24     15.9G     8% /
├─sda2
└─sda5               swap                             04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sdc                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sdd
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sde
sdf
{% endhighlight %}

Tal y como se puede apreciar en la parte derecha, el volumen lógico tiene un total de 470.6MB disponibles, y su punto de montaje es **/mnt/raid5**, tal y como era de esperar.

A continuación, vamos a crear un fichero con un texto en su interior (de nombre **prueba.txt**, por ejemplo). Para ello, ejecutaremos el comando:

{% highlight shell %}
echo "Esto es una prueba." > /mnt/raid5/prueba.txt
{% endhighlight %}

Para verificar que el fichero se ha creado con dicho texto en su interior, vamos a leer su contenido, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /mnt/raid5/prueba.txt
Esto es una prueba.
{% endhighlight %}

Como se puede apreciar, el fichero se ha creado correctamente.

Para hacer que el punto de montaje sea permanente, tenemos que configurar el fichero **/etc/fstab**. Este requerirá tener introducido el **UUID** del sistema de ficheros en cuestión, así que dado que anteriormente hemos visto los UUID de los sistemas de ficheros al ejecutar `lsblk -f`, sabemos que el UUID del XFS empieza por **4**, así que ejecutaremos el siguiente comando para añadirlo al final del documento /etc/fstab (en realidad sería más sencillo marcarlo con el cursor y hacer copia-pega, pero no siempre vamos a tener ratón a la hora de configurar un servidor):

{% highlight shell %}
lsblk -f | egrep '^4' | uniq >> /etc/fstab
{% endhighlight %}

El UUID ya se encuentra anexado, así que para comenzar a editar el fichero, ejecutaremos el comando:

{% highlight shell %}
nano /etc/fstab
{% endhighlight %}

{% highlight shell %}
UUID=983742b1-65a8-49d1-a148-a3865ea09e24       /        ext4        errors=remount-ro       0        1
UUID=04559374-06db-46f1-aa31-e7a4e6ec3286       none     swap        sw       0        0
/dev/sr0       /media/cdrom0     udf,iso9660        user,noauto       0        0
4f3d039c-2dc6-4cab-97d3-5751a3d75450
{% endhighlight %}

Como se puede apreciar, la última línea contiene el UUID en cuestión. Tenemos que dejar dicha línea con la siguiente estructura:

{% highlight shell %}
[UUID] [punto_de_montaje] [sistema_de_archivos] [opciones] [dump] [pass]
{% endhighlight %}

Siendo:

* **UUID** = Identificador único universal del sistema de ficheros (_4f3d039c-2dc6-4cab-97d3-5751a3d75450_).
* **punto_de_montaje** = Directorio donde se montará el sistema de archivos (_/mnt/raid5_).
* **sistema_de_archivos** = Tipo de sistema de ficheros (_xfs_).
* **opciones** = Opciones de arranque (_defaults_).
* **dump** = Indica si se hará copia de seguridad del sistema de ficheros, siendo 0 no y 1 sí (_0_).
* **pass** = Indica la prioridad con la que se revisa el sistema de ficheros con fsck, siendo 0 nunca, 1 para el raíz y 2 para el resto (_2_).

En este caso, el resultado debe ser:

{% highlight shell %}
UUID=983742b1-65a8-49d1-a148-a3865ea09e24       /        ext4        errors=remount-ro       0        1
UUID=04559374-06db-46f1-aa31-e7a4e6ec3286       none        swap        sw       0        0
/dev/sr0       /media/cdrom0     udf,iso9660        user,noauto       0        0
UUID=4f3d039c-2dc6-4cab-97d3-5751a3d75450       /mnt/raid5        xfs        defaults       0       2
{% endhighlight %}

Para comprobar que funciona, vamos a reiniciar la máquina, ejecutando el comando:

{% highlight shell %}
root@practicaraid:~# reboot
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
{% endhighlight %}

Si volvemos a ejecutar el comando `vagrant ssh`, nos conectaremos de nuevo a la máquina sin problemas. Tras ello, volveremos a ejecutar el comando `lsblk -f` para verificar que el sistema de ficheros se ha montado automáticamente:

{% highlight shell %}
vagrant@practicaraid:~$ lsblk -f
NAME                 FSTYPE            LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1               ext4                             983742b1-65a8-49d1-a148-a3865ea09e24     15.9G     8% /
├─sda2
└─sda5               swap                             04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sdc                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sdd                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sde
sdf
{% endhighlight %}

Efectivamente, así ha sido.

## Tarea 6: Marca un disco como estropeado. Muestra el estado del RAID para comprobar que un disco falla. ¿Podemos acceder al fichero?

Para marcar un disco como estropeado en el RAID 5, haremos uso de la sintaxis:

{% highlight shell %}
mdadm -f /dev/[nombreraid] /dev/[disco]
{% endhighlight %}

En este caso, marcaré como estropeado el disco **sdb**, por ejemplo. El comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mdadm -f /dev/md5 /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md5
{% endhighlight %}

Al parecer, el disco ya se ha marcado como estropeado, pero para verificarlo, vamos comprobar el estado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sdb[0](F) sdc[1] sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]

unused devices: <none>
{% endhighlight %}

Como podemos ver, el disco **sdb** tiene una **(F)** a su lado, indicando que su estado es **faulty** (estropeado). Aunque haya fallado un disco, vamos a comprobar si todavía podemos acceder al fichero contenido en el interior del RAID 5. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@practicaraid:~# cat /mnt/raid5/prueba.txt
Esto es una prueba.
{% endhighlight %}

Efectivamente, el fichero **prueba.txt** todavía es accesible.

## Tarea 7: Una vez marcado como estropeado, lo tenemos que retirar del RAID.

Para retirar un disco estropeado del RAID 5, haremos uso de la sintaxis:

{% highlight shell %}
mdadm --remove /dev/[nombreraid] /dev/[disco]
{% endhighlight %}

En este caso, eliminaré el disco estropeado **sdb**. El comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mdadm --remove /dev/md5 /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md5
{% endhighlight %}

Al parecer, el disco ya se ha eliminado del RAID 5 y podríamos desconectarlo de la máquina, pero para verificarlo, vamos comprobar el estado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sdc[1] sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]

unused devices: <none>
{% endhighlight %}

Efectivamente, el disco **sdb** ha sido eliminado del RAID 5 y ya no aparece entre los dispositivos activos, por lo que, en un caso real, podríamos desconectarlo de la máquina y tirarlo.

Si nos fijamos en la parte inferior, podemos ver que nos pone **[3/2] [_UU]**, indicando que actualmente el RAID 5 cuenta con 2 de los 3 discos necesarios, de manera que deberíamos incluir un nuevo disco de manera inmediata, o en caso de fallo de cualquiera de los dos discos actualmente funcionales, perderíamos información.

## Tarea 8: Imaginemos que lo cambiamos por un nuevo disco nuevo (el dispositivo de bloque se llama igual), añádelo al array y comprueba cómo se sincroniza con el anterior.

Para añadir un nuevo disco al RAID 5, haremos uso de la sintaxis:

{% highlight shell %}
mdadm --add /dev/[nombreraid] /dev/[disco]
{% endhighlight %}

En este caso, añadiré al RAID 5 el disco **sde**. El comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mdadm --add /dev/md5 /dev/sde
mdadm: added /dev/sde
{% endhighlight %}

Al parecer, el disco ya se ha añadido al RAID 5, pero para verificarlo, vamos comprobar el estado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sde[4] sdc[1] sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]
      [===========>.........]  recovery = 58.9% (617596/1046528) finish=0.0min speed=154399K/sec

unused devices: <none>
{% endhighlight %}

Dado que hemos ejecutado rápidamente la instrucción para comprobar el estado del RAID, hemos visto la barra de progreso de la recuperación de la información en el nuevo disco. Sin embargo, si volvemos a ejecutarlo una vez más, la barra de progreso habrá finalizado y nuestro RAID volverá a estar totalmente operativo.

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sde[4] sdc[1] sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
{% endhighlight %}

Nuestro RAID está totalmente operativo de nuevo.

Si nos volvemos a fijar, podemos ver que nos pone **[3/3] [UUU]**, indicando que actualmente el RAID 5 cuenta con 3 de los 3 discos necesarios, de manera que se encuentra totalmente funcional.

## Tarea 9: Añade otro disco como reserva. Vuelve a simular el fallo de un disco y comprueba cómo automáticamente se realiza la sincronización con el disco de reserva.

La instrucción a ejecutar para añadir un disco como reserva es la misma que hemos ejecutado en la tarea anterior. En este caso, añadiré como disco de reserva al RAID 5 el disco **sdf**. El comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# mdadm --add /dev/md5 /dev/sdf
mdadm: added /dev/sdf
{% endhighlight %}

Al parecer, el disco de reserva ya se ha añadido al RAID 5, pero para verificarlo, vamos comprobar el estado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sdf[5](S) sde[4] sdc[1] sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
{% endhighlight %}

Como podemos ver, el disco **sdf** tiene una **(S)** a su lado, indicando que su estado es **spare** (de repuesto). Para comprobar que el disco de reserva cumple su función, vamos a marcar uno de los discos como estropeado (el disco **sdc**, por ejemplo), ejecutando la instrucción vista anteriormente en la _Tarea 6_:

{% highlight shell %}
root@practicaraid:~# mdadm -f /dev/md5 /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md5
{% endhighlight %}

Al parecer, el disco ya se ha marcado como estropeado, pero para verificarlo, vamos comprobar el estado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sdf[5] sde[4] sdc[1](F) sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [===========>.........]  recovery = 56.5% (592436/1046528) finish=0.0min speed=296218K/sec

unused devices: <none>
{% endhighlight %}

Dado que hemos ejecutado rápidamente la instrucción para comprobar el estado del RAID, hemos visto la barra de progreso de la recuperación de la información en el disco de reserva, además de apreciar que el disco **sdf** ha dejado de tener la marca de reserva **(S)** y el disco **sdc** ahora tiene la marca de estropeado **(F)**. Sin embargo, si volvemos a ejecutarlo una vez más, la barra de progreso habrá finalizado y nuestro RAID volverá a estar totalmente operativo.

{% highlight shell %}
root@practicaraid:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md5 : active raid5 sdf[5] sde[4] sdc[1](F) sdd[3]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
{% endhighlight %}

Nuestro RAID está totalmente operativo de nuevo, aunque sería aconsejable eliminar el disco estropeado del RAID, ejecutando para ello el comando:

{% highlight shell %}
root@practicaraid:~# mdadm --remove /dev/md5 /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md5
{% endhighlight %}

El disco ya ha sido eliminado.

## Tarea 10: Redimensiona el volumen y el sistema de archivos de 500MB al tamaño del RAID.

Para redimensionar un volumen lógico LVM haremos uso de la sintaxis:

{% highlight shell %}
lvextend -l +[extents] /dev/[nombrevg]/[nombrelv]
{% endhighlight %}

En este caso, especificaremos **+100%FREE** como parámetro **-l**, para que así coja todos los extents disponibles y haga uso de todo el espacio libre. El comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# lvextend -l +100%FREE /dev/raid5/volumen1
   Size of logical volume raid5/volumen1 changed from 500.00 MiB (125 extents) to 1.99 GiB (510 extents).
   Logical volume raid5/volumen1 successfully resized.
{% endhighlight %}

El volumen lógico ya ha sido extendido, así que para verificarlo, vamos a ejecutar el comando `lsblk -f`:

{% highlight shell %}
root@practicaraid:~# lsblk -f
NAME                 FSTYPE            LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1               ext4                             983742b1-65a8-49d1-a148-a3865ea09e24     15.9G     8% /
├─sda2
└─sda5               swap                             04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
sdc                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
sdd                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sde                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
sdf                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450    470.6M     5% /mnt/raid5
{% endhighlight %}

Si nos fijamos, el espacio disponible sigue apareciendo que es de **470.6MB**. ¿A qué se debe esto? Bien, hemos redimensionado el volumen lógico, pero no hemos redimensionado lo más importante, el sistema de ficheros que hay en su interior. Para redimensionar un sistema de ficheros XFS haremos uso de la sintaxis:

{% highlight shell %}
xfs_growfs [puntomontaje]
{% endhighlight %}

En este caso, el comando a ejecutar sería:

{% highlight shell %}
root@practicaraid:~# xfs_growfs /mnt/raid5/
meta-data=/dev/mapper/raid5-volumen1    isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 128000 to 522240
{% endhighlight %}

El sistema de ficheros ya ha sido redimensionado, pues antes tenía **128000** bloques y ha pasado a tener **522240**. Volvemos a ejecutar el comando `lsblk -f` para verificar que el sistema de ficheros ha crecido.

{% highlight shell %}
root@practicaraid:~# lsblk -f
NAME                 FSTYPE            LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1               ext4                             983742b1-65a8-49d1-a148-a3865ea09e24     15.9G     8% /
├─sda2
└─sda5               swap                             04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
sdc                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
sdd                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450        2G     1% /mnt/raid5
sde                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450        2G     1% /mnt/raid5
sdf                  linux_raid_member practicaraid:5 9974942d-1a0a-0bc2-538d-513eb3cb1c91
└─md5                LVM2_member                      cAPGLD-kFBx-uLTX-a6Lw-N9X3-mFxc-hcAg3s
  └─raid5-volumen1   xfs                              4f3d039c-2dc6-4cab-97d3-5751a3d75450        2G     1% /mnt/raid5
{% endhighlight %}

Efectivamente, el tamaño del sistema de ficheros ha pasado de ser de **470.6MB** a ser de **2GB**.