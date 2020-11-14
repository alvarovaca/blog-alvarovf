---
layout: post
title:  "Compilación de un Kernel Linux a medida"
banner: "/assets/images/banners/kernel.jpg"
date:   2020-11-14 13:43:00 +0200
categories: sistemas
---
Se recomienda realizar una previa lectura del _post_ [Compilación de un programa en C utilizando un Makefile](http://www.alvarovf.com/sistemas/2020/10/31/compilacion-usando-makefile.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

El **_kernel_** del sistema operativo es el componente más interno después del _hardware_. Es la parte fundamental del sistema operativo y se encarga de manejar los recursos y permitir que los programas hagan uso de los mismos mediante peticiones de las distintas aplicaciones y procesos que se están ejecutando, siendo éstos recursos principalmente:

* **CPU**.
* **Memoria**.
* **Dispositivos de entrada/salida (I/O)**.

Es además el encargado de proporcionar protección mediante diferentes niveles de acceso o _rings_ (asegurando que las aplicaciones únicamente accedan donde deben) y acceso compartido multiplexado, es decir, que las aplicaciones crean que tienen a su disposición todos los recursos, pero en realidad estén siendo compartidos y se compita por los mismos.

Lo más habitual hoy en día es que en los _kernel_ existan al menos 2 niveles para acceder tanto a la CPU como a la memoria:

* **Kernel Mode**: Sin restricciones, es decir, privilegiado. Su ejecución se lleva a cabo de una forma mucho más sencilla, pero con las correspondientes consideraciones en cuanto a seguridad.
* **User Mode**: Restringido, es decir, no privilegiado. Su ejecución no es tan sencilla como el nivel anterior, pero es mucho más segura.

Dependiendo de que algo se ejecute en _Kernel Mode_ o en _User Mode_, tendrá o no acceso restringido a la CPU, es decir, el acceso a los recursos de la máquina de un proceso que se ejecute en _User Mode_ estará controlado a través del _kernel_, mientras que si ese mismo proceso se ejecutase en _Kernel Mode_, no existiría ningún tipo de restricción.

Generalmente, se suelen implementar **4** anillos a nivel de procesador, pero posteriormente los sistemas operativos únicamente suelen usar **2** de ellos: el más interno (**Anillo 0**) para procesos en **_Kernel Mode_** y el más externo (**Anillo 4**) para procesos en **_User Mode_**. En Internet se puede encontrar bastante información al respecto, pues su detallada explicación se saldría del objetivo de esta tarea.

Vamos a dejar de lado la teoría general de los _kernel_ de los sistemas operativos para centrarnos en el **_kernel_ Linux**. Se trata de un _kernel_ **monolítico** que incluye una parte muy importante de los componentes como módulos compilados con enlace **dinámico**, de manera que la parte que se carga en memoria **estáticamente** es una parte pequeña, pues también está pensado para pequeños dispositivos cuya cantidad de memoria sea mínima, por lo que debemos intentar que sea lo más pequeño posible, dejando para ello en la parte estática los componentes estrictamente esenciales.

Por definición, al tratarse de un _kernel_ monolítico, implica que todos (o casi todos) sus componentes se ejecutan en _Kernel Mode_, teniendo por tanto la ventaja de que al acceder sin restricciones, es mucho más sencillo operar con el mismo, con su correspondiente mejora en cuanto a rendimiento, al no tener que realizar llamadas entre los elementos del _kernel_. Dicha característica implicaría vulnerabilidades en un principio, pero al haber sido desarrollados todos los componentes por el mismo proyecto, no existe dicho problema, pues generalmente, no introducimos nada que no haya sido desarrollado por el propio _kernel_ Linux.

Tenemos la posibilidad de descargar el _kernel_ sin compilar desde [kernel.org](https://www.kernel.org/), aunque lo que generalmente se suele hacer es descargar una distribución, que ya contiene un _kernel_ totalmente funcional y trae consigo una paquetería base, de manera que se convierte en algo muy sencillo de usar por parte de los usuarios, al no requerir grandes conocimientos. Actualmente, el código fuente de rama _vanilla_ del _kernel_ ocupa **1.1 GiB**.

Anteriormente he hecho uso de los términos **estático** y **dinámico**. Ésto se debe a que los componentes del _kernel_ se pueden compilar (enlazar) de dos formas distintas:

* **Estáticamente**: Todos los componentes forman un único fichero binario que se carga en memoria, denominado _vmlinuz_ o _zImage_. Compilar todo el kernel de forma estática supondría compilar los 1.1 GiB (o una selección de dichos componentes), quedando un fichero binario de 300, 400, 500 MiB... que se cargaría en memoria. Esto no sería un problema para máquinas de hoy en día, pero para los dispositivos pequeños, sí, pues sus recursos son limitados.
* **Dinámicamente**: Para solventar el problema que acabamos de mencionar existe la parte dinámica, que se encuentra compuesta por ficheros objeto con extensión **.ko** (_kernel object_), y dichos componentes (también conocidos como **módulos**) se cargan a demanda en memoria (se requiere tener el sistema de ficheros donde están contenidos previamente montado). Ésta es la parte más pesada.

Una vez explicado lo necesario para conocer de manera superficial qué es el _kernel_, más concretamente el **_kernel_ Linux**, es hora de compilar nuestro propio _kernel_. Ésto no es algo que se haga frecuentemente hoy en día (aunque existen excepciones, como los sistemas embebidos o dispositivos empotrados), ya que las distribuciones Linux optan por compilar _kernel_ suficientemente genéricos que soporten la mayor cantidad de situaciones posibles, pero nunca está de más ver la estructura de nuestro sistema de una forma más interna e intentar dejarlo lo más pequeño posible (desintegrando para ello los componentes no necesarios y rechazando el uso de otro tipo de funcionalidades no esenciales), para ver dónde se encuentran los límites, y así aprender a realizar ésta técnica.

El primer paso a llevar a cabo será crear un directorio en el que vamos a trabajar, para así mantener una organización en todo momento. En mi caso, he generado uno de nombre **compkernel/**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ mkdir compkernel
{% endhighlight %}

Tras ello, accederemos al directorio que acabamos de crear, haciendo uso del comando:

{% highlight shell %}
alvaro@debian:~$ cd compkernel/
{% endhighlight %}

Para llevar a cabo la compilación, necesitaremos una determinada paquetería, entre la que se encuentra el compilador de C, _make_, algunas bibliotecas básicas... ésto tiene fácil solución, y es que en lugar de ir instalándolos uno por uno, instalaremos **build-essential**, que contiene todo lo necesario. Además, posteriormente utilizaremos una aplicación gráfica que nos hará mas amena la tarea de selección de componentes del _kernel_, que hace uso de las bibliotecas de _qt_, por lo que tendremos que instalar también el paquete **qtbase5-dev** (hay otras aplicaciones que hacen uso de otras bibliotecas como las de _gtk_ o _ncurses_, pero en este caso, no las usaremos). El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~/compkernel$ sudo apt install build-essential qtbase5-dev
{% endhighlight %}

Ya está todo listo para descargar el código fuente de nuestro _kernel_, pero primero tendremos que descubrir cuál estamos usando (en caso que no lo sepamos). Para ello, haremos uso de `uname`:

{% highlight shell %}
alvaro@debian:~/compkernel$ uname -r
5.7.0-0.bpo.2-amd64
{% endhighlight %}

Donde:

* **-r**: Le pedimos que nos muestre la _release_ en lugar del nombre.

Como se puede apreciar, la versión de núcleo que actualmente estoy utilizando es la **5.7.0**. Llegados a este punto, tenemos dos posibilidades para obtener el código fuente de nuestro _kernel_:

* **Descargarlo de repositorios**: Descargaremos el paquete de nombre **linux-image** correspondiente (podemos buscarlo con `apt search`). Si la versión concreta que estamos utilizando se encuentra en los repositorios oficiales, podemos instalarlo con `apt install`, que realmente lo que hará será ubicar un comprimido **.tar.xz** dentro de _/usr/src_, que contendrá el código fuente, para su posterior descompresión en el directorio de trabajo que hemos creado.
* **Descargarlo de kernel.org**: En mi caso, el _kernel_ que actualmente estoy utilizando lo descargué de la rama **_backports_**, debido a determinados problemas con el _hardware_ si hacía uso de la versión existente en **_stable_**. La cuestión es que dicha versión ya no está disponible, pues actualmente se encuentra disponible la **5.8**, por lo que tendré que hacer uso de ésta opción para la correspondiente descarga del código fuente, tal y como veremos a continuación.

Como anteriormente hemos mencionado, en [kernel.org](https://mirrors.edge.kernel.org/pub/linux/kernel/) tenemos las diferentes versiones de _kernel_ que han sido lanzadas, por lo que nuestra versión concreta (**5.7.0**) la podremos descargar desde [aquí](https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.7.tar.xz), haciendo para ello uso de `wget`:

{% highlight shell %}
alvaro@debian:~/compkernel$ wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.7.tar.xz
--2020-11-13 17:43:20--  https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.7.tar.xz
Resolviendo mirrors.edge.kernel.org (mirrors.edge.kernel.org)... 147.75.101.1, 2604:1380:2001:3900::1
Conectando con mirrors.edge.kernel.org (mirrors.edge.kernel.org)[147.75.101.1]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 112690468 (107M) [application/x-xz]
Grabando a: “linux-5.7.tar.xz”

linux-5.7.tar.xz                100%[==========================>]  107,47M   525KB/s    en 3m 15s

2020-11-04 08:51:36 (563 KB/s) - “linux-5.7.tar.xz” guardado [112690468/112690468]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`:

{% highlight shell %}
alvaro@debian:~/compkernel$ ls -l
total 110052
-rw-r--r-- 1 alvaro alvaro 112690468 nov  4 08:51 linux-source-5.7.tar.xz
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**linux-source-5.7.tar.xz**" con un peso total de **107.47 MB** (112690468 _bytes_). Sin embargo, nada podemos hacer hasta que no llevamos a cabo la extracción de los ficheros contenidos. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/compkernel$ tar -Jxf linux-source-5.7.tar.xz
{% endhighlight %}

Donde:

* **-J**: Utiliza xz para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.xz.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
alvaro@debian:~/compkernel$ ls
linux-5.7  linux-5.7.tar.xz
{% endhighlight %}

Como se puede apreciar, el resultado de la descompresión se encuentra ubicado en un directorio de nombre **linux-5.7/**, al que accederemos ejecutando el comando:

{% highlight shell %}
alvaro@debian:~/compkernel$ cd linux-5.7/
{% endhighlight %}

Una vez dentro del mismo, volveremos a listar el contenido, haciendo uso de `ls`:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ ls
arch   certs    CREDITS  Documentation  fs       init  Kbuild   kernel  LICENSES     Makefile  net     samples  security  tools  virt
block  COPYING  crypto   drivers        include  ipc   Kconfig  lib     MAINTAINERS  mm        README  scripts  sound     usr
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido, tal y como queríamos. Por curiosidad, vamos a comprobar el tamaño del directorio actual y de todos sus ficheros y directorios de forma recursiva, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ du -sh
1,1G	.
{% endhighlight %}

Donde:

* **-s**: Le indicamos que muestre el tamaño total, es decir, el tamaño recursivo.
* **-h**: Le indicamos que muestre el tamaño en formato humano, en lugar de mostrarlo en _bytes_.

Como se puede apreciar, el tamaño total tras la descompresión es de **1.1 GiB**, tal y como hemos mencionado con anterioridad.

Si nos fijamos en el contenido anteriormente mostrado, existe un fichero **Makefile** cuyo contenido es bastante complejo, pues contiene todas las instrucciones necesarias para llevar a cabo dicha compilación, pero la ventaja es que se nos proporciona un comando de ayuda para aprender los primeros pasos y las diferentes opciones y parámetros de los que disponemos a la hora de llevar a cabo la compilación. Dicho comando es `make help`.

Para las compilaciones se hace uso de un fichero de nombre **.config** que contiene la información sobre qué componentes se van a enlazar estáticamente, cuáles dinámicamente y cuáles no se van a enlazar. Dicho fichero no se encuentra todavía generado, así que para generarlo tomando como punto de partida nuestra configuración actual del núcleo (existente en **/boot/**, tal y como veremos más adelante), haremos uso del comando `make oldconfig`, que a pesar de que preguntará explícitamente si queremos incluir determinados componentes opcionales, en mi caso, dije que no a todos ellos, ya que no serán necesarios:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make oldconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/confdata.o
  HOSTCC  scripts/kconfig/expr.o
  LEX     scripts/kconfig/lexer.lex.c
...
exFAT filesystem support (EXFAT_FS) [M/n/y/?] m
  Default iocharset for exFAT (EXFAT_DEFAULT_IOCHARSET) [utf8] utf8
NTFS file system support (NTFS_FS) [N/m/y/?] (NEW) N
#
# configuration written to .config
#
{% endhighlight %}

La salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/AZxfa20Q) se puede ver al completo. Tras ello, listaremos el contenido del directorio actual, estableciendo un filtro por nombre para así verificar que dicho fichero ha sido correctamente generado:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ ls -la | egrep '.config'
-rw-r--r--   1 alvaro alvaro     59 jun  1 01:49 .cocciconfig
-rw-r--r--   1 alvaro alvaro 229434 nov  4 09:24 .config
-rw-r--r--   1 alvaro alvaro    595 jun  1 01:49 Kconfig
{% endhighlight %}

Efectivamente, el fichero **.config** ha sido generado, así que vamos a comprobar cuántos componentes del núcleo han sido enlazados estáticamente (**y**) y cuántos dinámicamente (**m**) en la configuración actual, leyendo el contenido del fichero, filtrando por líneas y posteriormente, contando las coincidencias para cada uno de los casos:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=y' | wc -l
2097

alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=m' | wc -l
3417
{% endhighlight %}

Como se puede apreciar, en la configuración del núcleo que actualmente estamos utilizando, existen un total de **2097** componentes incluidos dentro del **vmlinuz**, pues son enlaces estáticos que estarán siempre cargados en memoria, y un total de **3417** componentes dinámicos o módulos del mismo, que se cargarán en memoria bajo demanda.

Sería una tarea bastante larga y tediosa empezar a reducir elementos estáticos y dinámicos partiendo de este punto, ya que existen más de **5000** componentes enlazados en total, de manera que haremos uso de `make localmodconfig`, que comprobará la lista de componentes que están siendo utilizados en este preciso momento, y en base a dicha información, modificará el fichero **.config**, descartando los demás, pues se entiende que no están activos porque son prescindibles para nosotros, consiguiendo así reducir considerablemente el número de componentes. Existen algunos componentes que preguntará explícitamente si queremos incluirlos, pero en mi caso, dije que no a todos ellos, ya que no serán necesarios:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make localmodconfig
using config: '.config'
vboxdrv config not found!!
vboxnetflt config not found!!
vboxnetadp config not found!!
nvidia config not found!!
...
Audio support for the the Xilinx SPDIF (SND_SOC_XILINX_SPDIF) [N/m/?] n
XTFPGA I2S master (SND_SOC_XTFPGA_I2S) [N/m/?] n
ZTE ZX TDM Driver Support (ZX_TDM) [N/m/?] n
ASoC Simple sound card support (SND_SIMPLE_CARD) [N/m/?] n
#
# configuration written to .config
#
{% endhighlight %}

La salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/bPjPSYER) se puede ver al completo. Una vez más, contaremos el número de componentes tanto estáticos como dinámicos, para así apreciar las diferencias:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=y' | wc -l
1539

alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=m' | wc -l
271
{% endhighlight %}

Como se puede apreciar, el número de componentes estáticos ha sido reducido de **2097** a **1539**, es decir, un total de **558** componentes estáticos han sido eliminados, mientras que en los componentes dinámicos es donde podemos percibir la mayor diferencia, pues el número total ha sido reducido de **3417** a **271**, es decir, **3146** componentes dinámicos han sido eliminados.

Tras ello, ya podremos llevar a cabo nuestra primera compilación, que no tendrá ninguna diferencia, pues los componentes son exactamente los mismos que aquellos que están siendo utilizados en este preciso momento, pero así podremos verificar que todo funciona correctamente. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make -j12 bindeb-pkg
  UPD     include/config/kernel.release
sh ./scripts/package/mkdebian
dpkg-buildpackage -r"fakeroot -u" -a$(cat debian/arch)  -b -nc -uc
dpkg-buildpackage: información: paquete fuente linux-5.7.0
dpkg-buildpackage: información: versión de las fuentes 5.7.0-1
dpkg-buildpackage: información: distribución de las fuentes buster
dpkg-buildpackage: información: fuentes modificadas por alvaro <alvaro@debian>
dpkg-buildpackage: información: arquitectura del sistema amd64
 dpkg-source --before-build .
dpkg-checkbuilddeps: fallo: Unmet build dependencies: libelf-dev:native
dpkg-buildpackage: aviso: Las dependencias y conflictos de construcción no están satisfechas, interrumpiendo
dpkg-buildpackage: aviso: (Use la opción «-d» para anularlo.)
make[1]: *** [scripts/Makefile.package:83: bindeb-pkg] Error 3
make: *** [Makefile:1460: bindeb-pkg] Error 2
{% endhighlight %}

Donde:

* **-j**: Indicamos el número de subprocesos a generar, que generalmente será igual al número de hilos de nuestro procesador, pues cada uno de dichos subprocesos será acogido por un hilo para su correspondiente procesamiento. Con `nproc` podemos ver la cantidad de hilos existentes.
* **bindeb-pkg**: Indicamos que produzca el binario en un paquete **.deb** que posteriormente instalaremos en la máquina haciendo uso de `dpkg`.

Como se puede apreciar, la ejecución ha finalizado con un error de falta de dependencias, concretamente falta el paquete **libelf-dev**, así que procederemos a su instalación, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ sudo apt install libelf-dev
{% endhighlight %}

Una vez instalado el paquete, volveremos a tratar de realizar la compilación, ejecutando de nuevo el comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make -j12 bindeb-pkg
scripts/kconfig/conf  --syncconfig Kconfig
  UPD     include/config/kernel.release
sh ./scripts/package/mkdebian
dpkg-buildpackage -r"fakeroot -u" -a$(cat debian/arch)  -b -nc -uc
dpkg-buildpackage: información: paquete fuente linux-5.7.0
...
 dpkg-genbuildinfo --build=binary
 dpkg-genchanges --build=binary >../linux-5.7.0_5.7.0-1_amd64.changes
dpkg-genchanges: información: binary-only upload (no source code included)
 dpkg-source --after-build .
dpkg-buildpackage: información: subida sólo de binarios (no se incluye ninguna fuente)
{% endhighlight %}

La salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/EFX63x4g) se puede ver al completo. En este caso, la ejecución ha finalizado sin errores, así que el correspondiente fichero **.deb** ha sido generado, por lo que para verificarlo, listaremos el contenido del directorio padre (pues es donde se genera por defecto):

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ ls -lh ..
total 316M
drwxr-xr-x 25 alvaro alvaro 4,0K nov  5 09:33 linux-5.7
-rw-r--r--  1 alvaro alvaro 4,9K nov  5 09:35 linux-5.7.0_5.7.0-1_amd64.buildinfo
-rw-r--r--  1 alvaro alvaro 2,1K nov  5 09:35 linux-5.7.0_5.7.0-1_amd64.changes
-rw-r--r--  1 alvaro alvaro 7,3M nov  5 09:33 linux-headers-5.7.0_5.7.0-1_amd64.deb
-rw-r--r--  1 alvaro alvaro  13M nov  5 09:33 linux-image-5.7.0_5.7.0-1_amd64.deb
-rw-r--r--  1 alvaro alvaro 187M nov  5 09:35 linux-image-5.7.0-dbg_5.7.0-1_amd64.deb
-rw-r--r--  1 alvaro alvaro 1,1M nov  5 09:33 linux-libc-dev_5.7.0-1_amd64.deb
-rw-r--r--  1 alvaro alvaro 108M nov  5 08:44 linux-source-5.7.tar.xz
{% endhighlight %}

Como se puede apreciar, existen varios ficheros **.deb**, pero el que nos interesa es **linux-image-5.7.0_5.7.0-1_amd64.deb**, cuyo peso actualmente es de **13M**. Tras ello, tendremos que instalar el paquete para así poder arrancar con el nuevo núcleo, haciendo uso de `dpkg`, que desempaquetará, instalará y generará una nueva configuración de GRUB:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ sudo dpkg -i ../linux-image-5.7.0_5.7.0-1_amd64.deb
{% endhighlight %}

Donde:

* **-i**: Indicamos que queremos instalar el paquete introducido.

Una vez realizada la instalación, verificaremos que dicho paquete se encuentra actualmente entre nuestra paquetería instalada, ejecutando para ello `dpkg -l`, para así listar todos los paquetes existentes, especificando posteriormente un filtro por nombre:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ dpkg -l | egrep 'linux-image'
ii  linux-image-4.19.0-10-amd64           4.19.132-1                                   amd64        Linux 4.19 for 64-bit PCs (signed)
ii  linux-image-4.19.0-11-amd64           4.19.146-1                                   amd64        Linux 4.19 for 64-bit PCs (signed)
ii  linux-image-5.7.0                     5.7.0-1                                      amd64        Linux kernel, version 5.7.0
ii  linux-image-5.7.0-0.bpo.2-amd64       5.7.10-1~bpo10+1                             amd64        Linux 5.7 for 64-bit PCs (signed)
{% endhighlight %}

Efectivamente, tal y como se puede apreciar en la salida, tenemos un paquete instalado de nombre **linux-image-5.7.0**, que es el que acabamos de compilar e instalar.

El siguiente paso consistiría en reiniciar la máquina y cuando nos cargue el gestor de arranque GRUB, acceder a "**Opciones avanzadas para Debian GNU/Linux**" y una vez ahí, indicar que queremos iniciar con el nuevo _kernel_. En este caso, arrancará sin problemas ya que no hemos modificado manualmente la configuración, así que tras verificarlo, volveremos a reiniciar, arrancando ésta vez con el _kernel_ original.

Lo primero que tendremos que hacer antes de llevar a cabo una nueva compilación será eliminar los ficheros residuales de la anterior, ejecutando para ello el comando `make clean`. Tras ello, vamos a hacer una nueva compilación, pero ésta vez, modificando los componentes a mano, para así conseguir reducirlo al máximo. Como anteriormente hemos mencionado, vamos a hacer uso de una aplicación gráfica, así que en esta ocasión, en lugar de hacer uso de `make oldconfig` o `make localmodconfig`, vamos a utilizar `make xconfig`:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make xconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/images.o
  HOSTCC  scripts/kconfig/confdata.o
  HOSTCC  scripts/kconfig/expr.o
  LEX     scripts/kconfig/lexer.lex.c
  YACC    scripts/kconfig/parser.tab.[ch]
  HOSTCC  scripts/kconfig/lexer.lex.o
  HOSTCC  scripts/kconfig/parser.tab.o
  HOSTCC  scripts/kconfig/preprocess.o
  HOSTCC  scripts/kconfig/symbol.o
  HOSTCC  scripts/kconfig/util.o
*
* 'make xconfig' requires 'pkg-config'. Please install it.
*
make[1]: *** [scripts/kconfig/Makefile:210: scripts/kconfig/qconf-cfg] Error 1
make: *** [Makefile:588: xconfig] Error 2
{% endhighlight %}

Como se puede apreciar, la ejecución ha finalizado con un error de falta de dependencias, concretamente falta el paquete **pkg-config**, así que procederemos a su instalación, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ sudo apt install pkg-config
{% endhighlight %}

Una vez instalado el paquete, volveremos a tratar de realizar la apertura de la aplicación gráfica, ejecutando de nuevo el comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ make xconfig
{% endhighlight %}

![xconfig](https://i.ibb.co/dJCXS5n/Captura-de-pantalla-de-2020-11-14-09-22-35.png "Aplicación gráfica")

Como se puede apreciar, en esta ocasión no hemos tenido problemas para llevar a cabo la correspondiente apertura de la aplicación gráfica. Su uso es bastante sencillo e intuitivo, por lo que no considero necesario llevar a cabo una explicación al respecto, más allá de mencionar que aquellos componentes que estén marcados con un **·** son módulos, los marcados con **✓** son estáticos y los que no están marcados, no serán añadidos a la compilación. Cuando hayamos finalizado, pulsaremos en el símbolo del disquete en la parte superior para guardar los cambios.

Una vez más, contaremos el número de componentes tanto estáticos como dinámicos, para así apreciar las diferencias:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=y' | wc -l
1469

alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=m' | wc -l
263
{% endhighlight %}

Como se puede apreciar, tras desmarcar algunos componentes, el número total se ha visto reducido, aunque esto es una tarea lenta. Tras ello, volveremos a compilar, a instalar y a reiniciar, para verificar que arranca. Llegados a este punto, pueden ocurrir dos cosas:

* **Que el _kernel_ inicie correctamente**, por lo que cuando volvamos a nuestro _kernel_ original, haremos una copia de seguridad del fichero **_.config_**, por si acaso necesitásemos hacer uso del mismo posteriormente.
* **Que el _kernel_ no inicie**, de manera que volveremos a nuestro _kernel_ original y restauraremos la copia de seguridad del fichero **_.config_** para volver atrás, ya que hemos cometido el error de quitar un componente que era necesario para el correcto funcionamiento.

Como curiosidad, cada vez que compilemos, se generará un fichero **.deb** con un número secuencial que irá incrementándose, para así no sobreescribir al fichero de la anterior compilación. Sin embargo, a la hora de instalar dicho paquete, sobreescribirá al _kernel_ compilado que hemos instalado con anterioridad (recalco lo de compilado, ya que el nativo no lo toca). Ésto último es modificable (concretamente en el fichero **_Makefile_**), pero en mi caso, no lo he considerado necesario, ya que no me apetecía tener un _kernel_ instalado por cada compilación, de manera que he preferido sobreescribir el anterior.

Tras repetir reiteradas veces el procedimiento anteriormente mencionado, he dejado el fichero **.config** con la siguiente cantidad de componentes enlazados estáticamente y dinámicamente:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=y' | wc -l
513

alvaro@debian:~/compkernel/linux-5.7$ cat .config | egrep '=m' | wc -l
24
{% endhighlight %}

Como se puede apreciar, la diferencia es bastante grande, pues tras hacer el `make localmodconfig`, teníamos **1539** componentes estáticos, que han pasado a ser **513**, es decir, una diferencia de **1026** componentes estáticos, y de otro lado, teníamos **271** módulos inicialmente, que han pasado a ser únicamente **24**, suponiendo una diferencia de **247** módulos.

Finalmente, vamos a comprobar el tamaño del fichero **.deb** que se ha generado como consecuencia de ésta última compilación, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ ls -lh ../ | egrep 'linux-image'
-rw-r--r--  1 alvaro alvaro 3,1M nov 14 09:11 linux-image-5.7.0_5.7.0-135_amd64.deb
{% endhighlight %}

En este caso, el peso final es de **3.1M**, con respecto a los **13M** que pesaba el fichero en la primera compilación, lo que supone una disminución de tamaño de casi **10M**, es decir, de alrededor de un **76%** del peso inicial, una marca bastante considerable, pero realmente, lo que quiero verificar es el peso del fichero **vmlinuz** instalado en mi máquina, fichero que contiene todos los componentes enlazados estáticamente, así que listaremos el contenido del directorio **/boot/**, pues es donde se encuentra ubicado:

{% highlight shell %}
alvaro@debian:~/compkernel/linux-5.7$ ls -lh /boot/ | egrep 'vmlinuz'
total 217M
-rw-r--r-- 1 root root 202K jul 24 20:46 config-4.19.0-10-amd64
-rw-r--r-- 1 root root 202K sep 17 23:42 config-4.19.0-11-amd64
-rw-r--r-- 1 root root  62K nov 14 09:09 config-5.7.0
-rw-r--r-- 1 root root 225K jul 30 21:11 config-5.7.0-0.bpo.2-amd64
drwx------ 4 root root 1,0K ene  1  1970 efi
drwxr-xr-x 5 root root 1,0K nov 14 09:12 grub
-rw-r--r-- 1 root root  53M sep 25 15:55 initrd.img-4.19.0-10-amd64
-rw-r--r-- 1 root root  53M oct  3 09:10 initrd.img-4.19.0-11-amd64
-rw-r--r-- 1 root root  20M nov 14 09:12 initrd.img-5.7.0
-rw-r--r-- 1 root root  62M oct  8 09:23 initrd.img-5.7.0-0.bpo.2-amd64
drwx------ 2 root root  12K sep 23 18:05 lost+found
-rw-r--r-- 1 root root 3,3M jul 24 20:46 System.map-4.19.0-10-amd64
-rw-r--r-- 1 root root 3,3M sep 17 23:42 System.map-4.19.0-11-amd64
-rw-r--r-- 1 root root 1,8M nov 14 09:09 System.map-5.7.0
-rw-r--r-- 1 root root 4,1M jul 30 21:11 System.map-5.7.0-0.bpo.2-amd64
-rw-r--r-- 1 root root 5,1M jul 24 20:46 vmlinuz-4.19.0-10-amd64
-rw-r--r-- 1 root root 5,1M sep 17 23:42 vmlinuz-4.19.0-11-amd64
-rw-r--r-- 1 root root 1,9M nov 14 09:09 vmlinuz-5.7.0
-rw-r--r-- 1 root root 5,4M jul 30 21:11 vmlinuz-5.7.0-0.bpo.2-amd64
{% endhighlight %}

En esta ocasión, el peso del fichero **vmlinuz** correspondiente al _kernel_ que hemos instalado es de **1.9M**, a diferencia de los **5.4M** que pesa el fichero **vmlinuz** del _kernel_ original que estoy usando, una diferencia que aunque parezca insignificante, para dispositivos pequeños puede marcar una gran diferencia. Como curiosidad, en el directorio **/boot/** también se almacena una copia del fichero **.config** utilizado durante la compilación de cada uno de los _kernel_ existentes, ficheros de nombre __config-*__.

Me gustaría dejar por último, en forma de resumen, todos los componentes tanto estáticos como dinámicos que he deshabilitado en la aplicación gráfica para obtener los resultados anteriormente mostrados:

{% highlight shell %}
General setup > Allow slab caches to be merged
General setup > Auditing support
General setup > Automatic process group scheduling
General setup > CPU/Task time and stats accounting > BSD Process Accounting
General setup > CPU/Task time and stats accounting > Export task/process statistics through netlink
General setup > CPU/Task time and stats accounting > Pressure stall information tracking
General setup > Checkpoint/restore support
General setup > Compiler optimization level > Optimize for size (-Os)
General setup > Configure standard kernel features (expert users) > BUG() support
General setup > Configure standard kernel features (expert users) > Enable PC-Speaker support
General setup > Configure standard kernel features (expert users) > Enable full-sized data structures for core
General setup > Control Group support > CPU controller
General setup > Control Group support > Device controller
General setup > Control Group support > Freezer controller
General setup > Control Group support > IO controller
General setup > Control Group support > Memory controller
General setup > Control Group support > PIDs controller
General setup > Control Group support > Perf controller
General setup > Control Group support > RDMA controller
General setup > Control Group support > Simple CPU accounting controller
General setup > Enable SLUB debugging support
General setup > Enable VM event counters for /proc/vmstat
General setup > Enable bpf() system call
General setup > Enable membarrier() system call
General setup > Enable process_vm_readv/writev syscalls
General setup > Enable rseq() system call
General setup > Enable userfaultfd() system call
General setup > Harden slab freelist metadata
General setup > Initial RAM filesystem and RAM disk (initramfs/initrd) support > Support initial ramdisk/ramfs compressed using LZ4
General setup > Initial RAM filesystem and RAM disk (initramfs/initrd) support > Support initial ramdisk/ramfs compressed using LZMA
General setup > Initial RAM filesystem and RAM disk (initramfs/initrd) support > Support initial ramdisk/ramfs compressed using LZO
General setup > Initial RAM filesystem and RAM disk (initramfs/initrd) support > Support initial ramdisk/ramfs compressed using XZ
General setup > Initial RAM filesystem and RAM disk (initramfs/initrd) support > Support initial ramdisk/ramfs compressed using bzip2
General setup > Kernel .config support
General setup > Load all symbols for debugging/ksymoops > Include all symbols in kallsyms
General setup > Namespaces support
General setup > POSIX Message Queues
General setup > Page allocator randomization
General setup > Profiling support
General setup > SLAB freelist randomization
General setup > Support for paging of anonymous memory (swap)
General setup > System V IPC
General setup > Timers subsystem > High Resolution Timer Support
General setup > uselib syscall
Virtualization
File systems > Btrfs filesystem support
File systems > Direct Access (DAX) support
File systems > Dnotify support
File systems > Enable POSIX file locking API > Enable Mandatory file locking
File systems > Enable filesystem export operations for block IO
File systems > FS Encryption (Per-file encryption)
File systems > FS Verity (read-only file-based authenticity protection)
File systems > FUSE (Filesystem in Userspace) support
File systems > Filesystem wide access notification
File systems > JFS filesystem support
File systems > Kernel automounter support (supports v3, v4 and v5)
File systems > Miscellaneous filesystems > Apple Extended HFS file system support
File systems > Miscellaneous filesystems > Apple Macintosh file system support
File systems > Miscellaneous filesystems > Minix file system support
File systems > Miscellaneous filesystems > Persistent store support > DEFLATE (ZLIB) compression
File systems > Miscellaneous filesystems > QNX4 file system support (read only)
File systems > Miscellaneous filesystems > UFS file system support (read only)
File systems > Network File Systems
File systems > Pseudo filesystems > HugeTLB file system support
File systems > Pseudo filesystems > Include /proc/<pid>/task/<tid>/children file
File systems > Quota Support
File systems > The Extended 4 (ext4) filesystem > Ext4 POSIX Access Control Lists
File systems > The Extended 4 (ext4) filesystem > Use ext4 for ext2 file systems
File systems > XFS filesystem support
Bus options (PCI etc.) > ISA-style DMA support
Power management and ACPI options > ACPI (Advanced Configuration and Power Interface) Support
Power management and ACPI options > CPU Frequency scaling > CPU Frequency scaling
Power management and ACPI options > CPU Idle > CPU idle PM support
Power management and ACPI options > Device power management core functionality
Power management and ACPI options > SFI (Simple Firmware Interface) Support
Power management and ACPI options > Suspend to RAM and standby
Binary Emulations > IA32 Emulation
Binary Emulations > x32 ABI for 64-bit mode
Device drivers > Accessibility support
Device drivers > Android > Android Drivers
Device drivers > Character devices > /dev/mem virtual device support
Device drivers > Character devices > /dev/port character device
Device drivers > Character devices > Enable TTY > Automatically load TTY Line Disciplines
Device drivers > Character devices > Enable TTY > Non-standard serial port support
Device drivers > Character devices > Enable TTY > Serial drivers > 8250/16550 and compatible serial support > Support for Fintek F81216A LPC to 4 UART RS485 API
Device drivers > Character devices > Enable TTY > Serial drivers > Support for Synopsys DesignWare 8250 quirks
Device drivers > Character devices > Enable TTY > Virtual terminal > Support for console on virtual terminal
Device drivers > Character devices > Hardware Random Number Generator Core support
Device drivers > Character devices > IPMI top-level message handler
Device drivers > Character devices > TPM Hardware Support
Device drivers > Connector - unified userspace <-> kernelspace linker
Device drivers > DAX: direct access to differentiated memory
Device drivers > Fusion MTP device support
Device drivers > GPIO Support
Device drivers > Generic Driver Options > Allow device coredump
Device drivers > Generic Driver Options > Disable drivers features which enable custom firmware building
Device drivers > Generic Driver Options > Firmware loader > Firmware loading facility
Device drivers > Generic Driver Options > Select only drivers that don't need compile-time external firmware
Device drivers > Generic Dynamic Voltage and Frequency Scalling (DVFS) support
Device drivers > Graphics support > /dev/agpgart (AGP Support)
Device drivers > Graphics support > Console display driver support > Framebuffer Console support
Device drivers > IOMMU Hardware Support
Device drivers > Input device support > Hardware I/O ports > Raw access to serio ports
Device drivers > Input device support > Joystick interface
Device drivers > Input device support > Joysticks/Gamepads > Joysticks/Gamepads
Device drivers > Input device support > Mice > Mice
Device drivers > Input device support > Miscellaneous devices > Miscellaneous devices
Device drivers > Input device support > Mouse interface
Device drivers > Input device support > Sparse keymap support library
Device drivers > Input device support > Tablets > Tablets
Device drivers > Input device support > Touchscreens > Touchscreens
Device drivers > LED Support
Device drivers > MMC/SD/SDIO card support
Device drivers > Macintosh device drivers
Device drivers > Mailbox Hardware Support
Device drivers > Multimedia support
Device drivers > Multiple devices driver support (RAID and LVM) > Device mapper support > DM uevents
Device drivers > Multiple devices driver support (RAID and LVM) > RAID support
Device drivers > NVME Support > NVMe multipath support
Device drivers > NVMEM Support
Device drivers > Network device support > Ethernet Driver support > Todo excepto Atheros devices
Device drivers > Network device support > FDDI driver support
Device drivers > Network device support > HIPPI driver support
Device drivers > Network device support > ISDN support
Device drivers > Network device support > Network core driver support
Device drivers > Network device support > Wan interfaces support
Device drivers > Network device support > Wireless LAN
Device drivers > PCI Support > Enable PCI quirk workarounds
Device drivers > PCI Support > Message Signaled Interrupts (MSI and MSI-X)
Device drivers > PCI Support > PCI Express ASPM Control
Device drivers > PCI Support > PCI Express Port Bus Support
Device drivers > PCI Support > PCI Express Precision Time Measurement support
Device drivers > PCI Support > PCI IOV support
Device drivers > PCI Support > PCI PASID support
Device drivers > PCI Support > PCI PRI support
Device drivers > PCI Support > Support for PCI Hotplug
Device drivers > PHY Subsystem > PHY Core
Device drivers > Pin controllers
Device drivers > Platform support for Chrome hardware
Device drivers > Power supply class support
Device drivers > Pulse-Width Modulation (PWM) Support
Device drivers > Real Time Clock
Device drivers > Reliability, Availability and Serviceability (RAS) features
Device drivers > SCSI device support > Asynchronous SCSI scanning
Device drivers > SCSI device support > SCSI Device Handlers
Device drivers > SCSI device support > SCSI generic support
Device drivers > SCSI device support > SCSI logging facility
Device drivers > SCSI device support > SCSI low-level drivers
Device drivers > SCSI device support > Verbose SCSI error reporting (kernel size += 36K)
Device drivers > SPI support
Device drivers > Serial ATA and Pararell ATA drivers (libata) > "libata.force=" kernel parameter support
Device drivers > Serial ATA and Pararell ATA drivers (libata) > ACard AHCI variant (ATP 8620)
Device drivers > Serial ATA and Pararell ATA drivers (libata) > ATA SFF support (for legacy IDE and PATA)
Device drivers > Serial ATA and Pararell ATA drivers (libata) > SATA Port Multiplier support
Device drivers > Serial ATA and Pararell ATA drivers (libata) > Verbose ATA error reporting
Device drivers > Sony MemoryStick card support
Device drivers > Sound card support
Device drivers > Staging drivers
Device drivers > Thermal drivers
Device drivers > USB support
Device drivers > VHOST drivers
Device drivers > Virtio drivers
Device drivers > Virtualization drivers
Device drivers > Voltage and Current Regulator Support
Device drivers > Watchdog Timer Support
Device drivers > X86 Platform Specific Device Drivers
Networking support > Amateur Radio support
Networking support > Bluetooth subsystem support
Networking support > Netlink interface for ethtool
Networking support > Network light weight tunnels
Networking support > Networking options > Data Center Bridging support
Networking support > Networking options > L3 Master device support
Networking support > Networking options > MultiProtocol Label Switching
Networking support > Networking options > Network classid cgroup
Networking support > Networking options > Network packet filtering framework (Netfilter)
Networking support > Networking options > Network priority cgroup
Networking support > Networking options > Security Marking
Networking support > Networking options > Switch (and switch-ish) device support
Networking support > Networking options > TCP/IP networking > IP: TCP syncookie support
Networking support > Networking options > TCP/IP networking > IP: advanced router
Networking support > Networking options > TCP/IP networking > IP: multicasting
Networking support > Networking options > TCP/IP networking > TCP: MD5 Signature Option support (RFC2385)
Networking support > Networking options > TCP/IP networking > TCP: advanced congestion control > TCP: advanced congestion control
Networking support > Networking options > TCP/IP networking > The IPv6 protocol > The IPv6 protocol
Networking support > Networking options > enable BPF Just In Time compiler
Networking support > Wireless
Cryptographic API > AES cipher algorithms (AES-NI)
Cryptographic API > CRCT10DIF PCLMULQDQ hardware acceleration
Cryptographic API > Certificates for signature checking > Provide system-wide ring of blacklisted keys
Cryptographic API > Certificates for signature checking > Provide system-wide ring of trusted keys > Provide a keyring to which extra trustable keys may be added
Cryptographic API > GHASH hash function (CLMUL-NI accelerated)
Cryptographic API > Hardware crypto devices
Cryptographic API > LZO compression algorithm
Security options > Enable access key retention support > Diffie-Hellman operations on retained keys
Security options > Enable access key retention support > Enable register of persistent per-UID keyrings
Security options > Enable different security models
Security options > Enable the securityfs filesystem
Security options > Harden common str/mem functions against buffer overflows
Security options > Harden memory copies between kernel and userspace
Security options > Remove the kernel mapping in user mode
Security options > Restrict unprivileged access to the kernel syslog
Kernel hacking > Compile-time checks and compiler options > Compile the kernel with debug info
Kernel hacking > Compile-time checks and compiler options > Enable __must_check logic
Kernel hacking > Compile-time checks and compiler options > Make section mismatch errors non-fatal
Kernel hacking > Compile-time checks and compiler options > Strip assembler-generated symbols during link
Kernel hacking > Debug Oops, Lockups and Hangs > Detect Hard Lockups
Kernel hacking > Debug Oops, Lockups and Hangs > Detect Hung Tasks
Kernel hacking > Debug Oops, Lockups and Hangs > Detect Soft Lockups
Kernel hacking > Debug kernel data structures > Debug linked list manipulation
Kernel hacking > Debug kernel data structures > Trigger a BUG when data corruption is detected
Kernel hacking > Generic Kernel Debugging Instruments > Debug Filesystem
Kernel hacking > Generic Kernel Debugging Instruments > Magic SysRq key
Kernel hacking > Kernel Testing and Coverage > Memtest
Kernel hacking > Kernel Testing and Coverage > Runtime Testing
Kernel hacking > Kernel debugging > Miscellaneous debug code
Kernel hacking > Memory Debugging > Debug memory initialisation
Kernel hacking > Memory Debugging > Detect stack corruption on calls to schedule()
Kernel hacking > Memory Debugging > Extend memmap on extra space for more information on page
Kernel hacking > Memory Debugging > Poison pages after freeing
Kernel hacking > Scheduler Debugging > Collect scheduler debugging info
Kernel hacking > Scheduler Debugging > Collect scheduler statistics
Kernel hacking > Stack backtrace support
Kernel hacking > Tracers
Kernel hacking > printk and dmesg options > Support symbolic error names in printf
Kernel hacking > x86 Debugging > Early printk
Kernel hacking > x86 Debugging > Warn on W+X mappings at boot
Processor type and features > Avoid speculative indirect branches in kernel
Processor type and features > Build a relocatable kernel
Processor type and features > CPU microcode loading support
Processor type and features > DMA memory allocation support
Processor type and features > Enable DMI scanning
Processor type and features > Enable seccomp to safely compute untrusted bytecode
Processor type and features > Enable the LDT (local descriptor table)
Processor type and features > Enable vsyscall emulation
Processor type and features > IOPERM and IOPL Emulation
Processor type and features > Intel Memory Protection Keys
Processor type and features > Linux guest support
Processor type and features > MTRR (Memory Type Range Register) support
Processor type and features > Machine Check / overheating reporting
Processor type and features > Old AMD GART IOMMU support
Processor type and features > Performance monitoring > Intel cstate performance events
Processor type and features > Performance monitoring > Intel uncore performance events
Processor type and features > Reroute for broken boot IRQs
Processor type and features > Single-depth WCHAN output
Processor type and features > Supervisor Mode Access Prevention
Processor type and features > Symmetric multi-processing support
Processor type and features > User Mode Instruction Prevention
Processor type and features > kernel crash dumps
Processor type and features > kexec file based system call
Processor type and features > kexec system call
Processor type and features > x86 architectural random number generator
Library routines > IRQ polling library
Library routines > XZ decompression support
Firmware drivers > Add firmware-provided memory map to sysfs
General architecture-dependent options > Kprobes
General architecture-dependent options > Optimize very unlikely/likely branches
General architecture-dependent options > Provide system calls for 32-bit time_t
General architecture-dependent options > Stack Protector buffer overflow detection
General architecture-dependent options > Use a virtually-mapped stack
Enable loadable module support > Forced module loading
Enable loadable module support > Module signature verification
Enable loadable module support > Module unloading
Enable loadable module support > Module versioning support
Enable the block layer > Block layer SG support v4
Enable the block layer > Block layer SG support v4 helper lib
Enable the block layer > Block layer data integrity support
Enable the block layer > Enable support for block device writeback throttling
Enable the block layer > Logic for interfacing with Opal enabled SEDs
Enable the block layer > Partition Types > Advanced partition selection > Acorn partition support
Enable the block layer > Partition Types > Advanced partition selection > Alpha OSF partition support
Enable the block layer > Partition Types > Advanced partition selection > Amiga partition table support
Enable the block layer > Partition Types > Advanced partition selection > Atari partition table support
Enable the block layer > Partition Types > Advanced partition selection > Karma Partition support
Enable the block layer > Partition Types > Advanced partition selection > Macintosh partition map support
Enable the block layer > Partition Types > Advanced partition selection > PC BIOS (MSDOS partition tables) support
Enable the block layer > Partition Types > Advanced partition selection > SGI partition support
Enable the block layer > Partition Types > Advanced partition selection > Sun partition tables support
Enable the block layer > Partition Types > Advanced partition selection > Ultrix partition table support
Enable the block layer > Partition Types > Advanced partition selection > Windows Logical Disk Manager (Dynamic disk) support
Enable the block layer > Zoned block device support
IO Schedulers > MQ deadline I/O scheduler
Executable file formats > Enable core dump support
Memory Management options > Allow for memory compaction
Memory Management options > Allow for memory hot-add
Memory Management options > Common API for compressed memory storage
Memory Management options > Free page reporting
Memory Management options > Low (Up to 2x) density storage for compressed pages
Memory Management options > Sparse Memory virtual memmap
Memory Management options > Transparent Hugepage Support
{% endhighlight %}