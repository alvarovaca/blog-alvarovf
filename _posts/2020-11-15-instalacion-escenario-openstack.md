---
layout: post
title:  "Instalación y configuración del escenario OpenStack"
banner: "/assets/images/banners/openstack.png"
date:   2020-11-15 13:20:00 +0200
categories: hlc openstack
---
Se recomienda realizar una previa lectura del _post_ [Configuración de cliente OpenVPN con certificados X.509](http://www.alvarovf.com/hlc/2020/11/02/configuracion-cliente-openvpn.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de ésta tarea es el de crear el escenario de trabajo que se va a usar durante todo el curso en **OpenStack**, que va a constar inicialmente de 3 instancias con nombres relacionados con el libro "_Don Quijote de la Mancha_". La infraestructura a la que se pretende llegar es la siguiente:

![escenario1](https://i.ibb.co/0sTVtvM/escenario.png "Escenario OpenStack")

- **Dulcinea**:
    - _Debian Buster_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Acessible a través de la red externa _10.0.0.0/24_ con una IP flotante.
    - Conectada a la red interna _10.0.1.0/24_.
- **Sancho**:
    - _Ubuntu 20.04_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Conectada a la red interna _10.0.1.0/24_.
- **Quijote**:
    - _CentOS 7_ sobre un volumen de _10GB_ con sabor _m1.mini_.
    - Conectada a la red interna _10.0.1.0/24_.

Como se puede apreciar, en el escenario tendremos una red externa con direccionamiento **10.0.0.0/24** que cuenta con salida a Internet a través de un router situado en la dirección **10.0.0.1**. Una de las interfaces de **Dulcinea** se encontrará dentro de dicha red. La red en cuestión viene creada por defecto.

De otro lado, tendremos una red interna con direccionamiento **10.0.1.0/24**, en la que se encontrarán las tres máquinas. Si lo pensamos, dicha red no tiene salida a Internet, de manera que las máquinas que se encuentren ahí no podrán salir. Para solucionarlo, vamos a configurar _NAT_ en **Dulcinea**, concretamente un _SNAT_, para que así tengan conectividad a través de la misma. La red en cuestión hay que crearla manualmente, así que vamos a ponernos manos a la obra.

Partimos un punto en el que ya hemos configurado la VPN para poder tener acceso remoto a dichas máquinas, que se encuentran en el departamento de informática del IES Gonzalo Nazareno, así que accederemos a **jupiter.gonzalonazareno.org**, que resolverá estáticamente a **172.22.222.1**, de manera que accederemos con nuestro usuario y procederemos a crear la red.

## Creación de la red interna

Para ello, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, pulsaremos en el botón **Crear red** y estableceremos los siguientes parámetros:

- **Red**:
    - **Nombre de la red**: red interna de alvaro.vaca
    - **Estado de administración**: ARRIBA
    - **Compartido**: _No_
    - **Crear subred**: _Sí_
- **Subred**:
    - **Nombre de subred**: _Vacío_
    - **Direcciones de red**: 10.0.1.0/24
    - **Versión de IP**: IPv4
    - **Deshabilitar puerta de enlace**: _Sí_
- **Detalles de Subred**:
    - **Habilitar DHCP**: _Sí_
    - **Pools de asignación**: 10.0.1.2,10.0.1.254
    - **Servidores DNS**: 192.168.200.2

Gracias a la configuración mencionada, habremos evitado que el servidor DHCP sirva una puerta de enlace predeterminada a las máquinas (pues nos podría haber causado algunos conflictos con **Dulcinea**, ya que se encuentra dentro de las dos redes). Además, hemos habilitado el servidor DHCP para que sirva direcciones IP a las máquinas, ya que de primeras, necesitaremos una dirección para acceder a las mismas (aunque posteriormente configuraremos un direccionamiento estático). El resultado final sería:

![redes1](https://i.ibb.co/RQpcPLz/4.jpg "Creación de red")

Genial, ya hemos creado la red interna que vamos a necesitar, de manera que ya tenemos las dos redes (externa e interna) configuradas y operativas.

## Creación de las instancias

El primer paso será crear los volúmenes que utilizarán las correspondientes instancias. Para ello, accederemos en el menú izquierdo al apartado **COMPUTE**, dentro del cuál pulsaremos en **Volúmenes**. Una vez ahí, pulsaremos en el botón **Crear volumen** y estableceremos los siguientes parámetros para cada uno de los volúmenes:

- **Dulcinea**:
    - **Nombre del volumen**: Dulcinea
    - **Descripción**: _Vacío_
    - **Origen del volumen**: Imagen
    - **Utilizar una imagen como origen**: Debian Buster 10.6 (516,5 MB)
    - **Tipo**: Sin tipo de volumen
    - **Tamaño (GiB)**: 10
    - **Zona de Disponibilidad**: nova
- **Sancho**:
    - **Nombre del volumen**: Sancho
    - **Descripción**: _Vacío_
    - **Origen del volumen**: Imagen
    - **Utilizar una imagen como origen**: Ubuntu 20.04 LTS (focal fossa) (522,2 MB)
    - **Tipo**: Sin tipo de volumen
    - **Tamaño (GiB)**: 10
    - **Zona de Disponibilidad**: nova
- **Quijote**:
    - **Nombre del volumen**: Quijote
    - **Descripción**: _Vacío_
    - **Origen del volumen**: Imagen
    - **Utilizar una imagen como origen**: CentOS 7 (819,0 MB)
    - **Tipo**: Sin tipo de volumen
    - **Tamaño (GiB)**: 10
    - **Zona de Disponibilidad**: nova

El resultado final tras haber generado los 3 volúmenes sería:

![volumenes1](https://i.ibb.co/Ry1gDWR/extra4.jpg "Creación de volúmenes")

Los volúmenes que van a ser utilizados por las instancias ya se encuentran generados, así que no queda nada más por hacer antes de proceder a crear las correspondientes instancias, por lo que vamos a ponernos manos a la obra. Para ello, accederemos en el menú izquierdo al apartado **COMPUTE**, dentro del cuál pulsaremos en **Instancias**. Una vez ahí, pulsaremos en el botón **Lanzar instancia** y estableceremos los siguientes parámetros para cada una de las instancias:

- **Dulcinea**:
    - **Details**:
        - **Instance Name**: Dulcinea
        - **Zona de Disponibilidad**: nova
        - **Count**: 1
    - **Source**:
        - **Seleccionar Origen de arranque**: Volumen
        - **Delete Volume on Instance Delete**: No
        - **Allocated**: Dulcinea
    - **Sabor**:
        - **Allocated**: m1.mini
    - **Redes**:
        - **Allocated**: red de alvaro.vaca
        - **Allocated**: red interna de alvaro.vaca
    - **Par de claves**:
        - **Allocated**: Linux
- **Sancho**:
    - **Details**:
        - **Instance Name**: Sancho
        - **Zona de Disponibilidad**: nova
        - **Count**: 1
    - **Source**:
        - **Seleccionar Origen de arranque**: Volumen
        - **Delete Volume on Instance Delete**: No
        - **Allocated**: Sancho
    - **Sabor**:
        - **Allocated**: m1.mini
    - **Redes**:
        - **Allocated**: red interna de alvaro.vaca
    - **Par de claves**:
        - **Allocated**: Linux
- **Quijote**:
    - **Details**:
        - **Instance Name**: Quijote
        - **Zona de Disponibilidad**: nova
        - **Count**: 1
    - **Source**:
        - **Seleccionar Origen de arranque**: Volumen
        - **Delete Volume on Instance Delete**: No
        - **Allocated**: Quijote
    - **Sabor**:
        - **Allocated**: m1.mini
    - **Redes**:
        - **Allocated**: red interna de alvaro.vaca
    - **Par de claves**:
        - **Allocated**: Linux

Las instancias ya han sido generadas, pero todavía no tenemos acceso a ninguna de ellas, pues las redes a las que pertenecen no son accesibles por nosotros (al menos de forma directa), de manera que debemos asignarle a aquella instancia capaz de salir a Internet una IP flotante, en este caso, a **Dulcinea**. Una IP flotante no es más que un router virtual con dos interfaces, la primera de ellas conectada a la red accesible por nosotros, es decir, a **172.22.0.0/16** y la segunda, a la red no accesible directamente, es decir, a **10.0.0.0/24**.

Para ello, volveremos a la lista de nuestras instancias creadas y en pulsaremos en el símbolo **▾** en Dulcinea, para así abrir el desplegable. Tras ello, pulsaremos en la opción **Asociar IP flotante** y le asignaremos una de las disponibles. El resultado final sería:

![instancias1](https://i.ibb.co/DLkN9ZF/extra9.jpg "Creación de instancias")

Como se puede apreciar, las direcciones IP asignadas han sido las siguientes:

- **Dulcinea**:
    - 10.0.0.7 - 172.22.200.134
    - 10.0.1.9
- **Sancho**:
    - 10.0.1.4
- **Quijote**:
    - 10.0.1.5

Si recordamos, con anterioridad hemos añadido a las 3 instancias un par de claves de nombre **Linux**. Éste par de claves es personal y lo utilizaré para conectarme por primera vez a las instancias, ya que por defecto no tienen ninguna contraseña establecida. Si lo pensamos, hay un pequeño problema, y es que para conectarnos a las instancias **Sancho** y **Quijote**, no podremos hacerlo de forma directa desde la máquina anfitriona, dado que se encuentran en la red privada, sino que tendremos que establecer una conexión con **Dulcinea** y desde ahí, establecer la conexión con las otras dos instancias.

Es por ello que tendremos que hacer uso de una opción de `ssh`, para así poder usar el par de claves **Linux** en la máquina **Dulcinea** a la hora de conectarnos a las otras dos instancias, pero sin necesidad de copiar la clave privada a dicha máquina, pues supondría un agujero de seguridad bastante grande.

Lo primero que tendremos que hacer es añadir la clave privada a nuestro agente de claves de la sesión en nuestra máquina anfitriona, haciendo para ello uso de `ssh-add`, que recibirá la ruta de la clave privada que deseamos añadir:

{% highlight shell %}
alvaro@debian:~$ ssh-add ~/.ssh/linux.pem 
Identity added: /home/alvaro/.ssh/linux.pem (/home/alvaro/.ssh/linux.pem)
{% endhighlight %}

Como se puede apreciar en la salida del comando, la clave ha sido correctamente añadida, así que todo está preparado para realizar la conexión **ssh** a la dirección **172.22.200.134**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ ssh -A debian@172.22.200.134
Linux dulcinea 4.19.0-11-cloud-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
{% endhighlight %}

Donde:

* **-A**: Indica a SSH que herede el agente con las claves privadas almacenadas, que podrán ser utilizadas dentro de dicha instancia.

Ya nos encontramos dentro de la máquina. Para verificar que las interfaces de red se han generado correctamente, ejecutaremos el comando:

{% highlight shell %}
debian@dulcinea:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:e2:f6:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.7/24 brd 10.0.0.255 scope global dynamic eth0
       valid_lft 65310sec preferred_lft 65310sec
    inet6 fe80::f816:3eff:fee2:f6e7/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:b5:5d:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.9/24 brd 10.0.1.255 scope global dynamic eth1
       valid_lft 65310sec preferred_lft 65310sec
    inet6 fe80::f816:3eff:feb5:5de7/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Efectivamente, contamos con **dos** interfaces:

* **eth0**: Conectada a la red externa, con dirección IP **10.0.0.7**.
* **eth1**: Conectada a la red interna, con dirección IP **10.0.1.9**.

La configuración de red parece estar bien, pero todavía no hemos llevado a cabo la _prueba de fuego_, consistente en comprobar la conectividad con las otras dos máquinas, de manera que haremos uso del comando `ping` para probar la conectividad con **10.0.1.4** (_Sancho_) y **10.0.1.5** (_Quijote_):

{% highlight shell %}
debian@dulcinea:~$ ping 10.0.1.4
PING 10.0.1.4 (10.0.1.4) 56(84) bytes of data.
64 bytes from 10.0.1.4: icmp_seq=1 ttl=64 time=3.46 ms
64 bytes from 10.0.1.4: icmp_seq=2 ttl=64 time=1.21 ms
64 bytes from 10.0.1.4: icmp_seq=3 ttl=64 time=1.04 ms
^C
--- 10.0.1.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.044/1.904/3.462/1.104 ms

debian@dulcinea:~$ ping 10.0.1.5
PING 10.0.1.5 (10.0.1.5) 56(84) bytes of data.
64 bytes from 10.0.1.5: icmp_seq=1 ttl=64 time=1.53 ms
64 bytes from 10.0.1.5: icmp_seq=2 ttl=64 time=0.672 ms
64 bytes from 10.0.1.5: icmp_seq=3 ttl=64 time=0.724 ms
^C
--- 10.0.1.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 36ms
rtt min/avg/max/mdev = 0.672/0.973/1.525/0.392 ms
{% endhighlight %}

Efectivamente, existe conectividad entre las máquinas, así que antes de empezar a trabajar en las instancias, vamos a verificar que también podemos llevar a cabo una conexión `ssh` a las mismas, tratando de acceder primero a **10.0.1.4** (_Sancho_):

{% highlight shell %}
debian@dulcinea:~$ ssh ubuntu@10.0.1.4
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Nov 11 17:14:43 UTC 2020

  System load:  0.0               Processes:             95
  Usage of /:   12.8% of 9.52GB   Users logged in:       0
  Memory usage: 36%               IPv4 address for ens3: 10.0.1.4
  Swap usage:   0%

1 update can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
{% endhighlight %}

Efectivamente, **Dulcinea** ha podido llevar a cabo la conexión `ssh` con **Sancho**, así que cerraremos dicha conexión y volveremos a intentar conectarnos esta vez a **10.0.1.5** (_Quijote_):

{% highlight shell %}
debian@dulcinea:~$ ssh centos@10.0.1.5
[centos@quijote ~]$
{% endhighlight %}

Como era de esperar, la conexión **ssh** también se ha llevado a cabo correctamente, gracias a haber exportado nuestro agente de claves de la máquina anfitriona, de manera que **Dulcinea** puede hacer uso de las mismas.

## Definición de contraseña en todas las instancias

Como se ha mencionado con anterioridad, las instancias no vienen con una contraseña configurada, por lo que tendremos que establecerla manualmente para evitar perder el acceso a las mismas en caso que hubiese algún tipo de problema, pues en ese caso, podríamos acceder desde **Horizon**. La sintaxis para establecer una contraseña es (dicha sintaxis es genérica, por lo que nos servirá en las tres distribuciones):

{% highlight shell %}
passwd <usuario>
{% endhighlight %}

Donde:

* **usuario**: Indicamos el usuario cuya contraseña queremos modificar. En _Dulcinea_ es **debian**, en _Sancho_ es **ubuntu** y en _Quijote_ es **centos**.

### Dulcinea

{% highlight shell %}
debian@dulcinea:~$ sudo passwd debian 
New password: 
Retype new password: 
passwd: password updated successfully
{% endhighlight %}

### Sancho

{% highlight shell %}
ubuntu@sancho:~$ sudo passwd ubuntu 
New password: 
Retype new password: 
passwd: password updated successfully
{% endhighlight %}

### Quijote

{% highlight shell %}
[centos@quijote ~]$ sudo passwd centos
Changing password for user centos.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
{% endhighlight %}

## Configuración de NAT en Dulcinea

Actualmente, las máquinas **Sancho** y **Quijote** no tienen salida a Internet, puesto que van a salir a través de una máquina que se encuentra conectada a ambas redes (externa e interna), que va a actuar como router haciendo _SNAT_, en este caso **Dulcinea**. Dicha máquina viene configurada con un grupo de seguridad que se asigna por defecto durante su creación, es decir, cuenta con unas reglas de cortafuegos que no necesitamos, pues nosotros queremos configurar nuestras propias reglas para hacer _SNAT_. Dicha configuración no se puede llevar a cabo desde **Horizon**, sino que tendremos que usar el cliente desde la terminal, por lo que vamos a proceder a configurarlo.

El primer paso será crear un entorno virtual en Python en nuestra máquina anfitriona, de manera que debemos instalar previamente el paquete necesario para ello (**python3-venv**), además de otro paquete que requiere el cliente de OpenStack para funcionar (**python3-dev**), ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt install python3-venv python3-dev
{% endhighlight %}

En mi caso, me he movido al directorio **virtualenv/** (haciendo uso de `cd`), pues es donde tengo alojados todos los entornos virtuales, y por consecuencia, donde alojaré el que voy a crear. En ésta ocasión, el nombre que le voy a asignar al entorno virtual es **openstackclient**, así que lo generaré ejecutando el comando:

{% highlight shell %}
alvaro@debian:~/virtualenv$ python3 -m venv openstackclient
{% endhighlight %}

Una vez creado, tendremos que iniciarlo haciendo uso de `source` con el binario **activate** que se encuentra contenido en el directorio **bin/**:

{% highlight shell %}
alvaro@debian:~/virtualenv$ source openstackclient/bin/activate
{% endhighlight %}

Una vez activado, ya podremos proceder a instalar dicho cliente haciendo uso de `pip`. El paquete en cuestión se llama **python-openstackclient**, así que para instalarlo, haremos uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~$ pip install python-openstackclient
{% endhighlight %}

Una vez instalado, tendremos que descargar el fichero RC necesario para el funcionamiento de dicho cliente. Para ello, volveremos a **Horizon** y accederemos en el menú izquierdo al apartado **COMPUTE**, dentro del cuál pulsaremos en **Acceso y seguridad**. Una vez ahí, nos posicionaremos dentro de **Acceso a la API** y pulsaremos en el botón **Descargar fichero RC de OpenStack v3**.

Dado que actualmente se está utilizando una versión de OpenStack con _TLS_, es necesario indicar en dicho fichero una variable de entorno (**OS_CACERT**) con la ruta del certificado del IES Gonzalo Nazareno firmado por la autoridad certificadora:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ echo "export OS_CACERT=/etc/ssl/certs/gonzalonazareno.crt" >> ~/Descargas/Proyecto\ de\ alvaro.vaca-openrc.sh
{% endhighlight %}

La línea para declarar la correspondiente variable ya ha sido añadida, así que ya podremos ejecutar dicho script que llevará a cabo toda la configuración necesaria en la máquina anfitriona para establecer la conexión (configurará una serie de variables de entorno tales como la URL a la que conectar, el nombre de usuario...). Para ello, ejecutaremos el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ source ~/Descargas/Proyecto\ de\ alvaro.vaca-openrc.sh
Please enter your OpenStack Password for project Proyecto de alvaro.vaca as user alvaro.vaca:
{% endhighlight %}

Como se puede apreciar, nos ha pedido la contraseña de nuestro proyecto y la ha almacenado en una variable de entorno en texto plano. Tras ello, ya tendremos todo configurado y podremos proceder a listar las instancias existentes en nuestro proyecto para así verificar que funciona correctamente. El comando a ejecutar sería:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server list
+--------------------------------------+----------+---------+----------------------------------------------------------------------------------+--------------------------+---------+
| ID                                   | Name     | Status  | Networks                                                                         | Image                    | Flavor  |
+--------------------------------------+----------+---------+----------------------------------------------------------------------------------+--------------------------+---------+
| d5474d1a-51af-4dc5-935f-4756848bd88b | Quijote  | ACTIVE  | red interna de alvaro.vaca=10.0.1.5                                              | N/A (booted from volume) | m1.mini |
| 93673790-a461-49ee-94d6-f0aeff3b8bdb | Sancho   | ACTIVE  | red interna de alvaro.vaca=10.0.1.4                                              | N/A (booted from volume) | m1.mini |
| 2e1b3676-ae28-44a4-8748-9f59609c7ef7 | Dulcinea | ACTIVE  | red de alvaro.vaca=10.0.0.7, 172.22.200.134; red interna de alvaro.vaca=10.0.1.9 | N/A (booted from volume) | m1.mini |
| d73cbed2-1c32-46cf-98e1-e715a950688b | backup   | SHUTOFF | red de alvaro.vaca=10.0.0.5                                                      | Debian Buster 10.6       | m1.mini |
| 976b4ef3-55b2-468f-8243-ffb8db1fd521 | cms      | SHUTOFF | red de alvaro.vaca=10.0.0.8, 172.22.200.103                                      | Debian Buster 10.6       | m1.mini |
| 7c725323-c2e7-4d62-9a14-5f6dc42b4d96 | nginx2   | ACTIVE  | red de alvaro.vaca=10.0.0.13                                                     | Debian Buster 10.6       | m1.mini |
| da9c8fe2-3403-425c-8693-bfc65907bb0f | nginx    | ACTIVE  | red de alvaro.vaca=10.0.0.3, 172.22.200.108                                      | Debian Buster 10.6       | m1.mini |
+--------------------------------------+----------+---------+----------------------------------------------------------------------------------+--------------------------+---------+
{% endhighlight %}

Efectivamente, el cliente de OpenStack se encuentra actualmente operativo y nos ha mostrado la información referente a las instancias creadas en mi proyecto. Tras ello, procederemos a eliminar el grupo de seguridad por defecto (**default**) en **Dulcinea**, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server remove security group Dulcinea default
{% endhighlight %}

En teoría el grupo de seguridad ya ha sido eliminado, pero para verificarlo, vamos a listar la información detallada sobre la instancia **Dulcinea**, haciendo uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack server show Dulcinea
+-----------------------------+----------------------------------------------------------------------------------+
| Field                       | Value                                                                            |
+-----------------------------+----------------------------------------------------------------------------------+
| OS-DCF:diskConfig           | AUTO                                                                             |
| OS-EXT-AZ:availability_zone | nova                                                                             |
| OS-EXT-STS:power_state      | Running                                                                          |
| OS-EXT-STS:task_state       | None                                                                             |
| OS-EXT-STS:vm_state         | active                                                                           |
| OS-SRV-USG:launched_at      | 2020-11-11T11:20:56.000000                                                       |
| OS-SRV-USG:terminated_at    | None                                                                             |
| accessIPv4                  |                                                                                  |
| accessIPv6                  |                                                                                  |
| addresses                   | red de alvaro.vaca=10.0.0.7, 172.22.200.134; red interna de alvaro.vaca=10.0.1.9 |
| config_drive                |                                                                                  |
| created                     | 2020-11-11T11:20:32Z                                                             |
| flavor                      | m1.mini (12)                                                                     |
| hostId                      | 22a13c533c21963a8665798b1423dd32ca1eb3d0c754d2b3301fbfc3                         |
| id                          | 2e1b3676-ae28-44a4-8748-9f59609c7ef7                                             |
| image                       | N/A (booted from volume)                                                         |
| key_name                    | Linux                                                                            |
| name                        | Dulcinea                                                                         |
| progress                    | 0                                                                                |
| project_id                  | dab5156c32654875b6c54ce23c6712a2                                                 |
| properties                  |                                                                                  |
| status                      | ACTIVE                                                                           |
| updated                     | 2020-11-11T11:20:56Z                                                             |
| user_id                     | fc679f920848637f7d23fefdfbce7336b666cdc265f8bc32f51552cbcebbc3a7                 |
| volumes_attached            | id='72ae6142-f577-4753-a0f1-8bbfb3fd17bb'                                        |
+-----------------------------+----------------------------------------------------------------------------------+
{% endhighlight %}

Efectivamente, podemos concluir que el grupo de seguridad ha sido correctamente eliminado, ya que no aparece ningún apartado de nombre **security_groups**.

Ésto no es todo, ya que también tenemos que deshabilitar la seguridad en los correspondientes puertos de la máquina, ya que cuando una máquina no tiene ningún grupo de seguridad asignado, el _modus operandi_ por defecto es el de bloquear la conexión en todos los puertos asociados a la misma. Para ello, vamos a listar primero los puertos existentes en nuestro proyecto, ejecutando el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack port list
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                       | Status |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
| 0aab3e9c-c2c9-46c2-b6ed-e33447107554 |      | fa:16:3e:b5:5d:e7 | ip_address='10.0.1.9', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| 2dc799e2-63b1-44fe-b724-0888132a1e22 |      | fa:16:3e:3b:bb:e2 | ip_address='10.0.1.5', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| 488dc43f-3a9b-4981-8cf7-bc6cbcf82203 |      | fa:16:3e:e2:f6:e7 | ip_address='10.0.0.7', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 5929e26d-ebf9-42b3-81be-0ab52eac06bc |      | fa:16:3e:15:08:1b | ip_address='10.0.0.1', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 66ef0647-c0b1-4a2c-8bfe-dd228c2fc83e |      | fa:16:3e:56:ee:36 | ip_address='10.0.0.2', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| 9bb65414-27ad-427d-a6bb-08c1c4cb0d6a |      | fa:16:3e:c8:fd:80 | ip_address='10.0.0.5', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| a6b28726-da2a-4b2e-8a88-90bfc56e47a7 |      | fa:16:3e:51:3b:85 | ip_address='10.0.0.13', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299' | ACTIVE |
| c55c80c8-7561-43db-ae96-05f8efa43ce1 |      | fa:16:3e:6d:ae:1e | ip_address='10.0.1.2', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| c7c9209b-4df1-46c8-be51-226323baa9e8 |      | fa:16:3e:10:d8:23 | ip_address='10.0.1.4', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0'  | ACTIVE |
| d03ae5f2-8fa8-4fc3-bd57-6b20bc0f2fdc |      | fa:16:3e:f8:f2:e8 | ip_address='10.0.0.3', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
| eba43a24-dc3b-40dc-a700-3c3080e62b22 |      | fa:16:3e:83:7c:6e | ip_address='10.0.0.8', subnet_id='747952ea-a0a0-4bd5-8ad6-8dec0666a299'  | ACTIVE |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------+--------+
{% endhighlight %}

Aquí podemos ver todos los puertos existentes en nuestro proyecto, por lo que tendremos que buscar los identificadores (**ID**) de aquellos puertos asociados a Dulcinea, es decir, los asociados a las direcciones IP **10.0.0.7** y **10.0.1.9**:

* **10.0.0.7**: 488dc43f-3a9b-4981-8cf7-bc6cbcf82203
* **10.0.1.9**: 0aab3e9c-c2c9-46c2-b6ed-e33447107554

Tras ello, tendremos que deshabilitar la seguridad en dichos puertos, indicando para ello el identificador de los mismos, haciendo uso de los comandos:

{% highlight shell %}
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 488dc43f-3a9b-4981-8cf7-bc6cbcf82203
(openstackclient) alvaro@debian:~/virtualenv$ openstack port set --disable-port-security 0aab3e9c-c2c9-46c2-b6ed-e33447107554
{% endhighlight %}

Listo, ya hemos deshabilitado la seguridad en los mismos, de manera que ya tenemos accesible todo el rango de puertos, así que ya podemos comenzar a configurar NAT en **Dulcinea**. Es importante mencionar que deshabilitar el cortafuegos es una acción un tanto precipitada, por lo que únicamente debe llevarse a cabo en un entorno controlado.

Lo primero que tendremos que hacer será activar el **_bit de forward_**, permitiendo así que los paquetes puedan pasar de una interfaz a otra en el servidor, ya que por defecto, en Debian viene deshabilitado por razones de seguridad. Además, haremos uso de `sysctl` para que la modificación sea persistente tras un posible reinicio. Para ello, ejecutamos el comando (con permisos de administrador, ejecutando para ello el comando `sudo su -`):

{% highlight shell %}
root@dulcinea:~# echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
{% endhighlight %}

Gracias a la instrucción que acabamos de ejecutar, hemos añadido una línea al fichero de configuración **sysctl.conf**, el cuál contiene los valores de los parámetros del _kernel_, poniendo el valor del parámetro correspondiente para habilitar el **_bit de forward_** a **1**. Sin embargo, el cambio todavía no ha entrado en vigor, ya que tendremos que volver a leer el contenido de dicho fichero para así cargar el valor del parámetro en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# sysctl -p
net.ipv4.ip_forward = 1
{% endhighlight %}

Como se puede apreciar, la salida del comando nos ha informado de que se ha percatado del cambio que acabamos de realizar en el fichero, por lo que el **_bit de forward_** se encuentra ya habilitado.

Tras ello, tendremos que instalar el paquete que nos va a permitir hacer NAT. En este caso se trata de **nftables**, un cortafuegos que consta con dicha funcionalidad. Para ello, actualizaremos previamente la paquetería para asegurarnos de que estamos trabajando con las últimas versiones e instalaremos el paquete:

{% highlight shell %}
root@dulcinea:~# apt update && apt upgrade && apt install nftables
{% endhighlight %}

Cuando el paquete haya terminado de instalarse, tendremos que arrancar y habilitar el demonio, de manera que se inicie cada vez que se arranque la máquina y lea la configuración almacenada en el fichero (lo veremos a continuación). Para ello, ejecutamos los comandos:

{% highlight shell %}
root@dulcinea:~# systemctl start nftables.service
root@dulcinea:~# systemctl enable nftables.service
{% endhighlight %}

Una vez habilitado y arrancado el demonio, tendremos que crear una nueva tabla en _nftables_ de nombre **nat**. Para ello, ejecutamos el comando:

{% highlight shell %}
root@dulcinea:~# nft add table nat
{% endhighlight %}

Para verificar que la creación de dicha tabla se ha realizado correctamente, ejecutamos el comando:

{% highlight shell %}
root@dulcinea:~# nft list tables
table inet filter
table ip nat
{% endhighlight %}

Efectivamente, así ha sido. Dentro de dicha tabla tendremos que crear la cadena **POSTROUTING**, es decir, aquella que permite modificar paquetes justo antes de que salgan del equipo, permitiendo por tanto hacer **Source NAT** (_SNAT_), para posteriormente alojar dentro de la misma, la regla que necesitamos. Para crear dicha cadena ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
{% endhighlight %}

En este caso, hemos especificado que esta cadena sea poco prioritaria (a mayor sea el número, menor es la prioridad), de manera que si en un futuro quisiéramos añadir una cadena **PREROUTING**, es decir, aquella que permite modificar paquetes entrantes antes de que se tome una decisión de enrutamiento, permitiendo por tanto hacer **Destination NAT** (DNAT), bastaría con indicarle a ésta última una prioridad más alta, de manera que las reglas albergadas en el interior de la cadena **PREROUTING** se ejecuten antes que las de la cadena **POSTROUTING**.

De nuevo, vamos a verificar que dicha cadena se ha generado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# nft list chains
table inet filter {
    chain input {
            type filter hook input priority 0; policy accept;
    }
    chain forward {
            type filter hook forward priority 0; policy accept;
    }
    chain output {
            type filter hook output priority 0; policy accept;
    }
}
table ip nat {
    chain postrouting {
            type nat hook postrouting priority 100; policy accept;
    }
}
{% endhighlight %}

Efectivamente, así ha sido. Nos queda un último paso, añadir la regla necesaria a la cadena **POSTROUTING**, de manera que nos permita hacer _SNAT_ a la dirección de la interfaz de red perteneciente a la red externa, en este caso, la **10.0.0.7**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nft add rule ip nat postrouting oifname "eth0" ip saddr 10.0.1.0/24 counter snat to 10.0.0.7
{% endhighlight %}

En este caso, hemos especificado que se trata de una regla para la cadena **postrouting** (pues debe aplicarse justo antes de salir de la máquina), además, la interfaz (_oifname_) que debemos introducir será aquella por la que van a salir los paquetes, es decir, la que está conectada a Internet (aunque sea haciendo doble o triple NAT, no es algo relevante en estas circunstancias), en este caso, **eth0**, además de indicar que esta regla la vamos a aplicar a todos aquellos paquetes que provengan de la red (_ip saddr_) **10.0.1.0/24**. Por último, para aquellos paquetes que cumplan dicha regla, vamos a contarlos (**counter**) y a hacer _SNAT_ a la dirección IP perteneciente a la red exterior (_snat to_) **10.0.0.7**.

Por último, vamos a volver a verificar que dicha regla se ha añadido correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset
table inet filter {
    chain input {
            type filter hook input priority 0; policy accept;
    }

    chain forward {
            type filter hook forward priority 0; policy accept;
    }

    chain output {
            type filter hook output priority 0; policy accept;
    }
}
table ip nat {
    chain postrouting {
            type nat hook postrouting priority 100; policy accept;
            oifname "eth0" ip saddr 10.0.1.0/24 counter packets 0 bytes 0 snat to 10.0.0.7
    }
}
{% endhighlight %}

Efectivamente, así ha sido. Al igual que pasaba con el **_bit de forward_**, ésta configuración se encuentra cargada en memoria, por lo que para que conseguir que perdure en el tiempo, vamos a ejecutar el comando:

{% highlight shell %}
root@dulcinea:~# nft list ruleset > /etc/nftables.conf
{% endhighlight %}

Gracias a ello, habremos guardado la configuración en un fichero en **/etc/** de nombre **nftables.conf** que se importará de forma automática cuando reiniciemos la máquina gracias al _daemon_ que se encuentra habilitado. En caso de que no cargase de nuevo la configuración, podríamos hacerlo manualmente con el comando `nft -f /etc/nftables.conf`.

## Modificación de las instancias para que usen direccionamiento estático

Las direcciones IP asignadas a las interfaces de red de las máquinas han sido concedidas por DHCP, pero como todos sabemos, no es buena idea dejar direcciones dinámicas en los servidores, por lo que vamos a configurar direccionamiento estático en todas ellas (concretamente haciendo uso de las mismas direcciones que se nos han concedido por DHCP).

### Dulcinea

El fichero de configuración en el que debemos establecer la configuración de las interfaces de red de una máquina Debian Buster es **/etc/network/interfaces**, así que lo modificaremos ejecutando el comando:

{% highlight shell %}
root@dulcinea:~# nano /etc/network/interfaces
{% endhighlight %}

Dentro del mismo debemos introducir las siguientes líneas:

{% highlight shell %}
allow-hotplug eth0
iface eth0 inet static
        address 10.0.0.7
        netmask 255.255.255.0
        gateway 10.0.0.1

allow-hotplug eth1
iface eth1 inet static
        address 10.0.1.9
        netmask 255.255.255.0
{% endhighlight %}

Como se puede apreciar, hemos declarado dos interfaces de red (una perteneciente a la red externa, **eth0** y otra perteneciente a la red interna, **eth1**), cuya dirección IP estática es la misma que se le concedió en su momento por DHCP (**10.0.0.7** para **eth0** y **10.0.1.9** para **eth1**). Es importante mencionar que la puerta de enlace únicamente debe ser configurada para la interfaz **eth0**, que es la que tiene salida a Internet, concretamente a través del router situado en **10.0.0.1**.

Las instancias Debian Buster de OpenStack vienen con un fichero **/etc/resolv.conf** dinámico, lo que quiere decir que se genera dinámicamente en cada arranque. El contenido del mismo incluye los dos servidores DNS existentes en el instituto, situados en las direcciones **192.168.202.2** y **192.168.200.2**, pero personalmente, he tomado la decisión de añadir un tercer servidor DNS ubicado en Internet, para aumentar la disponibilidad. Al ser dinámico dicho fichero, no podemos modificarlo a mano ni tampoco indicar el servidor DNS en el **/etc/network/interfaces**, sino que tendremos que modificar el fichero **/etc/resolvconf/resolv.conf.d/base** e indicarlo ahí, de manera que lo tendrá en cuenta para la siguiente ocasión, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# nano /etc/resolvconf/resolv.conf.d/base
{% endhighlight %}

En este caso, el servidor DNS que he decidido añadir es **8.8.8.8**, por lo que la apariencia de dicho fichero sería:

{% highlight shell %}
nameserver 8.8.8.8
{% endhighlight %}

Genial, todos los cambios se han llevado a cabo, pero para que surtan efecto podemos reiniciar (comando `reboot`) o bien reiniciar el servicio correspondiente, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# systemctl restart networking
{% endhighlight %}

Tras unos segundos de espera mientras el servicio se reiniciaba, ya podremos apreciar los cambios que hemos realizado, ejecutando antes de nada el comando `ip a` para verificar que las direcciones IP estáticas se han asignado correctamente:

{% highlight shell %}
root@dulcinea:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:e2:f6:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.7/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee2:f6e7/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:b5:5d:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.9/24 brd 10.0.1.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:feb5:5de7/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, las direcciones han sido correctamente configuradas estáticamente (podemos deducir que son estáticas ya que su **valid_lft** es **forever**). Además, vamos a comprobar que la puerta de enlace predeterminada también se ha configurado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# ip r
default via 10.0.0.1 dev eth0 onlink 
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.7 
10.0.1.0/24 dev eth1 proto kernel scope link src 10.0.1.9 
169.254.169.254 via 10.0.0.1 dev eth0
{% endhighlight %}

Efectivamente, así ha sido. Por último, vamos a comprobar los servidores DNS que se encuentran indicados en el fichero **/etc/resolv.conf**, haciendo uso del comando:

{% highlight shell %}
root@dulcinea:~# cat /etc/resolv.conf 
nameserver 192.168.202.2
nameserver 192.168.200.2
nameserver 8.8.8.8
search openstacklocal
{% endhighlight %}

Como era de esperar, los servidores DNS se han configurado como deberían, además de haberse añadido el servidor **8.8.8.8**, tal y como lo hemos establecido.

### Sancho

Antes de llevar a cabo la configuración, vamos a comprobar que no tenemos conectividad con el exterior (todavía), tratando de hacer un `ping` a **8.8.8.8**, por ejemplo:

{% highlight shell %}
root@sancho:~# ping 8.8.8.8
ping: connect: Network is unreachable
{% endhighlight %}

Efectivamente, la red es inaccesible, ya que ni siquiera tenemos configurada una puerta de enlace predeterminada.

El fichero de configuración en el que debemos establecer la configuración de las interfaces de red de una máquina Ubuntu 20.04 es **/etc/netplan/50-cloud-init.yaml**, pero al igual que ocurría con el fichero **/etc/resolv.conf** en la máquina Debian Buster, éste también se genera dinámicamente durante el arranque gracias a un estándar de nombre **cloud-init**, que debemos desactivar ejecutando el comando:

{% highlight shell %}
root@sancho:~# echo 'network: {config: disabled}' > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
{% endhighlight %}

En realidad, lo que hemos hecho ha sido generar un fichero de nombre **99-disable-network-config.cfg** en **/etc/cloud/cloud.cfg.d/**, con un determinado contenido que evita por lo tanto que se genere el fichero de configuración de red durante cada arranque, permitiendo así llevar a cabo configuraciones estáticas que perduren en el tiempo. Una vez que lo hayamos desactivado, ya podremos proceder a modificar el fichero **/etc/netplan/50-cloud-init.yaml**, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# nano /etc/netplan/50-cloud-init.yaml
{% endhighlight %}

Dentro del mismo debemos introducir las siguientes líneas:

{% highlight shell %}
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: false
            match:
                macaddress: fa:16:3e:10:d8:23
            mtu: 8950
            set-name: ens3
            addresses: [10.0.1.4/24]
            gateway4: 10.0.1.9
            nameservers:
                addresses: [192.168.202.2, 192.168.200.2, 8.8.8.8]
{% endhighlight %}

Como se puede apreciar, hemos declarado la única interfaz de red (correspondiente a la red interna, **ens3**), cuya dirección IP estática es la misma que se le concedió en su momento por DHCP (**10.0.1.4**). Es importante mencionar que la puerta de enlace ha de ser la dirección de la máquina **Dulcinea** perteneciente a la red interna (concretamente, **10.0.1.9**), de manera que podrá hacer _SNAT_ con los paquetes que le lleguen. Por último, estableceremos los dos servidores DNS existentes en el instituto, **192.168.202.2** y **192.168.200.2**, además de un servidor extra para así aumentar la disponibilidad, como puede ser **8.8.8.8**.

Genial, todos los cambios se han llevado a cabo, pero para que surtan efecto podemos reiniciar (comando `reboot`) o bien volver a cargar la configuración del servicio correspondiente, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@sancho:~# netplan apply
{% endhighlight %}

Tras unos segundos de espera mientras la nueva configuración se aplicaba, ya podremos apreciar los cambios que hemos realizado, ejecutando antes de nada el comando `ip a` para verificar que la dirección IP estática se ha asignado correctamente:

{% highlight shell %}
root@sancho:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:10:d8:23 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.4/24 brd 10.0.1.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe10:d823/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la dirección ha sido correctamente configurada estáticamente (podemos deducir que es estática ya que su **valid_lft** es **forever**). Además, vamos a comprobar que la puerta de enlace predeterminada también se ha configurado correctamente, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# ip r
default via 10.0.1.9 dev ens3 proto static 
10.0.1.0/24 dev ens3 proto kernel scope link src 10.0.1.4
{% endhighlight %}

Efectivamente, así ha sido. Por último, vamos a comprobar los servidores DNS que se encuentran indicados en el fichero **/etc/resolv.conf**, haciendo uso del comando:

{% highlight shell %}
root@sancho:~# cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
{% endhighlight %}

En este caso, los servidores DNS no aparecen en el fichero **/etc/resolv.conf** ya que Ubuntu implementa por defecto un servidor DNS local, capaz de cachear las respuestas, consiguiendo por tanto una navegación más rápida, pero que internamente, hará uso de los servidores DNS que le hemos especificado anteriormente para dichas peticiones.

Para verificar que toda la configuración ha funcionado, vamos a tratar de hacer `ping` a **google.es**, para así verificar también que la resolución de nombres se realiza correctamente:

{% highlight shell %}
root@sancho:~# ping google.es
PING google.es (172.217.17.3) 56(84) bytes of data.
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=1 ttl=112 time=43.8 ms
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=2 ttl=112 time=50.7 ms
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=3 ttl=112 time=43.5 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2818ms
rtt min/avg/max/mdev = 43.478/45.970/50.651/3.312 ms
{% endhighlight %}

Como era de esperar, ya tenemos salida a Internet a través de **Dulcinea**, así que aprovecharemos y llevaremos a cabo una actualización de la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# apt update && apt upgrade
{% endhighlight %}

Tras unos segundos, la paquetería se habrá actualizado y ya estaremos utilizando las últimas versiones de los correspondientes paquetes.

### Quijote

Antes de llevar a cabo la configuración, vamos a comprobar que no tenemos conectividad con el exterior (todavía), tratando de hacer un `ping` a **8.8.8.8**, por ejemplo:

{% highlight shell %}
[root@quijote ~]# ping 8.8.8.8
connect: Network is unreachable
{% endhighlight %}

Efectivamente, la red es inaccesible, ya que ni siquiera tenemos configurada una puerta de enlace predeterminada.

El fichero de configuración en el que debemos establecer la configuración de las interfaces de red de una máquina CentOS 7 es **/etc/sysconfig/network-scripts/ifcfg-eth0**, pero al igual que ocurría en la máquina **Sancho**, éste fichero también se genera dinámicamente durante el arranque gracias al estándar **cloud-init**, que debemos desactivar ejecutando el comando:

{% highlight shell %}
[root@quijote ~]# touch /etc/cloud/cloud-init.disabled
{% endhighlight %}

En realidad, lo que hemos hecho ha sido generar un fichero de nombre **cloud-init.disabled** en **/etc/cloud/**, que evita por lo tanto que se genere el fichero de configuración de red durante cada arranque, permitiendo así llevar a cabo configuraciones estáticas que perduren en el tiempo. Una vez que lo hayamos desactivado, ya podremos proceder a modificar el fichero **/etc/sysconfig/network-scripts/ifcfg-eth0**, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth0
{% endhighlight %}

Dentro del mismo debemos introducir las siguientes líneas:

{% highlight shell %}
BOOTPROTO=none
DEVICE=eth0
HWADDR=fa:16:3e:3b:bb:e2
MTU=8950
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
IPADDR=10.0.1.5
PREFIX=24
GATEWAY=10.0.1.9
DNS1=192.168.202.2
DNS2=192.168.200.2
DNS3=8.8.8.8
{% endhighlight %}

Como se puede apreciar, hemos declarado la única interfaz de red (correspondiente a la red interna, **eth0**), cuya dirección IP estática es la misma que se le concedió en su momento por DHCP (**10.0.1.5**). Es importante mencionar que la puerta de enlace ha de ser la dirección de la máquina **Dulcinea** perteneciente a la red interna (concretamente, **10.0.1.9**), de manera que podrá hacer _SNAT_ con los paquetes que le lleguen. Por último, estableceremos los dos servidores DNS existentes en el instituto, **192.168.202.2** y **192.168.200.2**, además de un servidor extra para así aumentar la disponibilidad, como puede ser **8.8.8.8**.

Genial, todos los cambios se han llevado a cabo, pero para que surtan efecto podemos reiniciar (comando `reboot`) o bien reiniciar el servicio correspondiente, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# systemctl restart network
{% endhighlight %}

Tras unos segundos de espera mientras el servicio se reiniciaba, ya podremos apreciar los cambios que hemos realizado, ejecutando antes de nada el comando `ip a` para verificar que la dirección IP estática se ha asignado correctamente:

{% highlight shell %}
[root@quijote ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:3b:bb:e2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.5/24 brd 10.0.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe3b:bbe2/64 scope link 
       valid_lft forever preferred_lft forever
{% endhighlight %}

Como se puede apreciar, la dirección ha sido correctamente configurada estáticamente (podemos deducir que es estática ya que su **valid_lft** es **forever**). Además, vamos a comprobar que la puerta de enlace predeterminada también se ha configurado correctamente, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# ip r
default via 10.0.1.9 dev eth0 
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.5
{% endhighlight %}

Efectivamente, así ha sido. Por último, vamos a comprobar los servidores DNS que se encuentran indicados en el fichero **/etc/resolv.conf**, haciendo uso del comando:

{% highlight shell %}
[root@quijote ~]# cat /etc/resolv.conf
search openstacklocal
nameserver 192.168.202.2
nameserver 192.168.200.2
nameserver 8.8.8.8
{% endhighlight %}

Como era de esperar, los servidores DNS se han configurado como deberían, además de haberse añadido el servidor **8.8.8.8**, tal y como lo hemos establecido.

Para verificar que toda la configuración ha funcionado, vamos a tratar de hacer `ping` a **google.es**, para así verificar también que la resolución de nombres se realiza correctamente:

{% highlight shell %}
[root@quijote ~]# ping google.es
PING google.es (172.217.17.3) 56(84) bytes of data.
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=1 ttl=112 time=43.4 ms
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=2 ttl=112 time=43.5 ms
64 bytes from mad07s09-in-f3.1e100.net (172.217.17.3): icmp_seq=3 ttl=112 time=43.7 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 43.472/43.574/43.747/0.122 ms
{% endhighlight %}

Como era de esperar, ya tenemos salida a Internet a través de **Dulcinea**, así que aprovecharemos y llevaremos a cabo una actualización de la paquetería existente en la máquina, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# yum update
{% endhighlight %}

Tras unos segundos, la paquetería se habrá actualizado y ya estaremos utilizando las últimas versiones de los correspondientes paquetes.

## Modificación de la subred de la red interna, deshabilitando el servidor DHCP

Ya hemos configurado el direccionamiento estático en todas las máquinas, por lo que no será necesario tener activo el servidor DHCP, pues no le daremos ningún uso. Para deshabilitarlo, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, buscaremos la red interna (**red interna de alvaro.vaca**) y pulsaremos en el nombre de la misma. Cuando hayamos accedido a la red, tendremos varios subapartados, así que accederemos a **Subredes** y nos aparecerá la única que hemos creado, por lo que procederemos a editarla pulsando en **Editar subred**.

Se nos habrá abierto un pequeño menú, así que iremos a **Detalles de subred**, pues es donde se encuentra la configuración referente al servidor DHCP de la subred. Simplemente desmarcaremos la casilla **Habilitar DHCP** y guardaremos los cambios, pulsando para ello en **Guardar**. El servidor DHCP ya habrá sido deshabilitado, tal y como se puede apreciar:

![dhcp1](https://i.ibb.co/pKw0nwk/Captura-de-pantalla-de-2020-11-12-09-49-32.png "Deshabilitar DHCP")

## Creación del usuario profesor en todas las instancias

En este caso, vamos a generar un nuevo usuario **profesor** en las tres instancias, en el que añadiremos las claves públicas de los profesores para que puedan acceder. Además, llevaremos a cabo la correspondiente configuración para que dicho usuario pueda ejecutar `sudo` sin contraseña. La sintaxis para generar un nuevo usuario es (dicha sintaxis es genérica, por lo que nos servirá en las tres distribuciones):

{% highlight shell %}
useradd <usuario> -m -s <shell>
{% endhighlight %}

Donde:

* **usuario**: Indicamos el usuario que queremos crear. En este caso, **profesor**.
* **-m**: Especificamos que cree un directorio personal para dicho usuario en caso de que no exista.
* **-s**: Especificamos el nombre de la shell que utilizará para el login. En este caso, **/bin/bash**.

Suponemos que el usuario ya se encuentra creado, así que nos cambiaremos de usuario para hacer uso del mismo, siguiendo la sintaxis (dicha sintaxis es genérica, por lo que nos servirá en las tres distribuciones):

{% highlight shell %}
su - <usuario>
{% endhighlight %}

Donde:

* **usuario**: Indicamos el usuario al que nos queremos cambiar. En este caso, **profesor**.

Como bien sabemos, las claves públicas autorizadas a acceder a nuestra máquina se encuentran en **~/.ssh/**, dentro de un fichero de nombre **authorized_keys**, que actualmente no se encuentra creado, así que vamos a proceder a crear dicho directorio y dicho fichero e insertar las correspondientes claves públicas en su interior. Los comandos a ejecutar serían (dichos comandos son genéricos, por lo que nos servirán en las tres distribuciones, excepto en **CentOS 7**, que tendremos que utilizar `vi` en lugar de `nano`):

{% highlight shell %}
mkdir .ssh

nano .ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfk9mRtOHM3T1KpmGi0KiN2uAM6CDXM3WFcm1wkzKXx7RaLtf9pX+KCuVqHdy/N/9d9wtH7iSmLFX/4gQKQVG00jHiGf3ABufWeIpjmHtT1WaI0+vV47fofEIjDDfSZPlI3p5/c7tefHsIAK6GbQn31yepAcFYy9ZfqAh8H/Y5eLpf3egPZn9Czsvx+lm0I8Q+e/HSayRaiAPUukF57N2nnw7yhPZCHSZJqFbXyK3fVQ/UQVBeNS2ayp0my8X9sIBZnNkcYHFLIWBqJYdnu1ZFhnbu3yy94jmJdmELy3+54hqiwFEfjZAjUYSl8eGPixOfdTgc8ObbHbkHyIrQ91Kz rafa@eco
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmjoVIoZCx4QFXvljqozXGqxxlSvO7V2aizqyPgMfGqnyl0J9YXo6zrcWYwyWMnMdRdwYZgHqfiiFCUn2QDm6ZuzC4Lcx0K3ZwO2lgL4XaATykVLneHR1ib6RNroFcClN69cxWsdwQW6dpjpiBDXf8m6/qxVP3EHwUTsP8XaOV7WkcCAqfYAMvpWLISqYme6e+6ZGJUIPkDTxavu5JTagDLwY+py1WB53eoDWsG99gmvyit2O1Eo+jRWN+mgRHIxJTrFtLS6o4iWeshPZ6LvCZ/Pum12Oj4B4bjGSHzrKjHZgTwhVJ/LDq3v71/PP4zaI3gVB9ZalemSxqomgbTlnT jose@debian
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3AUDWjyPANntK+qwHmlJihKQZ1H+AGN02k06dzRHmkvWiNgou/VcCgowhMTGR+0I6nWVwgRSWKJEUEaMu1r9rEeL63GRtUSepCWpClHJG1CuySuJKVGtRdUq+/szDntpJnJW207a78hTeQLjQsyPvbOqkbulQG7xTRCycdT3bH2UO4JI2d+341gkOlxSG/stPQ52Dsbfb274oMRom5r5f2apD3wbfxE9A6qwm4m70G9NYS7T3uKgCiXegO/3GTJD4UbK0ylGUamG5obdS5yD8Ib12vRCCXWav23SAj/4f9MzAnXX8U4ATM/du2FHZBiIzWVH12LYvIEZpUIVYKPSf alberto@roma
{% endhighlight %}

El directorio **.ssh** y el fichero **authorized_keys** se encuentran ya creados, conteniendo éste último, además, las claves públicas de los profesores.

Los ficheros y directorios se crean por defecto con unos determinados permisos UNIX que no son los que nosotros queremos, pues dicha información ha de ser confidencial y únicamente accesible por el usuario **profesor**, de manera que cambiaremos los permisos del **directorio** a **700** y los permisos del **fichero** a **600**, ejecutando para ello los comandos (dichos comandos son genéricos, por lo que nos servirán en las tres distribuciones):

{% highlight shell %}
chmod 700 .ssh/

chmod 600 .ssh/authorized_keys
{% endhighlight %}

Listo, los permisos ya habrían sido modificados, así que nos queda un último paso, permitir que dicho usuario haga uso del comando `sudo` sin que se le solicite una contraseña (pues actualmente no tiene, de manera que el único método de acceso al mismo es mediante par de claves SSH). Para ello debemos modificar el fichero **/etc/sudoers** y añadir una línea de la siguiente forma:

{% highlight shell %}
<usuario> ALL=(ALL) NOPASSWD:ALL
{% endhighlight %}

Donde:

* **usuario**: Indicamos el usuario que queremos autorizar a ejecutar `sudo` sin contraseña. En este caso, **profesor**.

Gracias a `echo` podremos añadir dicha línea sin tener que modificar el fichero con un editor, ejecutando para ello el comando (dicho comando es genérico, por lo que nos servirá en las tres distribuciones):

{% highlight shell %}
echo "profesor ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
{% endhighlight %}

Tras ello, el usuario **profesor** ya podrá hacer uso de `sudo` sin que se le requiera una contraseña.

### Dulcinea

{% highlight shell %}
root@dulcinea:~# useradd profesor -m -s /bin/bash

root@dulcinea:~# su - profesor

profesor@dulcinea:~$ mkdir .ssh

profesor@dulcinea:~$ nano .ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfk9mRtOHM3T1KpmGi0KiN2uAM6CDXM3WFcm1wkzKXx7RaLtf9pX+KCuVqHdy/N/9d9wtH7iSmLFX/4gQKQVG00jHiGf3ABufWeIpjmHtT1WaI0+vV47fofEIjDDfSZPlI3p5/c7tefHsIAK6GbQn31yepAcFYy9ZfqAh8H/Y5eLpf3egPZn9Czsvx+lm0I8Q+e/HSayRaiAPUukF57N2nnw7yhPZCHSZJqFbXyK3fVQ/UQVBeNS2ayp0my8X9sIBZnNkcYHFLIWBqJYdnu1ZFhnbu3yy94jmJdmELy3+54hqiwFEfjZAjUYSl8eGPixOfdTgc8ObbHbkHyIrQ91Kz rafa@eco
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmjoVIoZCx4QFXvljqozXGqxxlSvO7V2aizqyPgMfGqnyl0J9YXo6zrcWYwyWMnMdRdwYZgHqfiiFCUn2QDm6ZuzC4Lcx0K3ZwO2lgL4XaATykVLneHR1ib6RNroFcClN69cxWsdwQW6dpjpiBDXf8m6/qxVP3EHwUTsP8XaOV7WkcCAqfYAMvpWLISqYme6e+6ZGJUIPkDTxavu5JTagDLwY+py1WB53eoDWsG99gmvyit2O1Eo+jRWN+mgRHIxJTrFtLS6o4iWeshPZ6LvCZ/Pum12Oj4B4bjGSHzrKjHZgTwhVJ/LDq3v71/PP4zaI3gVB9ZalemSxqomgbTlnT jose@debian
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3AUDWjyPANntK+qwHmlJihKQZ1H+AGN02k06dzRHmkvWiNgou/VcCgowhMTGR+0I6nWVwgRSWKJEUEaMu1r9rEeL63GRtUSepCWpClHJG1CuySuJKVGtRdUq+/szDntpJnJW207a78hTeQLjQsyPvbOqkbulQG7xTRCycdT3bH2UO4JI2d+341gkOlxSG/stPQ52Dsbfb274oMRom5r5f2apD3wbfxE9A6qwm4m70G9NYS7T3uKgCiXegO/3GTJD4UbK0ylGUamG5obdS5yD8Ib12vRCCXWav23SAj/4f9MzAnXX8U4ATM/du2FHZBiIzWVH12LYvIEZpUIVYKPSf alberto@roma

profesor@dulcinea:~$ chmod 700 .ssh/

profesor@dulcinea:~$ chmod 600 .ssh/authorized_keys
{% endhighlight %}

### Sancho

{% highlight shell %}
root@sancho:~# useradd profesor -m -s /bin/bash

root@sancho:~# su - profesor

profesor@sancho:~$ mkdir .ssh

profesor@sancho:~$ nano .ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfk9mRtOHM3T1KpmGi0KiN2uAM6CDXM3WFcm1wkzKXx7RaLtf9pX+KCuVqHdy/N/9d9wtH7iSmLFX/4gQKQVG00jHiGf3ABufWeIpjmHtT1WaI0+vV47fofEIjDDfSZPlI3p5/c7tefHsIAK6GbQn31yepAcFYy9ZfqAh8H/Y5eLpf3egPZn9Czsvx+lm0I8Q+e/HSayRaiAPUukF57N2nnw7yhPZCHSZJqFbXyK3fVQ/UQVBeNS2ayp0my8X9sIBZnNkcYHFLIWBqJYdnu1ZFhnbu3yy94jmJdmELy3+54hqiwFEfjZAjUYSl8eGPixOfdTgc8ObbHbkHyIrQ91Kz rafa@eco
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmjoVIoZCx4QFXvljqozXGqxxlSvO7V2aizqyPgMfGqnyl0J9YXo6zrcWYwyWMnMdRdwYZgHqfiiFCUn2QDm6ZuzC4Lcx0K3ZwO2lgL4XaATykVLneHR1ib6RNroFcClN69cxWsdwQW6dpjpiBDXf8m6/qxVP3EHwUTsP8XaOV7WkcCAqfYAMvpWLISqYme6e+6ZGJUIPkDTxavu5JTagDLwY+py1WB53eoDWsG99gmvyit2O1Eo+jRWN+mgRHIxJTrFtLS6o4iWeshPZ6LvCZ/Pum12Oj4B4bjGSHzrKjHZgTwhVJ/LDq3v71/PP4zaI3gVB9ZalemSxqomgbTlnT jose@debian
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3AUDWjyPANntK+qwHmlJihKQZ1H+AGN02k06dzRHmkvWiNgou/VcCgowhMTGR+0I6nWVwgRSWKJEUEaMu1r9rEeL63GRtUSepCWpClHJG1CuySuJKVGtRdUq+/szDntpJnJW207a78hTeQLjQsyPvbOqkbulQG7xTRCycdT3bH2UO4JI2d+341gkOlxSG/stPQ52Dsbfb274oMRom5r5f2apD3wbfxE9A6qwm4m70G9NYS7T3uKgCiXegO/3GTJD4UbK0ylGUamG5obdS5yD8Ib12vRCCXWav23SAj/4f9MzAnXX8U4ATM/du2FHZBiIzWVH12LYvIEZpUIVYKPSf alberto@roma

profesor@sancho:~$ chmod 700 .ssh/

profesor@sancho:~$ chmod 600 .ssh/authorized_keys
{% endhighlight %}

### Quijote

{% highlight shell %}
[root@quijote ~]# useradd profesor -m -s /bin/bash

[root@quijote ~]# su - profesor

[profesor@quijote ~]$ mkdir .ssh

[profesor@quijote ~]$ vi .ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfk9mRtOHM3T1KpmGi0KiN2uAM6CDXM3WFcm1wkzKXx7RaLtf9pX+KCuVqHdy/N/9d9wtH7iSmLFX/4gQKQVG00jHiGf3ABufWeIpjmHtT1WaI0+vV47fofEIjDDfSZPlI3p5/c7tefHsIAK6GbQn31yepAcFYy9ZfqAh8H/Y5eLpf3egPZn9Czsvx+lm0I8Q+e/HSayRaiAPUukF57N2nnw7yhPZCHSZJqFbXyK3fVQ/UQVBeNS2ayp0my8X9sIBZnNkcYHFLIWBqJYdnu1ZFhnbu3yy94jmJdmELy3+54hqiwFEfjZAjUYSl8eGPixOfdTgc8ObbHbkHyIrQ91Kz rafa@eco
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmjoVIoZCx4QFXvljqozXGqxxlSvO7V2aizqyPgMfGqnyl0J9YXo6zrcWYwyWMnMdRdwYZgHqfiiFCUn2QDm6ZuzC4Lcx0K3ZwO2lgL4XaATykVLneHR1ib6RNroFcClN69cxWsdwQW6dpjpiBDXf8m6/qxVP3EHwUTsP8XaOV7WkcCAqfYAMvpWLISqYme6e+6ZGJUIPkDTxavu5JTagDLwY+py1WB53eoDWsG99gmvyit2O1Eo+jRWN+mgRHIxJTrFtLS6o4iWeshPZ6LvCZ/Pum12Oj4B4bjGSHzrKjHZgTwhVJ/LDq3v71/PP4zaI3gVB9ZalemSxqomgbTlnT jose@debian
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3AUDWjyPANntK+qwHmlJihKQZ1H+AGN02k06dzRHmkvWiNgou/VcCgowhMTGR+0I6nWVwgRSWKJEUEaMu1r9rEeL63GRtUSepCWpClHJG1CuySuJKVGtRdUq+/szDntpJnJW207a78hTeQLjQsyPvbOqkbulQG7xTRCycdT3bH2UO4JI2d+341gkOlxSG/stPQ52Dsbfb274oMRom5r5f2apD3wbfxE9A6qwm4m70G9NYS7T3uKgCiXegO/3GTJD4UbK0ylGUamG5obdS5yD8Ib12vRCCXWav23SAj/4f9MzAnXX8U4ATM/du2FHZBiIzWVH12LYvIEZpUIVYKPSf alberto@roma

[profesor@quijote ~]$ chmod 700 .ssh/

[profesor@quijote ~]$ chmod 600 .ssh/authorized_keys
{% endhighlight %}

## Configuración de resolución estática en las tres instancias

Dado que todavía no hemos configurado un servidor DNS, tendremos que configurar la resolución estática de nombres en las máquinas, de manera que se conozcan entre ellas. Dicha información está almacenada en el fichero **/etc/hosts**, así que tendremos que incluir en el mismo líneas con la siguiente sintaxis:

{% highlight shell %}
<IP> <FQDN> <hostname>
{% endhighlight %}

Donde:

* **IP**: La IP de la máquina que queremos resolver.
* **FQDN**: El _Fully Qualified Domain Name_, que se encuentra compuesto por **[hostname].alvaro.vaca.gonzalonazareno.org**.
* **hostname**: El nombre corto de la máquina.

En cada máquina debemos introducir dos líneas (de manera que conozca las otras dos máquinas restantes, además de a sí misma, modificando o añadiendo la línea **127.0.1.1**), es decir:

* **Dulcinea**: Debe conocer a Sancho, a Quijote y a sí misma.
* **Sancho**: Debe conocer a Dulcinea, a Quijote y a sí misma.
* **Quijote**: Debe conocer a Dulcinea, a Sancho y a sí misma.

Para llevar a cabo dicha modificación, haremos uso del comando (dicho comando es genérico, excepto para **CentOS 7**, que tendremos que utilizar `vi` en lugar de `nano`):

{% highlight shell %}
nano /etc/hosts
{% endhighlight %}

Tras ello, ya podremos hacer `ping`, por ejemplo, y las máquinas deben responder en caso de hacer uso tanto del **FQDN** como del **hostname**.

### Dulcinea

En la máquina Debian Buster hay una pequeña peculiaridad, y es que el fichero **/etc/hosts** se genera dinámicamente durante el arranque, gracias al estándar **cloud-init**, así que para deshabilitarlo y así conseguir un fichero estático, tendremos que cambiar el valor de la directiva **manage_etc_hosts** a **false** en el fichero **/etc/cloud/cloud.cfg**. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# sed -i 's/manage_etc_hosts: true/manage_etc_hosts: false/g' /etc/cloud/cloud.cfg
{% endhighlight %}

Para verificar que el valor de dicha directiva ha sido modificado, vamos a visualizar el contenido de dicho fichero, estableciendo un filtro por nombre:

{% highlight shell %}
root@dulcinea:~# cat /etc/cloud/cloud.cfg | egrep 'manage_etc_hosts'
manage_etc_hosts: false
{% endhighlight %}

Efectivamente, su valor ha sido modificado, así que ya podemos proceder a modificar el fichero **/etc/hosts** y realizar las correspondientes pruebas:

{% highlight shell %}
root@dulcinea:~# nano /etc/hosts
127.0.1.1 dulcinea.alvaro.vaca.gonzalonazareno.org dulcinea.novalocal dulcinea
127.0.0.1 localhost
10.0.1.4 sancho.alvaro.vaca.gonzalonazareno.org sancho
10.0.1.5 quijote.alvaro.vaca.gonzalonazareno.org quijote

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@dulcinea:~# ping sancho
PING sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=64 time=2.31 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=64 time=1.21 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=64 time=1.17 ms
^C
--- sancho.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.173/1.562/2.308/0.527 ms

root@dulcinea:~# ping quijote.alvaro.vaca.gonzalonazareno.org
PING quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5) 56(84) bytes of data.
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=1 ttl=64 time=2.05 ms
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=2 ttl=64 time=0.875 ms
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=3 ttl=64 time=0.592 ms
^C
--- quijote.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.592/1.173/2.054/0.634 ms
{% endhighlight %}

### Sancho

{% highlight shell %}
root@sancho:~# sudo nano /etc/hosts
127.0.1.1 sancho.alvaro.vaca.gonzalonazareno.org sancho
127.0.0.1 localhost
10.0.1.9 dulcinea.alvaro.vaca.gonzalonazareno.org dulcinea
10.0.1.5 quijote.alvaro.vaca.gonzalonazareno.org quijote

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@sancho:~# ping dulcinea.alvaro.vaca.gonzalonazareno.org
PING dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=1 ttl=64 time=0.724 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=2 ttl=64 time=1.22 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=3 ttl=64 time=0.961 ms
^C
--- dulcinea.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.724/0.967/1.217/0.201 ms

root@sancho:~# ping quijote
PING quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5) 56(84) bytes of data.
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=1 ttl=64 time=3.37 ms
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=2 ttl=64 time=1.02 ms
64 bytes from quijote.alvaro.vaca.gonzalonazareno.org (10.0.1.5): icmp_seq=3 ttl=64 time=1.10 ms
^C
--- quijote.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.022/1.830/3.369/1.088 ms
{% endhighlight %}

### Quijote

En las máquinas CentOS 7 de OpenStack existe una peculiaridad, y es que el **_hostname_** que se asigna por defecto durante la creación es **[hostname].novalocal**, cosa que no necesitamos, ya que llega a provocar incluso conflictos con el FQDN. Para llevar a cabo dicha modificación, podríamos hacer uso del comando `hostnamectl` o bien, modificar a mano el fichero **/etc/hostname**, que es lo que haré en mi caso. Para ello, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# vi /etc/hostname
{% endhighlight %}

Y modificaremos el contenido a lo siguiente:

{% highlight shell %}
quijote
{% endhighlight %}

Genial, ya hemos llevado a cabo el cambio necesario, pero para que surta efecto podemos reiniciar (comando `reboot`) o bien cerrar la sesión y volver a abrirla, que es lo que nosotros haremos, pues estamos intentando simular una situación real, con servidores reales. En caso de que tampoco muestre el nuevo _hostname_, podríamos modificar directamente el fichero **/proc/sys/kernel/hostname**, de manera que el cambio entraría en vigor automáticamente. Tras ello, ya podremos proceder a llevar a cabo las modificaciones oportunas para establecer la resolución estática:

{% highlight shell %}
[root@quijote ~]# sudo vi /etc/hosts
127.0.1.1 quijote.alvaro.vaca.gonzalonazareno.org quijote
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.1.9 dulcinea.alvaro.vaca.gonzalonazareno.org dulcinea
10.0.1.4 sancho.alvaro.vaca.gonzalonazareno.org sancho

[root@quijote ~]# ping dulcinea
PING dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=1 ttl=64 time=0.466 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=2 ttl=64 time=0.688 ms
64 bytes from dulcinea.alvaro.vaca.gonzalonazareno.org (10.0.1.9): icmp_seq=3 ttl=64 time=0.651 ms
^C
--- dulcinea.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.466/0.601/0.688/0.101 ms

[root@quijote ~]# ping sancho.alvaro.vaca.gonzalonazareno.org
PING sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4) 56(84) bytes of data.
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=1 ttl=64 time=2.29 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=2 ttl=64 time=1.01 ms
64 bytes from sancho.alvaro.vaca.gonzalonazareno.org (10.0.1.4): icmp_seq=3 ttl=64 time=1.19 ms
^C
--- sancho.alvaro.vaca.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 1.011/1.501/2.295/0.566 ms
{% endhighlight %}

## Sincronización del reloj de las instancias utilizando un servidor NTP externo

El _Network Time Protocol_ (**NTP**) es un protocolo de Internet para sincronizar los relojes de los sistemas informáticos muy utilizado hoy en día, principalmente en empresas, pues es altamente recomendable que todas las máquinas existentes tengan la hora sincronizada entre sí.

### Dulcinea

Para comprobar si nuestro servidor tiene actualmente sincronizado el reloj haciendo uso de un servidor NTP, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# timedatectl
               Local time: Sat 2020-11-14 19:48:01 UTC
           Universal time: Sat 2020-11-14 19:48:01 UTC
                 RTC time: Sat 2020-11-14 19:48:02
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: no
{% endhighlight %}

Como se puede apreciar, nos ha mostrado la información referente a la hora de nuestra máquina. Sin embargo, podemos apreciar que el estado del **NTP service** es **inactive**, y que por tanto el reloj del sistema no se encuentra sincronizado (**System clock synchronized**).

Para tratar de solucionarlo, vamos a ver el estado del servicio **systemd-timesyncd.service**, uno de los _daemon_ que se puede encargar de gestionar la hora en las máquinas, para aquellas que estén corriendo **systemd** (pues no es necesario hacer uso del _daemon_ **ntpd**). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: inactive (dead)
     Docs: man:systemd-timesyncd.service(8)
{% endhighlight %}

Como se puede apreciar, el servicio se encuentra actualmente **inactivo**, por lo que trataremos de reiniciarlo para que así arranque, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# systemctl restart systemd-timesyncd.service
{% endhighlight %}

Una vez más, mostraremos el estado del servicio para verificar si ha podido arrancar:

{% highlight shell %}
root@dulcinea:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: inactive (dead)
Condition: start condition failed at Sat 2020-11-14 19:51:08 UTC; 5s ago
           └─ ConditionFileIsExecutable=!/usr/sbin/ntpd was not met
     Docs: man:systemd-timesyncd.service(8)

Nov 14 19:50:05 dulcinea systemd[1]: Condition check resulted in Network Time Synchronization being skipped.
Nov 14 19:51:08 dulcinea systemd[1]: Condition check resulted in Network Time Synchronization being skipped.
{% endhighlight %}

Como se puede apreciar, en ésta ocasión tampoco ha podido arrancar, ya que aparentemente hay un conflicto entre **ntpd** y **systemd-timesyncd**, por lo que vamos a eliminar el primero de ellos, ejecutando para ello el comando:

{% highlight shell %}
root@dulcinea:~# apt purge ntp
{% endhighlight %}

Antes de tratar de arrancar de nuevo el servicio, vamos a modificar el fichero de configuración del mismo (**/etc/systemd/timesyncd.conf**) para así indicarle el servidor que tendrá que usar para la sincronización. Lo recomendable es que sea lo más cercano a nosotros geográficamente hablando. En este caso, he elegido **es.pool.ntp.org**, así que para modificar dicho fichero ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# nano /etc/systemd/timesyncd.conf

[Time]
NTP=es.pool.ntp.org
{% endhighlight %}

Listo, el servidor que ha de usar ya ha sido especificado, así que volveremos a reiniciar el servicio y a comprobar su estado:

{% highlight shell %}
root@dulcinea:~# systemctl restart systemd-timesyncd.service

root@dulcinea:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
           └─disable-with-time-daemon.conf
   Active: active (running) since Sat 2020-11-14 19:54:09 UTC; 2s ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 1066 (systemd-timesyn)
   Status: "Synchronized to time server for the first time 162.159.200.123:123 (es.pool.ntp.org)."
    Tasks: 2 (limit: 562)
   Memory: 1.2M
   CGroup: /system.slice/systemd-timesyncd.service
           └─1066 /lib/systemd/systemd-timesyncd

Nov 14 19:54:09 dulcinea systemd[1]: Starting Network Time Synchronization...
Nov 14 19:54:09 dulcinea systemd[1]: Started Network Time Synchronization.
Nov 14 19:54:09 dulcinea systemd-timesyncd[1066]: Synchronized to time server for the first time 162.159.200.123:123 (es.pool.ntp.org).
{% endhighlight %}

En ésta ocasión, sí ha sido capaz de arrancar (**active**) y de sincronizarse con dicho servidor, tal y como podemos apreciar en el último mensaje mostrado. Todavía no hemos terminado, ya que cuando ejecuté el comando `timedatectl`, me mostraba una hora menos de la hora local, es decir, había un problema con la zona horaria, por lo que debemos modificarla manualmente. Para ver las zonas horarias disponibles, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# timedatectl list-timezones
{% endhighlight %}

En mi caso, me quedaré con la hora de España (**Europe/Madrid**), así que para modificar la zona horaria, ejecutaremos el comando:

{% highlight shell %}
root@dulcinea:~# timedatectl set-timezone Europe/Madrid
{% endhighlight %}

En un principio, la hora ya debe estar completamente sincronizada y adaptada a nuestra hora local, así que volveremos a ejecutar `timedatectl` para verificarlo:

{% highlight shell %}
root@dulcinea:~# timedatectl
               Local time: Sat 2020-11-14 20:57:16 CET
           Universal time: Sat 2020-11-14 19:57:16 UTC
                 RTC time: Sat 2020-11-14 19:57:17
                Time zone: Europe/Madrid (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
{% endhighlight %}

Efectivamente, el servicio NTP ya se encuentra activo y el reloj del sistema sincronizado, por lo que ya muestra la hora local real (**20:57:16**).

### Sancho

Para comprobar si nuestro servidor tiene actualmente sincronizado el reloj haciendo uso de un servidor NTP, ejecutaremos el comando:

{% highlight shell %}
root@sancho:~# timedatectl
               Local time: Sat 2020-11-14 19:59:49 UTC
           Universal time: Sat 2020-11-14 19:59:49 UTC
                 RTC time: Sat 2020-11-14 19:59:50    
                Time zone: Etc/UTC (UTC, +0000)       
System clock synchronized: yes                        
              NTP service: active                     
          RTC in local TZ: no
{% endhighlight %}

Como se puede apreciar, nos ha mostrado la información referente a la hora de nuestra máquina. Sin embargo, y a diferencia de la máquina anterior, podemos apreciar que el estado del **NTP service** es **active**, y que por tanto el reloj del sistema se encuentra sincronizado (**System clock synchronized**), pero no haciendo uso del servidor **es.pool.ntp.org**, sino de uno que viene por defecto, así que para una mayor precisión, procederemos a modificarlo manualmente, haciendo uso del fichero **/etc/systemd/timesyncd.conf**:

{% highlight shell %}
root@sancho:~# nano /etc/systemd/timesyncd.conf

[Time]
NTP=es.pool.ntp.org
{% endhighlight %}

Listo, el servidor que ha de usar ya ha sido especificado, así que procederemos a reiniciar el servicio y a comprobar su estado:

{% highlight shell %}
root@sancho:~# systemctl restart systemd-timesyncd.service

root@sancho:~# systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2020-11-14 20:01:17 UTC; 5s ago
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 1403 (systemd-timesyn)
     Status: "Initial synchronization to time server 90.165.120.190:123 (es.pool.ntp.org)."
      Tasks: 2 (limit: 533)
     Memory: 1.2M
     CGroup: /system.slice/systemd-timesyncd.service
             └─1403 /lib/systemd/systemd-timesyncd

Nov 14 20:01:16 sancho systemd[1]: Starting Network Time Synchronization...
Nov 14 20:01:17 sancho systemd[1]: Started Network Time Synchronization.
Nov 14 20:01:17 sancho systemd-timesyncd[1403]: Initial synchronization to time server 90.165.120.190:123 (es.pool.ntp.org).
{% endhighlight %}

En ésta ocasión, ha sido capaz de sincronizarse con dicho servidor, tal y como podemos apreciar en el último mensaje mostrado. Todavía no hemos terminado, ya que cuando ejecuté el comando `timedatectl`, me mostraba una hora menos de la hora local, es decir, había un problema con la zona horaria, por lo que debemos modificarla manualmente, tal y como hemos hecho en la anterior máquina, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# timedatectl set-timezone Europe/Madrid
{% endhighlight %}

En un principio, la hora ya debe estar completamente sincronizada y adaptada a nuestra hora local, así que volveremos a ejecutar `timedatectl` para verificarlo:

{% highlight shell %}
root@sancho:~# timedatectl
               Local time: Sat 2020-11-14 21:01:59 CET
           Universal time: Sat 2020-11-14 20:01:59 UTC
                 RTC time: Sat 2020-11-14 20:02:00    
                Time zone: Europe/Madrid (CET, +0100) 
System clock synchronized: yes                        
              NTP service: active                     
          RTC in local TZ: no
{% endhighlight %}

Efectivamente, el servicio NTP ya se encuentra activo y el reloj del sistema sincronizado, por lo que ya muestra la hora local real (**21:01:59**).

### Quijote

En el caso de **CentOS 7**, no se hace uso de **systemd-timesyncd** para la sincronización NTP, sino de **chronyd**, pero la filosofía es exactamente la misma.

Para comprobar si nuestro servidor tiene actualmente sincronizado el reloj haciendo uso de un servidor NTP, ejecutaremos el comando:

{% highlight shell %}
[root@quijote ~]# timedatectl 
      Local time: Sat 2020-11-14 20:02:59 UTC
  Universal time: Sat 2020-11-14 20:02:59 UTC
        RTC time: Sat 2020-11-14 20:02:58
       Time zone: UTC (UTC, +0000)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
{% endhighlight %}

Como se puede apreciar, nos ha mostrado la información referente a la hora de nuestra máquina. Al igual que ocurría en la máquina anterior, el reloj del sistema se encuentra sincronizado (**NTP synchronized**), pero no haciendo uso del servidor **es.pool.ntp.org**, sino de unos que vienen por defecto, así que para una mayor precisión, procederemos a modificarlo manualmente, haciendo uso del fichero **/etc/chrony.conf**, en el que comentaremos los anteriores servidores y estableceremos el nuevo:

{% highlight shell %}
[root@quijote ~]# vi /etc/chrony.conf

server es.pool.ntp.org iburst
{% endhighlight %}

Listo, el servidor que ha de usar ya ha sido especificado, así que procederemos a reiniciar el servicio y a comprobar su estado:

{% highlight shell %}
[root@quijote ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2020-11-14 20:10:16 UTC; 9s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 1223 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 1219 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 1221 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─1221 /usr/sbin/chronyd

Nov 14 20:10:16 quijote.novalocal systemd[1]: Stopped NTP client/server.
Nov 14 20:10:16 quijote.novalocal systemd[1]: Starting NTP client/server...
Nov 14 20:10:16 quijote.novalocal chronyd[1221]: chronyd version 3.4 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SIGND +ASYNCDNS +SECHASH +IPV6 +DEBUG)
Nov 14 20:10:16 quijote.novalocal chronyd[1221]: Frequency 58.512 +/- 0.277 ppm read from /var/lib/chrony/drift
Nov 14 20:10:16 quijote.novalocal systemd[1]: Started NTP client/server.
Nov 14 20:10:20 quijote.novalocal chronyd[1221]: Selected source 162.159.200.1
{% endhighlight %}

En ésta ocasión, ha sido capaz de sincronizarse con dicho servidor, tal y como podemos apreciar en el último mensaje mostrado. Todavía no hemos terminado, ya que cuando ejecuté el comando `timedatectl`, me mostraba una hora menos de la hora local, es decir, había un problema con la zona horaria, por lo que debemos modificarla manualmente, tal y como hemos hecho en las anteriores ocasiones, ejecutando para ello el comando:

{% highlight shell %}
[root@quijote ~]# timedatectl set-timezone Europe/Madrid
{% endhighlight %}

En un principio, la hora ya debe estar completamente sincronizada y adaptada a nuestra hora local, así que volveremos a ejecutar `timedatectl` para verificarlo:

{% highlight shell %}
[root@quijote ~]# timedatectl
      Local time: Sat 2020-11-14 21:10:59 CET
  Universal time: Sat 2020-11-14 20:10:59 UTC
        RTC time: Sat 2020-11-14 20:10:59
       Time zone: Europe/Madrid (CET, +0100)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: no
 Last DST change: DST ended at
                  Sun 2020-10-25 02:59:59 CEST
                  Sun 2020-10-25 02:00:00 CET
 Next DST change: DST begins (the clock jumps one hour forward) at
                  Sun 2021-03-28 01:59:59 CET
                  Sun 2021-03-28 03:00:00 CEST
{% endhighlight %}

Efectivamente, el servicio NTP ya se encuentra activo y el reloj del sistema sincronizado, por lo que ya muestra la hora local real (**21:10:59**).