---
layout: post
title:  "Integridad, firmas y autenticación"
banner: "/assets/images/banners/firmas.jpg"
date:   2020-11-09 21:07:00 +0200
categories: seguridad
---
Se recomienda realizar una previa lectura del _post_ [Cifrado asimétrico con gpg y openssl](http://www.alvarovf.com/seguridad/2020/10/11/cifrado-asimetrico-gpg-openssl.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Tarea 1: Firmas electrónicas

### Manda un documento y la firma electrónica del mismo a un compañero. Verifica la firma que tú has recibido.

Antes de comenzar, voy a listar las claves existentes en mi _keyring_ para así ver el punto desde el que partimos. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]
{% endhighlight %}

Como se puede apreciar, la única clave existente en mi anillo de claves es la mía personal, así que procederemos a subir dicha clave a un servidor de claves, como por ejemplo, **keys.gnupg.net** (haciendo uso del _fingerprint_ para así poder identificarla). Para ello, haremos uso del comando:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver keys.gnupg.net --send-key B02B578465B0756DFD271C733E0DA17912B9A4F8
gpg: enviando clave 3E0DA17912B9A4F8 a hkp://hkps.pool.sks-keyservers.net
{% endhighlight %}

La clave ya ha sido subida al servidor de claves, de manera que la persona que reciba mi fichero firmado podrá importarla para verificar mi firma.

Podríamos firmar un fichero e incluir la firma en el mismo, de manera que estaría unificado (opción `--sign`), o bien podríamos tener el fichero por un lado, y la firma por otro, tal y como vamos a hacer en esta ocasión (opción `--detach-sign`).

En este caso, vamos a firmar un fichero de nombre **hlc.pdf** y a separar la firma en un fichero aparte, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --detach-sign hlc.pdf
{% endhighlight %}

Para llevar a cabo el proceso de firmado, nos pedirá la frase de paso de nuestra clave privada, así que la introduciremos. Tras ello, listaremos el contenido del directorio actual, estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'hlc'
-rw-r--r-- 1 alvaro alvaro   3497882 nov  8 18:05 hlc.pdf
-rw-r--r-- 1 alvaro alvaro       438 nov  8 18:06 hlc.pdf.sig
{% endhighlight %}

Como se puede apreciar, el fichero original (**hlc.pdf**) y la firma (**hlc.pdf.sig**) se encuentran en ficheros separados, de manera que podríamos enviarlos a un compañero para que así verificase la integridad del fichero, haciendo uso de nuestra clave pública.

Como curiosidad, lo que se firma realmente es el **_hash_** del fichero, es decir, un "resumen" que se ha generado del mismo mediante la aplicación de un algoritmo, ya que firmar todo el fichero sería muy costoso computacionalmente, de manera que posteriormente, el receptor lo que hace es calcular el _hash_ del fichero recibido y verificar que coincida con el que nosotros hemos firmado, pudiendo asegurar así, que no ha sido manipulado.

Nuestro compañero ha realizado de forma paralela el mismo procedimiento que nosotros, por lo tanto, antes de tratar de verificar la firma de dicho fichero, tendré que importar la clave pública de la persona que lo ha firmado. Para ello, ejecutaré el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver pgp.rediris.es --recv-keys 08950F41
gpg: clave 7A01A1F808950F41: clave pública "celia garcia <cg.marquez95@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
{% endhighlight %}

Para verificar que la importación se ha llevado a cabo satisfactoriamente, volveremos a listar las claves públicas actualmente existentes en nuestro _keyring_:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]

pub   rsa3072 2020-11-04 [SC] [caduca: 2022-11-04]
      3A9AF138559F9BC6A146EEBE7A01A1F808950F41
uid        [desconocida] celia garcia <cg.marquez95@gmail.com>
uid        [desconocida] celia garcia
sub   rsa3072 2020-11-04 [E] [caduca: 2022-11-04]
{% endhighlight %}

Efectivamente, la nueva clave pública ha sido importada. En este caso, **Celia** me ha enviado dos ficheros, un fichero **empresa.pdf** que es el fichero original, y un fichero **empresa.pdf.sig**, que contiene la firma del mismo. Para verificar la firma, vamos a hacer uso de la opción `--verify`, pasando como parámetros ambos ficheros:

{% highlight shell %}
alvaro@debian:~$ gpg --verify empresa.pdf.sig empresa.pdf
gpg: Firmado el dom 08 nov 2020 16:33:22 CET
gpg:                usando RSA clave 3A9AF138559F9BC6A146EEBE7A01A1F808950F41
gpg: Firma correcta de "celia garcia <cg.marquez95@gmail.com>" [desconocido]
gpg:                 alias "celia garcia" [desconocido]
gpg: ATENCIÓN: ¡Esta clave no está certificada por una firma de confianza!
gpg:          No hay indicios de que la firma pertenezca al propietario.
Huellas dactilares de la clave primaria: 3A9A F138 559F 9BC6 A146  EEBE 7A01 A1F8 0895 0F41
{% endhighlight %}

Efectivamente, la firma es correcta, tal y como nos ha informado la salida del comando. Si nos fijamos, se ha devuelto además un mensaje informando que la clave no está certificada por una firma de confianza, de manera que no hay indicios de que la firma pertenezca al propietario. Esto se debe a que no tenemos validez en la clave pública que acabamos de importar, o dicho de otra manera, todavía no la hemos firmado ni tampoco tenemos a personas intermediarias con las que validar la clave de forma indirecta. Lo explicaremos con más detalle a continuación.

Antes de proseguir, he decidido eliminar la clave pública de mi anillo de claves, para así empezar desde un punto totalmente limpio, sin ninguna clave en mi anillo, haciendo para ello uso de `--delete-keys`:

{% highlight shell %}
alvaro@debian:~$ gpg --delete-keys 3A9AF138559F9BC6A146EEBE7A01A1F808950F41
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


pub  rsa3072/7A01A1F808950F41 2020-11-04 celia garcia <cg.marquez95@gmail.com>

¿Eliminar esta clave del anillo? (s/N) s
{% endhighlight %}

La clave pública de **Celia** ya ha sido eliminado de mi _keyring_.

### Crea un anillo de confianza entre los miembros de la clase.

En el uso cotidiano de los pares de claves _gpg_ surge un gran problema, la confianza. Necesitamos un mecanismo que asegure que las claves públicas que tenemos realmente pertenecen a quien dicen pertenecer, asegurándonos así que cuando encriptamos un fichero haciendo uso de una clave pública, el receptor va a ser quien debe ser, y no un impostor, o bien, cuando vamos a comprobar la firma de un fichero, realmente podamos corroborar que la persona que lo ha firmado es quien dice ser.

Una de las posibles maneras es construir un anillo de confianza, es decir, entre un grupo no muy elevado de personas y "conocidos" entre ellos, de manera que para expresar que realmente confiamos en la clave pública de alguien, lo que hacemos es firmar la huella digital (_fingerprint_) de dicha clave con nuestra clave privada. Dicha clave firmada se la devolveremos al remitente, de manera que podrá importar la nueva firma, para posteriormente volver a distribuirla con dicha firma añadida, aumentando así su autenticidad y confianza.

Gracias a este mecanismo, si una persona que nos ha enviado su clave (en la que nosotros confiamos) conoce a su vez a una tercera persona cuya clave ha validado (es decir, a la que ha firmado), y ésta persona nos envía su clave pública, es posible que podamos usarla con validez (dependiendo de cómo lo hayamos configurado), ya que tenemos una persona en común, de manera que la responsabilidad de la validación de las claves públicas recae en las personas en las que confiamos. Quizás, es un poco lioso, así que vamos a usar un ejemplo para que se vea más claro. Supongamos que:

* Adrián ha firmado la clave de Rocío.
* Rocío ha firmado las claves de Juan y de Alejandro.

Si Adrián **confía** lo suficiente en Rocío como para **validar** las claves que ella firma, entonces Adrián puede deducir que las claves de Juan y de Alejandro son válidas sin llegar a comprobarlas personalmente. Adrián se limitará a usar su copia validada de la clave pública de Rocío para comprobar que las firmas de Rocío sobre Juan y Alejandro son auténticas, que pasarán a ser válidas en caso de que el nivel de confianza sea el correcto.

Como se puede apreciar, estamos trabajando con dos conceptos diferentes, la **validez** (también conocida como _validity_) y la **confianza** (también conocida como _ownertrust_), conceptos muy abstractos que pueden llegar a ser complicados de entender. Digamos que la **validez** es algo que recáe sobre la propia clave (por ello cuando firmamos una clave, automáticamente pasa a tener una validez total para nosotros, de manera que podremos encriptar o comprobar firmas con dicha clave pública sin que nos muestre la advertencia de seguridad), y la **confianza**, algo que recáe sobre la persona a la que está asociada la clave, que dependiendo del nivel que le asociemos, permitirá o no validar las claves públicas de terceras personas. Existen varios niveles de confianza:

* **Absoluta**: Se debe utilizar únicamente para nuestras claves personales (a la hora de la creación, _gpg_ asigna dicha confianza automáticamente). Ésta es la razón por la que cuando firmamos la clave pública de un amigo, automáticamente se le asigna una validez total, a pesar de que cuando la importamos tenía una validez desconocida.
* **Total**: Se debe utilizar para las personas en las que confiamos lo suficiente como para que validemos las claves que dichas personas firman de forma automática. Es decir, cualquier clave firmada por una persona a la que hemos asignado una confianza total, será marcada como válida.
* **Un poco**: Confianza marginal. Es decir, cualquier clave firmada por al menos 3 personas a las que hemos asignado una confianza marginal, será marcada como válida. El número de personas necesarias se puede modificar.
* **No confío**: Es igual al nivel de confianza que se explicará a continuación, pero indicándolo de forma explícita. En mi opinión, es un poco inútil, ya que si no confías en la persona, lo más lógico es borrar dicha clave. Como es lógico, no tendrá ningún impacto para nosotros el hecho de que una persona en la que no confiamos firme la clave de una tercera persona.
* **Desconocida**: Es el nivel de confianza por defecto, ya que todavía no hemos asignado ninguno manualmente. Como es lógico, no tendrá ningún impacto para nosotros el hecho de que una persona con confianza desconocida firme la clave de una tercera persona.

Para modificar la confianza de las claves existentes en nuestro _keyring_ haremos uso de la opción `--edit-key` junto a la orden `trust`.

Tras esta explicación, vamos a pasar a resumir el desarrollo que se va a llevar a cabo para ésta tarea:

* Partimos del punto en el que ya tengo un par de claves generado y exportado a un servidor de claves para que 3 de mis compañeros puedan descargarla y firmarla (nosotros también descargaremos y firmaremos las de ellos, firmando todos las claves de todos).
* Tras ello, nos devolveremos las respectivas firmas para importarlas y posteriormente volver a exportar la clave con las firmas añadidas.
* Una vez exportada la clave con las firmas, volveremos a distribuirlas para que así todos tengamos conocimiento sobre las firmas que tiene cada uno.
* Por último, añadiremos a una tercera persona al anillo de confianza en la que nosotros no confiaremos, para así verificar que mediante un intermediario en el que tenemos confianza total, podemos validar la clave de esa tercera persona.

Actualmente, la única clave existente en mi anillo de claves es la mía personal, así que procederemos a importar las de mis compañeros a nuestro anillo de claves, ejecutando para ello los comandos:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver keys.gnupg.net --recv-keys 8352B9BB
gpg: clave 15E1B16E8352B9BB: clave pública "Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1

alvaro@debian:~$ gpg --keyserver pgp.surfnet.nl --recv-keys DC2F7A96
gpg: clave BEAECEA4DC2F7A96: clave pública "Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1

alvaro@debian:~$ gpg --keyserver keys.gnupg.net --recv-keys D4BB0593
gpg: clave 73986F40D4BB0593: clave pública "Adrián Rodríguez Povea <arodriguezpovea@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
{% endhighlight %}

Como se puede apreciar, las claves de **Juan Luis**, **Francisco Javier** y **Adrián** ya han sido importadas, aunque de todas formas, vamos a listar las claves existentes en nuestro _keyring_ para así verificarlo:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      4C220919DD2364BED7D49C3215E1B16E8352B9BB
uid        [desconocida] Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      D6BE9BF1C7D6A435BF1B9D38BEAECEA4DC2F7A96
uid        [desconocida] Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa4096 2020-10-14 [SC]
      04EF48BD3DD780EF5ED977F973986F40D4BB0593
uid        [desconocida] Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sub   rsa4096 2020-10-14 [E]
{% endhighlight %}

Efectivamente, las 3 claves han sido correctamente importadas, y si nos fijamos, su validez es actualmente **desconocida**, ya que todavía no hemos firmado las claves, lo que quiere decir que en caso de querer encriptar con dichas claves públicas o comprobar firmas, nos mostrará un mensaje de advertencia, indicando que no existen indicios que dichas personas sean realmente quien dicen ser.

El siguiente paso será firmar dichas claves haciendo uso del _fingerprint_ para identificarlas (o de sus 8 últimos dígitos). Para ello, ejecutaremos los comandos:

{% highlight shell %}
alvaro@debian:~$ gpg --sign-key 8352B9BB

pub  rsa3072/15E1B16E8352B9BB
     creado: 2020-10-07  caduca: 2022-10-07  uso: SC  
     confianza: desconocido   validez: desconocido
sub  rsa3072/988EDEB8C4FF299C
     creado: 2020-10-07  caduca: 2022-10-07  uso: E   
[desconocida] (1). Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>


pub  rsa3072/15E1B16E8352B9BB
     creado: 2020-10-07  caduca: 2022-10-07  uso: SC  
     confianza: desconocido   validez: desconocido
 Huella clave primaria: 4C22 0919 DD23 64BE D7D4  9C32 15E1 B16E 8352 B9BB

     Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>

Esta clave expirará el 2022-10-07.
¿Está realmente seguro de querer firmar esta clave
con su clave: "Álvaro Vaca Ferreras <avacaferreras@gmail.com>" (3E0DA17912B9A4F8)?

¿Firmar de verdad? (s/N) s

alvaro@debian:~$ gpg --sign-key DC2F7A96
...

alvaro@debian:~$ gpg --sign-key D4BB0593
...
{% endhighlight %}

Todas las claves han sido ya firmadas (es posible que nos haya pedido la frase de paso de nuestra clave privada a la hora de llevar a cabo el proceso de firmado), por lo que volveremos a listar las claves existentes en nuestro _keyring_ para así verificar que su validez es ahora **total**:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      4C220919DD2364BED7D49C3215E1B16E8352B9BB
uid        [   total   ] Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      D6BE9BF1C7D6A435BF1B9D38BEAECEA4DC2F7A96
uid        [   total   ] Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa4096 2020-10-14 [SC]
      04EF48BD3DD780EF5ED977F973986F40D4BB0593
uid        [   total   ] Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sub   rsa4096 2020-10-14 [E]
{% endhighlight %}

Efectivamente, su validez ha cambiado de **desconocida** a **total**, por lo que en caso de querer encriptar con dichas claves públicas o comprobar firmas, no se nos mostrará el mensaje de advertencia.

Tras ello, tendremos que exportar las claves de nuestros compañeros con la firma que acabamos de hacer y distribuirlas manualmente a sus propietarios para que las importen. En este caso, exporté las claves de mis tres compañeros ejecutando para ello los comandos:

{% highlight shell %}
alvaro@debian:~$ gpg --export -a D4BB0593 > clave-adrian.asc
alvaro@debian:~$ gpg --export -a 8352B9BB > clave-juanlu.asc
alvaro@debian:~$ gpg --export -a DC2F7A96 > clave-fran.asc
{% endhighlight %}

En un principio, las claves ya han sido correctamente exportadas a los correspondientes ficheros, así que para verificarlo, vamos a listar el contenido del directorio actual, estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'clave'
-rw-r--r-- 1 alvaro alvaro 3752 oct 21 14:32 clave-adrian.asc
-rw-r--r-- 1 alvaro alvaro 3086 oct 21 14:33 clave-fran.asc
-rw-r--r-- 1 alvaro alvaro 3082 oct 21 14:32 clave-juanlu.asc
{% endhighlight %}

Efectivamente, las tres claves públicas han sido correctamente exportadas con las firmas.

Mis compañeros han seguido exactamente los mismos pasos, así que ellos también me enviaron mi clave con las correspondientes firmas, que debemos importar. En mi caso, he almacenado dichas firmas dentro de un directorio llamado **firmas/**, por lo que nos moveremos dentro del mismo, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ cd firmas/
{% endhighlight %}

Una vez nos encontremos dentro del mismo, listaremos el contenido para verificar que las firmas se encuentran ahí almacenadas:

{% highlight shell %}
alvaro@debian:~/firmas$ ls -l
total 12
-rw-r--r-- 1 alvaro alvaro 3065 oct 23 10:04 clave-alvaro.asc
-rw-r--r-- 1 alvaro alvaro 3240 oct 23 10:04 clavealvarofirmadaadri.asc
-rw-r--r-- 1 alvaro alvaro 2204 oct 23 10:04 firma-alvaro.key
{% endhighlight %}

Efectivamente, existen 3 ficheros, uno por cada una de las firmas que se han realizado a mi clave pública, así que tendremos que importarlas, ejecutando para ello los comandos:

{% highlight shell %}
alvaro@debian:~/firmas$ gpg --import clave-alvaro.asc 
gpg: clave 3E0DA17912B9A4F8: "Álvaro Vaca Ferreras <avacaferreras@gmail.com>" 1 firma nueva
gpg: Cantidad total procesada: 1
gpg:         nuevas firmas: 1
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: nivel: 0  validez:   1  firmada:   3  confianza: 0-, 0q, 0n, 0m, 0f, 1u
gpg: nivel: 1  validez:   3  firmada:   0  confianza: 3-, 0q, 0n, 0m, 0f, 0u
gpg: siguiente comprobación de base de datos de confianza el: 2022-10-07

alvaro@debian:~/firmas$ gpg --import clavealvarofirmadaadri.asc 
...

alvaro@debian:~/firmas$ gpg --import firma-alvaro.key 
...
{% endhighlight %}

Todas las firmas han sido ya importadas, así que para verificarlo, vamos a listar las firmas de las claves públicas existentes en nuestro _keyring_, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/firmas$ gpg --list-sig
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig 3        3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          15E1B16E8352B9BB 2020-10-21  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig          73986F40D4BB0593 2020-10-21  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          BEAECEA4DC2F7A96 2020-10-21  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]
sig          3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      4C220919DD2364BED7D49C3215E1B16E8352B9BB
uid        [   total   ] Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig 3        15E1B16E8352B9BB 2020-10-07  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
sig          15E1B16E8352B9BB 2020-10-07  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      D6BE9BF1C7D6A435BF1B9D38BEAECEA4DC2F7A96
uid        [   total   ] Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sig 3        BEAECEA4DC2F7A96 2020-10-07  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
sig          BEAECEA4DC2F7A96 2020-10-07  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>

pub   rsa4096 2020-10-14 [SC]
      04EF48BD3DD780EF5ED977F973986F40D4BB0593
uid        [   total   ] Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig 3        73986F40D4BB0593 2020-10-14  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa4096 2020-10-14 [E]
sig          73986F40D4BB0593 2020-10-14  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
{% endhighlight %}

Como se puede apreciar, la salida es bastante grande, pero aun así, si nos fijamos, veremos que nuestra clave tiene un total de 3 firmas (además de la nuestra) correspondientes a las firmas realizadas por nuestros compañeros. Además, las 3 claves públicas de nuestros compañeros se encuentran firmadas por nosotros (además de por ellos mismos). Sin embargo, todavía no nos aparecen las firmas que los otros compañeros han realizado sobre dichas claves (ya que si recordamos, todos vamos a firmas las claves de todos). Ésto se debe a que debemos volver a importar las claves con las nuevas firmas.

En una situación ideal, hubiésemos volvido a subir nuestras claves con las nuevas firmas al servidor de claves, pero en ese momento, no funcionaba, así que tuvimos que exportar las claves y distribuirlas manualmente. Antes de ello, saldré del directorio **firmas/** para volver a situarme en el directorio personal, donde se encuentran todas las claves, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~/firmas$ cd
{% endhighlight %}

Una vez situados en el directorio personal, procederemos a exportar nuestra clave con todas las firmas importadas, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --export -a "Álvaro Vaca Ferreras" > clave.asc
{% endhighlight %}

Para verificar que se ha exportado correctamente, vamos a listar el contenido del directorio actual, estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'clave'
-rw-r--r-- 1 alvaro alvaro      3752 oct 21 14:32 clave-adrian.asc
-rw-r--r-- 1 alvaro alvaro      4426 oct 27 12:14 clave.asc
-rw-r--r-- 1 alvaro alvaro      3086 oct 21 14:33 clave-fran.asc
-rw-r--r-- 1 alvaro alvaro      3082 oct 21 14:32 clave-juanlu.asc
{% endhighlight %}

Como se puede apreciar, nuestra clave ha sido correctamente exportada con nombre **clave.asc**, así que volveremos a distribuirla a nuestros compañeros para que puedan importarla y así percatarse de las nuevas firmas que ha recibido nuestra clave pública. Nosotros, realizaremos el mismo proceso de importación con sus claves, las cuáles he descargado en el directorio actual, haciendo uso de los nombres anteriormente establecidos:

{% highlight shell %}
alvaro@debian:~$ gpg --import clave-fran.asc
gpg: clave BEAECEA4DC2F7A96: "Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>" 2 firmas nuevas
gpg: Cantidad total procesada: 1
gpg:         nuevas firmas: 2
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: nivel: 0  validez:   1  firmada:   3  confianza: 0-, 0q, 0n, 0m, 0f, 1u
gpg: nivel: 1  validez:   3  firmada:   0  confianza: 3-, 0q, 0n, 0m, 0f, 0u
gpg: siguiente comprobación de base de datos de confianza el: 2022-10-07

alvaro@debian:~$ gpg --import clave-juanlu.asc 
...

alvaro@debian:~$ gpg --import clave-adrian.asc 
...
{% endhighlight %}

Como se puede apreciar en los primeros mensajes devueltos, la clave ha sido correctamente importada con 2 nuevas firmas (pues la nuestra no se cuenta, ya que existía con anterioridad). De igual forma, volveremos a listar las firmas de las claves existentes en nuestro _keyring_, para así verificar que tenemos constancia de todas las firmas:

{% highlight shell %}
alvaro@debian:~$ gpg --list-sig
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig 3        3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          15E1B16E8352B9BB 2020-10-21  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig          73986F40D4BB0593 2020-10-21  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          BEAECEA4DC2F7A96 2020-10-21  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]
sig          3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      4C220919DD2364BED7D49C3215E1B16E8352B9BB
uid        [   total   ] Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig 3        15E1B16E8352B9BB 2020-10-07  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          73986F40D4BB0593 2020-10-21  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          BEAECEA4DC2F7A96 2020-10-21  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
sig          15E1B16E8352B9BB 2020-10-07  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      D6BE9BF1C7D6A435BF1B9D38BEAECEA4DC2F7A96
uid        [   total   ] Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sig 3        BEAECEA4DC2F7A96 2020-10-07  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          73986F40D4BB0593 2020-10-21  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          15E1B16E8352B9BB 2020-10-27  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
sig          BEAECEA4DC2F7A96 2020-10-07  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>

pub   rsa4096 2020-10-14 [SC]
      04EF48BD3DD780EF5ED977F973986F40D4BB0593
uid        [   total   ] Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig 3        73986F40D4BB0593 2020-10-14  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          3E0DA17912B9A4F8 2020-10-21  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          BEAECEA4DC2F7A96 2020-10-21  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sig          15E1B16E8352B9BB 2020-10-27  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sub   rsa4096 2020-10-14 [E]
sig          73986F40D4BB0593 2020-10-14  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
{% endhighlight %}

Si miramos con detenimiento la gran salida del comando, podremos verificar que todas las claves se encuentran firmadas por todos, por lo que todos hemos validado las claves de todos.

Antes de continuar, he considerado oportuno tratar de verificar la firma de un fichero de una persona cuya clave consideramos actualmente válida. En este caso, tengo un fichero de nombre **hola.txt.gpg** que ha sido firmado por **Francisco Javier**, así que vamos a hacer uso de la opción `--verify` para verificar la firma:

{% highlight shell %}
alvaro@debian:~$ gpg --verify hola.txt.gpg 
gpg: Firmado el mar 03 nov 2020 20:29:02 CET
gpg:                usando RSA clave D6BE9BF1C7D6A435BF1B9D38BEAECEA4DC2F7A96
gpg: Firma correcta de "Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>" [total]
{% endhighlight %}

Efectivamente, la firma se ha comprobado correctamente y no ha mostrado ningún mensaje de advertencia, pues la validez de la clave pública asociada a la clave privada con la que dicho fichero ha sido firmado es **total**.

Bien, hemos llegado a la parte interesante de la tarea. Vamos a introducir a una tercera persona en el anillo de confianza, pero nosotros no vamos a firmar su clave pública, sino que la va a firmar una persona en la que nosotros vamos a establecer una confianza total, por ejemplo, **Juan Luis**.

El primer paso será configurar dicha confianza, así que para comprobar las confianzas que actualmente tenemos establecidas, vamos a hacer uso de la opción `--export-ownertrust`, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --export-ownertrust
# Lista de valores de confianza asignados, creada mar 27 oct 2020 13:10:32 CET
# (Use "gpg --import-ownertrust" para restablecerlos)
B02B578465B0756DFD271C733E0DA17912B9A4F8:6:
{% endhighlight %}

Como se puede apreciar, únicamente tenemos confianza en mi par de claves, pues se asignó de forma automática una confianza absoluta (**6**) en el momento de la creación, de manera que debemos asignarle una confianza total al par de claves de **Juan Luis**, haciendo uso de la opción `--edit-key`, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --edit-key 8352B9BB
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


pub  rsa3072/15E1B16E8352B9BB
     creado: 2020-10-07  caduca: 2022-10-07  uso: SC  
     confianza: desconocido   validez: total
sub  rsa3072/988EDEB8C4FF299C
     creado: 2020-10-07  caduca: 2022-10-07  uso: E   
[   total   ] (1). Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>

gpg>
{% endhighlight %}

Como se puede apreciar, nos ha mostrado la información sobre la clave pública que vamos a editar, y nos ha informado de que la confianza actual es **desconocido**. Para modificar dicho valor, tendremos que introcir la orden `trust`:

{% highlight shell %}
gpg> trust
pub  rsa3072/15E1B16E8352B9BB
     creado: 2020-10-07  caduca: 2022-10-07  uso: SC  
     confianza: desconocido   validez: total
sub  rsa3072/988EDEB8C4FF299C
     creado: 2020-10-07  caduca: 2022-10-07  uso: E   
[   total   ] (1). Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>

Por favor, decida su nivel de confianza en que este usuario
verifique correctamente las claves de otros usuarios (mirando
pasaportes, comprobando huellas dactilares en diferentes fuentes...)


  1 = No lo sé o prefiero no decirlo
  2 = NO tengo confianza
  3 = Confío un poco
  4 = Confío totalmente
  5 = confío absolutamente
  m = volver al menú principal

¿Su decisión? 4

pub  rsa3072/15E1B16E8352B9BB
     creado: 2020-10-07  caduca: 2022-10-07  uso: SC  
     confianza: total         validez: absoluta
sub  rsa3072/988EDEB8C4FF299C
     creado: 2020-10-07  caduca: 2022-10-07  uso: E   
[  absoluta ] (1). Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
Ten en cuenta que la validez de clave mostrada no es necesariamente
correcta a menos de que reinicies el programa.

gpg> quit
{% endhighlight %}

En este caso, le he asignado una confianza **total** (ya que la confianza absoluta únicamente debe asignarse a nuestras claves personales). Para verificar que dicho cambio se ha llevado a cambio correctamente, volveremos a listar las confianzas actualmente establecidas:

{% highlight shell %}
alvaro@debian:~$ gpg --export-ownertrust
# Lista de valores de confianza asignados, creada dom 08 nov 2020 18:45:37 CET
# (Use "gpg --import-ownertrust" para restablecerlos)
B02B578465B0756DFD271C733E0DA17912B9A4F8:6:
4C220919DD2364BED7D49C3215E1B16E8352B9BB:5:
{% endhighlight %}

Efectivamente, a la clave pública de **Juan Luis** se le ha asignado una confianza total (**5**).

En este caso, la persona que vamos a introducir en el anillo de confianza es **Javier**, por lo que importaremos su clave pública con la firma de **Juan Luis** correctamente importada (clave que ha sido previamente exportada a un fichero de nombre **clave-javier.asc**), ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --import clave-javier.asc 
gpg: clave 6F7A456BC67662D3: clave pública "Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: nivel: 0  validez:   1  firmada:   3  confianza: 0-, 0q, 0n, 0m, 0f, 1u
gpg: nivel: 1  validez:   3  firmada:   1  confianza: 3-, 0q, 0n, 0m, 0f, 0u
gpg: siguiente comprobación de base de datos de confianza el: 2022-10-07
{% endhighlight %}

Al parecer, la clave ha sido correctamente importada, así que vamos a listar las firmas de las claves de nuestro _keyring_ para así verificar que ha sido correctamente firmada por **Juan Luis**:

{% highlight shell %}
alvaro@debian:~$ gpg --list-sig
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig 3        3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sig          15E1B16E8352B9BB 2020-10-21  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sig          73986F40D4BB0593 2020-10-21  Adrián Rodríguez Povea <arodriguezpovea@gmail.com>
sig          BEAECEA4DC2F7A96 2020-10-21  Francisco Javier Martín Núñez <franjaviermn17100@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]
sig          3E0DA17912B9A4F8 2020-10-13  Álvaro Vaca Ferreras <avacaferreras@gmail.com>

...

pub   rsa3072 2020-10-21 [SC] [caduca: 2022-10-21]
      76270D5E766E0F22D70466BE6F7A456BC67662D3
uid        [no definida] Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>
sig 3        6F7A456BC67662D3 2020-10-21  Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>
sig          15E1B16E8352B9BB 2020-10-27  Juan Luis Millan Hidalgo <juanluismillanhidalgo@gmail.com>
sub   rsa3072 2020-10-21 [E] [caduca: 2022-10-21]
sig          6F7A456BC67662D3 2020-10-21  Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>
{% endhighlight %}

Efectivamente, la clave de **Javier** está firmada por **Juan Luis**, así que lo único que queda es tratar de verificar la firma de un fichero firmado por **Javier**. En este caso, dicho fichero será **apariencia_dashtopanel.gpg**, por lo que haremos uso de la opción `--verify`:

{% highlight shell %}
alvaro@debian:~$ gpg --verify apariencia_dashtopanel.gpg 
gpg: Firmado el mar 27 oct 2020 12:50:41 CET
gpg:                usando RSA clave 76270D5E766E0F22D70466BE6F7A456BC67662D3
gpg: Firma correcta de "Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>" [total]
{% endhighlight %}

Como se puede apreciar, la firma ha sido correctamente verificada y sin advertencias de seguridad, ya que la clave pública de **Javier** ha sido correctamente validada, al tener una confianza total en **Juan Luis**, quién ha firmado dicha clave. Por último, vamos a listar las claves existentes en nuestro _keyring_ para así comprobar que la validez de su clave ha cambiado:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-13 [SC] [caduca: 2022-10-13]
      B02B578465B0756DFD271C733E0DA17912B9A4F8
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-13 [E] [caduca: 2022-10-13]

...

pub   rsa3072 2020-10-21 [SC] [caduca: 2022-10-21]
      76270D5E766E0F22D70466BE6F7A456BC67662D3
uid        [   total   ] Javier Pérez Hidalgo <javierperezhidalgo01@gmail.com>
sub   rsa3072 2020-10-21 [E] [caduca: 2022-10-21]
{% endhighlight %}

Efectivamente, la validez de la clave de **Javier** es ahora **total**, por el motivo anteriormente mencionado.

Si en lugar de haber asignado una confianza total a **Juan Luis**, le hubiésemos asignado una confianza marginal (**un poco**), necesitaríamos que 3 personas con dicho nivel de confianza firmasen la clave pública de **Javier** para que hubiese sido validada sin nuestra intervención.

## Tarea 2: Integridad de ficheros

En esta tarea, vamos a llevar a cabo un proceso de comprobación de integridad de una imagen ISO de Debian, un proceso que deberíamos llevar a cabo cada vez que descargamos ficheros desde fuentes oficiales y nos ofrecen métodos de comprobación fiables, como es este caso, pero que por comodidad y pereza, casi nunca hacemos.

El primer paso, como es lógico, consiste en descargar los ficheros necesarios, que podremos encontrar [aquí](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/). En este caso, descargaremos los siguientes ficheros haciendo uso de `wget`:

* **debian-10.6.0-amd64-netinst.iso**: La imagen ISO cuya integridad comprobaremos.
* **SHA256SUMS**: El fichero con el _hash_ de las distintas imágenes, tras aplicar el algoritmo SHA256.
* **SHA256SUMS.sign**: La firma haciendo uso de la clave privada de Debian sobre el _hash_ del fichero _SHA256SUMS_.
* **SHA512SUMS**: El fichero con el _hash_ de las distintas imágenes, tras aplicar el algoritmo SHA512.
* **SHA512SUMS.sign**: La firma haciendo uso de la clave privada de Debian sobre el _hash_ del fichero _SHA512SUMS_.

También podríamos descargar el fichero **SHA1SUMS**, que contiene el _hash_ de las imágenes, tras aplicar el algoritmo SHA1, pero es un algoritmo que ya no es fiable, al igual que tampoco lo es MD5, por lo que lo obviaremos. Para llevar a cabo las descargas, ejecutaremos los siguientes comandos:

{% highlight shell %}
alvaro@debian:~$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.6.0-amd64-netinst.iso
alvaro@debian:~$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA256SUMS
alvaro@debian:~$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA256SUMS.sign
alvaro@debian:~$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA512SUMS
alvaro@debian:~$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA512SUMS.sign
{% endhighlight %}

Para verificar que todos los ficheros han sido correctamente descargados, vamos a listar el contenido del directorio actual, aplicando un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep '(debian|SHA)'
-rw-r--r-- 1 alvaro alvaro 365953024 sep 26 13:51 debian-10.6.0-amd64-netinst.iso
-rw-r--r-- 1 alvaro alvaro       402 sep 27 02:19 SHA256SUMS
-rw-r--r-- 1 alvaro alvaro       833 sep 27 02:24 SHA256SUMS.sign
-rw-r--r-- 1 alvaro alvaro       658 sep 27 02:19 SHA512SUMS
-rw-r--r-- 1 alvaro alvaro       833 sep 27 02:24 SHA512SUMS.sign
{% endhighlight %}

Efectivamente, todos los ficheros se encuentran ya ubicados en el directorio personal, así que podremos empezar el procedimiento.

Si recordamos, anteriormente mencioné que cuando se firma un fichero, realmente no se firma el fichero en su totalidad, pues sería muy costoso computacionalmente, sino un resumen del mismo, llamado _hash_, resultante de la aplicación de un algoritmo concreto, de manera que posteriormente se calculaba el _hash_ del fichero original y se comparaba con el que había sido firmado.

Si extrapolamos la definición anterior a este ejemplo, tenemos un fichero original (**debian-10.6.0-amd64-netinst.iso**) cuyo _hash_ (junto al de otras imágenes ISO) se encuentra contenido en el fichero **SHA256SUMS** (algoritmo SHA256) y en el fichero **SHA512SUMS** (algoritmo SHA512).

En este caso, en lugar de firmar directamente el _hash_ del fichero ISO, lo que se firma es el _hash_ del fichero que contiene el resumen de todos los _hash_ de las imágenes (**SHA256SUMS.sign** y **SHA512SUMS.sign**), para así no tener que hacer una firma por cada una de las imágenes ISO.

Quizás, puede llegar a ser algo lioso, así que vamos a ponernos manos a la obra para verlo más claro. El primer paso consiste en obtener la clave pública de Debian, con la que verificaremos la firma realizada con su clave privada. Tras una pequeña búsqueda, el comando a ejecutar para importar la clave pública de Debian es:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver keyring.debian.org --recv-keys 6294BE9B
gpg: clave DA87E80D6294BE9B: clave pública "Debian CD signing key <debian-cd@lists.debian.org>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
{% endhighlight %}

Listo, ya tenemos la clave pública de Debian en nuestro anillo de claves para así poder usarla a continuación.

Antes de verificar las firmas, me gustaría aclarar algunos conceptos, y a mi parecer, lo mejor es ver qué contienen dichos ficheros, ejecutando para ello los comandos:

{% highlight shell %}
alvaro@debian:~$ cat SHA256SUMS
2af8f43d4a7ab852151a7f630ba596572213e17d3579400b5648eba4cc974ed0  debian-10.6.0-amd64-netinst.iso
fff31b94f193e094400281d3f01589456562c8017f11cb1744fc325d83ba2ed4  debian-10.6.0-amd64-xfce-CD-1.iso
48a35c92457c3d688d144915505dbae9e2e741e9416b01703093c1b4a08c5e7c  debian-edu-10.6.0-amd64-netinst.iso
89eca24c5fe87d0ed30ae72bd6e2531b00f13dbfbe1c3babdd045fff86c1e0c7  debian-mac-10.6.0-amd64-netinst.iso

alvaro@debian:~$ cat SHA512SUMS
cb74dcb7f3816da4967c727839bdaa5efb2f912cab224279f4a31f0c9e35f79621b32afe390195d5e142d66cedc03d42f48874eba76eae23d1fac22d618cb669  debian-10.6.0-amd64-netinst.iso
ebd7a29a47bb537f50c432bf4bdb57b4ea4c34f4137187a96171729bf7315944d0a933990f4ab84c9c0721af18f21a34e7925d354829797e6ab3d0c92066a444  debian-10.6.0-amd64-xfce-CD-1.iso
a848055ca9a728e79c19426268fc8dc87b2d3fde5d9421a15c9f9f2fad5e82172676772bfa62a96242549fa501387f9965168439e53b7680a88a9486e99232fc  debian-edu-10.6.0-amd64-netinst.iso
500caf3e49955a5547fef02620a0dd89d87895fbad70f508091c6612f8b74d275201d7094c12b5cb60e5271d8f1b2cd66c211fdbc2cd7ec366ec42a935594fe0  debian-mac-10.6.0-amd64-netinst.iso
{% endhighlight %}

Como se puede apreciar, cada fichero contiene 4 líneas que muestran el _hash_ de cada fichero (correspondientes a las 4 imágenes ISO disponibles para descargar), como resultado de la aplicación de los correspondientes algoritmos (**SHA256** en el primero y **SHA512** en el segundo).

Destaca el hecho de que el algoritmo SHA512 es bastante más seguro, pues está compuesto por 128 dígitos hexadecimales (512 bits), frente a los 64 dígitos hexadecimales (256 bits) del algoritmo SHA256, pero con la desventaja de que es más costoso computacionalmente de calcular.

De las 4 líneas existentes en cada fichero, únicamente nos interesa la primera, pues es la correspondiente al fichero cuya integridad vamos a comprobar, pero, ¿cómo podemos verificar entonces que el _hash_ de dicho fichero coincide con el que nos ha proporcionado Debian? Es fácil, únicamente tendremos que hacer uso de las utilidades para ello, `sha256sum` y `sha512sum`, que reciben un fichero y calculan su _hash_:

{% highlight shell %}
alvaro@debian:~$ sha256sum debian-10.6.0-amd64-netinst.iso 
2af8f43d4a7ab852151a7f630ba596572213e17d3579400b5648eba4cc974ed0  debian-10.6.0-amd64-netinst.iso

alvaro@debian:~$ sha512sum debian-10.6.0-amd64-netinst.iso 
cb74dcb7f3816da4967c727839bdaa5efb2f912cab224279f4a31f0c9e35f79621b32afe390195d5e142d66cedc03d42f48874eba76eae23d1fac22d618cb669  debian-10.6.0-amd64-netinst.iso
{% endhighlight %}

Éstos serían los resultados tras aplicar los correspondientes algoritmos, de manera que si los comparamos visualmente, podremos ver que coinciden con los que Debian nos ha proporcionado.

Quizás, hacerlo de la anterior manera se puede convertir en una tarea un poco tediosa, es por ello que haremos uso de la opción **-c**, que irá recorriendo un determinado fichero que contiene el resumen de los _hash_ e irá calculando y comparando de forma automática el resultado en el fichero origen:

{% highlight shell %}
alvaro@debian:~/comprobacion$ sha256sum -c SHA256SUMS 2> /dev/null | egrep 'debian-10.6.0-amd64-netinst.iso'
debian-10.6.0-amd64-netinst.iso: La suma coincide

alvaro@debian:~/comprobacion$ sha512sum -c SHA512SUMS 2> /dev/null | egrep 'debian-10.6.0-amd64-netinst.iso'
debian-10.6.0-amd64-netinst.iso: La suma coincide
{% endhighlight %}

Tras redirigir los errores a **/dev/null** para que no se muestren por pantalla (ya que únicamente tenemos 1 de los 4 ficheros, por lo que va a dar errores de ficheros no encontrados) y filtrar por el nombre del fichero que queremos verificar, podemos asegurar que la suma coincide con la que Debian ha proporcionado.

Realmente, esto que hemos hecho no es garantía de que los ficheros no hayan sido manipulados, ya que el impostor podría haber modificado también el fichero en el que se muestra el resumen de los _hash_, de manera que tendremos que verificar la firma existente para así corroborar que el fichero con el resumen no ha sido manipulado. Para ello, haremos uso de la opción `--verify`, a la que le pasaremos como parámetros el fichero resumen y la firma de dicho fichero para así verificar la firma:

{% highlight shell %}
alvaro@debian:~/comprobacion$ gpg --verify SHA256SUMS.sign SHA256SUMS
gpg: Firmado el dom 27 sep 2020 02:24:23 CEST
gpg:                usando RSA clave DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Firma correcta de "Debian CD signing key <debian-cd@lists.debian.org>" [desconocido]
gpg: ATENCIÓN: ¡Esta clave no está certificada por una firma de confianza!
gpg:          No hay indicios de que la firma pertenezca al propietario.
Huellas dactilares de la clave primaria: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B

alvaro@debian:~/comprobacion$ gpg --verify SHA512SUMS.sign SHA512SUMS
gpg: Firmado el dom 27 sep 2020 02:24:23 CEST
gpg:                usando RSA clave DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Firma correcta de "Debian CD signing key <debian-cd@lists.debian.org>" [desconocido]
gpg: ATENCIÓN: ¡Esta clave no está certificada por una firma de confianza!
gpg:          No hay indicios de que la firma pertenezca al propietario.
Huellas dactilares de la clave primaria: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
{% endhighlight %}

Efectivamente, la el _hash_ del fichero resumen coincide con el _hash_ del fichero resumen que Debian ha firmado, por lo que podemos concluir que no ha habido ninguna manipulación y las imágenes ISO están totalmente limpias. Dado que no tenemos validada la clave de Debian (no la hemos firmado ni tampoco tenemos confianza en una persona que la haya firmado), se nos ha mostrado la advertencia de seguridad, pero eso no tiene nada que ver, pues la firma ha sido correctamente verificada.

## Tarea 3: Integridad y autenticidad (apt secure)

### ¿Qué software utiliza apt secure para realizar la criptografía asimétrica?

La herramienta que se utiliza en _Secure-Apt_ para firmar los ficheros y comprobar sus firmas es **GPG** (_GNU Privacy Guard_), cuyo funcionamiento explicaremos con más detalle a continuación.

### ¿Para qué sirve el comando apt-key? ¿Qué muestra el comando apt-key list?

El comando `apt-key` es una herramienta que permite gestionar la lista de claves que utiliza _APT_ para autenticar paquetes, considerando paquetes de confianza a todos aquellos que hayan sido correctamente autenticados con dichas claves. Las opciones más utilizadas son:

* **add**: Añade una nueva clave a la lista claves de confianza desde un fichero que pasamos como parámetro.
* **del**: Elimina una clave de la lista de claves de confianza, indicando como parámetro el _fingerprint_ de la clave.
* **export**: Muestra la clave por la salida estándar, indicando como parámetro el _fingerprint_ de la misma.
* **exportall**: Muestra todas las claves de confianza por la salida estándar.
* **list**: Lista todas las claves de confianza.
* **finger**: Muestra los _fingerprint_ de todas las claves de confianza.

Vamos a mostrar por ejemplo, la salida del comando `apt-key list`, de manera que nos mostrará todas las claves consideradas de confianza para _APT_:

{% highlight shell %}
alvaro@debian:~$ apt-key list
/etc/apt/trusted.gpg.d/debian-archive-buster-automatic.gpg
----------------------------------------------------------
pub   rsa4096 2019-04-14 [SC] [caduca: 2027-04-12]
      80D1 5823 B7FD 1561 F9F7  BCDD DC30 D7C2 3CBB ABEE
uid        [desconocida] Debian Archive Automatic Signing Key (10/buster) <ftpmaster@debian.org>
sub   rsa4096 2019-04-14 [S] [caduca: 2027-04-12]

/etc/apt/trusted.gpg.d/debian-archive-buster-security-automatic.gpg
-------------------------------------------------------------------
pub   rsa4096 2019-04-14 [SC] [caduca: 2027-04-12]
      5E61 B217 265D A980 7A23  C5FF 4DFA B270 CAA9 6DFA
uid        [desconocida] Debian Security Archive Automatic Signing Key (10/buster) <ftpmaster@debian.org>
sub   rsa4096 2019-04-14 [S] [caduca: 2027-04-12]

/etc/apt/trusted.gpg.d/debian-archive-buster-stable.gpg
-------------------------------------------------------
pub   rsa4096 2019-02-05 [SC] [caduca: 2027-02-03]
      6D33 866E DD8F FA41 C014  3AED DCC9 EFBF 77E1 1517
uid        [desconocida] Debian Stable Release Key (10/buster) <debian-release@lists.debian.org>

/etc/apt/trusted.gpg.d/debian-archive-jessie-automatic.gpg
----------------------------------------------------------
pub   rsa4096 2014-11-21 [SC] [caduca: 2022-11-19]
      126C 0D24 BD8A 2942 CC7D  F8AC 7638 D044 2B90 D010
uid        [desconocida] Debian Archive Automatic Signing Key (8/jessie) <ftpmaster@debian.org>

/etc/apt/trusted.gpg.d/debian-archive-jessie-security-automatic.gpg
-------------------------------------------------------------------
pub   rsa4096 2014-11-21 [SC] [caduca: 2022-11-19]
      D211 6914 1CEC D440 F2EB  8DDA 9D6D 8F6B C857 C906
uid        [desconocida] Debian Security Archive Automatic Signing Key (8/jessie) <ftpmaster@debian.org>

/etc/apt/trusted.gpg.d/debian-archive-jessie-stable.gpg
-------------------------------------------------------
pub   rsa4096 2013-08-17 [SC] [caduca: 2021-08-15]
      75DD C3C4 A499 F1A1 8CB5  F3C8 CBF8 D6FD 518E 17E1
uid        [desconocida] Jessie Stable Release Key <debian-release@lists.debian.org>

/etc/apt/trusted.gpg.d/debian-archive-stretch-automatic.gpg
-----------------------------------------------------------
pub   rsa4096 2017-05-22 [SC] [caduca: 2025-05-20]
      E1CF 20DD FFE4 B89E 8026  58F1 E0B1 1894 F66A EC98
uid        [desconocida] Debian Archive Automatic Signing Key (9/stretch) <ftpmaster@debian.org>
sub   rsa4096 2017-05-22 [S] [caduca: 2025-05-20]

/etc/apt/trusted.gpg.d/debian-archive-stretch-security-automatic.gpg
--------------------------------------------------------------------
pub   rsa4096 2017-05-22 [SC] [caduca: 2025-05-20]
      6ED6 F5CB 5FA6 FB2F 460A  E88E EDA0 D238 8AE2 2BA9
uid        [desconocida] Debian Security Archive Automatic Signing Key (9/stretch) <ftpmaster@debian.org>
sub   rsa4096 2017-05-22 [S] [caduca: 2025-05-20]

/etc/apt/trusted.gpg.d/debian-archive-stretch-stable.gpg
--------------------------------------------------------
pub   rsa4096 2017-05-20 [SC] [caduca: 2025-05-18]
      067E 3C45 6BAE 240A CEE8  8F6F EF0F 382A 1A7B 6500
uid        [desconocida] Debian Stable Release Key (9/stretch) <debian-release@lists.debian.org>
{% endhighlight %}

La salida del mismo es grande, pues hay varios _keyring_ extra dentro del directorio **/etc/apt/trusted.gpg.d/** que contienen a su vez más claves de confianza.

### ¿En que fichero se guarda el anillo de claves que usa la herramienta apt-key?

El anillo de claves que gestiona `apt-key` se encuentra en **/etc/apt/trusted.gpg**, de manera que las nuevas claves se irán añadiendo a dicho fichero, aunque como hemos visto en el ejemplo anterior, existen varios _keyring_ extra dentro del directorio **/etc/apt/trusted.gpg.d/** que contienen a su vez más claves de confianza.

### ¿Qué contiene el archivo Release de un repositorio de paquetes? ¿Y el archivo Release.gpg? Explica el proceso por el cual el sistema nos asegura que los ficheros que estamos descargando son legítimos.

Los repositorios Debian contienen varios ficheros, entre los que se encuentran **Release** y **Release.gpg**.

El primero de ellos, contiene una lista de los archivos **Packages** (incluyendo sus formas comprimidas, **Packages.gz** y **Packages.xz**, así como las versiones incrementales), junto con sus _hash_ MD5, SHA1 y SHA256. Gracias a esto, podremos verificar que dichos ficheros **Packages** no han sido manipulados, llevando a cabo una comparación del _hash_ que se nos ha facilitado con el _hash_ que nosotros calculamos sobre el fichero que hemos descargado, de manera que en caso de que el _hash_ coincida, podremos asegurar que dicho fichero no ha sido manipulado.

De nada serviría éste método si el posible impostor lograse modificar también el fichero **Release**, haciéndonos creer que dado que los _hash_ coinciden, no ha habido manipulación de los mismos, de manera que para ello, se lleva a cabo un proceso de firmado del _hash_ del fichero **Release**, cuyo resultado se encuentra en **Release.gpg**, de manera que sería imposible falsificar dicha información. Es por ello, que _APT_ hace uso de las claves públicas contenidas en **/etc/apt/trusted.gpg** y **/etc/apt/trusted.gpg.d/** para así verificar la firma del fichero **Release**.

El contenido de dichos ficheros **Packages** es una lista de los paquetes disponibles en la réplica junto con sus _hash_, lo que asegura, a su vez, que el contenido de dichos paquetes tampoco ha sido modificado.

En resumen, cuando llevamos a cabo un `apt update`, se descargan los ficheros **Packages.gz**, **Release** y **Release.gpg**, ficheros que son necesarios para la hora de la descarga del paquete, pues lo primero que se hace es comprobar que el _hash_ del paquete **.deb** coincide con el indicado en el fichero **Packages.gz**, comprobando a su vez que coincide con el indicado en el fichero **Release**, comprobando por último la firma de dicho fichero, que se encuentra contenida en el fichero **Release.gpg**.

Si nos fijamos, se establece una estructura de firmado jerárquica, de manera que ante cualquier mínima manipulación, nos daríamos cuenta, pues el hecho de modificar un único bit haría que el _hash_ resultante fuese totalmente distinto.

Como curiosidad, el uso de los ficheros **Release** y **Release.gpg** está en decadencia, ya que se está llevando a cabo una transición al uso del fichero **InRelease**, pues éste último está firmado criptográficamente en línea, mientras que para el otro caso, se necesita hacer uso de dos ficheros, al tener la firma por separado.

### Añade de forma correcta el repositorio de VirtualBox junto a su clave pública.

Lo primero que haremos será tratar de instalar VirtualBox, para así mostrar que dado que no se encuentra en los repositorios oficiales de Debian, no podremos llevar a cabo su descarga e instalación:

{% highlight shell %}
alvaro@debian:~$ sudo apt install virtualbox-6.1
Leyendo lista de paquetes... Hecho
Creando árbol de dependencias       
Leyendo la información de estado... Hecho
E: No se ha podido localizar el paquete virtualbox-6.1
E: No se pudo encontrar ningún paquete usando «*» con «virtualbox-6.1»
E: No se pudo encontrar ningún paquete con la expresión regular «virtualbox-6.1»
{% endhighlight %}

Efectivamente, la instalación ha fallado ya que no ha podido encontrar el paquete. Para ello, tendremos que añadir los repositorios de VirtualBox al fichero en el que se encuentran reflejados los repositorios disponibles, es decir, a **/etc/apt/sources.list**, ejecutando para ello el comando (con permisos de administrador, ejecutando el comando `su -`):

{% highlight shell %}
root@debian:~# echo "deb https://download.virtualbox.org/virtualbox/debian buster contrib" >> /etc/apt/sources.list
{% endhighlight %}

El repositorio ya ha sido añadido. Muy importante hacer uso de **>>** en lugar de **>**, ya que de lo contrario, sobreescribiría dicho fichero.

Tras ello, tal y como hemos visto con anterioridad, tendremos que añadir la clave pública de VirtualBox a nuestro anillo de claves de `apt-key`, ya que de lo contrario, no podría verificar la firma del fichero **Release** que descargará de dicho repositorio. Para ello, ejecutaremos los comandos:

{% highlight shell %}
root@debian:~# wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | apt-key add -
OK

root@debian:~# wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | apt-key add -
OK
{% endhighlight %}

Las claves públicas han sido correctamente importadas, así que ya estamos en posición de llevar a cabo una actualización de la paquetería disponible, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# apt update
Obj:1 http://security.debian.org/debian-security buster/updates InRelease
Obj:2 http://deb.debian.org/debian buster InRelease                              
Obj:3 http://deb.debian.org/debian buster-updates InRelease                      
Des:4 https://download.virtualbox.org/virtualbox/debian buster InRelease [7.736 B]
Des:5 https://download.virtualbox.org/virtualbox/debian buster/contrib amd64 Packages [1.775 B]
Des:6 https://download.virtualbox.org/virtualbox/debian buster/contrib amd64 Contents (deb) [4.171 B]
Descargados 13,7 kB en 1s (17,8 kB/s)       
Leyendo lista de paquetes... Hecho
Creando árbol de dependencias       
Leyendo la información de estado... Hecho
Se pueden actualizar 2 paquetes. Ejecute «apt list --upgradable» para verlos.
{% endhighlight %}

Como se puede apreciar, ha descargado el fichero **InRelease** (ya que como he mencionado, el uso de **Release** y **Release.gpg** cada vez es menor), junto al fichero **Packages**, que contiene la información sobre los paquetes que pueden instalarse. Tras ello, volveremos a intentar instalar VirtualBox, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# apt install virtualbox-6.1
{% endhighlight %}

Para verificar que la instalación se ha completado, haremos uso de `dpkg -s`, filtrando por el estado actual del paquete:

{% highlight shell %}
root@debian:~# dpkg -s virtualbox-6.1 | egrep 'Status'
Status: install ok installed
{% endhighlight %}

Efectivamente, hemos conseguido instalar VirtualBox, añadiendo el repositorio junto con las claves públicas necesarias para verificar que el paquete no ha sido manipulado.

## Tarea 4: Autentificación: ejemplo SSH

### Explica los pasos que se producen entre el cliente y el servidor para que el protocolo cifre la información que se transmite. ¿Para qué se utiliza la criptografía simétrica? ¿Y la asimétrica?

La ventaja significativa ofrecida por el protocolo SSH sobre sus predecesores es el uso del cifrado para asegurar la transferencia segura de información entre el servidor remoto y el cliente.

Cuando un cliente intenta conectarse al servidor a través de TCP, el servidor presenta los protocolos de cifrado y las respectivas versiones que soporta. Si el cliente tiene un par similar de protocolo y versión, se alcanza un acuerdo y se inicia la conexión con el protocolo aceptado, de manera que ambas partes producen pares de claves temporales e intercambian la clave pública, es decir, se hace uso de la criptografía asimétrica. El cliente ha de autentificar al servidor, comprobando el _host key_ que ha recibido, preguntando al usuario si confía en el mismo si es la primera vez que se conecta, o bien comparándolo con los _host key_ existentes en un archivo que almacena aquellos en los que se confía.

Una vez que esto se establece, las dos partes usan lo que se conoce como **Algoritmo de Intercambio de Claves _Diffie-Hellman_** para crear una clave simétrica. Este algoritmo permite que tanto el cliente como el servidor lleguen a una clave de cifrado compartida que se utilizará en adelante para cifrar toda la sesión de comunicación, ya que mantener una conexión cifrada mediante el uso de criptografía asimétrica sería totalmente inviable, debido el gran coste computacional que ésto conlleva.

Una vez establecido el cifrado simétrico para asegurar las comunicaciones entre el servidor y el cliente, el cliente debe autenticarse para permitir el acceso.

### Explica los dos métodos principales de autentificación: por contraseña y utilizando un par de claves públicas y privadas.

La etapa final antes de que se conceda al usuario acceso al servidor es la autenticación de sus credenciales, de manera que se hace uso de dos métodos principales de autentificación:

* **Contraseña**: Se le pide al usuario la contraseña asociada al usuario al que está tratando de conectarse. Éstas credenciales pasan con seguridad a través del túnel cifrado simétricamente, así que no hay ninguna posibilidad de que sean capturadas por un tercero.

* **Par de claves**: El usuario tiene su clave pública almacenada en el servidor (concretamente en el fichero **~/.ssh/authorized_keys**), de manera que a la hora de realizar la conexión, el usuario se presenta y le indica al servidor que es el dueño de dicha clave. El servidor desconfía, por lo que le hace un "desafío", consistente en encriptar un mensaje con dicha clave pública, enviándoselo encriptado al usuario que trata de acceder, el cuál debe descifrarlo haciendo uso de la clave privada asociada a dicha clave pública.

### En el cliente, ¿para qué sirve el contenido que se guarda en el fichero ~/.ssh/known_hosts?

Tal y como hemos mencionado con anterioridad, cuando el cliente lleva a cabo la conexión TCP con el servidor, éste debe ser autenticado por el cliente. Si es la primera vez que se lleva a cabo dicha conexión, se mostrará el siguiente mensaje de advertencia:

{% highlight shell %}
$ ssh debian@172.22.200.74
The authenticity of host '172.22.200.74 (172.22.200.74)' can't be established.
ECDSA key fingerprint is SHA256:7ZoNZPCbQTnDso1meVSNoKszn38ZwUI4i6saebbfL4M.
Are you sure you want to continue connecting (yes/no)?
{% endhighlight %}

El cliente tiene la posibilidad de decidir si quiere continuar o no con la conexión, dependiendo si confía en el otro extremo (en el servidor). En caso afirmativo, se añadirá el _host key_ al fichero situado en **~/.ssh/known_hosts**, de manera que para la siguiente ocasión, no se preguntará al cliente si confía, sino que mirará en dicho fichero para comprobar su previa existencia.

También puede darse el caso que el _host key_ asociado a una dirección IP cambie (puede darse por ejemplo porque la máquina asociada a dicha dirección IP ahora sea otra o bien haya sido reinstalada), de manera que a la hora de realizar una conexión SSH, saltarán las alarmas, pues el _host key_ actual no coincide con el almacenado para dicha dirección IP en el fichero, mostrando el siguiente mensaje y denegando la conexión:

{% highlight shell %}
$ ssh debian@172.22.200.74
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@     WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!    @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:W05RrybmcnJxD3fbwJOgSNNWATkVftsQl7EzfeKJgNc.
Please contact your system administrator.
Add correct host key in /home/jose/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/jose/.ssh/known_hosts:103
remove with:
ssh-keygen -f "/home/jose/.ssh/known_hosts" -R "172.22.200.74"
ECDSA host key for 172.22.200.74 has changed and you have requested strict checking.
{% endhighlight %}

Para solucionar éste problema tendríamos que borrar la línea correspondiente a dicho _host_ en el fichero **~/.ssh/known_hosts**. Generalmente, en el propio mensaje de advertencia te dice explícitamente el comando a ejecutar para borrar dicha línea.