---
layout: post
title:  "Cifrado asimétrico con gpg y openssl"
banner: "/assets/images/banners/cifrado.jpg"
date:   2020-10-11 10:05:00 +0200
categories: cifrado seguridad
---
## Tarea 1: Generación de claves (gpg).

### 1. Genera un par de claves (pública y privada). ¿En que directorio se guarda las claves de un usuario?

Para generar el par de claves haremos uso de la opción `--gen-key` de **gpg**:

{% highlight shell %}
alvaro@debian:~$ gpg --gen-key
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Nota: Usa "gpg --full-generate-key" para el diálogo completo de generación de clave.

GnuPG debe construir un ID de usuario para identificar su clave.

Nombre y apellidos:
{% endhighlight %}

Lo primero que nos pedirá será nuestro **nombre** junto con los **apellidos**, logrando así tener una forma para identificar la clave sobre el resto. Introduciremos nuestro nombre y apellidos y continuaremos.

{% highlight shell %}
Nombre y apellidos: Álvaro Vaca Ferreras
Dirección de correo electrónico:
{% endhighlight %}

Tras ello, nos pedirá también nuestra dirección de **correo electrónico**, así que la introduciremos y continuamos.

{% highlight shell %}
Dirección de correo electrónico: avacaferreras@gmail.com
Está usando el juego de caracteres 'utf-8'.
Ha seleccionado este ID de usuario:
    "Álvaro Vaca Ferreras <avacaferreras@gmail.com>"

¿Cambia (N)ombre, (D)irección o (V)ale/(S)alir?
{% endhighlight %}

Una vez introducida dicha información, nos mostrará los datos y nos pedirá una confirmación. En caso de que los datos mostrados sean correctos, elegiremos la opción **V**.

{% highlight shell %}
¿Cambia (N)ombre, (D)irección o (V)ale/(S)alir? V

Es necesario generar muchos bytes aleatorios. Es una buena idea realizar
alguna otra tarea (trabajar en otra ventana/consola, mover el ratón, usar
la red y los discos) durante la generación de números primos. Esto da al
generador de números aleatorios mayor oportunidad de recoger suficiente
entropía.
{% endhighlight %}

Una vez llegados a este punto se nos pedirá una **frase de paso** para proteger nuestra clave privada, así que la introduciremos dos veces y continuaremos para que comience la generación del par de claves. Tal y como se nos ha indicado en el mensaje, es aconsejable realizar alguna que otra tarea en nuestra máquina durante su generación.

{% highlight shell %}
gpg: clave CC02797F092855F6 marcada como de confianza absoluta
gpg: creado el directorio '/home/alvaro/.gnupg/openpgp-revocs.d'
gpg: certificado de revocación guardado como '/home/alvaro/.gnupg/openpgp-revocs.d/A0BE5CA7A9DC70AD3D619467CC02797F092855F6.rev'
claves pública y secreta creadas y firmadas.

pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid                      Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
{% endhighlight %}

Nuestro par de claves ya se encuentra generado y añadido a nuestro keyring **pubring.kbx**, que se encuentra almacenado en nuestro directorio personal, dentro de un directorio de nombre **.gnupg/**. Además, ha generado de forma automática un certificado de revocación dentro de **.gnupg/openpgp-revocs.d/** por si nuestra clave privada llegase a malas manos o simplemente quisiésemos dejar de utilizar dicho par de claves, de manera que se notificará a otros usuarios que la clave pública no debe ser usada nunca más para cifrar.

### 2. Lista las claves públicas que tienes en tu almacén de claves. Explica los distintos datos que nos muestra. ¿Cómo deberías haber generado las claves para indicar, por ejemplo, que tenga un 1 mes de validez?

Para listar las claves públicas haremos uso de la opción `--list-keys` de **gpg**:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
{% endhighlight %}

En lugar de explicar ahora los datos que nos ha mostrado, vamos a esperar al siguiente ejercicio y así podemos hacer una explicación más completa y detallada sobre la información que se está mostrando.

Para generar claves con una determinada validez, tendríamos que haber hecho uso de la opción `--full-gen-key` de **gpg**, donde nos habríamos encontrado un mensaje de la siguiente forma:

{% highlight shell %}
Por favor, especifique el período de validez de la clave.
         0 = la clave nunca caduca
      <n>  = la clave caduca en n días
      <n>w = la clave caduca en n semanas
      <n>m = la clave caduca en n meses
      <n>y = la clave caduca en n años
¿Validez de la clave (0)?
{% endhighlight %}

Tras ello, introduciríamos la validez deseada siguiendo el esquema que nos ha mostrado, en este caso, para que tenga un mes de validez, podríamos haber indicado **30**, **4w** o **1m**, pues todas hacen referencia al mismo lapso de tiempo.

### 3. Lista las claves privadas de tu almacén de claves.

Para listar las claves privadas haremos uso de la opción `--list-secret-keys` de **gpg**:

{% highlight shell %}
alvaro@debian:~$ gpg --list-secret-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
sec   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
ssb   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
{% endhighlight %}

Bien, es hora de explicar qué significan las diferentes abreviaturas devueltas por ambos comandos.

Cuando hemos listado las **claves públicas** hemos obtenido las siguientes abreviaturas:

* **pub** -- **pub**lic primary key.
* **uid** -- **u**nique **id**entifier
* **sub** -- public **sub**-key

Cuando hemos listado las **claves privadas** hemos obtenido las siguientes abreviaturas:

* **sec** -- **sec**ret primary key
* **uid** -- **u**nique **id**entifier
* **ssb** -- **s**ecret **s**u**b**-key

En la criptografía asimétrica siempre trabajamos con pares de claves: una clave **pública** para encriptar/comprobar firmas y una clave **privada** (secreta) para desencriptar/firmar, respectivamente.

Cuando generamos un par de claves OpenPGP con GnuPG, por defecto se genera un par de claves **primario** (también conocido como **master-key**) y un par de claves **secundario**. El par de claves primario contiene uno o más user-IDs (nombre, apellidos, correo electrónico) y se usa para para **firmar/comprobar firmas**, pues una prueba de identidad. Es por esto que debe guardarse con tanto cuidado la clave privada de nuestro par de claves maestro. De otro lado, el par de claves secundario se encuentra firmado por dicho par de claves primario, lo que confirma que pertenece a dicho user-ID y se usa para **encriptar/desencriptar**.

La idea principal de tener pares de claves maestros y pares de claves secundarios es que estos últimos pueden ser revocados independientemente de los pares de claves maestros, y también almacenados de forma separada. En otras palabras, los pares de claves secundarios son pares de claves independientes pero asociados a su vez al par de claves maestro.

Volviendo a la información que se nos ha mostrado a la hora de ejecutar los comandos, ahora podemos entenderla de una forma mucho más clara. Al ejecutar el comando `gpg --list-keys` se nos ha mostrado la información sobre las claves públicas del par de claves maestro (**pub**) y del par de claves secundario (**sub**), mientras que al ejecutar el comando `gpg --list-secret-keys` se nos ha mostrado la información sobre las claves privadas del par de claves maestro (**sec**) y del par de claves secundario (**ssb**). Además, en ambas ocasiones se ha mostrado un **uid**, que hace referencia a la identidad (user-ID) del usuario en concreto. En [este](https://wiki.debian.org/Subkeys) enlace se puede encontrar más información al respecto.

De esta manera, el enlace existente entre los pares de claves es el siguiente:

* **pub** -- **sec**
* **sub** -- **ssb**

En este caso se nos ha mostrado el algoritmo que usa dicho par de claves (**rsa3072**), la fecha de creación del par de claves (**2020-10-07**), los flags (los explicaremos a continuación) y la fecha de caducidad (**2022-10-07**).

Tal y como he mencionado anteriormente, el par de claves maestro es usado para **firmar o comprobar firmas**, mientras que el par de claves secundario es utilizado para **encriptar/desencriptar**. Si nos fijamos, en la ejecución de los comandos, el par de claves maestro tenía como flags **[SC]** y el par de claves secundario tenía como flag **[E]**. Los significados de los flags son los siguientes:

* **S:** Signing (firmar archivos).
* **C:** Certify (firmar una clave, también conocido como certificación).
* **A:** Authenticate (autenticarse en una máquina, como por ejemplo loguearse).
* **E:** Encrypt (encriptar información).

De manera que podemos concluir que el par de claves maestro permite firmar y comprobar firmas de archivos y claves (**[SC]**), mientras que el par de claves secundario permite encriptar y desencriptar información (**[E]**).

## Tarea 2: Importar/exportar clave pública (gpg).

### 1. Exporta tu clave pública en formato ASCII, guárdalo en un archivo "nombre_apellido.asc" y envíalo al compañero con el que vas a hacer esta práctica.

Para exportar nuestras claves públicas haremos uso de la opción `--export` de **gpg** junto con la opción `-a <nombre>`, para posteriormente redireccionar la salida a un fichero. En este caso, el **nombre** que debemos introducir es el mismo que hemos introducido a la hora de generar el par de claves. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --export -a "Álvaro Vaca Ferreras" > alvaro_vaca.asc
{% endhighlight %}

El comando ejecutado no ha devuelto ninguna salida por pantalla, así que para verificar que se ha generado correctamente, haremos uso del comando `ls`:

{% highlight shell %}
alvaro@debian:~$ ls
 alvaro_vaca.asc   Escritorio   Música       vagrant           virtualenv
 Descargas         GitHub       Plantillas   Vídeos
 Documentos        Imágenes     Público     'VirtualBox VMs'
{% endhighlight %}

Efectivamente, las claves públicas se encuentran correctamente exportadas dentro del fichero **alvaro_vaca.asc**.

Esta práctica la he realizado junto a **Alejandro Gutiérrez**, así que le he enviado mis claves públicas exportadas a través de Discord:

![envio1](https://i.ibb.co/FY7pPmX/2-1.png "Envío de claves")

### 2. Importa las claves públicas recibidas de vuestro compañero.

En este caso, las claves públicas de Alejandro las he recibido a través de Discord:

![envio2](https://i.ibb.co/b55Bjck/2-2.png "Envío de claves")

Para importar las claves públicas de nuestro compañero haremos uso de la opción `--import <fichero>` de **gpg**. En este caso, el **fichero** que debemos introducir es el que acabamos de recibir, es decir, **alejandro_gutierrez.asc**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --import alejandro_gutierrez.asc
gpg: clave C3B291882C4EE5DF: clave pública "Alejandro Gutierrez Valencia <tojandro@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
{% endhighlight %}

En este caso, nos ha devuelto un mensaje por pantalla informando que las claves públicas de Alejandro han sido correctamente importadas.

### 3. Comprueba que las claves se han incluido correctamente en vuestro keyring.

De nuevo, volveremos a hacer uso de opción `--list-keys` de **gpg** (ya que no tendría sentido usar la opción `--list-secret-keys`, pues lo que hemos importado son claves públicas, no privadas) para verificar que las claves públicas han sido correctamente importadas a nuestro keyring:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa3072 2020-10-08 [SC] [caduca: 2020-11-07]
      443D661D9AAF3ABAEDCA93E1C3B291882C4EE5DF
uid        [desconocida] Alejandro Gutierrez Valencia <tojandro@gmail.com>
sub   rsa3072 2020-10-08 [E] [caduca: 2020-11-07]
{% endhighlight %}

Efectivamente, las claves públicas de Alejandro han sido correctamente importadas a nuestro keyring en **.gnupg/pubring.kbx**.

## Tarea 3: Cifrado asimétrico con claves públicas (gpg).

### 1. Cifraremos un archivo cualquiera y lo remitiremos por email a uno de nuestros compañeros que nos proporcionó su clave pública.

En este caso, he descargado un fichero .pdf de nombre **Apuntes.pdf** que es el que voy a encriptar con la clave pública de mi compañero para posteriormente enviárselo. Para cifrar haremos uso de la opción `-e` de **gpg** junto con la opción `-u <remitente>` y la opción `-r <destinatario>`, para posteriormente indicar el fichero a cifrar. En este caso, el **remitente** sería yo y el **destinatario**, Alejandro, de manera que introduciendo su nombre como destinatario, buscará entre las claves públicas existentes y usará la suya, para que no haya confusión. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg -e -u "Álvaro Vaca Ferreras" -r "Alejandro Gutierrez Valencia" Apuntes.pdf
gpg: 3C5DBE21F6961E37: No hay seguridad de que esta clave pertenezca realmente
al usuario que se nombra

sub  rsa3072/3C5DBE21F6961E37 2020-10-08 Alejandro Gutierrez Valencia <tojandro@gmail.com>
 Huella clave primaria: 443D 661D 9AAF 3ABA EDCA  93E1 C3B2 9188 2C4E E5DF
      Huella de subclave: 224B 7529 F826 29C0 6D70  B6A5 3C5D BE21 F696 1E37

No es seguro que la clave pertenezca a la persona que se nombra en el
identificador de usuario. Si *realmente* sabe lo que está haciendo,
puede contestar sí a la siguiente pregunta.

¿Usar esta clave de todas formas? (s/N) s
{% endhighlight %}

Tras mostrarnos una advertencia de seguridad y confirmar que queremos usar dicha clave, el fichero ya se encontrará cifrado, por lo que vamos a proceder a ejecutar el comando `ls -l | grep Apuntes` para asegurarnos de que así ha sido.

{% highlight shell %}
alvaro@debian:~$ ls -l | grep Apuntes
-rw-r--r-- 1 alvaro alvaro 3497882 oct  8 17:30  Apuntes.pdf
-rw-r--r-- 1 alvaro alvaro 3449802 oct  8 17:41  Apuntes.pdf.gpg
{% endhighlight %}

Como se puede apreciar ahora tenemos dos ficheros, el original (**Apuntes.pdf**) y el cifrado (**Apuntes.pdf.gpg**), que cuenta con la extensión **.gpg**.

Tras ello, le enviaré dicho fichero a Alejandro por correo electrónico:

![envio3](https://i.ibb.co/Zf0tX1y/3-1.png "Envío de ficheros")

### 2. Nuestro compañero, a su vez, nos remitirá un archivo cifrado para que nosotros lo descifremos.

En este caso, he recibido un fichero cifrado de nombre **Triggers_1.pdf.gpg** por parte de Alejandro a través del correo electrónico:

![envio4](https://i.ibb.co/xh4fbxh/3-2.png "Envío de ficheros")

### 3. Tanto nosotros como nuestro compañero comprobaremos que hemos podido descifrar los mensajes recibidos respectivamente.

Para descifrar haremos uso de la opción `-d` de **gpg**, para posteriormente indicar el fichero a descifrar. En este caso, el fichero a descifrar sería el que acabamos de recibir por parte de Alejandro, es decir, **Triggers_1.pdf.gpg**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg -d Triggers_1.pdf.gpg
{% endhighlight %}

Tras ejecutar el comando, se nos pedirá la frase de paso para desbloquear la clave privada con la que vamos a descifrar el fichero (pues anteriormente fue cifrado con la clave pública asociada a dicha clave privada):

![decrypt1](https://i.ibb.co/sKVHxn8/3-3.png "Frase de paso")

{% highlight shell %}
alvaro@debian:~$ gpg -d Triggers_1.pdf.gpg
�JC8�a��G���	#��`v`�)�L�s	k�:��a9�j�PT�b@eC���]�)��P��Ŧ��c"H^�L���ͨ1&P��*�D��^}�	X#�Ԅ@��f�H�' $9��x~v�ߟ�$I����/Y���e�mNˁ�S�?=�'��S,�H�;���wC��1�
cv���4f��1�0Z��hIj1��f�F>�x;��2�.�o&O����O��=ի[� Ey����HQ����#�Y�jtYg��Dz�,r�Q��4j}u�_�qO*�ߕ����������"h�.,��<�ԃU���D~���̋�l�-v��bG���l")��=2�s5��\���.���2ɯ�*
...
e�X�ޒ�*��y���i�u��+�>ǈ���s��-�Q���iLΘ⚒���9u��s��J�s�/-�,y����(t������-�z$L�,�$s)3�QC��沤*D�N�.7jS���re���)ܜ"�oP
$K���/���M��l��^�y�M���3�c�mL�}Ψ_��z6�V�
                                        |�a[��������(7fR��Iqb�<j�!����5���'s�8띜ӤFNU7��hp2s�3�
{% endhighlight %}

Como se puede apreciar, el contenido desencriptado es un poco extraño. Esto se debe a que el contenido no está en **texto plano**, sino que es un **.pdf**. Para visualizarlo correctamente, volveremos a ejecutar el comando pero esta vez, redirigiendo la salida a un fichero de formato .pdf (por ejemplo, a uno de nombre **desencriptado.pdf**), para posteriormente abrirlo haciendo doble click, consiguiendo así que use la aplicación de Adobe para su correcta visualización.

{% highlight shell %}
alvaro@debian:~$ gpg -d Triggers_1.pdf.gpg > desencriptado.pdf
gpg: cifrado con clave de 3072 bits RSA, ID 94A4BCC489CBC9DA, creada el 2020-10-07
      "Álvaro Vaca Ferreras <avacaferreras@gmail.com>"
{% endhighlight %}

![decrypt2](https://i.ibb.co/DWDczQQ/3-3-1.png "Contenido desencriptado")

Como se puede apreciar, el contenido es ahora totalmente visible. En el caso de Alejandro, también ha podido desencriptar y visualizar el contenido correctamente.

### 4. Por último, enviaremos el documento cifrado a alguien que no estaba en la lista de destinatarios y comprobaremos que este usuario no podrá descifrar este archivo.

Para este punto, he solicitado ayuda a **Francisco Javier Martín**, al cuál envié dicho fichero cifrado (**Apuntes.pdf.gpg**) y al tratar de descifrarlo, obtuvo el siguiente error por pantalla:

{% highlight shell %}
francisco@debian10:~/Descargas$ gpg -d Apuntes.pdf.gpg
gpg: cifrado con clave RSA, ID 3C5DBE21F6961E37
gpg: descifrado fallido: No secret key
{% endhighlight %}

Como se puede apreciar, el descifrado ha fallado dado que no ha encontrado ninguna clave privada para descifrar dicho fichero.

### 5. Para terminar, indica los comandos necesarios para borrar las claves públicas y privadas que posees.

Este ejercicio lo he dejado para el final ya que quería mostrar cómo se lleva a cabo la eliminación de dichas claves, no indicar simplemente los comandos a usar. Es importante mencionar que en caso de que no vayamos a usar más un par de claves, sería recomendable [revocarlo](https://jorgegarciadev.gitlab.io/post/revocar_una_clave_gpg/) primero, pero en este caso vamos a aprender a eliminarlo únicamente.

Es necesario seguir un determinado orden a la hora de llevar a cabo la eliminación de dicho par de claves, ya que de lo contrario, nos devolverá un error. Primero hay que eliminar la clave **privada** y posteriormente, la clave **pública**.

Para eliminar la clave privada haremos uso de la opción `--delete-secret-key <nombre>` de **gpg**. En este caso, el **nombre** que debemos introducir es el mismo que hemos introducido a la hora de generar el par de claves. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --delete-secret-key "Álvaro Vaca Ferreras"
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


sec  rsa3072/CC02797F092855F6 2020-10-07 Álvaro Vaca Ferreras <avacaferreras@gmail.com>

¿Eliminar esta clave del anillo? (s/N) s
¡Es una clave secreta! ¿Eliminar realmente? (s/N) s
{% endhighlight %}

Tras un par de confirmaciones, nuestra clave privada se encuentra eliminada, así que por último, vamos a eliminar la clave pública. Para ello, haremos uso de la opción `--delete-key <nombre>` de **gpg**. En este caso, el **nombre** que debemos introducir es el mismo que hemos introducido a la hora de generar el par de claves. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --delete-key "Álvaro Vaca Ferreras"
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


pub  rsa3072/CC02797F092855F6 2020-10-07 Álvaro Vaca Ferreras <avacaferreras@gmail.com>

¿Eliminar esta clave del anillo? (s/N) s
{% endhighlight %}

Listo. Nuestro par de claves ya se encuentra eliminado, pero para verificarlo, volveremos a ejecutar los comandos:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-08 [SC] [caduca: 2020-11-07]
      443D661D9AAF3ABAEDCA93E1C3B291882C4EE5DF
uid        [desconocida] Alejandro Gutierrez Valencia <tojandro@gmail.com>
sub   rsa3072 2020-10-08 [E] [caduca: 2020-11-07]

alvaro@debian:~$ gpg --list-secret-keys
{% endhighlight %}

Como se puede apreciar, no ha quedado ningún rastro de nuestro par de claves, la única clave existente actualmente en nuestra máquina es la clave pública de Alejandro.

## Tarea 4: Exportar clave a un servidor público de claves PGP (gpg).

### 1. Genera la clave de revocación de tu clave pública para utilizarla en caso de que haya problemas.

En realidad, **gpg** ha generado una clave de revocación de forma automática cuando se generó el par de claves, pero para mostrarlo, vamos a generar uno nuevo. Para ello, haremos uso de la opción `--gen-revoke <ID>` de **gpg**. En este caso, el **ID** que debemos introducir es aquel identificador (comunmente conocido como _fingerprint_) de 40 dígitos que encontramos al ejecutar el comando `gpg --list-keys`, aunque si especificamos los últimos 8 dígitos del mismo también sería válido. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --gen-revoke A0BE5CA7A9DC70AD3D619467CC02797F092855F6

sec  rsa3072/CC02797F092855F6 2020-10-07 Álvaro Vaca Ferreras <avacaferreras@gmail.com>

¿Crear un certificado de revocación para esta clave? (s/N) s
Por favor elija una razón para la revocación:
  0 = No se dio ninguna razón
  1 = La clave ha sido comprometida
  2 = La clave ha sido reemplazada
  3 = La clave ya no está en uso
  Q = Cancelar
(Probablemente quería seleccionar 1 aquí)
¿Su decisión?
{% endhighlight %}

En este caso, nos está pidiendo un motivo para generar el certificado de revocación. En este caso, voy a introducir la opción **0**.

{% highlight shell %}
¿Su decisión? 0
Introduzca una descripción opcional; acábela con una línea vacía:
>
{% endhighlight %}

Además, nos pedirá una descripción opcional. En este caso, voy a introducir **Práctica de criptografía** como descripción.

{% highlight shell %}
> Práctica de criptografía.
> 
Razón para la revocación: No se dio ninguna razón
Práctica de criptografía.
¿Es correcto? (s/N)
{% endhighlight %}

Por último, confirmaremos la generación del certificado de revocación introduciendo una **s** y la frase de paso y el certificado se generará.

{% highlight shell %}
¿Es correcto? (s/N) s
se fuerza salida con armadura ASCII.
-----BEGIN PGP PUBLIC KEY BLOCK-----
Comment: This is a revocation certificate

iQHRBCABCgA7FiEEoL5cp6nccK09YZRnzAJ5fwkoVfYFAl9/OiYdHQBQcsOhY3Rp
Y2EgZGUgY3JpcHRvZ3JhZsOtYS4ACgkQzAJ5fwkoVfZxxQwAtV3pH+CyoNCbcGUV
7lQp7ajpJx+oQ6Lv8fzBaapiwz45yXCBH0+6B1YZCf5sfxQ/Ae28piFxbYBhCUMs
r7NSFyZsRp12mRvTB0nTWvyZfp/3u/akcrwdgIQgrwB0YAutrb14cVXhV0u5NKiM
YPBnCBJeZeKG620Z6f8ECheDV4rXe3azxIIcO9hv/F3gEEhLqibMq+32dbpkT+Jt
cbrtviWBt6K072ZPyJD1aQNECVxrHygtWTqynrgAbmE+1CG3tpukp7baQTfOqLRb
PI8YYjke/zOsJ3pjP+sXYOMnyn/MBR/ucyR9yT0X9D4tXQZWDApPW8M0yHqoPRtX
C2by4Q+uoaVBInbpaOxvLZerY8ytkZbzAr+Lh59H3tfCdMmV3XeBrGGl0YYSQrO7
uVVFZCotBwm9tjcTAPv1iKgha1vqaFOJW0JI6wYckqiShNIjRzzXVVfygdCKUv+x
lY6Z98vB0NkgZqkuznOzX4mR2pwRMjiybh8ayjIGerYfiR26
=B7ll
-----END PGP PUBLIC KEY BLOCK-----
Certificado de revocación creado.

Por favor consérvelo en un medio que pueda esconder; si alguien consigue
acceso a este certificado puede usarlo para inutilizar su clave.
Es inteligente imprimir este certificado y guardarlo en otro lugar, por
si acaso su medio resulta imposible de leer. Pero precaución: ¡el sistema
de impresión de su máquina podría almacenar los datos y hacerlos accesibles
a otras personas!
{% endhighlight %}

El certificado de revocación es aquello contenido entre **-----BEGIN PGP PUBLIC KEY BLOCK-----** y **-----END PGP PUBLIC KEY BLOCK-----**, por lo que es necesario guardarlo en un lugar seguro por si en algún momento necesitásemos revocar nuestro par de claves (soy consciente de que estoy mostrando el certificado, pero no es algo relevante, pues el par de claves fue generado únicamente para la práctica y ya ha sido revocado).

### 2. Exporta tu clave pública al servidor pgp.rediris.es.

Para exportar la clave pública a un servidor de claves públicas haremos uso de la opción `--keyserver <servidor>` de **gpg** junto con la opción `--send-key <ID>`. En este caso, el **servidor** que debemos introducir es **pgp.rediris.es** y el **ID** es aquel identificador (_fingerprint_) de 40 dígitos que encontramos al ejecutar el comando `gpg --list-keys`, aunque si especificamos los últimos 8 dígitos del mismo también sería válido. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver pgp.rediris.es --send-key A0BE5CA7A9DC70AD3D619467CC02797F092855F6
gpg: enviando clave CC02797F092855F6 a hkp://pgp.rediris.es
{% endhighlight %}

Nuestra clave ya ha sido enviada al servidor de claves públicas **pgp.rediris.es**, pero para verificarlo, vamos a acceder al mismo mediante el navegador. Dentro, encontraremos un apartado para buscar claves públicas introduciendo un **string**, que puede ser nuestro nombre o nuestro correo electrónico, por ejemplo. En este caso, voy a introducir mi correo electrónico:

![servidor1](https://i.ibb.co/gSP3BWz/4-2.png "Servidor de claves públicas")

Tras ello, pulsaremos en "**Search for a key**" para ver los resultados y así verificar que la clave pública se encuentra correctamente subida:

![servidor2](https://i.ibb.co/55Kj3P6/4-2-1.png "Servidor de claves públicas")

Como se puede apreciar, ha aparecido una coincidencia referente a la clave que acabamos de subir, por lo que podemos concluir que la subida se ha completado satisfactoriamente.

### 3. Borra la clave pública de alguno de tus compañeros de clase e impórtala ahora del servidor público de rediris.

Tal y como hemos visto en uno de los ejercicios anteriores, para eliminar una clave pública de nuestro keyring, haremos uso de la opción `--delete-key <nombre>` de **gpg**. En este caso, el **nombre** que debemos introducir es el de Alejandro. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --delete-key "Alejandro Gutierrez Valencia"
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


pub  rsa3072/C3B291882C4EE5DF 2020-10-08 Alejandro Gutierrez Valencia <tojandro@gmail.com>

¿Eliminar esta clave del anillo? (s/N) s
{% endhighlight %}

Listo. La clave pública de Alejandro ya se encuentra eliminada, pero para verificarlo, volveremos a ejecutar el comando:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]
{% endhighlight %}

Como se puede apreciar, no ha quedado ningún rastro de su clave pública.

Una vez que la clave pública ha sido totalmente eliminada, es hora de volver a importarla, pero esta vez, desde el servidor de claves públicas. Para ello, haremos uso de la opción `--keyserver <servidor>` de **gpg** junto con la opción `--recv-keys <ID>`. En este caso, el **servidor** que debemos introducir es **pgp.rediris.es** y el **ID** es aquel identificador (_fingerprint_) de 40 dígitos de la clave de Alejandro, aunque si especificamos los últimos 8 dígitos del mismo también sería válido. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ gpg --keyserver pgp.rediris.es --recv-keys 443D661D9AAF3ABAEDCA93E1C3B291882C4EE5DF
gpg: clave C3B291882C4EE5DF: clave pública "Alejandro Gutierrez Valencia <tojandro@gmail.com>" importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
{% endhighlight %}

Al parecer, la clave pública de Alejandro ha sido importada de nuevo, por lo que volveremos a listar las claves públicas existentes en nuestro keyring para verificar que así ha sido:

{% highlight shell %}
alvaro@debian:~$ gpg --list-keys
/home/alvaro/.gnupg/pubring.kbx
-------------------------------
pub   rsa3072 2020-10-07 [SC] [caduca: 2022-10-07]
      A0BE5CA7A9DC70AD3D619467CC02797F092855F6
uid        [  absoluta ] Álvaro Vaca Ferreras <avacaferreras@gmail.com>
sub   rsa3072 2020-10-07 [E] [caduca: 2022-10-07]

pub   rsa3072 2020-10-08 [SC] [caduca: 2020-11-07]
      443D661D9AAF3ABAEDCA93E1C3B291882C4EE5DF
uid        [desconocida] Alejandro Gutierrez Valencia <tojandro@gmail.com>
sub   rsa3072 2020-10-08 [E] [caduca: 2020-11-07]
{% endhighlight %}

Efectivamente, la clave de Alejandro ha vuelto a ser importada satisfactoriamente a nuestro keyring y ya podemos volver a hacer uso de la misma para cifrar ficheros.

## Tarea 5: Cifrado asimétrico con openssl.

### 1. Genera un par de claves (pública y privada).

Para generar el par de claves vamos a hacer uso del algoritmo RSA, por lo que tendremos que indicar la opción `genrsa`. Además, vamos a hacer que la clave privada tenga que ser desbloqueada haciendo uso de una frase de paso, que utilizará el algoritmo AES128, por lo que tendremos que indicar la opción `-aes128`. Este par de claves se generará en un único fichero **.pem**, por lo que para indicar dicho fichero, haremos uso de la opción `-out <fichero>`. En este caso, el **fichero** de salida será **key.pem**. Por último, especificaremos el tamaño de la clave, que es recomendable que sea de al menos `2048` bits. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ sudo openssl genrsa -aes128 -out key.pem 2048
[sudo] password for alvaro: 
Generating RSA private key, 2048 bit long modulus (2 primes)
..................+++++
...............+++++
e is 65537 (0x010001)
Enter pass phrase for key.pem:
Verifying - Enter pass phrase for key.pem:
{% endhighlight %}

Tras habernos preguntado en dos ocasiones la frase de paso que queremos utilizar, el par de claves habrá sido generado en un fichero **.pem**, pero para verificarlo, vamos a listar el contenido del directorio actual con el comando `ls`:

{% highlight shell %}
alvaro@debian:~$ ls
 Descargas    Escritorio   Imágenes   Música       Público   Vídeos            virtualenv
 Documentos   GitHub       key.pem    Plantillas   vagrant  'VirtualBox VMs'
{% endhighlight %}

Efectivamente, así ha sido. El par de claves se ha generado correctamente con nombre **key.pem**.

### 2. Envía tu clave pública a un compañero.

Dado que **openssl** genera el par de claves (tanto privada como pública) en un mismo fichero, tendremos que llevar a cabo la extracción de la clave pública para enviársela a nuestro compañero, ya que no es un procedimiento seguro el enviar el fichero directamente. Para ello, haremos uso de la opción `-in <pardeclaves>` para indicar el par de claves del que queremos extraer la clave pública, junto con la opción `-pubout` para indicar que extraiga la clave pública y la opción `-out <ficherosalida>` para extraer la clave a un fichero **.public.pem**. Además, tendremos que indicar el algoritmo, en este caso, `rsa`. En este caso, el **pardeclaves** será **key.pem** y el **ficherosalida** será **key.public.pem**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ sudo openssl rsa -in key.pem -pubout -out key.public.pem
Enter pass phrase for key.pem:
writing RSA key
{% endhighlight %}

Nos ha solicitado la frase de paso y a continuación ha generado el fichero con la clave pública. Para verificarlo, vamos a leer el contenido haciendo uso del comando `cat <fichero>`:

{% highlight shell %}
alvaro@debian:~$ cat key.public.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxYom3ubmu4VAzf9an0j4
MDWRS1X51k8L/Zak59RyHePfz3wiu1wzcOS0NAbUJ8uOncbZPKl8wxmuskqtDhZf
+6AJPC9Ai5b/lPQHE0tph0+dQxFFYsVw1ETLYJyA6KewOCbS0brm5bJZjHz3QiQG
ISWkEEmgryQnmGI7jEcIo327Pl5Yy55kD6aWYOjsWtr+vC4r0jn7FMTsn5o1v7ZX
zBF5tB8/EFawN258j0oifhCiQXibwFGIrtCNWXCZpaZIIX20h+3mDrXMOVEQobM2
eZ9Kt4QaZgeM5K7dqKzn5g2ae83Moo5Y+DFyHXbgAiNJKzg606TsgO+rXRUFMTBz
mQIDAQAB
-----END PUBLIC KEY-----
{% endhighlight %}

Efectivamente, así ha sido. La clave pública ha sido correctamente extraída, así que tras ello, le he enviado la clave pública a Alejandro a través de Discord:

![envio5](https://i.ibb.co/MppD2MK/5-2-2.png "Envío de clave")

De la misma forma, Alejandro también me ha enviado su clave pública a través de Discord, con nombre **clave.public.pem**:

![envio6](https://i.ibb.co/qRLJSfK/5-2-1.png "Envío de clave")

### 3. Utilizando la clave pública cifra un fichero de texto y envíalo a tu compañero.

Lo primero que tendremos que hacer será generar un fichero de texto con un determinado contenido que deseemos. En este caso, voy a hacer uso de `echo` para redirigir la salida a un fichero de texto (por ejemplo, a uno de nombre **fichero.txt**). El comando a ejecutar sería:

{% highlight shell %}
echo "Esto es un fichero cifrado." > fichero.txt
{% endhighlight %}

Tras ello, podremos proceder a cifrar dicho fichero. Para ello, haremos uso de la opción `-encrypt` junto con la opción `-in <fichero>` para indicar el fichero a encriptar, la opción `-out <ficherosalida>` para indicar el fichero de salida de la encriptación (**.enc**), la opción `-inkey <clavepublica>` para indicar la clave pública con la que cifrar y la opción `-pubin` para indicar que vamos a firmar con una clave pública. Además, tenemos que indicar que vamos a usar RSA para firmar, verificar, cifrar o descifrar, por lo que tendremos que introducir la opción `rsautl`. En este caso, el fichero de entrada sería **fichero.txt**, el fichero de salida será **fichero.enc** y la **clavepublica** será la de Alejandro, es decir, **clave.public.pem**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ openssl rsautl -encrypt -in fichero.txt -out fichero.enc -inkey clave.public.pem -pubin
{% endhighlight %}

Para verificar que el fichero se ha cifrado correctamente, vamos a ejecutar de nuevo el comando `ls` para ver si se encuentra creado en nuestro directorio actual:

{% highlight shell %}
alvaro@debian:~$ ls
 clave.public.pem   Escritorio    GitHub     key.public.pem   Público  'VirtualBox VMs'
 Descargas          fichero.enc   Imágenes   Música           vagrant   virtualenv
 Documentos         fichero.txt   key.pem    Plantillas       Vídeos
{% endhighlight %}

Efectivamente, el fichero de nombre **fichero.enc** se encuentra correctamente generado, así que es hora mandarlo a Alejandro para que lo desencripte y lea su contenido. En este caso, el envío lo he realizado a través de Discord:

![envio7](https://i.ibb.co/Zc3rzJk/5-3-1.png "Envío de fichero")

De la misma forma, Alejandro también me ha enviado su fichero encriptado a través de Discord, con nombre **secreto.enc**:

![envio8](https://i.ibb.co/nL0gP5b/5-3-2.png "Envío de fichero")

### 4. Tu compañero te ha mandado un fichero cifrado, muestra el proceso para el descifrado.

Para descifrar un fichero cifrado, haremos uso de la opción `-decrypt` junto con la opción `-in <fichero>` para indicar el fichero a desencriptar, la opción `-out <ficherosalida>` para indicar el fichero de salida de la desencriptación y la opción `-inkey <claveprivada>` para indicar la clave privada con la que descifrar. Además, tenemos que indicar que vamos a usar RSA para firmar, verificar, cifrar o descifrar, por lo que tendremos que introducir la opción `rsautl`. En este caso, el fichero de entrada sería **secreto.enc**, el fichero de salida será **secreto.txt** y la **claveprivada** será la que hemos generado anteriormente, es decir, **key.pem**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ sudo openssl rsautl -decrypt -in secreto.enc -out secreto.txt -inkey key.pem
{% endhighlight %}

Para verificar que el fichero se ha desencriptado correctamente, vamos a leer el contenido del nuevo fichero generado, haciendo uso del comando `cat <fichero>`:

{% highlight shell %}
alvaro@debian:~$ cat secreto.txt 
Prueba de fichero encriptado.
{% endhighlight %}

Efectivamente, el fichero ha sido correctamente desencriptado y podemos leer su contenido.