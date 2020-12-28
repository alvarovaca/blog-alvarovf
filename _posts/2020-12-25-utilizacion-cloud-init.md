---
layout: post
title:  "Utilización de cloud-init"
banner: "/assets/images/banners/openstack.png"
date:   2020-12-25 11:25:00 +0200
categories: hlc openstack
---
Se recomienda realizar una previa lectura del _post_ [Modificación del escenario OpenStack](https://www.alvarovf.com/hlc/openstack/2020/12/12/modificacion-escenario-openstack.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El objetivo de esta tarea es el de continuar con la configuración del escenario de trabajo previamente generado en **OpenStack**, concretamente, eliminando una de las instancias existentes y volviéndola a crear utilizando **cloud-init**, un estándar pensado para personalizar instancias _cloud_ en las que se hace uso de una imagen base genérica para posteriormente adaptarla a cada una de las situaciones.

De una forma más interna, dicho estándar trabaja con metadatos que descarga de un servidor de metadatos ubicado en la dirección **169.254.169.254** durante el arranque, y que posteriormente utilizará para llevar a cabo la configuración de la instancia. Dependiendo del metadato en cuestión, se utilizará únicamente para el primer arranque de la instancia o también para aquellos arranques posteriores.

La configuración por parte de **cloud-init** se lleva a cabo durante varias etapas que se ejecutan secuencialmente:

* La primera de ellas consiste en activar **cloud-init** gracias a **systemd**, pues al fin y al cabo es un proceso que se ejecuta en la máquina. En caso de existir un fichero de nombre **/etc/cloud/cloud-init.disabled** o haberse utilizado durante el arranque la opción del _kernel_ **cloud-init=disabled**, el servicio no se iniciará y por tanto, habremos deshabilitado el estándar en nuestra máquina.

* En la segunda fase se obtiene la información para la configuración de la red. En caso de no obtener ninguna, se hará una petición DHCP. Una vez configurada la red, se llevará a cabo la **configuración del hostname**, **montaje de sistemas de ficheros**, **generación de claves ssh**...

* Durante la tercera fase se llevan a cabo configuraciones más específicas como la de **ntp**, **apt**, **contraseñas de usuarios**...

* En la cuarta y última fase se producen las configuraciones finales, así como la **actualización de la paquetería**, **ejecución de scripts del proveedor (_vendor-data_)** y **ejecución de scripts del usuario (_user-data_)**.

Es recomendable mirar los registros (_logs_) durante el arranque de la máquina en cuestión para así comprobar que la configuración por parte de **cloud-init** se ha producido correctamente y en caso contrario, visualizar el fallo ocurrido.

En este caso, vamos a eliminar la máquina **Sancho** y la volveremos a crear haciendo uso del estándar que acabamos de mencionar, no sin antes llevar a cabo una serie de consideraciones previas. Es importante mencionar que no hay riesgo alguno ya que la máquina está contenida en un volumen, por lo que el hecho de eliminar la instancia no supone la pérdida de los datos existentes, pues para la siguiente creación, haremos uso del volumen actual.

Si recordamos, en el artículo referente a la modificación del escenario OpenStack deshabilitamos el servidor DHCP en la subred de la red interna, pero ahora nos será necesario tenerlo de nuevo activo, para que así se asignen automáticamente direcciones dentro del rango a las máquinas conectadas, concretamente a **Sancho**, la máquina que tendremos que volver a crear.

Para habilitarlo, accederemos en el menú izquierdo al apartado **RED**, dentro del cuál pulsaremos en **Redes**. Una vez ahí, buscaremos la red interna (**red interna de alvaro.vaca**) y pulsaremos en el nombre de la misma. Cuando hayamos accedido a la red, tendremos varios subapartados, así que accederemos a **Subredes** y nos aparecerá la única que hemos creado, por lo que procederemos a editarla pulsando en **Editar subred**.

Se nos habrá abierto un pequeño menú, así que iremos a **Detalles de subred**, pues es donde se encuentra la configuración referente al servidor DHCP de la subred. Simplemente marcaremos la casilla **Habilitar DHCP** y guardaremos los cambios, pulsando para ello en **Guardar**. El servidor DHCP ya habrá sido habilitado, tal y como se puede apreciar:

![dhcp1](https://i.ibb.co/D47TZtb/dhcpenabled.jpg "Habilitar DHCP")

Dado que a la hora de llevar a cabo la configuración inicial en la máquina **Sancho** hemos deshabilitado **cloud-init** (o al menos una parte del mismo), tendremos que revertir dichos cambios para así volverlo a habilitar. En mi caso, deshabilité la parte referente a la configuración de la red mediante la creación de un fichero de nombre **/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg** con un determinado contenido, de manera que procederé a su eliminación ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# rm /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
{% endhighlight %}

Tras ello, el estándar estará totalmente habilitado de nuevo, de manera que podremos salir de la máquina **Sancho** y volver a la anfitriona, para así proceder con los pasos previos a la eliminación de la instancia.

De otro lado, para la posterior creación de la instancia será necesario hacer uso del cliente **openstack** de línea de comandos, además de tener que trabajar con volúmenes, por lo que tendremos que indicar en el fichero RC previamente descargado una nueva variable de entorno (**OS_VOLUME_API_VERSION**) con la versión de la API que vamos a utilizar, por lo que haremos uso del comando:

{% highlight shell %}
alvaro@debian:~/Descargas$ echo "export OS_VOLUME_API_VERSION=2" >> Proyecto\ de\ alvaro.vaca-openrc.sh 
{% endhighlight %}

La línea para declarar la correspondiente variable ya ha sido añadida, así que ya podremos ejecutar dicho script que llevará a cabo toda la configuración necesaria en la máquina anfitriona para establecer la conexión. Para ello, reutilizaremos el entorno virtual creado con anterioridad y todavía existente en la máquina anfitriona junto al fichero RC que acabamos de modificar para así acceder a nuestro proyecto.

Una vez dentro del mismo, podremos proceder a listar las instancias existentes en nuestro proyecto para así verificar que funciona correctamente. El comando a ejecutar sería:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack server list
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
| ID                                   | Name       | Status  | Networks                                                                                                       | Image                    | Flavor  |
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
| 02978cea-fee1-4050-93cf-5213833652b2 | Freston    | ACTIVE  | red interna de alvaro.vaca=10.0.1.7                                                                            | N/A (booted from volume) | m1.mini |
| 24163226-9ada-445c-9b6c-18ca6ec73b62 | DNSCliente | SHUTOFF | red de alvaro.vaca=10.0.0.13, 172.22.200.106                                                                   | Debian Buster 10.6       | m1.mini |
| c997cff3-4f83-43a9-a797-a779425d7da0 | maquina    | SHUTOFF | red de alvaro.vaca=10.0.0.4, 172.22.201.19                                                                     | Debian Buster 10.6       | m1.mini |
| dc914d4c-8a26-4ac8-ae9a-d0ae2eeafb1d | Django     | SHUTOFF | red de alvaro.vaca=10.0.0.16, 172.22.200.186                                                                   | Debian Buster 10.6       | m1.mini |
| ee6e1fde-da68-44e4-89b2-536d0758f2d2 | DNS        | ACTIVE  | red de alvaro.vaca=10.0.0.15, 172.22.200.184                                                                   | Debian Buster 10.6       | m1.mini |
| d5474d1a-51af-4dc5-935f-4756848bd88b | Quijote    | ACTIVE  | DMZ de alvaro.vaca=10.0.2.6                                                                                    | N/A (booted from volume) | m1.mini |
| 93673790-a461-49ee-94d6-f0aeff3b8bdb | Sancho     | ACTIVE  | red interna de alvaro.vaca=10.0.1.4                                                                            | N/A (booted from volume) | m1.mini |
| 2e1b3676-ae28-44a4-8748-9f59609c7ef7 | Dulcinea   | ACTIVE  | DMZ de alvaro.vaca=10.0.2.12; red de alvaro.vaca=10.0.0.7, 172.22.200.134; red interna de alvaro.vaca=10.0.1.9 | N/A (booted from volume) | m1.mini |
+--------------------------------------+------------+---------+----------------------------------------------------------------------------------------------------------------+--------------------------+---------+
{% endhighlight %}

Efectivamente, el cliente de OpenStack se encuentra actualmente operativo y nos ha mostrado la información referente a las instancias creadas en mi proyecto.

Tras ello, procederemos a eliminar la instancia **Sancho**, cuyo volumen asociado quedará libre y disponible para ser usado en la siguiente creación, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack server delete Sancho
{% endhighlight %}

En teoría la instancia ya ha sido eliminada, pero para verificarlo, vamos a listar la información detallada sobre la misma, haciendo uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack server show Sancho
No server with a name or ID of 'Sancho' exists.
{% endhighlight %}

Efectivamente, tal y como nos ha devuelto la ejecución del comando, podemos concluir que la máquina **Sancho** ha sido correctamente eliminada de nuestro proyecto.

Si lo pensamos, cuando vayamos a crear la nueva instancia, el servidor DHCP existente en la subred de la red interna nos va a proporcionar una nueva dirección IP que actualmente no conocemos dentro del rango, lo que supondría tener que llevar a cabo modificaciones en las zonas del servidor DNS para que así la nueva dirección sea conocida por el resto de máquinas.

Para evitar este problema, crearemos a mano un **puerto** de OpenStack, al que asignaremos una dirección fija preestablecida y que asociaremos a la máquina durante la creación. Lo primero que haremos será listar los puertos actualmente existentes en la red interna, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack port list --network 'red interna de alvaro.vaca'
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                      | Status |
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------+--------+
| 0aab3e9c-c2c9-46c2-b6ed-e33447107554 |      | fa:16:3e:b5:5d:e7 | ip_address='10.0.1.9', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' | ACTIVE |
| 9f1f5eea-a302-4126-ad9b-7f2872e2ae91 |      | fa:16:3e:99:2b:18 | ip_address='10.0.1.7', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' | ACTIVE |
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------+--------+
{% endhighlight %}

Donde:

* **--network**: Indicamos el nombre de la red para filtrar los resultados.

Como se puede apreciar, actualmente existen un total de 2 puertos en la red interna, uno con dirección **10.0.1.9** referente a **Dulcinea** y otro con dirección **10.0.1.7** referente a **Freston**, de manera que vamos a crear dentro de dicha subred un nuevo puerto de nombre **port-sancho** con la dirección que previamente tenía asignada dicha máquina, es decir, la **10.0.1.4**, haciendo para ello uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack port create --network 'red interna de alvaro.vaca' --fixed-ip subnet=34a05a46-937e-4e91-b95f-f4bcad633af0,ip-address=10.0.1.4 --no-security-group --disable-port-security port-sancho
+-------------------------+-------------------------------------------------------------------------+
| Field                   | Value                                                                   |
+-------------------------+-------------------------------------------------------------------------+
| admin_state_up          | UP                                                                      |
| allowed_address_pairs   |                                                                         |
| binding_host_id         | None                                                                    |
| binding_profile         | None                                                                    |
| binding_vif_details     | None                                                                    |
| binding_vif_type        | None                                                                    |
| binding_vnic_type       | normal                                                                  |
| created_at              | 2020-12-20T10:14:45Z                                                    |
| data_plane_status       | None                                                                    |
| description             |                                                                         |
| device_id               |                                                                         |
| device_owner            |                                                                         |
| dns_assignment          | None                                                                    |
| dns_domain              | None                                                                    |
| dns_name                | None                                                                    |
| extra_dhcp_opts         |                                                                         |
| fixed_ips               | ip_address='10.0.1.4', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' |
| id                      | 87b0f183-329f-49e0-ad9e-2c422f79f9da                                    |
| ip_allocation           | None                                                                    |
| mac_address             | fa:16:3e:c1:0e:f2                                                       |
| name                    | port-sancho                                                             |
| network_id              | 879a4550-118a-4d30-83c4-2a47d0ad581e                                    |
| numa_affinity_policy    | None                                                                    |
| port_security_enabled   | False                                                                   |
| project_id              | dab5156c32654875b6c54ce23c6712a2                                        |
| propagate_uplink_status | None                                                                    |
| qos_network_policy_id   | None                                                                    |
| qos_policy_id           | None                                                                    |
| resource_request        | None                                                                    |
| revision_number         | 4                                                                       |
| security_group_ids      |                                                                         |
| status                  | DOWN                                                                    |
| tags                    |                                                                         |
| trunk_details           | None                                                                    |
| updated_at              | 2020-12-20T10:14:45Z                                                    |
+-------------------------+-------------------------------------------------------------------------+
{% endhighlight %}

Donde:

* **--network**: Indicamos el nombre de la red a la que va a pertenecer el puerto.
* **--fixed-ip**: Indicamos que la dirección IP asignada va a ser fija, debiendo indicarle el identificador de la subred y la dirección deseada.
* **--no-security-group**: Indicamos que no se le asigne ningún grupo de seguridad al puerto, para así evitar tener que deshabilitarlo posteriormente.
* **--disable-port-security**: Indicamos que se deshabilite la seguridad del puerto, para así evitar tener que hacerlo posteriormente.

El puerto ya ha sido generado, pero para verificarlo, vamos a listar una vez más los puertos existentes en la red interna, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack port list --network 'red interna de alvaro.vaca'
+--------------------------------------+-------------+-------------------+-------------------------------------------------------------------------+--------+
| ID                                   | Name        | MAC Address       | Fixed IP Addresses                                                      | Status |
+--------------------------------------+-------------+-------------------+-------------------------------------------------------------------------+--------+
| 0aab3e9c-c2c9-46c2-b6ed-e33447107554 |             | fa:16:3e:b5:5d:e7 | ip_address='10.0.1.9', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' | ACTIVE |
| 87b0f183-329f-49e0-ad9e-2c422f79f9da | port-sancho | fa:16:3e:c1:0e:f2 | ip_address='10.0.1.4', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' | DOWN   |
| 9f1f5eea-a302-4126-ad9b-7f2872e2ae91 |             | fa:16:3e:99:2b:18 | ip_address='10.0.1.7', subnet_id='34a05a46-937e-4e91-b95f-f4bcad633af0' | ACTIVE |
+--------------------------------------+-------------+-------------------+-------------------------------------------------------------------------+--------+
{% endhighlight %}

Efectivamente, podemos concluir que el puerto de nombre **port-sancho** con dirección **10.0.1.4** ha sido correctamente generado en la subred en cuestión.

Dado que en el momento de la creación de la instancia tendremos que indicar el identificador del volumen a utilizar, vamos a listar todos los volúmenes existentes en nuestro proyecto, haciendo para ello uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack volume list
+--------------------------------------+----------+-----------+------+-----------------------------------+
| ID                                   | Name     | Status    | Size | Attached to                       |
+--------------------------------------+----------+-----------+------+-----------------------------------+
| be6a9dc9-10d4-4be9-ad7b-cc46f612f18d | Freston  | in-use    |   10 | Attached to Freston on /dev/vda   |
| 78e5a390-4073-4495-a3af-12da6e5f1dcb | Quijote  | in-use    |   10 | Attached to Quijote on /dev/vda   |
| 58ae1ee9-d665-419c-8ef0-e51947ec6f89 | Sancho   | available |   10 |                                   |
| 72ae6142-f577-4753-a0f1-8bbfb3fd17bb | Dulcinea | in-use    |   10 | Attached to Dulcinea on /dev/vda  |
+--------------------------------------+----------+-----------+------+-----------------------------------+
{% endhighlight %}

Como era de esperar, en mi proyecto existen un total de 4 volúmenes creados, uno por cada una de las máquinas existentes en el escenario, siendo aquel de nombre **Sancho**, como es lógico, el único disponible para ser utilizado.

Ya conocemos toda la información necesaria para llevar a cabo la creación de la instancia, a falta del fichero en el que indiquemos la configuración que deseamos realizar, de manera que lo generaremos en el directorio actual con el nombre **cloud-config.yaml**, ejecutando para ello el comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ nano cloud-config.yaml
{% endhighlight %}

En mi caso concreto, el contenido establecido en dicho fichero tras haber leído la [documentación](https://cloudinit.readthedocs.io/en/latest/topics/modules.html) necesaria se puede encontrar [aquí](https://pastebin.com/uKaXHdLf), siendo el mismo bastante básico, ya que las posibilidades de **cloud-init** son limitadas en el sentido de que únicamente nos permite llevar a cabo una configuración inicial, pues el resto, como por ejemplo configurar un servidor web, tendríamos que hacerlo a mano o utilizando otras tecnologías.

Todo está listo para proceder con la creación de la instancia **Sancho**, de manera que haremos uso del comando:

{% highlight shell %}
(openstackclient) alvaro@debian:~/Descargas$ openstack server create --volume 58ae1ee9-d665-419c-8ef0-e51947ec6f89 --flavor m1.mini --key-name Linux --security-group default --port port-sancho --user-data cloud-config.yaml Sancho
+-----------------------------+------------------------------------------------------------------+
| Field                       | Value                                                            |
+-----------------------------+------------------------------------------------------------------+
| OS-DCF:diskConfig           | MANUAL                                                           |
| OS-EXT-AZ:availability_zone |                                                                  |
| OS-EXT-STS:power_state      | NOSTATE                                                          |
| OS-EXT-STS:task_state       | scheduling                                                       |
| OS-EXT-STS:vm_state         | building                                                         |
| OS-SRV-USG:launched_at      | None                                                             |
| OS-SRV-USG:terminated_at    | None                                                             |
| accessIPv4                  |                                                                  |
| accessIPv6                  |                                                                  |
| addresses                   |                                                                  |
| adminPass                   | eX7Z2diAUEni                                                     |
| config_drive                |                                                                  |
| created                     | 2020-12-21T16:03:41Z                                             |
| flavor                      | m1.mini (12)                                                     |
| hostId                      |                                                                  |
| id                          | 326d3022-e424-4cb6-ab5c-f0db1bd244b8                             |
| image                       | N/A (booted from volume)                                         |
| key_name                    | Linux                                                            |
| name                        | Sancho                                                           |
| progress                    | 0                                                                |
| project_id                  | dab5156c32654875b6c54ce23c6712a2                                 |
| properties                  |                                                                  |
| security_groups             | name='3d8d6ff9-874d-451f-b24a-d9fdaba21eea'                      |
| status                      | BUILD                                                            |
| updated                     | 2020-12-21T16:03:41Z                                             |
| user_id                     | fc679f920848637f7d23fefdfbce7336b666cdc265f8bc32f51552cbcebbc3a7 |
| volumes_attached            | id='58ae1ee9-d665-419c-8ef0-e51947ec6f89'                        |
+-----------------------------+------------------------------------------------------------------+
{% endhighlight %}

Donde:

* **--volume**: Indicamos el identificador del volumen a ser utilizado por la instancia.
* **--flavor**: Indicamos el sabor deseado para la instancia.
* **--key-name**: Indicamos el par de claves que queremos inyectar en la instancia. No es relevante ya que va a ser sobreescrito por la configuración indicada en el fichero **cloud-config.yaml**.
* **--security-group**: Indicamos un grupo de seguridad para asignar a la instancia. No es relevante ya que no hemos asignado ninguno en el momento de la creación del puerto.
* **--port**: Indicamos el nombre del puerto que queremos asociar, que será el que hemos generado con anterioridad.
* **--user-data**: Indicamos el fichero que contiene la configuración por parte del usuario para ser aplicada gracias a **cloud-init**.

La máquina ya se encuentra generada y operativa, de manera que volveremos a **Dulcinea** para llevar a cabo una conexión a la misma. En este caso, voy a utilizar la autenticación mediante contraseña, para así verificar que la nueva contraseña ha sido correctamente establecida al usuario **ubuntu**, de manera que ejecutaremos el comando:

{% highlight shell %}
debian@dulcinea:~$ ssh ubuntu@sancho
ubuntu@sancho's password: 
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Dec 21 17:08:46 CET 2020

  System load:  0.03              Processes:             106
  Usage of /:   26.5% of 9.52GB   Users logged in:       0
  Memory usage: 49%               IPv4 address for ens3: 10.0.1.4
  Swap usage:   0%


0 updates can be installed immediately.
0 of these updates are security updates.
{% endhighlight %}

Como se puede apreciar, la conexión **ssh** con autenticación mediante contraseña se ha producido correctamente, además de haber utilizado resolución DNS para el nombre **sancho**, pues su dirección IP no ha variado y por tanto, no ha sido necesario realizar ninguna modificación en las zonas DNS.

Lo primero que haremos será listar la información referente a las interfaces de red existentes en la máquina, así como las tablas de enrutamiento, haciendo para ello uso de los comandos:

{% highlight shell %}
ubuntu@sancho:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:c1:0e:f2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.4/24 brd 10.0.1.255 scope global dynamic ens3
       valid_lft 85986sec preferred_lft 85986sec
    inet6 fe80::f816:3eff:fec1:ef2/64 scope link 
       valid_lft forever preferred_lft forever

ubuntu@sancho:~$ ip r
10.0.1.0/24 dev ens3 proto kernel scope link src 10.0.1.4 
169.254.169.254 via 10.0.1.2 dev ens3 proto dhcp src 10.0.1.4 metric 100
{% endhighlight %}

Como se puede apreciar en la salida de ambos comandos, la interfaz **ens3** tiene configurada la dirección **10.0.1.4**, habiendo sido obtenida por el DHCP de OpenStack, ya que su validez no es infinita, que es lo que debería ocurrir cuando se está utilizando una configuración estática. De otro lado, la máquina no tiene puerta de enlace predeterminada, por lo que actualmente no tiene conectividad con el exterior.

Ambos problemas son debidos a que en el fichero **cloud-config.yaml** no hemos establecido ninguna configuración referente a la red. Sin embargo, esto no es un problema, ya que en caso de no encontrar dicha configuración en el fichero pasado como parámetro durante la correspondiente etapa de **cloud-init**, buscará además en un fichero local de nombre **/etc/cloud/cloud.cfg.d/50-curtin-networking.cfg**, que es el que nosotros utilizaremos.

En este caso, vamos a generar dicho fichero que contendrá exactamente la misma configuración de red que en anteriores artículos hemos asignado a la máquina, ejecutando para ello el comando:

{% highlight shell %}
root@sancho:~# nano /etc/cloud/cloud.cfg.d/50-curtin-networking.cfg
{% endhighlight %}

La única diferencia con respecto a la configuración de red asignada en anteriores artículos es que tendremos que indicar correctamente la dirección MAC del nuevo puerto, quedando de la siguiente manera:

{% highlight shell %}
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: false
            match:
                macaddress: fa:16:3e:c1:0e:f2
            mtu: 8950
            set-name: ens3
            addresses: [10.0.1.4/24]
            gateway4: 10.0.1.9
            nameservers:
                addresses: [10.0.1.7, 192.168.202.2, 192.168.200.2]
                search: ["alvaro.gonzalonazareno.org"]
{% endhighlight %}

Tras ello, guardaremos los cambios y con la finalidad de aplicarlos, reiniciaremos la máquina, forzando a **cloud-init** a cargar la nueva configuración establecida, haciendo para ello uso del comando:

{% highlight shell %}
root@sancho:~# cloud-init clean -r
Connection to sancho closed by remote host.
Connection to sancho closed.
{% endhighlight %}

Donde:

* **-r**: Indicamos a la máquina que se reinicie para aplicar la nueva configuración.

Como era de esperar, la conexión ha sido cerrada por el _host_ remoto, de manera que tras el reinicio podremos a llevar a cabo una nueva conexión **ssh**, siendo algo normal que muestre una advertencia al haber sido regeneradas las claves **ssh**, como consecuencia de forzar a **cloud-init** a generar una nueva configuración.

Una vez dentro de la máquina, volveremos a listar la información correspondiente a las interfaces de red existentes en la máquina, así como las tablas de enrutamiento, ejecutando para ello los comandos:

{% highlight shell %}
ubuntu@sancho:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8950 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:c1:0e:f2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.4/24 brd 10.0.1.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fec1:ef2/64 scope link 
       valid_lft forever preferred_lft forever

ubuntu@sancho:~$ ip r
default via 10.0.1.9 dev ens3 proto static 
10.0.1.0/24 dev ens3 proto kernel scope link src 10.0.1.4
{% endhighlight %}

Como se puede apreciar, la máquina tiene configurada una interfaz **ens3** con direccionamiento privado **10.0.1.4** configurado de forma estática, ya que su **valid_lft** es **forever**.

De otro lado, la configuración de enrutamiento es la correcta, por lo que en un principio debería tener salida al exterior. Para comprobarlo, vamos a hacer `ping` a una dirección que sabemos que nos va a responder, como por ejemplo a **google.es**:

{% highlight shell %}
ubuntu@sancho:~$ ping google.es
PING google.es (172.217.168.163) 56(84) bytes of data.
64 bytes from mad07s10-in-f3.1e100.net (172.217.168.163): icmp_seq=1 ttl=112 time=44.5 ms
64 bytes from mad07s10-in-f3.1e100.net (172.217.168.163): icmp_seq=2 ttl=112 time=43.5 ms
64 bytes from mad07s10-in-f3.1e100.net (172.217.168.163): icmp_seq=3 ttl=112 time=43.6 ms
^C
--- google.es ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4642ms
rtt min/avg/max/mdev = 43.464/43.862/44.493/0.451 ms
{% endhighlight %}

Como era de esperar, la máquina tiene conectividad con el exterior, además de haber logrado resolver correctamente el nombre **google.es**.

Vamos a indagar un poco más, visualizando para ello el contenido del fichero con la información sobre la configuración de red a utilizar, de nombre **/etc/netplan/50-cloud-init.yaml** y generado dinámicamente a partir del fichero **/etc/cloud/cloud.cfg.d/50-curtin-networking.cfg** que anteriormente hemos creado. Para ello, haremos uso del comando:

{% highlight shell %}
ubuntu@sancho:~$ cat /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        ens3:
            addresses:
            - 10.0.1.4/24
            dhcp4: false
            gateway4: 10.0.1.9
            match:
                macaddress: fa:16:3e:c1:0e:f2
            mtu: 8950
            nameservers:
                addresses:
                - 10.0.1.7
                - 192.168.202.2
                - 192.168.200.2
                search:
                - alvaro.gonzalonazareno.org
            set-name: ens3
    version: 2
{% endhighlight %}

Como se puede apreciar, toda la configuración de red que previamente hemos establecido para que **cloud-init** utilice, ha sido correctamente plasmada en dicho fichero, pudiendo por tanto concluir que la configuración de red se ha realizado correctamente.

De otro lado, vamos a visualizar el contenido del fichero en el que se establecen los servidores DNS a consultar, es decir, el fichero **/etc/resolv.conf**, ejecutando para ello el comando:

{% highlight shell %}
ubuntu@sancho:~$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search alvaro.gonzalonazareno.org
{% endhighlight %}

En este caso, los servidores DNS configurados no aparecen reflejados en dicho fichero ya que Ubuntu implementa por defecto un servidor DNS local, capaz de cachear las respuestas, consiguiendo por tanto una navegación más rápida, pero que internamente, hará uso de los servidores DNS que le hemos especificado anteriormente para dichas peticiones. Este es además, el motivo por el que no tiene sentido configurar dicho fichero con **cloud-init**, ya que lo que nos interesa es que el servidor DNS a utilizar sea el local.

Para finalizar con la configuración de red, vamos a hacer `ping` a algunas máquinas y servicios nombrados en la zona DNS interna, para así verificar que hace uso correctamente del servidor DNS ubicado en **Freston**:

{% highlight shell %}
ubuntu@sancho:~$ ping dulcinea
PING dulcinea.alvaro.gonzalonazareno.org (10.0.1.9) 56(84) bytes of data.
64 bytes from dulcinea.1.0.10.in-addr.arpa (10.0.1.9): icmp_seq=1 ttl=64 time=0.707 ms
64 bytes from dulcinea.1.0.10.in-addr.arpa (10.0.1.9): icmp_seq=2 ttl=64 time=1.16 ms
64 bytes from dulcinea.1.0.10.in-addr.arpa (10.0.1.9): icmp_seq=3 ttl=64 time=1.14 ms
^C
--- dulcinea.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.707/1.003/1.161/0.209 ms

ubuntu@sancho:~$ ping quijote
PING quijote.alvaro.gonzalonazareno.org (10.0.2.6) 56(84) bytes of data.
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=1 ttl=63 time=1.89 ms
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=2 ttl=63 time=1.94 ms
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=3 ttl=63 time=1.56 ms
^C
--- quijote.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.556/1.794/1.939/0.169 ms

ubuntu@sancho:~$ ping freston
PING freston.alvaro.gonzalonazareno.org (10.0.1.7) 56(84) bytes of data.
64 bytes from freston.1.0.10.in-addr.arpa (10.0.1.7): icmp_seq=1 ttl=64 time=0.487 ms
64 bytes from freston.1.0.10.in-addr.arpa (10.0.1.7): icmp_seq=2 ttl=64 time=0.739 ms
64 bytes from freston.1.0.10.in-addr.arpa (10.0.1.7): icmp_seq=3 ttl=64 time=0.626 ms
^C
--- freston.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.487/0.617/0.739/0.103 ms

ubuntu@sancho:~$ ping www
PING quijote.alvaro.gonzalonazareno.org (10.0.2.6) 56(84) bytes of data.
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=1 ttl=63 time=2.00 ms
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=2 ttl=63 time=1.65 ms
64 bytes from quijote.2.0.10.in-addr.arpa (10.0.2.6): icmp_seq=3 ttl=63 time=1.29 ms
^C
--- quijote.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.293/1.648/2.001/0.289 ms

ubuntu@sancho:~$ ping bd
PING sancho.alvaro.gonzalonazareno.org (127.0.1.1) 56(84) bytes of data.
64 bytes from sancho.alvaro.gonzalonazareno.org (127.0.1.1): icmp_seq=1 ttl=64 time=0.027 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (127.0.1.1): icmp_seq=2 ttl=64 time=0.086 ms
64 bytes from sancho.alvaro.gonzalonazareno.org (127.0.1.1): icmp_seq=3 ttl=64 time=0.060 ms
^C
--- sancho.alvaro.gonzalonazareno.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.027/0.057/0.086/0.024 ms
{% endhighlight %}

Efectivamente, es capaz de resolver todos los nombres introducidos, al estar haciendo uso correctamente del servidor DNS ubicado en **Freston**.

Tras finalizar con la configuración de red, vamos a proceder con las últimas dos pruebas, siendo la primera de ellas referente al nombre de la máquina, por lo que vamos a visualizar el _hostname_ y el _FQDN_ asignado, haciendo para ello uso de los comandos:

{% highlight shell %}
ubuntu@sancho:~$ hostname
sancho

ubuntu@sancho:~$ hostname -f
sancho.alvaro.gonzalonazareno.org
{% endhighlight %}

Una vez más, la configuración es correcta, de manera que para finalizar, voy a verificar que el reloj del sistema esté correctamente sincronizado, haciendo para ello uso del comando:

{% highlight shell %}
ubuntu@sancho:~$ timedatectl
               Local time: Mon 2020-12-21 17:25:11 CET
           Universal time: Mon 2020-12-21 16:25:11 UTC
                 RTC time: Mon 2020-12-21 16:25:12    
                Time zone: Europe/Madrid (CET, +0100) 
System clock synchronized: yes                        
              NTP service: active                     
          RTC in local TZ: no
{% endhighlight %}

Como era de esperar, el reloj se encuentra correctamente sincronizado (**System clock synchronized**) con la zona horaria de España (**Europe/Madrid**), tal y como hemos establecido en el correspondiente fichero **cloud-config.yaml**.

A pesar de ello, también hemos indicado el servidor NTP con el que queremos realizar la sincronización, así que vamos a comprobar que se ha hecho uso del mismo, ejecutando para ello el comando:

{% highlight shell %}
ubuntu@sancho:~$ systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2020-12-21 17:21:16 CET; 4min 9s ago
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 461 (systemd-timesyn)
     Status: "Initial synchronization to time server 193.145.15.15:123 (es.pool.ntp.org)."
      Tasks: 2 (limit: 533)
     Memory: 1.6M
     CGroup: /system.slice/systemd-timesyncd.service
             └─461 /lib/systemd/systemd-timesyncd

Dec 21 17:21:16 sancho systemd[1]: Starting Network Time Synchronization...
Dec 21 17:21:16 sancho systemd[1]: Started Network Time Synchronization.
Dec 21 17:21:19 sancho systemd-timesyncd[461]: Network configuration changed, trying to establish connection.
Dec 21 17:21:21 sancho systemd-timesyncd[461]: Network configuration changed, trying to establish connection.
Dec 21 17:21:50 sancho systemd-timesyncd[461]: Initial synchronization to time server 193.145.15.15:123 (es.pool.ntp.org).
{% endhighlight %}

Como se puede apreciar en el último mensaje mostrado, el servidor utilizado para la sincronización ha sido **es.pool.ntp.org**, de manera que podemos concluir que toda la configuración llevada a cabo mediante **cloud-init** ha sido correcta.