---
layout: post
title:  "Instalación Debian 10"
banner: "/assets/images/banners/debian.jpg"
date:   2020-10-03 19:42:00 +0200
categories: sistemas
---
Esta tarea he optado por separarla en 3 partes claramente diferenciadas.

* **Pre-instalación**: Preparación de Debian y configuraciones en UEFI.
* **Instalación**: Proceso de instalación y configuración de LVM.
* **Post-instalación**: Configuraciones en UEFI e instalación de firmware/drivers.

## Pre-instalación
---

Antes de nada, creo que es necesario explicar la estructura de almacenamiento de mi máquina:

{% highlight shell %}
Portátil (MSI GP63-8RE)
├─Disco SSD NVMe Western Digital (250 GB) -> Windows 10
└─Disco HDD Seagate Barracuda (1000 GB)
  ├─Partición 1 (500 GB) -> Almacenamiento para Windows 10
  └─Partición 2 (500 GB) -> Debian 9
{% endhighlight %}

En mi caso, he partido de un punto en el que tenía ciertos problemas con el disco duro HDD, debido a restos de anteriores instalaciones y problemas que tuve con Debian durante el primer curso de ASIR, pues el GRUB me daba fallos de vez en cuando, no detectaba los sistemas operativos, tenía que instalar con LEGACY ya que UEFI daba problemas... Así que lo primero que hice fue **formatear a bajo nivel** el disco HDD, con la finalidad de reestablecer todos sus sectores y dejarlos de fábrica, de manera que no quedase ningún rastro, aunque con borrar la tabla de particiones y haber generado una nueva, hubiese sido suficiente.

Este procedimiento lo he hecho con un software que llevo usando varios años cada vez que necesitaba vender un disco duro o tramitar una garantía para el mismo, asegurándome que no quedase dentro ningún tipo de información personal. Se trata de [HDD LLF Low Level Format Tool](https://hddguru.com/software/HDD-LLF-Low-Level-Format-Tool/).

![lowlevel](https://i.ibb.co/FxFd5gv/HDD.jpg "HDD LLF Low Level Format Tool")

Una vez que el disco se encuentra completamente limpio, ya podemos dar el siguiente paso. Lo siguiente que hice fue descargar la ISO Netinst de [Debian 10](https://www.debian.org/distrib/netinst) de la página oficial, para la arquitectura **amd64**.

Una vez descargada, creé un medio arrancable para poder realizar la instalación, en mi caso decidí hacerlo mediante un USB usando [Rufus](https://rufus.ie/), que permite hacerlo de una manera muy sencilla. Lo único que tuve que especificar fue el USB, la imagen ISO y el resto de la configuración, bastará con dejarla por defecto, pues generará un esquema de particiones MBR enfocado a un sistema destino BIOS o UEFI (nosotros vamos a instalar en UEFI).

Una creado el medio de instalación, ya podremos proceder a apagar la máquina y volverla a encender, pulsando **SUPR** durante el arranque (en mi caso) para poder acceder a la UEFI. Una vez dentro, nos iremos al apartado **Boot** y marcaremos el **pendrive** como **primera opción**, para que al volver a arrancar, inicie el instalador de Debian. Tendremos que asegurarnos que el **Boot mode** es **UEFI with CSM**, de manera que el GRUB podrá reconocer también a Windows, pues también fue instalado en modo UEFI with CSM, a diferencia de la anterior instalación, que tenía que cambiar las opciones de arranque cada vez que quisiera cambiar de sistema operativo, dado que tuve que instalar Debian 9 con LEGACY.

![uefi1](https://i.ibb.co/M1mhtLY/20201003-131329.jpg "UEFI")

Tras ello, iremos al apartado **Save & Exit** y guardaremos los cambios, reiniciendo por consecuencia la máquina, para que así arranque desde el USB.

## Instalación
---

Una vez se nos muestre el menú del instalador, seleccionaremos la opción **Install**, para así no usar ningún interfaz gráfico. Si nos fijamos, nada más intentar la **detección del hardware de red** nos dirá que nos falta firmware, y leyendo el nombre de los paquetes podemos deducir que se trata de la **tarjeta de red**. Nos preguntará si queremos cargar dichos controladores desde un medio extraíble, pero le diremos que no, ya que podemos instalarlo posteriormente sin mayor problema (lo único que pasa es que durante la instalación tendremos que usar el cable de red, pero no pasa nada, es el pan de cada día).

![firmware](https://i.ibb.co/b7N1K8v/20200923-155329.jpg "Firmware instalación")

Seguidamente seleccionaremos nuestro idioma preferido, en este caso **Español** y la distribución de teclado, en este caso, también **Español**. Hay que tener cierta precaución de no elegir la distribución de teclado de otro país ya que la distribución de los símbolos cambia, y en caso de necesitar escribir algo, habría que probar tecla por tecla hasta encontrar el símbolo deseado.

Los siguientes pasos consisten en elegir un nombre de máquina, en este caso **debian**, un nombre de usuario, en este caso **alvaro** y una ubicación para la zona horaria, en este caso **Península**. Se nos solicitarán también las credenciales de acceso.

Tras ello, llegaremos a la parte importante de la instalación, donde debemos seleccionar qué **método de particionado** queremos usar, así que seleccionaremos **Manual** para configurar **LVM** a nuestro gusto.

Actualmente, se nos mostrarán solo **dos** opciones en la parte superior del menú:

![lvm1](https://i.ibb.co/CWSJ3Yp/11.jpg "LVM")

Esto se debe a que uno de los dispositivos (el **HDD** en este caso) todavía no tiene una **tabla de particiones** en su interior, al haberle realizado un formateo a bajo nivel. Para crearla, nos posicionaremos sobre el mismo y al pulsar **INTRO** nos dará la opción de crear la tabla de particiones.

Cuando la tabla de particiones haya sido creada en su interior, nos habrán aparecido **3** nuevas opciones en la parte superior del menú:

![lvm2](https://i.ibb.co/vXLHBp8/13.jpg "LVM")

Todavía no podemos empezar a configurar LVM, ya que necesitamos particionar el disco HDD en dos particiones de más o menos el mismo tamaño (ya que me gustaría mantener la estructura que tenía previamente), para albergar en una de ellas ficheros tales como películas, juegos... para Windows y en la otra, el nuevo sistema operativo Debian 10 que vamos a instalar, así que nos volvemos a posicionar sobre el disco en cuestión y al pulsar, nos saldrá la opción de **Crear una partición nueva**. Tras ello, nos preguntará por el **tamaño** que le queremos asignar, en este caso, **500 GB**, seguido del **tipo** de partición, que será una partición **Primaria** y de la **ubicación** de la partición, que nos es indiferente, por lo que la posicionaré al **Principio**. Es muy importante marcar las particiones como **"no utilizar"**, para que no cree ningún sistema de ficheros en su interior todavía.

Volveremos a repetir el mismo procedimiento para generar la otra partición de **500 GB**, siendo otra partición primaria. Por defecto, se posicionará físicamente después de la primera partición creada, por lo que no nos preguntará la ubicación. Esta partición también la marcaremos como "no utilizar".

Nuestras dos particiones físicas ya están creadas, así que es hora de configurar LVM. Para ello, nos iremos a la parte superior del menú y accederemos a la opción **Configurar el Gestor de Volúmenes Lógicos (LVM)**. Acto seguido, nos pedirá confirmar la **escritura** de los cambios en el disco, ya que hasta ahora, estaban en memoria. Una vez escritos los cambios en disco, nos aparecerá este menú:

![lvm3](https://i.ibb.co/vqHtqbn/24.jpg "LVM")

Hemos llegado a la parte más importante de la instalación, así que considero oportuno explicar el funcionamiento de LVM antes de configurarlo.

El **Logical Volume Manager** o **LVM** es un gestor de volúmenes lógicos para el _kernel_ Linux. En castellano, significa que vamos a poder gestionar nuestro almacenamiento de una forma mucho más flexible que hasta ahora. Estoy seguro que no soy el primero ni el último que se ha quedado sin espacio en una partición física y ha tenido que borrarla para crear una de mayor tamaño. Este problema se acaba con LVM, ya que nos permite añadir nuestros dispositivos de bloques (no tiene por qué ser un disco duro físico como tal, también puede ser una partición del mismo, un RAID...) como **volúmenes físicos** (PV) que añadiremos a su vez a una especie de "contenedor" llamado **grupo de volúmenes** (VG), y de ahí es de donde posteriormente cogeremos la capacidad para crear un **volumen lógico** (LV), que sería lo equivalente a una partición.

Esto nos permite, por ejemplo, crear un volumen lógico de 1.5 TB, uniendo dos discos de 1 TB y dejar 500 GB libres, por si en algún momento necesitamos aumentar la capacidad de dicho volumen. Esa es la principal ventaja de LVM: si nos quedamos sin espacio en un volumen lógico, simplemente bastaría con comprar un nuevo disco duro, añadirlo como volumen físico y al grupo de volúmenes y posteriormente, extender el volumen lógico. Finalmente, redimensionamos el sistema de ficheros y listo. Sin embargo, esto no es posible con las particiones físicas, ya que al estar delimitado por sectores, el hecho de extender la partición puede afectar a los datos que se encuentren contiguos.

Una vez explicado y entendido para qué sirve LVM, volvemos a la instalación. Si nos fijamos en el menú, tenemos una opción llamada **Crear grupo de volúmenes**. Esa es la que debemos seleccionar, para añadir ahí el volumen físico (en este caso una de las dos particiones de 500 GB que hemos creado) y posteriormente crear los volúmenes lógicos. Nos pedirá un **nombre** para identificar el grupo de volúmenes, en este caso, he optado por llamarlo **"sistema"**, aunque hubiese funcionado con cualquier otro.

Seguidamente nos va a pedir añadir al menos un volumen físico para el grupo de volúmenes, que en este caso añadiré **sda1**, la primera de las dos particiones de 500 GB. Para marcarla, lo haremos con la **barra espaciadora** y seguidamente, pulsaremos **INTRO**. De nuevo, nos volverá a pedir confirmación para guardar los cambios al disco, así que volveremos a confirmar. El menú ahora tendrá el siguiente aspecto:

![lvm4](https://i.ibb.co/1Qsp087/28.jpg "LVM")

Nos han aparecido **3** nuevas opciones, pero la que ahora mismo nos interesa es la llamada **Crear un volumen lógico**. Nos pedirá seleccionar un grupo de volúmenes del cuál cogerá el espacio disponible para crear el volumen lógico en cuestión, así que seleccionaremos el único existente. Los volúmenes lógicos que vamos a crear son los siguientes:

{% highlight shell %}
Portátil
├─Disco SSD NVMe Western Digital (250 GB) -> Windows 10
└─Disco HDD Seagate Barracuda (1000 GB)
  ├─Partición 1 (500 GB) -> Almacenamiento para Windows 10
  └─Partición 2 (500 GB) -> Debian 9
    └─sistema (VG) (500 GB)
      ├─root (LV) (200 GB)
      ├─boot (LV) (2 GB)
      ├─swap (LV) (4 GB)
      └─ESPACIO LIBRE (294 GB)
{% endhighlight %}

Así que tendremos que crear 3 volúmenes lógicos en este caso, repitiendo para ello, 3 veces el proceso de creación de un volumen lógico. El primero de nombre **root**, con una capacidad de **200 GB** (como se puede apreciar, no utilizo todo el tamaño de la partición del disco, únicamente lo imprescindible, para no hacer el sistema de ficheros más grande de lo que realmente voy a necesitar), el segundo de nombre **boot**, con una capacidad de **2 GB** y el tercero de nombre **swap**, con una capacidad de **4 GB**.

Una vez finalizada la creación de los volúmenes lógicos, el menú se verá de la siguiente manera:

![lvm5](https://i.ibb.co/8gfqkJP/36.jpg "LVM")

Como se puede apreciar, ahora se nos muestra que tenemos 3 volúmenes lógicos creados, así que podemos proceder a marcar la opción **Terminar** para escribir los cambios en el disco.

Tras ello, veremos la lista de dispositivos de una manera un poco diferente, pues nos habrán aparecido los 3 volúmenes lógicos que acabamos de crear. Todavía no hemos terminado, pues lo que hemos hecho hasta ahora ha sido crear los volúmenes lógicos, pero todavía no les hemos dado formato.

En mi caso, los volúmenes lógicos tendrán los siguientes formatos:

{% highlight shell %}
sistema-root -> Sistema de ficheros ext4 transaccional, con punto de montaje en /. Es donde se va a instalar Debian.
sistema-boot -> Sistema de ficheros ext4 transaccional, con punto de montaje en /boot. Es donde se va a instalar el GRUB, consiguiendo así evitar que en un futuro surjan los mismos problemas que tuve con la anterior instalación.
sistema-swap -> Área de intercambio (tengo 16 GB de RAM, por lo que diría que no es necesario tener área swap, pero por si acaso, no cuesta nada crearla).
{% endhighlight %}

En mi caso, el resumen de los dispositivos quedaría de la siguiente forma:

![particiones](https://i.ibb.co/GRHFCJC/20201003-131604.jpg "Tabla de particiones")

Una vez que nos hemos asegurado de que las particiones y volúmenes lógicos son correctos, pulsaremos en **Finalizar el particionado y escribir los cambios en el disco**, y confirmaremos una vez más para que se empiece a dar formato a los mismos.

Tras la instalación del sistema base, se nos preguntará por el país para la réplica de Debian, que en este caso, será **España**. El repositorio en cuestión será **deb.debian.org**, aunque da igual el que se elija, pues son espejos unos de otros. Tras ello, se ejecutará **tasksel** y tendremos que elegir los programas que instalar. En mi caso, dado que lo voy a usar para mi portátil, necesitaré el **Entorno de escritorio Debian**, además del **SSH Server**, ya que así me ahorro instalarlo posteriormente, además de lógicamente, las **Utilidades estándar del sistema**. Tras ello, se comenzarán a instalar los programas seleccionados.

## Post-instalación
---

Una vez que la instalación ha finalizado, se reiniciará el equipo, así que una vez más, volveremos a presionar **SUPR** durante el arranque para acceder a la UEFI. Esta vez, tendremos que dejarla como la teníamos antes de comenzar con la instalación de Debian, es decir, dejando que el disco duro tenga mayor prioridad durante el arranque que el USB, de lo contrario, volvería a iniciar desde ahí.

En mi caso, el orden que he puesto ha sido el siguiente:

![uefi2](https://i.ibb.co/WxDtTPh/20201003-173737.jpg "UEFI")

Esto no es todo, ya que además, tendremos que configurar la prioridad de las opciones de arranque dentro de **UEFI Hard Disk Drive BBS Priorities**, ya que de lo contrario, arrancará Windows y no iniciará el GRUB. En mi caso, he especificado que lo primero que se arranque sea el GRUB de Linux, pues es compatible con el de Windows y deja seleccionar durante el mismo qué sistema operativo arrancar:

![uefi3](https://i.ibb.co/XDHwsq9/20201003-131342.jpg "UEFI")

Ya hemos finalizado la configuración, así que ya podremos guardar los cambios y reiniciar, para que arranque el GRUB.

Una vez dentro del GRUB, seleccionamos que arranque Debian 10, pero durante el arranque nos muestra los siguientes errores:

![arranque1](https://i.ibb.co/hstmhxg/20200923-172501.png "Errores arranque")

![arranque2](https://i.ibb.co/sH28Hxj/20200924-154824.png "Errores arranque")

Los fallos de **iwlwifi** de la primera imagen eran de esperar completamente, pues nos avisó durante la instalación que nos faltaba el firmware de la tarjeta de red, sin embargo, también ha devuelto algunos fallos de firmware de **Bluetooth**, de la **gráfica integrada** y de la **tarjeta gráfica NVIDIA**.

Sin embargo, el problema más grande no es ese, es el que viene ahora. Tras terminar el arranque, la máquina no era capaz de mostrar la imagen, pues se quedaba con una línea en la parte superior izquierda y no reaccionaba:

![arranque3](https://i.ibb.co/gTT1whF/20201003-090556.jpg "Errores arranque")

Al ver que no podía hacer nada, hice un REISUB y accedí al **modo Recovery**. Tras leer un poco por Internet, descubrí que lo que estaba dando conflicto eran los drivers de código abierto de la tarjeta gráfica NVIDIA (**nouveau**), así que accedí al fichero de configuración del grub (**/etc/default/grub**) para desactivar el **_Kernel_ Mode Setting (KMS)**, un método que permite fijar la resolución y la profundidad de la pantalla en el espacio del _kernel_, en vez de en el espacio de usuario, que es lo que nos estaba causando el conflicto. Este es el fichero de configuración del grub por defecto:

{% highlight shell %}
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet" #Esta es la línea que debemos cambiar.
GRUB_CMDLINE_LINUX=""

# Uncomment to enable BadRAM filtering, modify to suit your needs
# This works with Linux (no patch required) and with any kernel that obtains
# the memory map information from GRUB (GNU Mach, kernel of FreeBSD ...)
#GRUB_BADRAM="0x01234567,0xfefefefe,0x89abcdef,0xefefefef"

# Uncomment to disable graphical terminal (grub-pc only)
#GRUB_TERMINAL=console

# The resolution used on graphical terminal
# note that you can use only modes which your graphic card supports via VBE
# you can see them in real GRUB with the command `vbeinfo'
#GRUB_GFXMODE=640x480

# Uncomment if you don't want GRUB to pass "root=UUID=xxx" parameter to Linux
#GRUB_DISABLE_LINUX_UUID=true

# Uncomment to disable generation of recovery mode menu entries
#GRUB_DISABLE_RECOVERY="true"

# Uncomment to get a beep at grub start
#GRUB_INIT_TUNE="480 440 1"
{% endhighlight %}

Para desactivarlo, debemos cambiar la línea **GRUB_CMDLINE_LINUX_DEFAULT="quiet"** por **GRUB_CMDLINE_LINUX_DEFAULT="nouveau.modeset=0 quiet"**. Tras ello, guardamos los cambios y ejecutamos el siguiente comando para generar un nuevo grub:

{% highlight shell %}
update-grub
{% endhighlight %}

Una vez generado el nuevo grub, volví a reiniciar y ya podía acceder al entorno de escritorio, pero ésta no era la solución ni mucho menos, era algo simplemente para salir del paso. Lo primero que decidí hacer fue instalar el firmware de la tarjeta de red, pero había un problema, que no son de código abierto, por eso mismo no los ha instalado Debian. La solución es sencilla, tenemos que añadir la rama **non-free** a nuestros repositorios (_/etc/apt/sources.list_).

Dado que íbamos a añadir la rama **non-free**, decidí añadir también la rama **contrib** y los **backports**, por si me hiciesen falta más adelante a la hora de instalar el resto de firmware/drivers. El _sources.list_ quedó de la siguiente forma:

{% highlight shell %}
deb http://deb.debian.org/debian/ buster main contrib non-free
deb http://security.debian.org/debian-security buster/updates main
deb http://deb.debian.org/debian/ buster-updates main contrib non-free
deb http://deb.debian.org/debian buster-backports main contrib non-free
{% endhighlight %}

Lógicamente, dado que hemos modificado los repositorios, tendremos que hacer un **update** tanto de `apt` como de `apt-file` (lo usaremos para saber en qué paquete se encuentra el firmware):

{% highlight shell %}
apt update && apt-file update
{% endhighlight %}

Genial, tras ello, vamos a proceder a buscar qué paquete contiene el firmware de la tarjeta de red para poder instalarlo. Como sabemos que el firmware tiene como nombre **iwlwifi-9000**, vamos a introducirlo en `apt-file`:

{% highlight shell %}
alvaro@debian:~$ apt-file search iwlwifi-9000
firmware-iwlwifi: /lib/firmware/iwlwifi-9000-pu-b0-jf-b0-34.ucode
firmware-iwlwifi: /lib/firmware/iwlwifi-9000-pu-b0-jf-b0-38.ucode
firmware-iwlwifi: /lib/firmware/iwlwifi-9000-pu-b0-jf-b0-41.ucode
firmware-iwlwifi: /lib/firmware/iwlwifi-9000-pu-b0-jf-b0-46.ucode
{% endhighlight %}

Como se puede apreciar, el firmware de la tarjeta de red se encuentra contenido en el paquete **firmware-iwlwifi**. Repetí el mismo proceso para el firmware del Bluetooth (_ibt-17-16-1.sfi_) y también se encuentra contenido en el mismo paquete.

Llegado a este punto, instalé la versión del paquete que se encuentra en **stable**, pero seguía fallando, así que lo volví a instalar pero desde **backports**, ya que como podemos apreciar, son versiones distintas:

{% highlight shell %}
alvaro@debian:~$ apt policy firmware-iwlwifi
firmware-iwlwifi:
  Instalados: (ninguno)
  Candidato:  20190114-2
  Tabla de versión:
     20200817-1~bpo10+1 100
        100 http://deb.debian.org/debian buster-backports/non-free amd64 Packages
     20190114-2 500
        500 http://deb.debian.org/debian buster/non-free amd64 Packages
{% endhighlight %}

Para instalar el paquete **firmware-iwlwifi** desde **backports**:

{% highlight shell %}
apt install -t buster-backports firmware-iwlwifi
{% endhighlight %}

Tras ello, reinicié y todos los fallos que se mostraban durante el arranque referentes a la tarjeta de red y el Bluetooth desaparecieron. Todavía nos falta instalar el de la gráfica integrada y el de la tarjeta gráfica NVIDIA. Primero, vamos a buscar en qué paquete está contenido el firmware de la gráfica integrada (**bxt_dmc**, por ejemplo).

{% highlight shell %}
alvaro@debian:~$ apt-file search bxt_dmc
firmware-misc-nonfree: /lib/firmware/i915/bxt_dmc_ver1.bin
firmware-misc-nonfree: /lib/firmware/i915/bxt_dmc_ver1_07.bin
{% endhighlight %}

Como se puede apreciar, el paquete que contiene el firmware es **firmware-misc-nonfree**. Antes de instalar, realicé la misma búsqueda para el firmware de la tarjeta gráfica NVIDIA (_sw_nonctx.bin_) y me devolvió el mismo paquete, así que procedí a la instalación:

{% highlight shell %}
apt install firmware-iwlwifi
{% endhighlight %}

Volví a reiniciar y todos los fallos habían desaparecido, pero el problema de _nouveau_ persistía, es decir, tenía que desactivar el KMS para poder arrancar. Estuve unos cuantos días probando varias cosas para intentar arreglarlo. Lo primero que hice fue buscar en Internet qué paquete descargar, así que vi que mi GPU (GTX 1060) estaba soportada por el driver **nvidia-driver**, que actualmente se encuentra en la versión **418.152** en estable. También descargué el paquete **nvidia-detect** y lo ejecuté para que me recomendase el driver que instalar, y recomendó el mismo. Algunas de las cosas que probé fueron:

* Instalar el paquete **nvidia-driver** desde stable sin realizar ninguna configuración extra.
* Instalar el paquete **nvidia-driver** desde stable junto a **nvidia-xconfig** y ejecutar este último para generar el fichero _xorg.conf_ dentro de _/etc/X11/_.
* Pasar al _kernel_ 5.7 (se encuentra en los backports), ya que yo me encontraba en el 4.19 y volver a hacer lo mencionado anteriormente.

Sin embargo, nada funcionaba, pues se quedaba en la pantalla de arranque mostrando un error **"[OK] Started GNOME Display Manager"** hasta que hice lo siguiente:

Primero instalé de nuevo el _kernel_ 5.7 junto a los correspondientes _headers_ desde backports:

{% highlight shell %}
apt install -t buster-backports linux-image-amd64 linux-headers-amd64
{% endhighlight %}

Una vez el _kernel_ 5.7 estaba instalado, lógicamente era necesario reiniciar para cargar ese nuevo _kernel_ en memoria, ya que ahora mismo estaba cargado el 4.19 y no iba a notar diferencia, pero antes de ello, decidí volver a modificar el fichero _/etc/default/grub_ y dejar la línea que habia modificado como estaba en un principio, es decir, pasar de **GRUB_CMDLINE_LINUX_DEFAULT="nouveau.modeset=0 quiet"** a **GRUB_CMDLINE_LINUX_DEFAULT="quiet"**. Tras ello, generé un nuevo grub con el comando `update-grub`.

Acto seguido, decidí reiniciar la máquina para cargar el nuevo _kernel_ instalado. Mi sorpresa fue que consiguió llegar al entorno de escritorio, pero aun así, quería instalar los drivers propietarios de NVIDIA. Para verificar que el _kernel_ se había cargado correctamente en memoria y estaba trabajando con el 5.7, ejecuté el comando:

{% highlight shell %}
alvaro@debian:~$ uname -r
5.7.0-0.bpo.2-amd64
{% endhighlight %}

Efectivamente, está correctamente cargado, así que es hora de instalar los drivers propietarios de NVIDIA, pero para que no me pasase lo que me había estado pasando antes, intenté instalarlos de backports (versión **450.66**):

{% highlight shell %}
apt install -t buster-backports nvidia-driver
{% endhighlight %}

Durante la instalación se me mostró el siguiente mensaje de advertencia:

![drivers](https://i.ibb.co/vxj9CyF/20201003-095014.jpg "Drivers NVIDIA")

Nunca antes había llegado tan lejos en la instalación de los drivers, así que me puse bastante contento, pero no todo estaba hecho, todavía me quedaba reiniciar y rezar para que arrancase el entorno de escritorio. ¡Y así fue!, conseguí que funcionase.

Para verificar que los módulos de NVIDIA se habían cargado en lugar de los _nouveau_, ejecuté el siguiente comando:

{% highlight shell %}
alvaro@debian:~$ lsmod | egrep nvidia
nvidia_drm             53248  0
nvidia_modeset       1101824  2 nvidia_drm
nvidia              18702336  54 nvidia_modeset
ipmi_msghandler        73728  2 ipmi_devintf,nvidia
drm_kms_helper        249856  2 nvidia_drm,i915
drm                   602112  8 drm_kms_helper,nvidia_drm,i915
alvaro@debian:~$ lsmod | egrep nouveau
{% endhighlight %}

Efectivamente, no hay módulos _nouveau_ cargados.

Considero también algo totalmente necesario en las instalaciones de Debian el hecho de modificar el valor de la **_swappiness_**. Dicho valor se encuentra almacenado en un parámetro del _kernel_, concretamente en **/proc/sys/vm/swappiness**, e indica en qué momento nuestra máquina empezará a usar la partición de _swap_ en lugar de la RAM. Su valor se encuentra entre **0** y **100**, siendo **0** un uso constante de RAM y **100**, un uso constante de _swap_. Por defecto tiene valor **60**, es decir, la _swap_ comenzará a usarse cuando nuestra RAM tenga un **40%** o más de uso, un valor totalmente absurdo si hablamos de máquinas de hoy en día, que cuentan con gran cantidad de memoria RAM. Vamos a comprobar el valor de _swappiness_ que tiene actualmente nuestra máquina, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ cat /proc/sys/vm/swappiness 
60
{% endhighlight %}

Como se puede apreciar, el valor actual es totalmente absurdo para una máquina que cuenta con **16GB** de RAM. Para modificar dicho valor, podríamos modificar directamente el fichero anteriormente mencionado, pero en lugar de ello, haremos uso de `sysctl` para que la modificación sea persistente tras un reinicio. Para ello, ejecutamos el comando (con permisos de administrador, ejecutando para ello el comando `su -`):

{% highlight shell %}
root@debian:~# echo "vm.swappiness = 10" >> /etc/sysctl.conf
{% endhighlight %}

Gracias a la instrucción que acabamos de ejecutar, hemos añadido una línea al fichero de configuración **sysctl.conf**, el cuál contiene los valores de los parámetros del _kernel_, poniendo el valor del parámetro correspondiente para la _swappiness_ a **10**, pues considero que es un valor razonabñe. Sin embargo, el cambio todavía no ha entrado en vigor, ya que tendremos que volver a leer el contenido de dicho fichero para así cargar el valor del parámetro en memoria, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# sysctl -p
vm.swappiness = 10
{% endhighlight %}

Como se puede apreciar, la salida del comando nos ha informado de que se ha percatado del cambio que acabamos de realizar en el fichero, por lo que el valor de la _swappiness_ se encuentra ya modificado a **10**. Aun así, vamos a volver a mostrar el contenido del fichero **/proc/sys/vm/swappiness** para asegurarnos:

{% highlight shell %}
root@debian:~# cat /proc/sys/vm/swappiness 
10
{% endhighlight %}

Efectivamente, el valor de la _swappiness_ es ahora **10**, por lo que la zona de _swap_ únicamente se utilizará cuando el porcentaje de uso de la memoria RAM sea superior o igual al **90%**, por lo que la instalación de Debian finaliza aquí.