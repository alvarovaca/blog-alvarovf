---
layout: post
title:  "Compilación de un programa en C utilizando un Makefile"
banner: "/assets/images/banners/compilacion.jpg"
date:   2020-10-31 20:56:00 +0200
categories: sistemas
---
Antes de comenzar a compilar un programa en C, he considerado oportuno realizar una serie de aclaraciones para saber qué es lo que estamos haciendo y por qué lo hacemos de una manera concreta.

Es conocido por todos que cuando desarrollamos un programa en un determinado lenguaje de programación, lo hacemos a un nivel "fácilmente" legible por una persona humana, es decir, alto nivel, pero la máquina, no es capaz de comprenderlo, y por tanto, debe llevarse a cabo un proceso de traducción de dichas instrucciones para convertirlas a un formato legible por la máquina (bajo nivel). Este procedimiento, dependiendo del lenguaje de programación que hayamos usado, lo lleva a cabo un **compilador** o un **intérprete**.

¿En qué se diferencian estos dos programas? Bien, un **intérprete** procesa el código fuente durante el tiempo de ejecución, es decir, mientras el software se está ejecutando, de manera que actúa como una "interfaz" entre el proyecto y el procesador, que procesa el código línea por línea, analizando y preparando cada secuencia de forma consecutiva para el procesador. Para llevar a cabo esta operación, el intérprete recurre a sus propias bibliotecas internas, por lo que en cuanto una línea de código fuente se ha traducido a los correspondientes comandos legibles por la máquina, esta se envía directamente al procesador. El proceso de conversión finaliza cuando se ha interpretado todo el código (a excepción de que ocurra un fallo durante el procesamiento, de manera que la línea de código problemática se detecta inmediatamente, facilitando así la búsqueda de errores). Tiene un proceso de desarrollo relativamente sencillo y fácilmente modificable, pero con el inconveniente de que ha de interpretarse una vez por cada ejecución, afectando a la eficiencia. _Python_, _Ruby_ y _PHP_ son ejemplos de lenguajes interpretados.

En el otro lado, tenemos a los **compiladores**, que hacen justo lo contrario, pues la traducción del código fuente (fichero **.c**) se lleva a cabo antes de ejecutarlo, pues no necesitan un intérprete, posibilitando la ejecución una vez que la compilación ha finalizado, es decir, una vez que todas las instrucciones han sido traducidas a código máquina. Sin embargo, en muchos casos, durante el propio proceso de compilación tiene lugar un paso intermedio, pues se suele convertir el código fuente en un lenguaje legible por la máquina, que se encuentra contenido en un fichero intermedio que suele tener la extensión **.o** o **.obj**, conocido como objetos. Ya hemos realizado la mitad del proceso, así que el protagonista de esta segunda parte es el **_linker_**, que tiene la finalidad de combinar diferentes ficheros objeto en un único ejecutable (binario), existiendo dos tipos claramente diferenciados:

* **Enlazado estático**: En el binario se incluye el propio programa y todas las dependencias necesarias para su correcto funcionamiento. Para llevar a cabo este tipo de enlazado es necesario tener en la máquina el código fuente del programa y de todas sus dependencias, por lo tanto, cuando se ejecute el binario, no hará falta tener nada instalado en la máquina en cuanto a dependencias se refiere, ya que todas ellas se encuentran contenidas en el propio binario, para posteriormente cargarse en memoria junto al código del programa. Como consecuencia, el binario es más grande, y cualquier cambio en dichas librerías supondría tener que volver a compilar. Además, requiere que las licencias de todos los componentes sean compatibles con la del programa que las utiliza, para su posterior distribución.

* **Enlazado dinámico**: En el binario únicamente se incluye el propio programa y una referencia a las dependencias. Para llevar a cabo este tipo de enlazado es necesario tener en la máquina el código fuente del programa y los ficheros de cabeceras de sus dependencias (en Debian suelen tener la terminación **-dev**), por lo tanto, cuando se ejecute el binario, es necesario que en el equipo estén instaladas las dependencias en formato binario, compiladas individualmente, en muchos casos como bibliotecas de enlace dinámico (en el caso de Linux, aquellos ficheros con extensión **.so**, es decir, _shared object_). Como consecuencia, el binario es más pequeño, y cualquier cambio en dichas librerías no supondría tener que volver a compilar, ya que no se encuentran contenidas en el mismo, sino que el código necesario se añade en tiempos de ejecución, permitiendo así, una mayor flexibilidad en la gestión de los bugs. Es el método más adoptado por las distribuciones Linux, al no haber problema en incluir todo el código necesario como dependencias al ser código libre, pero no todo es tan bonito, y es que para gestionar dichas dependencias, se requiere un sistema avanzado y complejo como dpkg o rpm.

El uso de compiladores supone una mayor velocidad de ejecución, al tener el código máquina al completo antes de su ejecución, pero con la desventaja de que cualquier modificación del código requiere volver a realizar la compilación. _C_, _C++_ y _Pascal_ son ejemplos de lenguajes que utilizan compiladores.

En el proyecto GNU existe un conjunto de compiladores de código abierto de nombre **gcc** (_GNU Compiler Collection_, es decir, colección de compiladores GNU), que originalmente únicamente compilaba el lenguaje **C**, pero que posteriormente se extendió para compilar **C++**, **Fortran**, **Ada** y otros. La gran característica es que **gcc** permite compilar y posteriormente enlazar (_linkar_), haciendo uso de un conjunto de aplicaciones conocido como **binutils** (por defecto lleva a cabo ambos procesos, a no ser que le indiquemos lo contrario).

En proyectos un poco "complicados" que utilicen más de un archivo o librerías, debemos compilar primero los ficheros **.c**, haciendo uso de la opción **-c** y posteriormente _linkarlos_. Por esa misma razón, cuando queremos compilar programas más grandes, en los que hay gran cantidad de ficheros fuente (**.c**), _headers_ (**.h**) y librerías, sería totalmente inviable compilarlos todos a mano para posteriormente _linkar_ todos los **.o** en un único ejecutable. Es por ello que recurrimos al comando **make**, que nos permite hacerlo de una manera mucho más eficiente e inteligente gracias a un fichero de nombre **Makefile**, en el que se define mediante instrucciones, el orden en el que se compilarán los ficheros **.c** y _linkarán_ los ficheros **.o**. Otra característica muy importante es que en caso de llevar a cabo una modificación en cualquiera de los ficheros fuente, únicamente será necesario recompilar aquellos ficheros que dependan del que hemos modificado, evitando así errores y disminuyendo el tiempo de compilación.

Para tener una idea general, pues no vamos a entrar en profundidad en el tema ya que se saldría del objetivo de este artículo, el fichero **Makefile** tiene entradas de la siguiente forma:

{% highlight shell %}
OBJETIVOS: REQUISITOS
    REGLA PARA CONSEGUIRLO
{% endhighlight %}

Por ejemplo, esto sería un fichero **Makefile** (muy simple, pero nos vale para entender la filosofía):

{% highlight shell %}
all: a.out
a.out: suid.o
    gcc suid.o
suid.o: suid.c
    gcc -c suid.c
{% endhighlight %}

Como se puede apreciar, el objetivo final es el de obtener el fichero ejecutable **a.out** resultante del _linkado_ (**gcc**) del fichero objeto, pero para ello, tenemos que obtener previamente el fichero objeto **suid.o**, ¿pero cómo obtenemos dicho fichero objeto? Pues como resultado de la compilación (**gcc -c**) del código fuente **suid.c**. Como se puede apreciar, son una serie de instrucciones entrelazadas, facilitándonos todo el proceso, pues únicamente tendríamos que ejecutar el comando `make`.

A la hora de compilar un programa con **make** para Linux, lo habitual es que esté preparado para aprovechar al máximo la compilación dinámica, por lo que en la configuración (si está bien hecha), va a verificar que estén instalados en el equipo todas las bibliotecas necesarias para que pueda producir un binario sin que le falte nada (script **configure** que veremos a continuación). En caso de que la configuración no esté bien hecha del todo, el error lo mostrará en la compilación, quejándose de que falta alguna biblioteca que está referenciada en el código y no puede seguir hasta que la instalemos.

Creo que ha quedado bastante claro qué es lo que buscamos y cómo vamos a hacerlo, así que vamos a ponernos manos a la obra.

En este caso, el programa que quiero compilar es [emacs](https://www.gnu.org/software/emacs/), un editor de texto con gran cantidad de funciones. Lo primero que necesitaremos es el código fuente del mismo, así que llegados a este punto se nos plantean dos opciones:

* **Descargar la release** desde la [web](http://ftp.gnu.org/gnu/emacs/) (podemos hacerlo a mano, con `wget`...) para posteriormente ejecutar el script **configure** (del que hablaremos a continuación). Tendremos que hacer caso a las instrucciones del fichero **INSTALL**.
* **Clonar desde el repositorio** de [GitHub](https://github.com/emacs-mirror/emacs) o [Savannah](https://savannah.gnu.org/projects/emacs/) para posteriormente ejecutar **autogen.sh** para que aparezca el famoso **configure**. Tendremos que hacer caso a las instrucciones del fichero **INSTALL.REPO**.

Vamos a hablar un poco del script **configure**. Dicho script es el responsable de dejarlo todo preparado para la posterior construcción del software en nuestro sistema en específico. Se asegura de que todas las dependencias para el resto del proceso de construcción e instalación estén disponibles y descubre todo lo que necesita saber para usar dichas dependencias, como por ejemplo, averiguar cómo se llama el compilador de C existente en el sistema y dónde encontrarlo. En caso de no encontrar una dependencia necesaria, nos lo informará y tras resolverla, podremos volver a ejecutarlo. Dicho script hace uso de una plantilla llamada **Makefile.in**, dando lugar finalmente al fichero **Makefile** específico para nuestro sistema. Este _modus operandi_ se debe a que no es posible crear un Makefile genérico que sea válido para todos los sistemas, pues la instalación depende en una gran parte de nuestra estructura.

Sin embargo, en caso de clonar desde el repositorio, tenemos que hacer un paso previo para generar el script **configure**, ya que no viene incluido. Dicho paso consiste en la ejecución del script **autogen.sh**, que llevará a cabo determinadas tareas para facilitar el trabajo de los mantenedores, ya que generar un script **configure** a mano (junto a algunos otros ficheros), les llevaría mucho tiempo, debida la longitud del mismo. Efectivamente, **autogen.sh** está enfocado para los desarrolladores que deseen hacer cambios en el fichero **configure.ac**, fichero que es usado como plantilla para generar el script **configure**. Para ello, necesitamos tener instalado en la máquina el paquete **autotools**.

En este caso, dado que lo único que queremos es llevar a cabo una compilación, no es necesario clonar desde el repositorio, pues no deseamos modificar el fichero **configure.ac**, de manera que agilizaremos el proceso descargando directamente la release desde la web, es decir, desde el [servidor FTP](https://ftp.gnu.org/gnu/) del proyecto GNU.

Me encuentro en una máquina recién instalada sin interfaz gráfica, así que antes de proseguir, vamos a verificar que todos los paquetes instalados en la máquina se encuentran actualizados (deberían estarlo ya que se acaba de instalar), ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt update && sudo apt upgrade
{% endhighlight %}

Una vez verificado, vamos a verificar también que no tenemos instalado **emacs** (aunque de haberlo tenido, tampoco habría inconveniente, pues la instalación la vamos a llevar a cabo en un directorio que no pisaría la instalación actual), haciendo para ello uso del comando `dpkg -s`:

{% highlight shell %}
alvaro@debian:~$ dpkg -s emacs
dpkg-query: package 'emacs' is not installed and no information is available
Use dpkg --info (= dpkg-deb --info) to examine archive files.
{% endhighlight %}

Efectivamente, **emacs** no se encuentra instalado, así que lo primero que vamos a hacer, para mantener la organización, es generar un directorio (de nombre **emacs**, por ejemplo) en el que vamos a situar todos los ficheros necesarios para la compilación. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ mkdir emacs
{% endhighlight %}

Para comprobar que la generación de dicho directorio se ha llevado a cabo satisfactoriamente, vamos a listar el contenido del directorio actual, haciendo uso de `ls -l`:

{% highlight shell %}
alvaro@debian:~$ ls -l
total 4
drwxr-xr-x 2 alvaro alvaro 4096 Oct 24 15:19 emacs
{% endhighlight %}

Como era de esperar, el directorio **emacs/** ha sido correctamente generado en el directorio personal, así que nos moveremos dentro del mismo, ejecutando la instrucción:

{% highlight shell %}
alvaro@debian:~$ cd emacs/
{% endhighlight %}

Ya estamos dentro, así que es hora de descargar el fichero comprimido de _emacs_, que en este caso, descargaré la última versión disponible a día de hoy, la [27.1](http://ftp.gnu.org/gnu/emacs/emacs-27.1.tar.xz). Tenemos varias posibilidades para descargarlo, pero por comodidad, haré uso de `wget`, que descargará el fichero que le indiquemos en el directorio actual, pasándole el enlace de descarga:

{% highlight shell %}
alvaro@debian:~/emacs$ wget http://ftp.gnu.org/gnu/emacs/emacs-27.1.tar.xz
--2020-10-28 08:11:41--  http://ftp.gnu.org/gnu/emacs/emacs-27.1.tar.xz
Resolving ftp.gnu.org (ftp.gnu.org)... 209.51.188.20, 2001:470:142:3::b
Connecting to ftp.gnu.org (ftp.gnu.org)|209.51.188.20|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 43752012 (42M) [application/x-xz]
Saving to: ‘emacs-27.1.tar.xz’

emacs-27.1.tar.xz   100%[===================>]  41.72M  6.81MB/s    in 6.4s

2020-10-28 08:11:48 (6.57 MB/s) - ‘emacs-27.1.tar.xz’ saved [43752012/43752012]
{% endhighlight %}

De nuevo, vamos a listar el contenido del directorio actual para verificar que la descarga se ha completado satisfactoriamente:

{% highlight shell %}
alvaro@debian:~/emacs$ ls -l
total 42728
-rw-r--r-- 1 alvaro alvaro 43752012 Aug 10 22:45 emacs-27.1.tar.xz
{% endhighlight %}

Efectivamente, se ha descargado el fichero **emacs-27.1.tar.xz** de peso total **41.72M** (43752012 bytes). Si nos fijamos, el paquete es un comprimido, por lo que no podemos llevar a cabo la compilación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello el fichero comprimido ya que no nos hará falta. Para descomprimir un paquete **.tar.xz** necesitamos las **xz-utils** instaladas en la máquina, así que antes de tratar de instalarlas, vamos a verificar si se encuentran ya instaladas, ejecutando el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ dpkg -s xz-utils
dpkg-query: package 'xz-utils' is not installed and no information is available
Use dpkg --info (= dpkg-deb --info) to examine archive files.
{% endhighlight %}

En este caso, el paquete no se encuentra instalado, por lo que no podemos realizar la descompresión. Para ello, tendremos que instalar manualmente el paquete **xz-utils**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install xz-utils
{% endhighlight %}

Ya está todo listo para realizar la descompresión y la posterior eliminación del comprimido para liberar espacio. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ tar -Jxf emacs-27.1.tar.xz --strip 1 && rm -r emacs-27.1.tar.xz
{% endhighlight %}

Donde:

* **-J**: Utiliza xz para descomprimir el fichero.
* **-x**: Indica a tar que desempaquete el fichero.
* **-f**: Indica a tar que el siguiente argumento es el nombre del fichero .tar.xz.
* **--strip 1**: Saltamos el primer directorio, ya que dentro del comprimido hay un directorio padre que no necesitamos.

Para verificar que el fichero se ha descomprimido correctamente, haremos uso del comando:

{% highlight shell %}
alvaro@debian:~/emacs$ ls
aclocal.m4  BUGS       ChangeLog.1  config.bat    CONTRIBUTE  etc          INSTALL       lib      lwlib      Makefile.in  msdos     oldXMenu   src
admin       build-aux  ChangeLog.2  configure     COPYING     GNUmakefile  INSTALL.REPO  lib-src  m4         MANIFEST     nextstep  README     test
autogen.sh  ChangeLog  ChangeLog.3  configure.ac  doc         info         leim          lisp     make-dist  modules      nt        site-lisp
{% endhighlight %}

Efectivamente, todo el contenido se ha descomprimido tal y como queríamos (en lugar de descomprimir un directorio de nombre **emacs-27.1** del que posteriormente tendríamos que mover los ficheros contenidos al directorio actual). Además, si nos fijamos, existe el script **autogen.sh**, que usa el fichero **configure.ac** para generar el script **configure**. En este caso, tal y como se ha mencionado anteriormente, no sería necesario hacer uso de **autogen.sh** ya que **configure** viene ya generado. Sin embargo, sí tendremos que hacer uso de **configure** para así generar el **Makefile** haciendo uso de la plantilla **Makefile.in**, ya que dicho fichero no viene generado, al no ser genérico (pues depende de la estructura existente en cada caso).

Por curiosidad, vamos a ver el peso del contenido tras su descompresión, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh .
246M    .
{% endhighlight %}

En este momento, el peso (recursivo) de todos los ficheros del directorio actual es de **246M**, tamaño que irá en aumento conforme vayamos avanzando en la tarea.

Una vez que tenemos todo descomprimido, nos podemos fijar que ha aparecido un fichero de nombre **README**. Como su propio nombre indica, debemos leerlo, pues contendrá información bastante útil para la posterior compilación. Para ello, haremos uso de un paginador como `less`:

{% highlight shell %}
less README
{% endhighlight %}

Como sería poco práctico copiar y pegar aquí el contenido de dicho fichero, voy a tratar de resumir de la mejor forma posible los aspectos más importantes del mismo. En dicho documento, se nos explica la funcionalidad de los distintos scripts tales como **autogen.sh** y **configure**, indicando los ficheros que usan respectivamente como plantillas, para hacer su trabajo. Además, se nos explica el contenido de cada uno de los directorios existentes, siendo los más importantes **src/** y **lib/**.

* **src/**: Contiene el código fuente en C para _emacs_. El contenido es el siguiente:

{% highlight shell %}
alvaro@debian:~/emacs$ ls src/
alloc.c       ChangeLog.11  commands.h     dynlib.c           frame.h        json.c       mini-gmp-emacs.c  ptr-bounds.h    systime.h    unexelf.c      w32notify.c     xmenu.c
atimer.c      ChangeLog.12  composite.c    dynlib.h           fringe.c       keyboard.c   mini-gmp.h        puresize.h      systty.h     unexhp9k800.c  w32proc.c       xml.c
atimer.h      ChangeLog.13  composite.h    editfns.c          ftcrfont.c     keyboard.h   module-env-25.h   ralloc.c        syswait.h    unexmacosx.c   w32reg.c        xrdb.c
bidi.c        ChangeLog.2   config.in      emacs.c            ftfont.c       keymap.c     module-env-26.h   README          term.c       unexsol.c      w32select.c     xselect.c
bignum.c      ChangeLog.3   conf_post.h    emacsgtkfixed.c    ftfont.h       keymap.h     module-env-27.h   regex-emacs.c   termcap.c    unexw32.c      w32select.h     xsettings.c
bignum.h      ChangeLog.4   COPYING        emacsgtkfixed.h    ftxfont.c      kqueue.c     msdos.c           regex-emacs.h   termchar.h   vm-limit.c     w32term.c       xsettings.h
bitmaps       ChangeLog.5   cygw32.c       emacs-icon.h       getpagesize.h  lastfile.c   msdos.h           region-cache.c  termhooks.h  w16select.c    w32term.h       xsmfns.c
blockinput.h  ChangeLog.6   cygw32.h       emacs-module.c     gfilenotify.c  lcms.c       nsfns.m           region-cache.h  terminal.c   w32.c          w32uniscribe.c  xterm.c
buffer.c      ChangeLog.7   data.c         emacs-module.h.in  gmalloc.c      lisp.h       nsfont.m          scroll.c        terminfo.c   w32common.h    w32xfns.c       xterm.h
buffer.h      ChangeLog.8   dbusbind.c     epaths.in          gnutls.c       lread.c      nsgui.h           search.c        termopts.h   w32console.c   widget.c        xwidget.c
bytecode.c    ChangeLog.9   decompress.c   eval.c             gnutls.h       macfont.h    nsimage.m         sheap.c         textprop.c   w32cygwinx.c   widget.h        xwidget.h
callint.c     character.c   deps.mk        fileio.c           gtkutil.c      macfont.m    nsmenu.m          sheap.h         thread.c     w32fns.c       widgetprv.h
callproc.c    character.h   dired.c        filelock.c         gtkutil.h      macros.c     nsselect.m        sound.c         thread.h     w32font.c      window.c
casefiddle.c  charset.c     dispextern.h   firstfile.c        hbfont.c       macros.h     nsterm.h          syntax.c        timefns.c    w32font.h      window.h
casetab.c     charset.h     dispnew.c      floatfns.c         image.c        macuvs.h     nsterm.m          syntax.h        tparam.c     w32gui.h       xdisp.c
category.c    chartab.c     disptab.h      fns.c              indent.c       Makefile.in  pdumper.c         sysdep.c        tparam.h     w32.h          xfaces.c
category.h    cm.c          dmpstruct.awk  font.c             indent.h       marker.c     pdumper.h         sysselect.h     undo.c       w32heap.c      xfns.c
ccl.c         cmds.c        doc.c          font.h             inotify.c      menu.c       print.c           syssignal.h     unexaix.c    w32heap.h      xfont.c
ccl.h         cm.h          doprnt.c       fontset.c          insdel.c       menu.h       process.c         sysstdio.h      unexcoff.c   w32inevt.c     xftfont.c
ChangeLog.1   coding.c      dosfns.c       fontset.h          intervals.c    minibuf.c    process.h         systhread.c     unexcw.c     w32inevt.h     xgselect.c
ChangeLog.10  coding.h      dosfns.h       frame.c            intervals.h    mini-gmp.c   profiler.c        systhread.h     unexec.h     w32menu.c      xgselect.h
{% endhighlight %}

Como se puede apreciar, existen una gran cantidad de ficheros código fuente (**.c**), así como _headers_ (**.h**), siendo la finalidad de estos últimos la de definir las funciones que existen en las librerías (prototipos), que contiene el tipo de dato devuelto, el nombre de la función y los parámetros, pues en C, es necesario que el programa conozca dicha definición. Dichas cabeceras son posteriormente incluidas (**#include**) en el programa principal para que sean conocidas.

* **lib/**: Contiene el código fuente de las librerías usadas por _emacs_ y sus utilidades. El contenido es el siguiente:

{% highlight shell %}
alvaro@debian:~/emacs2$ ls lib
acl_entries.c        cloexec.h               errno.in.h        fsync.c            ieee754.in.h        mktime-internal.h  regex_internal.h   stdlib.in.h      timespec-add.c
acl-errno-valid.c    close-stream.c          euidaccess.c      ftoastr.c          ignore-value.h      _Noreturn.h        root-uid.h         stpcpy.c         timespec.c
acl.h                close-stream.h          execinfo.c        ftoastr.h          intprops.h          nstrftime.c        save-cwd.c         strftime.h       timespec.h
acl-internal.c       copy-file-range.c       execinfo.in.h     getdtablesize.c    inttypes.in.h       openat-die.c       save-cwd.h         string.in.h      timespec-sub.c
acl-internal.h       COPYING                 explicit_bzero.c  getgroups.c        libc-config.h       openat.h           set-permissions.c  strnlen.c        u64.c
alloca.in.h          count-leading-zeros.c   faccessat.c       getloadavg.c       limits.in.h         openat-priv.h      sha1.c             strtoimax.c      u64.h
allocator.c          count-leading-zeros.h   fcntl.c           getopt1.c          localtime-buffer.c  openat-proc.c      sha1.h             strtol.c         unistd.c
allocator.h          count-one-bits.c        fcntl.in.h        getopt.c           localtime-buffer.h  open.c             sha256.c           strtoll.c        unistd.in.h
arg-nonnull.h        count-one-bits.h        fdopendir.c       getopt-cdefs.in.h  lstat.c             pathmax.h          sha256.h           str-two-way.h    unlocked-io.h
at-func.c            count-trailing-zeros.c  filemode.c        getopt-core.h      Makefile.in         pipe2.c            sha512.c           symlink.c        utimens.c
binary-io.c          count-trailing-zeros.h  filemode.h        getopt-ext.h       malloca.c           pselect.c          sha512.h           sys_select.in.h  utimens.h
binary-io.h          c-strcasecmp.c          filevercmp.c      getopt.in.h        malloca.h           pthread_sigmask.c  sig2str.c          sys_stat.in.h    verify.h
byteswap.in.h        c-strcase.h             filevercmp.h      getopt_int.h       md5.c               putenv.c           sig2str.h          sys_time.in.h    vla.h
canonicalize-lgpl.c  c-strncasecmp.c         fingerprint.c     getopt-pfx-core.h  md5.h               qcopy-acl.c        signal.in.h        sys_types.in.h   warn-on-use.h
careadlinkat.c       diffseq.h               fingerprint.h     getopt-pfx-ext.h   memmem.c            readlinkat.c       stat-time.c        tempname.c       xalloc-oversized.h
careadlinkat.h       dirent.in.h             flexmember.h      get-permissions.c  mempcpy.c           readlink.c         stat-time.h        tempname.h
c-ctype.c            dirfd.c                 fpending.c        gettext.h          memrchr.c           regcomp.c          stdalign.in.h      timegm.c
c-ctype.h            dosname.h               fpending.h        gettime.c          min-max.h           regex.c            stddef.in.h        time.in.h
c++defs.h            dtoastr.c               fstatat.c         gettimeofday.c     minmax.h            regexec.c          stdint.in.h        time-internal.h
cdefs.h              dtotimespec.c           fsusage.c         gnulib.mk.in       mkostemp.c          regex.h            stdio-impl.h       time_r.c
cloexec.c            dup2.c                  fsusage.h         group-member.c     mktime.c            regex_internal.c   stdio.in.h         time_rz.c
{% endhighlight %}

Al igual que en el caso anterior, existen una gran cantidad de ficheros código fuente (**.c**), así como _headers_ (**.h**), necesarios para las librerías usadas por _emacs_, así como sus utilidades.

En dicho fichero, a su vez, se hace referencia a otro, **INSTALL**, un fichero que nos indicará los pasos a seguir durante la compilación (en caso de haber clonado desde repositorio, el nombre del fichero sería **INSTALL.REPO**). De nuevo, realizaremos la lectura de dicho fichero ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ less INSTALL
{% endhighlight %}

Partamos del punto de que ya hemos ejecutado correctamente el script **configure** y ha finalizado, de manera que ya ha generado el **Makefile**. Todavía nos falta compilar y _linkar_ todos los ficheros previamente vistos, función que llevaremos a cabo con el comando **make**, pues hará uso de ese Makefile, tal y como hemos visto anteriormente.

Bien, una vez que todo se haya compilado y enlazado correctamente, se generará el binario (ejecutable) dentro de **src/**, pero es poco práctico tener que acceder a dicho directorio cada vez que queramos usar _emacs_, y es por ello, que tendremos que instalarlo. El proceso de instalación consiste en simplemente ubicar una serie de ficheros (binarios, librerías...) en una serie de directorios (que por defecto cuelgan de **/usr/local**), pudiendo utilizar el programa normalmente a partir de este momento. Para ello, haremos uso de **make install**, pero todavía no hemos llegado a esa parte, así que vamos a empezar el proceso.

Lo primero que tendremos que hacer, tal y como hemos visto, es hacer uso del script **configure** para generar así el **Makefile** adaptado a nuestra máquina. Sin embargo, antes de ello, me quiero asegurar de algo, y es que al tratarse de una máquina "minimalista", muy posiblemente no venga incorporada con las utilidades necesarias para llevar a cabo la compilación, tales como **gcc** o **make**. Para comprobarlo, volveremos a hacer uso de `dpkg -s`:

{% highlight shell %}
alvaro@debian:~/emacs$ dpkg -s gcc
dpkg-query: package 'gcc' is not installed and no information is available
Use dpkg --info (= dpkg-deb --info) to examine archive files.

alvaro@debian:~/emacs$ dpkg -s make
dpkg-query: package 'make' is not installed and no information is available
Use dpkg --info (= dpkg-deb --info) to examine archive files.
{% endhighlight %}

Como suponía, dichos paquetes no se encuentran instalados. Al igual que dichos paquetes faltan (son los esenciales), faltarán muchos otros, tales como librerías. Es por ello que tenemos dos opciones:

* Ejecutar el script **configure** e ir instalando las utilidades y dependencias una por una, conforme el script vaya notificando los errores.

* Agilizar el proceso e instalar **build-essential**, un paquete que como su propio nombre indica, contiene las utilidades esenciales para compilar un paquete en Debian. Nosotros optaremos por instalar dicho paquete, ya que nos evitará tener que instalar una gran cantidad de paquetes a mano.

Para realizar la instalación del paquete anteriormente mencionado, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install build-essential
{% endhighlight %}

Una vez instalado el paquete, ya podremos lanzar el script, sin embargo, tal y como se ha mencionado con anterioridad, cuando vayamos a realizar la instalación tras haber compilado (con **make install**), se instalará por defecto en **/usr/local**, un directorio que está destinado a albergar el software instalado localmente por el administrador, generalmente como resultado de una compilación, de manera que no sobreescriba el paquete original, que se encontrará en **/usr**. Más adelante entraremos con mayor profundidad en este tema.

Sin embargo, tras informarme un poco sobre el tema, a la hora de desinstalar el paquete con **make uninstall** (también lo veremos con más detalle a continuación), dependemos en cierta forma de lo bien que haya hecho el desarrollador el script de desinstalación, que en ciertas ocasiones, puede dejar algún rastro en la máquina no deseado (suponiendo que haya hecho dicho script, ya que hay algunos que no lo proporcionan). Para solucionar esto, tenemos varias opciones:

* Ejecutar el comando **make -n install**, de manera que nos informará de los pasos que ha seguido **make** para instalar el software, permitiéndonos realizarlos de forma inversa, llevando a cabo por tanto la desinstalación de forma manual. Esta opción puede llegar a ser algo tediosa.
* Usar **make checkinstall** en lugar de **make install**, que generará un fichero **.deb** que podremos instalar con nuestro gestor de paquetes favorito, como por ejemplo **dpkg**.
* Realizar la instalación en un directorio apartado que nosotros creemos (generalmente dentro de **/opt/**), por lo que cuando hagamos la desinstalación, en caso de que quedase algún rastro, con eliminar dicho directorio cortaríamos de raíz el problema. Para esta ocasión, voy a optar por esta posibilidad.

En este caso, voy a generar un directorio dentro de **/opt/**, de nombre **emacs/**. En realidad, podría instalarse en otro directorio, por ejemplo, en un subdirectorio en el directorio personal, pero no es lo óptimo, ya que eso debería hacerse sólo en caso de que el software únicamente vaya a ser usado por dicho usuario, cosa de la que no tenemos certeza, así que lo pondremos disponible para todos los usuarios, instalándolo por tanto, dentro de **/opt**. Para llevar a cabo la generación de dicho directorio, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo mkdir /opt/emacs
{% endhighlight %}

Para verificar que dicho directorio se ha creado correctamente, listaremos el contenido de **/opt/**, haciendo uso de `ls`:

{% highlight shell %}
alvaro@debian:~/emacs$ ls /opt/
emacs
{% endhighlight %}

Efectivamente, el directorio se encuentra creado, así que ahora sí que sí, vamos a pasar a ejecutar el script **configure**, no sin antes indicarle con el parámetro **--prefix** el directorio que posteriormente usará para la instalación (pues podemos personalizar la instalación a través de determinados parámetros), en este caso, **/opt/emacs/**:

{% highlight shell %}
alvaro@debian:~/emacs$ ./configure --prefix=/opt/emacs/
checking for xcrun... no
checking for GNU Make... make
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking for gcc... gcc
...
checking for dladdr... yes
checking for dlfunc... no
configure: error: The following required libraries were not found:
     gnutls
Maybe some development libraries/packages are missing?
To build anyway, give:
     --with-gnutls=ifavailable
as options to configure.
{% endhighlight %}

La salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/yPc45nsd) se puede ver al completo. La ejecución del mismo ha finalizado debido a un error de falta de librerías. En este caso, la librería que nos falta es **gnutls**, por lo que debemos instalarla antes de volver a ejecutar el script (también podríamos seguir con la instalación obviando este error, pero no es lo aconsejable).

Tras buscar un poco de información al respecto, tenemos que instalar el paquete de desarrollo de **gnutls**, que encontraremos ejecutando el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ apt-cache search 'libgnutls.*-dev'
libgnutls28-dev - GNU TLS library - development files
{% endhighlight %}

En este caso, el paquete que debemos instalar es **libgnutls28-dev**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install libgnutls28-dev
{% endhighlight %}

Tras la instalación del paquete, volví a ejecutar el script **configure**, pero para mi sorpresa, el problema persistía. Tras un buen rato peleándome con esto, descubrí que existe un paquete llamado **pkg-config**, que provee una interfaz unificada para llamar bibliotecas instaladas cuando se está compilando un programa a partir del código, o en castellano, lo que ha ocurrido es que el script **configure**, no era capaz de encontrar dicha librería a pesar de estar instalada, y es por ello que le proporcionaremos la ayuda de **pkg-config**, que conoce la ruta en la que se encuentra. Para su correspondiente instalación, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install pkg-config
{% endhighlight %}

Genial, ya podemos volver a ejecutar el script **configure**, haciendo uso del comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/emacs$ ./configure --prefix=/opt/emacs/
checking for xcrun... no
checking for GNU Make... make
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking for gcc... gcc
...
checking for posix_openpt... yes
checking for library containing tputs... no
configure: error: The required function 'tputs' was not found in any library.
The following libraries were tried (in order):
  libtinfo, libncurses, libterminfo, libcurses, libtermcap
Please try installing whichever of these libraries is most appropriate
for your system, together with its header files.
For example, a libncurses-dev(el) or similar package.
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/U78LR1PE) se puede ver al completo. Una vez más, la ejecución del mismo ha finalizado debido a un error. En este caso, ha intentado llamar a una función de nombre **tputs** y no la ha encontrado a pesar de haber probado en cinco librerías distintas. Para solucionarlo, tendremos que instalar cualquiera de dichas librerías, que proporciona la función. En este caso, he optado por instalar **libncurses-dev**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install libncurses-dev
{% endhighlight %}

Una vez más, trataremos de ejecutar el script **configure**, haciendo uso del comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/emacs$ ./configure --prefix=/opt/emacs/
checking for xcrun... no
checking for GNU Make... make
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking for gcc... gcc
...
config.status: executing doc/emacs/emacsver.texi commands
config.status: executing etc-refcards-emacsver.tex commands
configure: WARNING: This configuration installs a 'movemail' program
that does not retrieve POP3 email.  By default, Emacs 25 and earlier
installed a 'movemail' program that retrieved POP3 email via only
insecure channels, a practice that is no longer recommended but that
you can continue to support by using './configure --with-pop'.
configure: You might want to install GNU Mailutils
<https://mailutils.org> and use './configure --with-mailutils'.
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/gZ2XC1Pu) se puede ver al completo. En este caso, la ejecución del mismo ha finalizado pero mostrando una advertencia de que no se debería usar **movemail** para la recepción de emails, ya que lo hacía mediante canales inseguros, sino **mailutils**, por lo que se recomienda su instalación. Realmente, no es algo muy importante en este caso, pero por curiosidad, lo instalé:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo apt install mailutils
{% endhighlight %}

Una vez más, trataremos de ejecutar el script **configure**, haciendo uso del comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/emacs$ ./configure --prefix=/opt/emacs/
checking for xcrun... no
checking for GNU Make... make
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking for gcc... gcc
...
config.status: executing src/epaths.h commands
config.status: executing src/.gdbinit commands
config.status: executing doc/emacs/emacsver.texi commands
config.status: executing etc-refcards-emacsver.tex commands
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/Yr1YfNdm) se puede ver al completo. En este caso, la ejecución del mismo ha finalizado correctamente, sin ningún tipo de error ni advertencia, así que volveremos a listar el contenido del directorio actual, haciendo uso de `ls`:

{% highlight shell %}
alvaro@debian:~/emacs$ ls
aclocal.m4  BUGS       ChangeLog.1  config.bat     configure     COPYING  GNUmakefile  INSTALL.REPO  lib-src  m4         Makefile.in  msdos     oldXMenu   src
admin       build-aux  ChangeLog.2  config.log     configure.ac  doc      info         leim          lisp     make-dist  MANIFEST     nextstep  README     test
autogen.sh  ChangeLog  ChangeLog.3  config.status  CONTRIBUTE    etc      INSTALL      lib           lwlib    Makefile   modules      nt        site-lisp
{% endhighlight %}

Si nos fijamos con detenimiento, se han generado dos nuevos ficheros referentes al script **configure** (**config.log** y **config.status**), referentes a los logs del script y el estado en el que se encuentra actualmente, pero dichos ficheros no son los relevantes, sino el fichero **Makefile** que se ha generado. Podremos mirar el contenido del mismo con `less`:

{% highlight shell %}
alvaro@debian:~/emacs$ less Makefile
{% endhighlight %}

Es muy posible que no se entienda nada, pues es un fichero bastante complejo, ya que si lo pensamos, contiene todas las instrucciones necesarias para compilar y _linkar_ un programa tan grande como _emacs_.

Una vez que ya hemos resuelto todas las dependencias, ya lo tendremos todo listo para pasar a la compilación y al enlazado, haciendo uso del **Makefile** que acabamos de generar. Tal y como hemos explicado anteriormente, dicho proceso lo llevaremos a cabo con **make**, que generará el ejecutable en el directorio **src/**:

{% highlight shell %}
alvaro@debian:~/emacs$ make
make -C lib all
make[1]: Entering directory '/home/alvaro/emacs/lib'
  GEN      alloca.h
  GEN      dirent.h
  GEN      fcntl.h
...
make[2]: Entering directory '/home/alvaro/emacs/doc/misc'
make[2]: Nothing to be done for 'info'.
make[2]: Leaving directory '/home/alvaro/emacs/doc/misc'
make[1]: Nothing to be done for 'info-dir'.
make[1]: Leaving directory '/home/alvaro/emacs'
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/YFFR58sE) se puede ver al completo. En este caso, la ejecución del mismo ha finalizado correctamente, sin ningún tipo de error ni advertencia, así que podemos asegurar que la compilación y el proceso de enlazado ha terminado, por lo que listaremos el contenido de **src/**, pues los ficheros resultantes se habrán alojado ahí:

{% highlight shell %}
alvaro@debian:~/emacs$ ls src/
alloc.c               ccl.c         coding.o       dynlib.c           font.c         insdel.o     menu.h            puresize.h      temacs         vm-limit.c      xdisp.c
alloc.o               ccl.h         commands.h     dynlib.h           font.h         intervals.c  menu.o            ralloc.c        term.c         w16select.c     xdisp.o
atimer.c              ccl.o         composite.c    dynlib.o           font.o         intervals.h  minibuf.c         README          termcap.c      w32.c           xfaces.c
atimer.h              ChangeLog.1   composite.h    editfns.c          fontset.c      intervals.o  minibuf.o         regex-emacs.c   termchar.h     w32common.h     xfaces.o
atimer.o              ChangeLog.10  composite.o    editfns.o          fontset.h      json.c       mini-gmp.c        regex-emacs.h   termhooks.h    w32console.c    xfns.c
bidi.c                ChangeLog.11  config.h       emacs              frame.c        keyboard.c   mini-gmp-emacs.c  regex-emacs.o   terminal.c     w32cygwinx.c    xfont.c
bidi.o                ChangeLog.12  config.in      emacs-27.1.1       frame.h        keyboard.h   mini-gmp.h        region-cache.c  terminal.o     w32fns.c        xftfont.c
bignum.c              ChangeLog.13  conf_post.h    emacs-27.1.1.pdmp  frame.o        keyboard.o   module-env-25.h   region-cache.h  terminfo.c     w32font.c       xgselect.c
bignum.h              ChangeLog.2   COPYING        emacs.c            fringe.c       keymap.c     module-env-26.h   region-cache.o  terminfo.o     w32font.h       xgselect.h
bignum.o              ChangeLog.3   cygw32.c       emacsgtkfixed.c    ftcrfont.c     keymap.h     module-env-27.h   scroll.c        term.o         w32gui.h        xmenu.c
bitmaps               ChangeLog.4   cygw32.h       emacsgtkfixed.h    ftfont.c       keymap.o     msdos.c           scroll.o        termopts.h     w32.h           xml.c
blockinput.h          ChangeLog.5   data.c         emacs-icon.h       ftfont.h       kqueue.c     msdos.h           search.c        textprop.c     w32heap.c       xml.o
bootstrap-emacs       ChangeLog.6   data.o         emacs-module.c     ftxfont.c      lastfile.c   nsfns.m           search.o        textprop.o     w32heap.h       xrdb.c
bootstrap-emacs.pdmp  ChangeLog.7   dbusbind.c     emacs-module.h     getpagesize.h  lastfile.o   nsfont.m          sheap.c         thread.c       w32inevt.c      xselect.c
buffer.c              ChangeLog.8   decompress.c   emacs-module.h.in  gfilenotify.c  lcms.c       nsgui.h           sheap.h         thread.h       w32inevt.h      xsettings.c
buffer.h              ChangeLog.9   decompress.o   emacs-module.o     globals.h      lcms.o       nsimage.m         sound.c         thread.o       w32menu.c       xsettings.h
buffer.o              character.c   deps           emacs.o            gl-stamp       lisp.h       nsmenu.m          sound.o         timefns.c      w32notify.c     xsmfns.c
buildobj.h            character.h   deps.mk        emacs.pdmp         gmalloc.c      lisp.mk      nsselect.m        syntax.c        timefns.o      w32proc.c       xterm.c
bytecode.c            character.o   dired.c        epaths.h           gnutls.c       lread.c      nsterm.h          syntax.h        tparam.c       w32reg.c        xterm.h
bytecode.o            charset.c     dired.o        epaths.in          gnutls.h       lread.o      nsterm.m          syntax.o        tparam.h       w32select.c     xwidget.c
callint.c             charset.h     dispextern.h   eval.c             gnutls.o       macfont.h    pdumper.c         sysdep.c        undo.c         w32select.h     xwidget.h
callint.o             charset.o     dispnew.c      eval.o             gtkutil.c      macfont.m    pdumper.h         sysdep.o        undo.o         w32term.c
callproc.c            chartab.c     dispnew.o      fileio.c           gtkutil.h      macros.c     pdumper.o         sysselect.h     unexaix.c      w32term.h
callproc.o            chartab.o     disptab.h      fileio.o           hbfont.c       macros.h     print.c           syssignal.h     unexcoff.c     w32uniscribe.c
casefiddle.c          cm.c          dmpstruct.awk  filelock.c         image.c        macros.o     print.o           sysstdio.h      unexcw.c       w32xfns.c
casefiddle.o          cmds.c        doc.c          filelock.o         indent.c       macuvs.h     process.c         systhread.c     unexec.h       widget.c
casetab.c             cmds.o        doc.o          firstfile.c        indent.h       Makefile     process.h         systhread.h     unexelf.c      widget.h
casetab.o             cm.h          doprnt.c       floatfns.c         indent.o       Makefile.in  process.o         systhread.o     unexhp9k800.c  widgetprv.h
category.c            cm.o          doprnt.o       floatfns.o         inotify.c      marker.c     profiler.c        systime.h       unexmacosx.c   window.c
category.h            coding.c      dosfns.c       fns.c              inotify.o      marker.o     profiler.o        systty.h        unexsol.c      window.h
category.o            coding.h      dosfns.h       fns.o              insdel.c       menu.c       ptr-bounds.h      syswait.h       unexw32.c      window.o
{% endhighlight %}

Como se puede apreciar en esta gran salida, se han generado una gran cantidad de ficheros objeto (**.o**) como resultado de la compilación de los ficheros fuente. Además, si nos fijamos entre todos los ficheros existentes, hay uno de nombre **emacs** que es el binario resultante de la compilación y el enlazado, por lo que si tratamos de ejecutarlo, debería de abrir _emacs_:

{% highlight shell %}
alvaro@debian:~/emacs$ src/emacs
{% endhighlight %}

![emacs1](https://i.ibb.co/kBT5pbH/1.jpg "Ejecución binario")

Efectivamente, el binario se ha ejecutado y funciona correctamente, sin embargo, vamos a tratar de ejecutar el comando sin especificar la ruta del mismo:

{% highlight shell %}
alvaro@debian:~/emacs$ emacs
-bash: emacs: command not found
{% endhighlight %}

Como se puede apreciar, nos ha devuelto un error ya que no ha sido capaz de encontrar la ruta en la que se encuentra el binario. Esto tiene una razón, y es que en Linux existe una variable de entorno de nombre **PATH** ($PATH) que indica las rutas en las que buscar los binarios (y en qué orden). Vamos a mostrar el valor de dicha variable haciendo uso de `echo`:

{% highlight shell %}
alvaro@debian:~$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
{% endhighlight %}

Como se puede apreciar, la primera ruta que mirará es **/usr/local/bin**, ya que como hemos mencionado, es donde se encuentra el software instalado localmente, de manera que si te has tomado la molestia de compilarlo a mano, es porque quieres usarlo con una mayor preferencia. Sin embargo, en caso de no encontrar el binario ahí, mirará en **/usr/bin**, que es donde se encuentra el software instalado por los gestores de paquetes... y así sucesivamente.

Antes de seguir y proceder con la instalación, vamos a comprobar una vez más el tamaño total del directorio actual, ejecutando el comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh .
433M    .
{% endhighlight %}

Como se puede apreciar, el tamaño total ha aumentado de **246M** a **433M**, es decir, el proceso de compilación y enlazado ha supuesto un aumento en el tamaño de **187M**.

Sin embargo, vamos a comprobar el contenido del directorio en el que vamos a proceder a realizar la instalación (**/opt/emacs/**), para así verificar que todavía no se ha generado dentro del mismo ningún fichero:

{% highlight shell %}
alvaro@debian:~/emacs$ ls /opt/emacs/
{% endhighlight %}

Efectivamente, el directorio no tiene ningún contenido, por lo que podemos asegurar que el proceso de compilación y enlazado ha generado ficheros únicamente en el directorio **emacs/** existente en el directorio personal.

Una vez que tengamos todos los ficheros fuente compilados y enlazados, ya se habrán generado los correspondientes binarios, que podremos instalar en la ruta que hemos especificado en el parámetro **--prefix** del script **configure**. Para ello, haremos uso del comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo make install
make -C lib all
make[1]: Entering directory '/home/alvaro/emacs/lib'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory '/home/alvaro/emacs/lib'
make -C lib-src all
...
cd "/opt/emacs/bin" && ln -s "`echo emacs-27.1 | sed 's,x,x,'`" "`echo emacs | sed 's,x,x,'`"
make -C lib-src maybe-blessmail
make[1]: Entering directory '/home/alvaro/emacs/lib-src'
make[1]: Nothing to be done for 'maybe-blessmail'.
make[1]: Leaving directory '/home/alvaro/emacs/lib-src'
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/DNiUHpSr) se puede ver al completo. En este caso, la instalación del mismo ha finalizado correctamente, sin ningún tipo de error ni advertencia, así que podemos asegurar que los ficheros necesarios se han ubicado correctamente en el correspondiente directorio, por lo que listaremos el contenido de `/opt/emacs/`, pues los ficheros resultantes se habrán alojado ahí:

{% highlight shell %}
alvaro@debian:~/emacs$ ls /opt/emacs/
bin  include  lib  libexec  share
{% endhighlight %}

Efectivamente, se ha generado la correspondiente estructura de directorios, con los binarios dentro de **bin/**. Antes de listar el contenido de dicho directorio, vamos a volver a comprobar el tamaño del directorio **emacs/**, existente en el directorio personal para así asegurar que su tamaño no ha variado y que por tanto, no se han generado nuevos ficheros dentro del mismo:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh .
433M    .
{% endhighlight %}

Como era de esperar, el tamaño no ha variado. Sin embargo, si hacemos la misma comprobación en el directorio **/opt/emacs/** veremos que el resultado no es el mismo, y que esta vez, el tamaño sí ha variado:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh /opt/emacs/
132M    /opt/emacs/
{% endhighlight %}

Vamos a indagar un poco en los ficheros que ha generado el proceso de instalación, así que vamos a comenzar por listar el contenido del directorio **/opt/emacs/bin/**:

{% highlight shell %}
alvaro@debian:~/emacs$ ls -l /opt/emacs/bin/
total 24688
-rwxr-xr-x 1 root root   803456 Oct 28 10:03 ctags
-rwxr-xr-x 1 root root   210544 Oct 28 10:03 ebrowse
lrwxrwxrwx 1 root root       10 Oct 28 10:03 emacs -> emacs-27.1
-rwxr-xr-x 1 root root 23248240 Oct 28 10:03 emacs-27.1
-rwxr-xr-x 1 root root   213608 Oct 28 10:03 emacsclient
-rwxr-xr-x 1 root root   791000 Oct 28 10:03 etags
{% endhighlight %}

Como se puede apreciar, se han generado varios binarios, que cuentan con permisos de ejecución para todos los usuarios, de manera que cualquier usuario en la máquina podría hacer uso de los mismos. Como curiosidad, cuando ejecutemos **emacs** en ese directorio, realmente lo que estamos ejecutando es un enlace simbólico que apunta al binario real de nombre **emacs-27.1**.

De todas formas, dado que la ruta **/opt/emacs/bin/** no se encuentra contemplada en la variable de entorno **PATH**, aunque ejecutemos **emacs**, no será capaz de encontrar el binario. Es por ello que vamos a crear un enlace simbólico a todos los binarios dentro de **/usr/local/bin**, que sí está contemplada como primera opción en la variable **PATH**, de manera que aunque tuviésemos _emacs_ instalado en **/usr/bin**, el enlace simbólico tendría prioridad. Antes de crear dichos enlaces simbólicos, vamos a listar el contenido de **/usr/local/bin/**, para ver qué hay:

{% highlight shell %}
alvaro@debian:~/emacs$ ls /usr/local/bin/
{% endhighlight %}

Como se puede apreciar, el contenido de dicho directorio es nulo, pues nunca antes se ha instalado un programa compilado a mano en esta máquina. Para llevar a cabo dicha creación, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo ln -s /opt/emacs/bin/* /usr/local/bin/
{% endhighlight %}

Donde:

* **-s**: Indica que se cree un enlace simbólico (ya que por defecto, **ln** crea enlaces duros).

De nuevo, volveremos a listar el contenido de dicho directorio, para verificar la creación de los enlaces simbólicos:

{% highlight shell %}
alvaro@debian:~/emacs$ ls -l /usr/local/bin/
total 0
lrwxrwxrwx 1 root root 20 Oct 28 10:11 ctags -> /opt/emacs/bin/ctags
lrwxrwxrwx 1 root root 22 Oct 28 10:11 ebrowse -> /opt/emacs/bin/ebrowse
lrwxrwxrwx 1 root root 20 Oct 28 10:11 emacs -> /opt/emacs/bin/emacs
lrwxrwxrwx 1 root root 25 Oct 28 10:11 emacs-27.1 -> /opt/emacs/bin/emacs-27.1
lrwxrwxrwx 1 root root 26 Oct 28 10:11 emacsclient -> /opt/emacs/bin/emacsclient
lrwxrwxrwx 1 root root 20 Oct 28 10:11 etags -> /opt/emacs/bin/etags
{% endhighlight %}

Efectivamente, los enlaces simbólicos han sido correctamente generados, de manera que si ahora mismo ejecutásemos **emacs**, sería capaz de encontrar el binario, gracias a dicho enlace:

{% highlight shell %}
alvaro@debian:~/emacs$ emacs
{% endhighlight %}

![emacs2](https://i.ibb.co/kBT5pbH/1.jpg "Ejecución binario")

Como se puede apreciar, ya hemos sido capaces de encontrar el binario y su ejecución se ha llevado a cabo correctamente.

Para verificar que el binario lo está encontrando en la ruta en la que hemos ubicado el enlace simbólico, haremos uso del comando `which`, que nos devolverá la ruta del mismo:

{% highlight shell %}
alvaro@debian:~/emacs$ which emacs
/usr/local/bin/emacs
{% endhighlight %}

Como era de esperar, la ruta del binario es aquella en la que se encuentra ubicado el enlace simbólico que apunta al binario real.

Sin embargo, todo es muy bonito, pero no nos hemos parado a pensar en el hipotético caso de que necesitásemos leer la documentación de ayuda de _emacs_, haciendo uso del comando `man`. Vamos a intentarlo:

{% highlight shell %}
alvaro@debian:~/emacs$ man emacs
No manual entry for emacs
See 'man 7 undocumented' for help when manual pages are not available.
{% endhighlight %}

Como se puede apreciar, nos ha informado de que no existe ninguna entrada en el manual para _emacs_. ¿A qué se debe esto? Bien, tiene una sencilla explicación, y es que hemos creado enlaces simbólicos para los binarios pero no para las páginas del manual (que se encuentran en **/opt/emacs/share/man/man1/**). Tras leer un poco de documentación (principalmente del fichero **/etc/manpath.config**), he descubierto que las páginas del manual para aquel software instalado localmente se ubican en **/usr/local/man/**, dentro de diferentes subdirectorios, que van desde **man1/** hasta **man8/**, teniendo diferentes finalidades cada uno de ellos:

{% highlight shell %}
man1/ - Comandos del usuario
man2/ - Llamadas del sistema
man3/ - Funciones de librerías de C
man4/ - Dispositivos y ficheros especiales
man5/ - Formatos de ficheros y convenciones
man6/ - Juegos
man7/ - Miscelánea
man8/ - Herramientas de administración del sistema y demonios
{% endhighlight %}

Dado que lo que nosotros hemos instalado es un comando para el usuario, tendremos que incluir dichas páginas del manual en **man1/**, así que lo primero que tendremos que hacer será crear dicho subdirectorio en caso de que no se encuentre ya creado (como es mi caso), ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo mkdir /usr/local/man/man1
{% endhighlight %}

Una vez que el subdirectorio esté generado, ya podremos realizar la creación de dichos enlaces simbólicos que apunten a los correspondientes ficheros dentro de **/opt/emacs/share/man/man1/**, haciendo uso del comando visto con anterioridad:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo ln -s /opt/emacs/share/man/man1/* /usr/local/man/man1/
{% endhighlight %}

Lo siguiente que haremos será listar el contenido de dicho directorio, para verificar la creación de los enlaces simbólicos:

{% highlight shell %}
alvaro@debian:~/emacs$ ls -l /usr/local/man/man1/
total 0
lrwxrwxrwx 1 root root 36 Nov  1 13:05 ctags.1.gz -> /opt/emacs/share/man/man1/ctags.1.gz
lrwxrwxrwx 1 root root 38 Nov  1 13:05 ebrowse.1.gz -> /opt/emacs/share/man/man1/ebrowse.1.gz
lrwxrwxrwx 1 root root 36 Nov  1 13:05 emacs.1.gz -> /opt/emacs/share/man/man1/emacs.1.gz
lrwxrwxrwx 1 root root 42 Nov  1 13:05 emacsclient.1.gz -> /opt/emacs/share/man/man1/emacsclient.1.gz
lrwxrwxrwx 1 root root 36 Nov  1 13:05 etags.1.gz -> /opt/emacs/share/man/man1/etags.1.gz
{% endhighlight %}

Efectivamente, los enlaces simbólicos han sido correctamente generados, de manera que si ahora mismo ejecutásemos **man emacs**, sería capaz de encontrar la página del manual, gracias a dicho enlace:

{% highlight shell %}
alvaro@debian:~/emacs$ man emacs
{% endhighlight %}

![emacs3](https://i.ibb.co/FJB3rrL/Captura-de-pantalla-de-2020-11-01-18-15-28.png "Página del manual")

Como se puede apreciar, ya hemos sido capaces de encontrar la correspondiente página del manual y su apertura se ha llevado a cabo correctamente.

Para verificar que la página del manual la está encontrando en la ruta que realmente debería, haremos uso del comando `man -w`, que nos devolverá la ruta del mismo:

{% highlight shell %}
alvaro@debian:~/emacs$ man -w emacs
/opt/emacs/share/man/man1/emacs.1.gz
{% endhighlight %}

En este caso, la ruta que nos ha devuelto es la ruta real, como consecuencia de haber seguido el enlace simbólico que habíamos generado anteriormente.

Tras finalizar con la configuración de los enlaces simbólicos, listaremos de forma recursiva y un poco más visual el contenido del directorio **/opt/emacs/lib/** haciendo uso del comando `tree`, para ver las librerías existentes:

{% highlight shell %}
alvaro@debian:~/emacs$ tree /opt/emacs/lib/
/opt/emacs/lib/
└── systemd
    └── user
        └── emacs.service

2 directories, 1 file
{% endhighlight %}

Si nos fijamos en la salida del comando, dentro de dicho directorio existe otro directorio de nombre **systemd/**, que contiene a su vez otro directorio **user/**, dentro del cuál existe un fichero de nombre **emacs.service**. Lo primero que podemos deducir de todo esto es que no hay librerías que únicamente sean utilizadas por _emacs_, o bien, han sido enlazadas estáticamente en el propio binario. La explicación de la existencia de dicho fichero anteriormente mencionado, se debe a que **systemd** nos permite tener servicios específicos para cada usuario, además de todos los existentes en el sistema, que pueden ser administrados como haríamos normalmente con `systemctl`. El gestor de servicios del sistema no conoce la existencia de dicho servicio, por lo que para que sea visible, tendremos que usar el parámetro **--user** durante la ejecución del comando **systemctl**.

Tras una pequeña investigación, descubrí que existía otro directorio, cuanto menos curioso. Se trata de **libexec/**, cuyo contenido es el siguiente:

{% highlight shell %}
alvaro@debian:~/emacs$ tree /opt/emacs/libexec/
/opt/emacs/libexec/
└── emacs
    └── 27.1
        └── x86_64-pc-linux-gnu
            ├── emacs.pdmp
            ├── hexl
            └── rcs2log

3 directories, 3 files
{% endhighlight %}

Como se puede apreciar, contiene dos binarios, **hexl** y **rcs2log**. Esto se debe a que dicho directorio está pensado para albergar aquellos binarios que no están destinados a ser ejecutados directamente por los usuarios o scripts de shell, pero sí por otros binarios. Para hacer un ejemplo, y verificar que son capaces de hacer uso de dichos binarios, vamos a hacer una prueba del modo **Hexl** de _emacs_, que suele ser usado para editar ficheros en binario, de manera que leerá un fichero y nos lo mostrará en hexadecimal, permitiendo editar la traducción para posteriormente traducirlo de nuevo a binario y guardarlo.

Lo primero que haremos será generar un fichero de prueba, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/emacs$ echo "Hola" > ../fichero.txt
{% endhighlight %}

Una vez generado, vamos a probarlo. Para hacer uso de este modo, tendremos que acceder a _emacs_:

{% highlight shell %}
alvaro@debian:~/emacs$ emacs
{% endhighlight %}

El modo Hexl es accesible ejecutando la combinación de teclas **ALT + x** y escribiendo posteriormente **hexl-find-file**. Tras ello, pulsaremos **INTRO** y tendremos que ver algo similar a esto en la parte inferior:

![emacs4](https://i.ibb.co/K77g4S6/Captura.jpg "Hexl mode")

Ahí tendremos que escribir la ruta al fichero en cuestión, que en este caso, se encuentra en el directorio padre (**../**), con el nombre de **fichero.txt**:

![emacs5](https://i.ibb.co/Pwp9dcB/Captura2.jpg "Hexl mode")

Como se puede apreciar, la traducción se ha llevado a cabo correctamente haciendo uso del binario **hexl** situado en **/opt/emacs/libexec/**. Por cierto, para salir de _emacs_ hay que ejecutar la combinación **CTRL + x** seguido de **CTRL + c**, pues me llevé más de 10 minutos tratando de averiguar cómo salir, debido a mis nulos conocimientos del programa (por ahora).

Mi curiosidad todavía seguía latente, así que seguí investigando y descubrí la existencia del comando `ldd`, que muestra las librerías compartidas (**.so**, es decir, _shared object_) que son requeridas por un programa. En otras palabras, muestras las dependencias enlazadas dinámicamente de un binario, así que decidí probarlo con _emacs_:

{% highlight shell %}
alvaro@debian:~/emacs$ ldd /opt/emacs/bin/emacs
        linux-vdso.so.1 (0x00007ffeeadd3000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f32c62b8000)
        libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007f32c6288000)
        libgnutls.so.30 => /lib/x86_64-linux-gnu/libgnutls.so.30 (0x00007f32c60d8000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f32c60b0000)
        libanl.so.1 => /lib/x86_64-linux-gnu/libanl.so.1 (0x00007f32c60a8000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f32c5f20000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f32c5f18000)
        libgmp.so.10 => /lib/x86_64-linux-gnu/libgmp.so.10 (0x00007f32c5e90000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f32c5cc8000)
        libp11-kit.so.0 => /lib/x86_64-linux-gnu/libp11-kit.so.0 (0x00007f32c5b98000)
        libidn2.so.0 => /lib/x86_64-linux-gnu/libidn2.so.0 (0x00007f32c5b78000)
        libunistring.so.2 => /lib/x86_64-linux-gnu/libunistring.so.2 (0x00007f32c59f0000)
        libtasn1.so.6 => /lib/x86_64-linux-gnu/libtasn1.so.6 (0x00007f32c57d8000)
        libnettle.so.6 => /lib/x86_64-linux-gnu/libnettle.so.6 (0x00007f32c57a0000)
        libhogweed.so.4 => /lib/x86_64-linux-gnu/libhogweed.so.4 (0x00007f32c5760000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f32c6880000)
        libffi.so.6 => /lib/x86_64-linux-gnu/libffi.so.6 (0x00007f32c5750000)
{% endhighlight %}

Como se puede apreciar, existen unas cuantas librerías como dependencias enlazadas dinámicamente al binario, de manera que será un requisito que estén instaladas en el sistema a la hora de la ejecución del mismo. En este caso, dichas librerías se encuentran en **/lib**, directorio que contiene las librerías "esenciales" que podrían ser necesitadas incluso antes de que el directorio **/usr** se encuentre montado.

Por último, vamos a llevar a cabo la desinstalación de _emacs_. Lo óptimo sería hacerlo en el orden inverso al que hemos hecho la instalación, de manera que lo último que hicimos fue generar los correspondientes enlaces simbólicos en **/usr/local/man/man1/**, así que empezaremos por borrarlos. Para ello, nos moveremos a dicho directorio:

{% highlight shell %}
alvaro@debian:~/emacs$ cd /usr/local/man/man1/
{% endhighlight %}

Una vez dentro del mismo, listaremos el contenido para comprobar que se encuentran creados los enlaces simbólicos:

{% highlight shell %}
alvaro@debian:/usr/local/man/man1$ ls -l
total 0
lrwxrwxrwx 1 root root 36 Nov  1 13:05 ctags.1.gz -> /opt/emacs/share/man/man1/ctags.1.gz
lrwxrwxrwx 1 root root 38 Nov  1 13:05 ebrowse.1.gz -> /opt/emacs/share/man/man1/ebrowse.1.gz
lrwxrwxrwx 1 root root 36 Nov  1 13:05 emacs.1.gz -> /opt/emacs/share/man/man1/emacs.1.gz
lrwxrwxrwx 1 root root 42 Nov  1 13:05 emacsclient.1.gz -> /opt/emacs/share/man/man1/emacsclient.1.gz
lrwxrwxrwx 1 root root 36 Nov  1 13:05 etags.1.gz -> /opt/emacs/share/man/man1/etags.1.gz
{% endhighlight %}

Efectivamente, se encuentran creados, así que vamos a hacer uso de `xargs` para su correspondiente eliminación, que ejecuta una determinada acción para aquello que le llegue por **stdin**, de manera que podemos listar el contenido de **/opt/emacs/share/man/man1/** y eliminar todas las coincidencias una por una (en realidad, con hacer un __rm *__ bastaría, pero lo hago suponiendo que hubiesen más ficheros en dicho directorio):

{% highlight shell %}
alvaro@debian:/usr/local/man/man1$ ls /opt/emacs/share/man/man1/ | sudo xargs rm
{% endhighlight %}

Tras ello, vamos a listar el contenido de nuevo para verificar que se han eliminado:

{% highlight shell %}
alvaro@debian:/usr/local/man/man1$ ls -l
total 0
{% endhighlight %}

Todavía nos queda repetir el mismo proceso para el directorio donde se encuentran los enlaces simbólicos de los binarios, es decir, **/usr/local/bin/**, así que nos moveremos dentro del mismo, ejecutando para ello la instrucción:

{% highlight shell %}
alvaro@debian:/usr/local/man/man1$ cd /usr/local/bin/
{% endhighlight %}

Una vez dentro del mismo, listaremos el contenido para comprobar que se encuentran creados los enlaces simbólicos:

{% highlight shell %}
alvaro@debian:/usr/local/bin$ ls -l
total 0
lrwxrwxrwx 1 root root 20 Oct 28 10:11 ctags -> /opt/emacs/bin/ctags
lrwxrwxrwx 1 root root 22 Oct 28 10:11 ebrowse -> /opt/emacs/bin/ebrowse
lrwxrwxrwx 1 root root 20 Oct 28 10:11 emacs -> /opt/emacs/bin/emacs
lrwxrwxrwx 1 root root 25 Oct 28 10:11 emacs-27.1 -> /opt/emacs/bin/emacs-27.1
lrwxrwxrwx 1 root root 26 Oct 28 10:11 emacsclient -> /opt/emacs/bin/emacsclient
lrwxrwxrwx 1 root root 20 Oct 28 10:11 etags -> /opt/emacs/bin/etags
{% endhighlight %}

Efectivamente, se encuentran creados, así que vamos a volver a hacer uso de `xargs` para su correspondiente eliminación, de manera que podemos listar el contenido de **/opt/emacs/bin/** y eliminar todas las coincidencias una por una (en realidad, con hacer un __rm *__ bastaría, pero lo hago suponiendo que hubiesen más ficheros en dicho directorio):

{% highlight shell %}
alvaro@debian:/usr/local/bin$ ls /opt/emacs/bin/ | sudo xargs rm
{% endhighlight %}

Tras ello, vamos a listar el contenido de nuevo para verificar que se han eliminado:

{% highlight shell %}
alvaro@debian:/usr/local/bin$ ls -l
total 0
{% endhighlight %}

Efectivamente, todos los enlaces simbólicos se encuentran ya eliminados, así que podremos volver al directorio **emacs/** de nuestro directorio personal para proseguir con la desinstalación:

{% highlight shell %}
alvaro@debian:/usr/local/bin$ cd ~/emacs
{% endhighlight %}

Ha llegado la hora de llevar a cabo la desinstalación, así que vamos a hacer uso de **make uninstall**, que en teoría debe eliminar todo el contenido sin dejar rastro alguno:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo make uninstall
make -C doc/emacs uninstall-dvi
make[1]: Entering directory '/home/alvaro/emacs/doc/emacs'
for file in emacs.dvi emacs-xtra.dvi; do \
  rm -f "/opt/emacs/share/doc/emacs/${file}"; \
done
...
     "hicolor/scalable/mimetypes/`echo emacs | sed 's,x,x,'`-document23.svg"; \
fi)
rm -f "/opt/emacs/share/applications/`echo emacs | sed 's,x,x,'`.desktop"
rm -f "/opt/emacs/share/metainfo/`echo emacs | sed 's,x,x,'`.appdata.xml"
rm -f "/opt/emacs/lib/systemd/user/`echo emacs | sed 's,x,x,'`.service"
{% endhighlight %}

De nuevo, la salida del comando es muy grande, así que la he recortado para no ensuciar, pero [aquí](https://pastebin.com/0T2hHVAW) se puede ver al completo. En este caso, la desinstalación del mismo ha finalizado correctamente, sin ningún tipo de error ni advertencia, por lo que vamos a ver el tamaño del contenido de **/opt/emacs/**, para verificar que se ha visto reducido como consecuencia de la desinstalación:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh /opt/emacs/
136K    /opt/emacs/
{% endhighlight %}

Efectivamente, el contenido del mismo se ha visto reducido considerablemente, pero todavía pesa **136K**, indicativo de que quedan algunos ficheros en su interior. Vamos a listar el contenido de dicho directorio de manera recursiva y gráfica con `tree`:

{% highlight shell %}
alvaro@debian:~/emacs$ tree /opt/emacs/
/opt/emacs/
├── bin
├── include
├── lib
│   └── systemd
│       └── user
├── libexec
│   └── emacs
└── share
    ├── applications
    ├── emacs
    │   └── site-lisp
    │       └── subdirs.el
    ├── icons
    │   └── hicolor
    │       ├── 128x128
    │       │   └── apps
    │       ├── 16x16
    │       │   └── apps
    │       ├── 24x24
    │       │   └── apps
    │       ├── 32x32
    │       │   └── apps
    │       ├── 48x48
    │       │   └── apps
    │       └── scalable
    │           ├── apps
    │           └── mimetypes
    ├── info
    │   └── dir
    ├── man
    │   └── man1
    └── metainfo
{% endhighlight %}

Como me temía, ha dejado algunos archivos residuales (pocos), así que gracias a la elección que hicimos de llevar a cabo la instalación en un directorio independiente, ahora podremos eliminar dicho directorio sin riesgo alguno, asegurándonos de que se ha llevado a cabo una desinstalación completamente limpia:

{% highlight shell %}
alvaro@debian:~/emacs$ sudo rm -r /opt/emacs/
{% endhighlight %}

Listo. El directorio ha sido eliminado, pero para asegurarnos, vamos a listar el contenido de **/opt/**:

{% highlight shell %}
alvaro@debian:~/emacs$ ls /opt/
{% endhighlight %}

Como era de esperar, el directorio **/opt/** se encuentra ahora vacío. Por último, vamos a comprobar el tamaño del directorio actual, para comprobar que la desinstalación no ha afectado al mismo:

{% highlight shell %}
alvaro@debian:~/emacs$ du -sh .
433M    .
{% endhighlight %}

En efecto, sigue pesando exactamente lo mismo. Llegados a este punto podríamos eliminar también el directorio que contiene los ficheros de la compilación, o bien ejecutar un `make clean`, el cuál eliminará todos los ficheros resultantes del proceso de compilación y enlazado, permitiendo hacer una nueva compilación, e incluso si queremos ir un paso más allá, podemos ejecutar un `make distclean`, que hace lo mismo que el comando anterior pero además elimina la configuración. Existen algunas otras posibilidades, pero esas son las más comunes.