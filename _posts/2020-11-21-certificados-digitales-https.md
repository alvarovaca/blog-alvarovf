---
layout: post
title:  "Certificados digitales y HTTPS"
banner: "/assets/images/banners/certdig.jpg"
date:   2020-11-21 22:14:00 +0200
categories: seguridad
---
Se recomienda realizar una previa lectura del _post_ [Integridad, firmas y autenticación](http://www.alvarovf.com/seguridad/2020/11/09/integridad-firmas-autenticacion-gpg.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

## Certificado digital de persona física

Hasta ahora, para solventar el problema de la confianza existente en el mundo de la criptografía asimétrica hemos hecho uso de los **anillos de confianza**, pero conforme el anillo va creciendo, la técnica va perdiendo validez, por lo que recurrimos a los **certificados digitales**, en los cuales existe una entidad certificadora en la que todos confiamos (pues no tenemos más remedio), en este caso, la _Fábrica Nacional de Moneda y Timbre_ (**FNMT**).

Al fin y al cabo, un certificado digital es un documento que contiene información de una persona o entidad, su clave pública y la firma digital de una _Autoridad de Certificación_. Permite firmar gestiones electrónicamente a través de Internet (y posteriormente comprobar dicha firma, pues la clave pública se adjunta al fichero firmado), como por ejemplo los correos electrónicos, e incluso autenticarnos en algunos sitios que permitan dicho mecanismo, como veremos a continuación. Se almacena en un fichero (__\*.pfx__, __\*.cer__) o en una tarjeta inteligente (como el **DNIe**) y se instala en el navegador web, desde donde podremos importarlo o exportarlo según nuestras necesidades.

Todo suena muy bonito, pero tiene una gran desventaja, y es que es un método centralizado, es decir, dicha entidad certificadora tiene un par de claves pública/privada y en el hipotético caso de que perdiese la clave privada, habría un gran problema.

Para trabajar con certificados digitales necesitaremos una infraestructura, principalmente compuesta por:

* **Autoridad de certificación (_CA_)**: Vincula una clave pública a una entidad particular (servidor, persona, equipo...), pasando por un previo proceso de identificación. Realmente, lo que hace es firmar unos certificados subordinados pertenecientes al siguiente eslabón de la cadena, la _Autoridad de Registro (RA)_. Es el caso de la _FNMT_.
* **Autoridad de registro (_RA_)**: Son aquellas entidades autorizadas a firmar los certificados de suscriptor (finales), de manera que firmarán la clave pública del usuario con su clave privada, otorgándole por lo tanto una validez, firma que posteriormente será posible comprobar haciendo uso de la clave pública de la _CA_. Es el caso de los _Ayuntamientos_, las _oficinas de Hacienda_... es decir, cualquier lugar en el que puedas validar tu identidad para recibir el certificado digital.
* **Software necesario**: El navegador en el que instalaremos el certificado digital. Puede ser _Google Chrome_, _Firefox_, _Internet Explorer_...
* **Seguridad telemática**: Protocolo HTTPS. Lo veremos con más detalle a continuación.

El resultado del uso de dichos certificados digitales tendremos que someterlos también a un proceso de validación, en el que se comprueba la identidad del firmante, la integridad del documento firmado y la validez temporal del certificado utilizado. Las dos primeras las podemos comprobar sin necesidad de conexión a Internet, pero no podemos saber si el certificado es válido, si estaba revocado en el momento de la firma o si la autoridad que lo remitió es de confianza. Dicha verificación o validación podemos llevarla a cabo a través de un [servicio](https://www.sede.fnmt.gob.es/certificados/persona-fisica/verificar-estado) que proporciona la autoridad que ha emitido el certificado.

### Tarea 1: Obtención e instalación del certificado

El proceso de obtención del certificado digital es bastante sencillo, y se compone de los siguientes pasos:

* Antes de solicitar el certificado tendremos que instalar el [software](https://www.sede.fnmt.gob.es/certificados/persona-fisica/obtener-certificado-software/configuracion-previa) necesario para la generación de claves. Es muy importante que durante todo el proceso se haga uso del mismo equipo y usuario, y obviamente, no formatear la máquina hasta que haya finalizado el proceso en su totalidad.
* Tras ello, tendremos que solicitar el certificado, rellenando para ello el correspondiente [formulario](https://www.sede.fnmt.gob.es/certificados/persona-fisica/obtener-certificado-software/solicitar-certificado), en el que se solicitará determinada información, como el número de documento de identificación, primer apellido y correo electrónico. Durante la solicitud, se pedirá una contraseña para así proteger las claves del certificado durante el proceso. Una vez solicitado el certificado, se recibirá un correo electrónico a los pocos minutos indicando el _Código de Solicitud_, que necesitaremos para el siguiente paso.
* El tercer paso consiste en dirigirnos físicamente a una _Autoridad de registro_ para acreditar nuestra identidad, haciendo uso del _Código de Solicitud_ obtenido en el paso anterior y de un documento que nos identifique, como puede ser el _Documento Nacional de Identidad_ (DNI), pasaporte o carné de conducir. [Aquí](http://mapaoficinascert.appspot.com/) se puede consultar las oficinas más cercanas que ofrezcan dicho servicio.
* Una vez acreditada nuestra identidad, tendremos que esperar unos minutos (en mi caso, en menos de dos minutos ya estaba disponible) para proceder a descargar e instalar nuestro certificado. El proceso es muy intuitivo, simplemente tendremos que rellenar el siguiente [formulario](https://www.sede.fnmt.gob.es/certificados/persona-fisica/obtener-certificado-software/descargar-certificado) y aceptar la instalación del certificado en el navegador, solicitándose para ello la contraseña previamente configurada para proteger las claves.

Por último, para verificar que el certificado ha sido correctamente instalado, podremos dirigirnos en el caso de _Google Chrome_ al icono de los **3 puntos** en la barra superior para seguidamente pulsar en **Configuración**. Una vez ahí, accederemos al apartado **Privacidad y Seguridad** ubicado en la parte izquierda de la pantalla. Dentro del mismo, pulsaremos en **Seguridad** y acto seguido en **Gestionar certificados**. Se nos abrirá una nueva ventana, que tendrá el siguiente aspecto:

![certificado1](https://i.ibb.co/j5k0yZH/13.png "Instalación certificado")

Como se puede apreciar, tenemos instalado un certificado digital de tipo **Personal** emitido por **AC FNMT Usuarios**, cuya validez es de 4 años a partir del momento de expedición.

Ya tenemos el certificado en nuestra máquina, ¿pero qué ocurre si quisiésemos tenerlo también en nuestro teléfono móvil o en otro ordenador? Para ello, tendríamos que exportarlo (también conocido como crear una copia de seguridad). El proceso es también bastante sencillo, simplemente tendremos que marcar el que queramos exportar y pulsar en la opción **Exportar...**, siguiendo para ello los siguientes pasos:

* Cuando se abra el asistente, es muy importante marcar la opción de **_Exportar la clave privada_**, que por defecto no viene seleccionada, ya que de lo contrario, no podríamos llevar a cabo firmas digitales una vez importado en otro dispositivo, pues se exportaría únicamente la clave pública, la cuál podemos enviar a las personas para que encripten mensajes que únicamente podremos descifrar nosotros haciendo uso de la clave privada asociada. Vuelvo a hacer hincapié en este punto, ya que es el más importante.
* Tras ello, se nos pedirá el formato de archivo en el que se exportará el certificado digital. En éste caso, dado que contiene la clave privada, únicamente es posible exportarlo en formato **.pfx**. En mi caso, únicamente he dejado marcada la opción **_Incluir todos los certificados en la ruta de certificación (si es posible)_**.
* El siguiente paso consiste en asignar una contraseña para así preservar la seguridad de la clave privada tras la exportación, además de dejarnos elegir el algoritmo de cifrado de la misma, que en mi caso, he dejado el que viene por defecto (**_TripleDES-SHA1_**).
* El último paso consiste en indicar la ruta en la que se va a exportar el certificado digital, que es _a gusto del consumidor_.

### Tarea 2: Validación del certificado

Para validar el certificado que acabamos de instalar, vamos a hacer uso de la utilidad [VALIDe](https://valide.redsara.es/valide/) que nos proporciona el _Gobierno de España_. Para ello, accederemos al apartado **[Validar Certificado](https://valide.redsara.es/valide/validarCertificado/ejecutar.html)** y tras pulsar en **_Seleccionar Certificado_**, se nos abrirá una ventana emergente para seleccionar el certificado que queremos validar:

![valide1](https://i.ibb.co/0shK2P8/Captura-de-pantalla-de-2020-11-17-10-56-50.png "VALIDe")

En éste caso, seleccionare el único certificado existente. Tras ello, tendremos que introducir un _Captcha_ y por último, pulsar en **_Validar_**. En caso de que la validación haya sido exitosa, obtendremos el siguiente mensaje:

![valide2](https://i.ibb.co/mBQ4hWd/Captura-de-pantalla-de-2020-11-17-10-57-14.png "VALIDe")

Como se puede apreciar, el certificado es totalmente válido y se muestra la información referente al mismo.

A continuación, vamos a proceder a descargar una aplicación de escritorio llamada **AutoFirma**, que como su propio nombre indica, nos permitirá firmar electrónicamente. La página de descarga la podemos encontrar [aquí](https://firmaelectronica.gob.es/Home/Descargas.html), de manera que en éste caso, haré uso de `wget` para descargar la versión de [Linux](https://estaticos.redsara.es/comunes/autofirma/currentversion/AutoFirma_Linux.zip):

{% highlight shell %}
alvaro@debian:~$ wget https://estaticos.redsara.es/comunes/autofirma/currentversion/AutoFirma_Linux.zip
--2020-11-17 10:38:12--  https://estaticos.redsara.es/comunes/autofirma/currentversion/AutoFirma_Linux.zip
Resolviendo estaticos.redsara.es (estaticos.redsara.es)... 185.73.172.87
Conectando con estaticos.redsara.es (estaticos.redsara.es)[185.73.172.87]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 155509984 (148M) [application/zip]
Grabando a: “AutoFirma_Linux.zip”

AutoFirma_Linux.zip                100%[==========================>]  148,31M   829KB/s    en 3m 52s  

2020-11-17 10:42:04 (654 KB/s) - “AutoFirma_Linux.zip” guardado [155509984/155509984]
{% endhighlight %}

Para verificar que el comprimido se ha descargado correctamente, haremos uso del comando `ls -l`, estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'AutoFirma'
-rw-r--r-- 1 alvaro alvaro 155509984 sep 19  2019 AutoFirma_Linux.zip
{% endhighlight %}

Efectivamente, se ha descargado un paquete de nombre "**AutoFirma_Linux.zip**" con un peso total de **148.31 MB** (155509984 _bytes_).

Al estar comprimido el fichero, no podemos llevar a cabo la instalación hasta que no hagamos una extracción de los ficheros contenidos. Además, para liberar espacio, borraremos tras ello dicho fichero comprimido ya que no nos hará falta. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ unzip AutoFirma_Linux.zip && rm AutoFirma_Linux.zip
Archive:  AutoFirma_Linux.zip
  inflating: AF_manual_instalacion_usuarios_ES.pdf  
 extracting: autofirma-1.6.5-1.noarch.rpm  
 extracting: AutoFirma_1_6_5.deb     
  inflating: autofirma-1.6.5-1.noarch_SUSE.rpm
{% endhighlight %}

Como se puede apreciar en la salida del comando, se han descomprimido un total de 4 ficheros, pero en nuestro caso, únicamente necesitaremos el **.deb**, de manera que comprobaremos que ha sido correctamente extraído, listando para ello el contenido del directorio actual y estableciendo una vez más, un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'Auto.*\.deb'
-rw-r--r-- 1 alvaro alvaro  51131568 abr 11  2019 AutoFirma_1_6_5.deb
{% endhighlight %}

Efectivamente, el fichero **.deb** se ha descomprimido correctamente y está listo para su instalación, pero dicho software necesita _Java_ (**default-jdk**) y **libnss3-tools** para funcionar, así que procederemos a instalar dichos paquetes previamente, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo apt install default-jdk libnss3-tools
{% endhighlight %}

Una vez instaladas dichas dependencias, ya podremos proceder a instalar **AutoFirma**, haciendo para ello uso de `dpkg`:

{% highlight shell %}
alvaro@debian:~$ sudo dpkg -i AutoFirma_1_6_5.deb
{% endhighlight %}

Tras ello, podremos acceder a nuestro menú de aplicaciónes y buscar el nuevo software, de manera que tras abrirlo tendrá el siguiente aspecto:

![autofirma1](https://i.ibb.co/s67sfLT/Captura-de-pantalla-de-2020-11-17-10-55-16.png "AutoFirma")

Como se puede apreciar, la aplicación de escritorio **AutoFirma** ha sido correctamente instalada y se encuentra totalmente operativa.

### Tarea 3: Firma electrónica

Para ésta tarea, vamos a generar dos ficheros para posteriormente firmarlos con nuestro certificado digital:

* El primero de ellos, tendrá como nombre **ficherovalide.txt** y como su nombre indica, lo firmaremos con la herramienta online **VALIDe**.
* El segundo, tendrá nombre **ficheroautofirma.txt** y en ésta ocasión, firmaremos dicho fichero con la aplicación de escritorio **AutoFirma** que acabamos de instalar.

Para generar dichos ficheros haremos uso de los comandos:

{% highlight shell %}
alvaro@debian:~$ echo "Fichero firmado por VALIDe" > ficherovalide.txt
alvaro@debian:~$ echo "Fichero firmado por AutoFirma" > ficheroautofirma.txt
{% endhighlight %}

Para verificar que los ficheros han sido correctamente generados, vamos a listar el contenido del directorio actual, estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'fichero.*\.txt'
-rw-r--r-- 1 alvaro alvaro    30 nov 17 11:00 ficheroautofirma.txt
-rw-r--r-- 1 alvaro alvaro    27 nov 17 11:00 ficherovalide.txt
{% endhighlight %}

Efectivamente, ambos ficheros han sido generados, así que vamos a proceder a firmar el primero de ellos, por lo que accederemos a [VALIDe](https://valide.redsara.es/valide/), pulsando en ésta ocasión en el apartado **[Realizar Firma](https://valide.redsara.es/valide/firmar/ejecutar.html)**.

Dentro del mismo, pulsaremos en **_Firmar_** y acto seguido nos pedirá el fichero que deseamos firmar. En éste caso, tendremos que elegir el fichero **ficherovalide.txt**, indicando además el certificado que queremos utilizar para ello.

Finalmente, si todo ha funcionado correctamente obtendremos el siguiente mensaje informativo:

![valide3](https://i.ibb.co/zZykLKF/Captura-de-pantalla-de-2020-11-17-11-00-49.png "VALIDe")

Como se puede apreciar, se ha indicado que la firma se ha realizado correctamente, por lo que pulsaremos en **_Guardar Firma_** para así almacenar en nuestra máquina el fichero resultante de dicho proceso. En mi caso, he decidido asignarle el nombre **ficherovalide.txt_signed.csig**, para así diferenciarlo.

Bien, ya hemos firmado uno de los dos, así que para el siguiente, volveremos a la aplicación de escritorio **AutoFirma** y elegiremos en éste caso el fichero **ficheroautofirma.txt**. Tras ello, pulsaremos en **_Firmar_** y nos pedirá el certificado a usar para la firma, seguido de la ruta donde almacenar el fichero resultante, que en este caso, he decidido asignarle el nombre **ficheroautofirma.txt_signed.csig**.

Finalmente, si todo ha funcionado correctamente obtendremos el siguiente mensaje informativo:

![autofirma2](https://i.ibb.co/0hHNFT2/Captura-de-pantalla-de-2020-11-17-11-03-07.png "AutoFirma")

Por último, para verificar que los ficheros se han almacenado correctamente, volveremos a listar el contenido del directorio actual y a establecer un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'fichero.*\.csig'
-rw-r--r-- 1 alvaro alvaro  6157 nov 17 11:02 ficheroautofirma.txt_signed.csig
-rw-r--r-- 1 alvaro alvaro  4739 nov 17 11:03 ficherovalide.txt_signed.csig
{% endhighlight %}

Efectivamente, ambos ficheros resultantes del proceso de firmado se encuentran almacenados en el directorio actual.

Genial, los ficheros han sido firmados, pero tendremos que comprobar que dicha firma se puede validar, así que sería poco relevante tratar de validar mi propia firma, por lo que he enviado dichos ficheros a un compañero y él me ha enviado los suyos, resultantes de seguir exactamente el mismo proceso que nosotros.

De nuevo, vamos a comprobar que dichos ficheros se encuentran ubicados en el directorio actual, listando el contenido y estableciendo un filtro por nombre:

{% highlight shell %}
alvaro@debian:~$ ls -l | egrep 'javi'
-rw-r--r-- 1 alvaro alvaro  6177 nov 17 16:00 javiautofirma_signed.csig
-rw-r--r-- 1 alvaro alvaro  6174 nov 17 16:00 javivalide.txt_signed.csig
{% endhighlight %}

Ambos ficheros con extensión **.csig** se encuentran contenidos en el directorio actual. Como se puede apreciar, no disponemos de los ficheros originales ni de su clave pública, sino de aquellos resultantes del proceso de firmado, pues es lo único que necesitaremos, ya que contienen tanto el fichero original como la clave pública necesaria para comprobar la firma.

Para validar dichas firmas, vamos a acceder a [VALIDe](https://valide.redsara.es/valide/), pulsando en ésta ocasión en el apartado **[Validar Firma](https://valide.redsara.es/valide/validarFirma/ejecutar.html)**.

Dentro del mismo, seleccionaremos el fichero firmado que deseamos validar. En éste caso, vamos a elegir el fichero **javivalide.txt_signed.csig**, indicando además el _Captcha_ que se nos solicita y por último, pulsando en **_Validar_**.

Finalmente, si todo ha funcionado correctamente obtendremos el siguiente mensaje informativo:

![valide4](https://i.ibb.co/ng18FrN/Captura-de-pantalla-de-2020-11-17-16-03-39.png "VALIDe")

Como se puede apreciar, la firma ha sido correctamente validada, por lo que podemos asegurar que dicho fichero ha sido firmado por mi compañero, que no puede repudiar dicha acción. Repetiremos el mismo proceso para el fichero **javiautofirma_signed.csig**, obteniendo el siguiente resultado:

![valide5](https://i.ibb.co/pfvwL2z/Captura-de-pantalla-de-2020-11-17-16-04-09.png "VALIDe")

Efectivamente, éste fichero también ha podido ser validado correctamente, gracias a la clave pública contenida en los mismos, pero hasta ahora, no hemos visto el contenido de dichos ficheros, así que para ello, tendremos que acceder a [VALIDe](https://valide.redsara.es/valide/), pulsando en ésta ocasión en el apartado **[Visualizar Firma](https://valide.redsara.es/valide/visor/ejecutar.html)**.

Dentro del mismo, seleccionaremos el fichero firmado cuyo contenido queremos visualizar. En éste caso, vamos a elegir el fichero **javivalide.txt_signed.csig**, indicando además el _Captcha_ que se nos solicita y por último, pulsando en **_Visualizar_**.

Finalmente, si todo ha funcionado correctamente podremos visualizar el contenido del mismo:

![valide6](https://i.ibb.co/VS9gHBd/Captura-de-pantalla-de-2020-11-17-16-06-27.png "VALIDe")

Como se puede apreciar, hemos podido leer el contenido del fichero firmado sin ningún problema. Repetiremos el mismo proceso para el fichero **javiautofirma_signed.csig**, obteniendo el siguiente resultado:

![valide7](https://i.ibb.co/DbR6Myf/Captura-de-pantalla-de-2020-11-17-16-07-28.png "VALIDe")

Efectivamente, éste fichero también ha podido ser visualizado correctamente. La peculiaridad de ésto, es que en el margen izquierdo del **PDF** mostrado, podemos encontrar un mensaje que muestra la información de la persona que ha firmado dicho fichero, de manera que cuenta con toda la validez legal que ello conlleva:

![valide8](https://i.ibb.co/JnXqrXR/Captura-de-pantalla-de-2020-11-17-16-06-30.png "VALIDe")

Me gustaría mencionar que dichas comprobaciones las hemos podido llevar a cabo gracias a dos requisitos indispensables:

* Tenemos una conexión a Internet activa, ya que internamente, éste proceso de validación comprueba que el certificado de la persona firmante no haya sido revocado (información que se almacena en las denominadas _listas de revocación_), así como verificar que la autoridad firmante sea de confianza. Es decir, en caso de no tener conexión a Internet, podríamos validar la identidad del firmante y la integridad del documento firmado, pero no la validez temporal del certificado utilizado.
* Tenemos la clave pública de la persona firmante, pues se ha adjuntado en el propio fichero resultante del proceso de firmado, de manera que haremos uso de dicha clave para así verificar que la firma se ha realizado con la clave privada asociada.

Por último, hemos decidido hacer una prueba que es bastante común en una situación real, aquella en la que varias personas deben firmar un mismo fichero. En éste caso, mi compañero ha sido el primero en firmarlo, por lo que me ha enviado el fichero firmado de nombre **firmadodos_signed.csig**, que he procedido a firmar utilizando **AutoFirma**, proceso que no considero necesario volver a explicar, obteniendo por tanto el siguiente resultado:

![autofirma3](https://i.ibb.co/1GsCNz0/Captura-de-pantalla-de-2020-11-17-16-21-04.png "AutoFirma")

Tal y como se puede apreciar en la parte inferior, concretamente en el apartado **Árbol de firmas del documento**, aparecemos ambos, pero aun así, para verlo de una forma más clara, vamos a acceder a [VALIDe](https://valide.redsara.es/valide/), pulsando en ésta ocasión en el apartado **[Validar Firma](https://valide.redsara.es/valide/validarFirma/ejecutar.html)**, repitiendo el mismo proceso que anteriormente, para obtener así el siguiente resultado:

![valide9](https://i.ibb.co/X8q3Dcg/Captura-de-pantalla-de-2020-11-17-16-21-32.png "VALIDe")

Como se puede apreciar, ambas firmas son válidas y aunque éste ejemplo lo hemos hecho con dos personas, podría hacerse en una situación real con muchas más.

### Tarea 4: Autenticación

Tal y como hemos mencionado con anterioridad, la única utilidad de los certificados digitales no es la de firmar documentos electrónicamente, sino que también nos permite, haciendo uso de la clave privada, autenticarnos en aquellos sitios que admitan dicho mecanismo, proporcionando un nivel de validez superior a aquel que tendríamos con la común autenticación basada en usuario/contraseña.

En este caso, he accedido a la web de la [DGT](https://sede.dgt.gob.es/es/permisos-de-conducir/consulta-tus-puntos/) para consultar mis puntos del carnet. En la misma, podremos apreciar que tenemos varios métodos para acceder, pero para hacer uso del certificado digital, pulsaremos en **_Cl@ve_**. Seguidamente, pulsaremos en **_DNIe / Certificado electrónico_** y se abrirá una ventana emergente como la siguiente:

![dgt1](https://i.ibb.co/xhVLqws/Captura-de-pantalla-de-2020-11-17-11-05-25.png "DGT")

En dicha ventana emergente tendremos que seleccionar aquel certificado que queremos utilizar para la autenticación, así que en mi caso, seleccionaré el único disponible, de manera que si no ha habido ningún problema, nos habremos autenticado en la página de la _Dirección General de Tráfico_ sin tener que introducir ningún tipo de credenciales:

![dgt2](https://i.ibb.co/Jpf2qrB/Captura-de-pantalla-de-2020-11-17-11-05-31.png "DGT")

Como se puede apreciar, nos hemos autenticado correctamente y se ha mostrado la información referente a mi saldo actual de puntos.

## HTTPS / SSL

Basado en la filosofía que hemos visto hasta ahora, surge el _Secure Sockets Layer_ (**SSL**), un protocolo que proporciona la posibilidad de cifrar el contenido que transmitimos en cualquier aplicación basada en **TCP** (_HTTP_, _SMTP_...), es decir, ofrece seguridad en la capa de transporte.

Dicho protocolo se utiliza entre navegadores de Internet y servidores, e incluso entre servidores, es por ello que en aquellos sitios web que tienen dicha característica correctamente configurada, accedemos a través de **https://**.

Los servidores web tienen un certificado emitido por alguna autoridad de certificación de confianza. El certificado contiene la clave pública del servidor y es enviado al cliente en el momento de la petición.

Dicho certificado sirve para ponerse de acuerdo con el cliente de forma asimétrica de la clave de cifrado que van a utilizar para la comunicación, ya que sería muy costoso computacionalmente mantener una comunicación utilizando cifrado asimétrico. El cliente elegirá y cifrará dicha clave simétrica utilizando la clave pública del servidor. Una vez cifrada, se la enviará de vuelta al servidor, siendo éste el único capaz de descifrar dicha clave, pues es quién posee la clave privada asociada. A partir de éste momento, toda la sesión está cifrada simétricamente.

Además del cifrado de datos, otro de los usos de dicho certificado es el de autenticar al servidor, garantizando al usuario que aquel servidor al que se conecta es quién dice ser. Para ello, los navegadores web incluyen una lista de claves públicas de autoridades de certificación (_CA_) de confianza, de manera que gracias a la misma, se puede comprobar la firma del certificado enviado por el servidor. En caso de que no se pueda verificar dicha firma por parte de la _CA_, el navegador mostrará una advertencia al usuario, que podrá continuar bajo su propia responsabilidad, aunque la conexión seguirá siendo cifrada. Generalmente, se suele establecer, al igual que ocurría con los certificados digitales de personas físicas, una estructura jerárquica, pues posiblemente la autoridad certificadora que ha firmado dicho certificado, haya sido previamente firmada por otra de nivel superior.

### Tarea 1: Configurar HTTPS

Para este ejercicio vamos a trabajar con un compañero que creará y actuará como una entidad certificadora, que nos firmará un certificado para así poder configurar HTTPS en nuestro servidor. Lógicamente, en un caso práctico, ésta entidad certificadora no tendría ninguna validez, ya que dicha validez es proporcional a la reputación de la empresa que hay detrás, que en éste caso, es nula, pero es un tema un poco extenso que se sale del objetivo de éste artículo.

El primer paso, como es obvio, será instalar un servidor web, _apache2_ en éste caso, que trataremos de configurar para que utilice el protocolo HTTPS en sus conexiones. Para ello, me encuentro en una máquina virtual en el cloud, así que lo primero que haremos será actualizar su paquetería e instalar _apache2_, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# apt update && apt upgrade && apt install apache2
{% endhighlight %}

Bien, el servidor web se encuentra ya instalado, y como es bien sabido por todos, _apache2_ crea un VirtualHost por defecto cuyo DocumentRoot se encuentra en **/var/www/html**, así que lo aprovecharemos para no tener que crear otro, pues no será necesario. Lo único que tendremos que hacer será eliminar el fichero **index.html** que viene por defecto para así dejar el directorio vacío, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# rm /var/www/html/index.html
{% endhighlight %}

Es hora de modificar el VirtualHost por defecto de _apache2_ para hacer uso de un ServerName concreto, como puede ser **alvaro.iesgn.org**, para así simular una situación más real. Si todo esto suena un poco extraño, es porque todavía no has leído el artículo [VirtualHosting con Apache](http://www.alvarovf.com/servicios/2020/10/17/virtualhosting-con-apache.html). Para modificarlo, haremos uso del comando:

{% highlight shell %}
root@https:~# nano /etc/apache2/sites-available/000-default.conf
{% endhighlight %}

Dentro del mismo, tendremos que descomentar la línea **ServerName** e indicar el nombre de dominio a través del que queremos acceder al servidor, de manera que a la hora de comparar la cabecera **Host** existente en la petición HTTP, encontrará coincidencia con dicho VirtualHost y lo mostrará. En éste caso, no importaría, ya que no existen más VirtualHost para mostrar, pero vamos a hacerlo bien, quedando el siguiente resultado:

{% highlight shell %}
<VirtualHost *:80>
        ServerName alvaro.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
{% endhighlight %}

Tras ello, guardaremos los cambios y volveremos a cargar la configuración del servidor web, para que los cambios surtan efecto, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# systemctl reload apache2
{% endhighlight %}

El VirtualHost ya está totalmente operativo, pero todavía no existe ningún contenido a servir en el DocumentRoot, así que en mi caso, copiaré en el mismo una página web existente en una VPS contratada, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# scp -rp debian@vps.iesgn19.es:/srv/iesgn19/principal/* /var/www/html/
debian@vps.iesgn19.es's password: 
index.html                                 100% 1302    20.2KB/s   00:00    
info.php                                   100%   20     0.3KB/s   00:00    
bullet.png                                 100%  989    15.3KB/s   00:00    
style.css                                  100% 5814    83.8KB/s   00:00    
Thumbs.db                                  100% 8704   119.5KB/s   00:00    
transparent.png                            100%  199     3.1KB/s   00:00    
pattern.png                                100%  111     1.8KB/s   00:00
{% endhighlight %}

Donde:

* **-r**: Indicamos que la copia se haga de forma recursiva, para que en caso de existir directorios, los copie junto a su contenido.
* **-p**: Mantiene los atributos de los ficheros, tales como marcas temporales, permisos...

Antes de tratar de acceder a la web, tendremos que configurar en la máquina anfitriona la resolución estática de nombres, de manera que sea capaz de resolver **alvaro.iesgn.org** a la dirección IP de la máquina servidora y acceder por tanto, al VirtualHost que hemos configurado. Para ello, tendremos que modificar el fichero **/etc/hosts**, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ sudo nano /etc/hosts
{% endhighlight %}

Dentro del mismo, tendremos que añadir una línea de la forma **[IP] [Nombre]** para el nuevo sitio que hemos creado, siendo **[IP]** la dirección IP de la máquina servidora (en este caso, **172.22.200.186**) y **[Nombre]** el nombre de dominio que vamos a resolver posteriormente (en este caso, **alvaro.iesgn.org**), de manera que la configuración final sería la siguiente:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
172.22.200.186  alvaro.iesgn.org

::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Tras ello, ya estará todo listo para realizar la conexión. Para ello, accederemos desde el navegador web a **alvaro.iesgn.org**:

![https1](https://i.ibb.co/71tvdRB/Captura-de-pantalla-de-2020-11-17-16-45-41.png "HTTPS")

Como se puede apreciar, la página web se ha mostrado correctamente, por lo que podremos proceder a configurar **HTTPS** en nuestro servidor web _apache2_, ya que actualmente estamos accediendo a través de **HTTP**.

Lo primero que tendremos que hacer será crear una solicitud de firma de certificado (CSR o _Certificate Signing Request_). En este caso, vamos a hacerlo con **openssl**, pero se podría hacer con otras múltiples opciones de software.

Para crear una solicitud de firma de certificado, primero debemos tener una clave privada que se asociará al mismo, así que generaremos una clave privada RSA de 4096 bits, que será almacenada en **/etc/ssl/private/**, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# openssl genrsa 4096 > /etc/ssl/private/alvaro.key
Generating RSA private key, 4096 bit long modulus (2 primes)
............++++
......++++
e is 65537 (0x010001)
{% endhighlight %}

Una vez generada, cambiaremos los permisos de la misma a **400**, de manera que únicamente el propietario pueda leer el contenido, pues se trata de una clave privada. Éste paso no es obligatorio pero sí recomendable por seguridad. Para ello, haremos uso de `chmod`:

{% highlight shell %}
root@https:~# chmod 400 /etc/ssl/private/alvaro.key
{% endhighlight %}

Tras ello, crearemos un fichero **.csr** de solicitud de firma de certificado para que sea firmado por la autoridad certificadora (_CA_) creada por nuestro compañero. Dicho fichero no contiene información confidencial, así que no importará la ruta donde lo almacenemos ni los permisos asignados. En mi caso, lo almacenaré en el directorio actual, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# openssl req -new -key /etc/ssl/private/alvaro.key -out alvaro.csr
{% endhighlight %}

Donde:

* **-new**: Indicamos que la creación de la solicitud de firma de certificado sea interactiva, pues nos pedirá determinados parámetros.
* **-key**: Indicamos la clave privada a asociar a dicha solicitud de firma de certificado. En este caso, la generada en el paso anterior, **/etc/ssl/private/alvaro.key**.
* **-out**: Especificamos dónde se va a almacenar la solicitud de firma de certificado. En este caso, en el directorio actual, con nombre **alvaro.csr**.

Durante la ejecución, nos pedirá una serie de valores para identificar al certificado, que tendremos que rellenar de la siguiente manera:

{% highlight shell %}
Country Name (2 letter code) [AU]:ES
State or Province Name (full name) [Some-State]:Sevilla
Locality Name (eg, city) []:Dos Hermanas
Organization Name (eg, company) [Internet Widgits Pty Ltd]:JavierPerez Corp
Organizational Unit Name (eg, section) []:Informatica
Common Name (e.g. server FQDN or YOUR name) []:alvaro.iesgn.org
Email Address []:avacaferreras@gmail.com
{% endhighlight %}

Dicha información nos la tendrá que dar la autoridad certificadora, excepto los dos últimos valores, que en el caso del **Common Name**, he puesto el _FQDN_ de la web en la que queremos configurar HTTPS, en este caso, **alvaro.iesgn.org**. Además, tendremos que introducir nuestro correo electrónico en el apartado **Email Address**, en este caso, **avacaferreras@gmail.com**. Tras ello, se pedirán una serie de valores cuya introducción es opcional. En mi caso, no los rellené.

Para verificar que el fichero de solicitud de firma ha sido correctamente generado, listaremos el contenido del directorio actual, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# ls -l
total 12
-rw-r--r-- 1 root root 1789 Nov 17 18:39 alvaro.csr
{% endhighlight %}

Efectivamente, existe un fichero de nombre **alvaro.csr** que debemos enviar a nuestro compañero, para que así sea firmado por la correspondiente autoridad certificadora que ha creado. Es normal que estos puntos sean un poco confusos, pero no hay problema, ya que a continuación invertiremos los papeles y seremos nosotros los que firmemos dichos certificados.

Además de dicho certificado firmado, nos debe enviar la clave pública de la entidad certificadora, es decir, el certificado de la misma, para así poder verificar su firma sobre nuestro certificado.

En mi caso, he almacenado ambos ficheros en **/etc/ssl/certs/**, así que para comprobar que han sido correctamente ubicados, listaremos el contenido de dicho directorio estableciendo a su vez un filtro por nombre:

{% highlight shell %}
root@https:~# ls -l /etc/ssl/certs/ | egrep '(alvaro|cacert)'
-rw-r--r-- 1 root root   6283 Nov 17 18:49 alvaro.crt
-rw-r--r-- 1 root root   4698 Nov 17 18:49 cacert.pem
{% endhighlight %}

Como se puede apreciar, existe un fichero de nombre **alvaro.crt** que es el resultado de la firma de la solicitud de firma de certificado que previamente le hemos enviado, y otro de nombre **cacert.pem**, que es el certificado de la entidad certificadora, con el que posteriormente se comprobará la firma de la autoridad certificadora sobre dicho certificado del servidor.

Al igual que _apache2_ incluía un VirtualHost por defecto para las peticiones entrantes por el puerto 80 (**HTTP**), contiene otro por defecto para las peticiones entrantes por el puerto 443 (**HTTPS**), de nombre **default-ssl**, que por defecto viene deshabilitado, así que procederemos a modificarlo ejecutando para ello el comando:

{% highlight shell %}
root@https:~# nano /etc/apache2/sites-available/default-ssl.conf
{% endhighlight %}

Dentro del mismo debemos configurar las siguientes directivas:

* **ServerName**: Al igual que en el VirtualHost anterior, tendremos que indicar el nombre de dominio a través del cuál accederemos al servidor.
* **SSLEngine**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **SSLCertificateFile**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/ssl/certs/alvaro.crt**.
* **SSLCertificateKeyFile**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/ssl/private/alvaro.key**.
* **SSLCACertificateFile**: Indicamos la ruta del certificado de la _CA_ con el que comprobaremos la firma de nuestro certificado. En este caso, **/etc/ssl/certs/cacert.pem**.

De manera que el resultado final sería:

{% highlight shell %}
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin webmaster@localhost
                ServerName alvaro.iesgn.org
                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/alvaro.crt
                SSLCertificateKeyFile /etc/ssl/private/alvaro.key
                SSLCACertificateFile /etc/ssl/certs/cacert.pem

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>
        </VirtualHost>
</IfModule>
{% endhighlight %}

Dado que éste VirtualHost no viene habilitado por defecto, tendremos que hacerlo manualmente, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# a2ensite default-ssl
Enabling site default-ssl.
To activate the new configuration, you need to run:
  systemctl reload apache2
{% endhighlight %}

El nuevo VirtualHost ya ha sido habilitado, pero además, si nos fijamos en el contenido del fichero de configuración del VirtualHost, podremos apreciar que se hace uso de la directiva **IfModule**, es decir, se requiere que el módulo **ssl** se encuentre habilitado.

Para verificar si dicho módulo se encuentra ya cargado en memoria, listaremos el contenido del directorio **/etc/apache2/mods-enabled/**, que contiene enlaces simbólicos para aquellos módulos habilitados, por lo que estableceremos el correspondiente filtro por nombre:

{% highlight shell %}
root@https:~# ls /etc/apache2/mods-enabled/ | egrep 'ssl'
{% endhighlight %}

Como se puede apreciar, el filtro no ha devuelto ningún resultado, por lo que podemos asegurar que el módulo no se encuentra habilitado. Para habilitarlo, haremos uso de `a2enmod`:

{% highlight shell %}
root@https:~# a2enmod ssl
Enabling module ssl.
To activate the new configuration, you need to run:
  systemctl restart apache2
{% endhighlight %}

Además, lo que queremos hacer es forzar el uso de HTTPS (**https://**), de manera que estableceremos una redirección permanente en el VirtualHost accesible en el puerto 80 para que se así no se permita servir la página por HTTP (**http://**), ejecutando para ello el comando:

{% highlight shell %}
root@https:~# nano /etc/apache2/sites-available/000-default.conf
{% endhighlight %}

La creación de una redirección es bastante sencilla. En caso de que no sepas hacerlo todavía, te recomiendo leer el artículo [Mapear URL en Apache](http://www.alvarovf.com/servicios/2020/10/20/mapear-url-en-apache.html). El resultado final sería:

{% highlight shell %}
<VirtualHost *:80>
        ServerName alvaro.iesgn.org

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        Redirect 301 / https://alvaro.iesgn.com/
</VirtualHost>
{% endhighlight %}

El nuevo VirtualHost y el nuevo el módulo ya han sido habilitados, además del VirtualHost por defecto configurado para forzar HTTPS, pero para que los cambios surtan efecto, tendremos que reiniciar el servicio, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# systemctl restart apache2
{% endhighlight %}

Tras ello, ya estará todo listo para acceder a **alvaro.iesgn.org** desde el navegador. El resultado sería el siguiente:

![https2](https://i.ibb.co/gT7frB0/Captura-de-pantalla-de-2020-11-17-20-15-01.png "HTTPS")

Como se puede apreciar, se nos ha mostrado una advertencia de seguridad, por lo que podemos asegurar que la redirección de **http://** a **https://** ha funcionado correctamente.

Ésta advertencia es debida a que el navegador no ha podido comprobar la firma por parte de la _CA_ en el certificado recibido por el servidor, básicamente porque no tiene la clave pública o certificado de la _CA_, por lo que debemos importarlo manualmente en el navegador. En una situación cotidiana, ésto no suele hacerse, ya que las claves públicas de las _CA_ más conocidas vienen importadas por defecto, pero al ser una entidad certificadora totalmente nueva y sin reputación alguna, los navegadores como es obvio, no la han importado. De igual manera, la conexión sigue estando cifrada ya que el certificado ha sido correctamente transferido por parte del servidor, lo único es que no se ha podido comprobar la firma de la _CA_.

Para importar el certificado de la _CA_ en un navegador _Firefox_, tendremos que pulsar en el icono de las **3 barras** en la barra superior, seguidamente en **Preferencias** y una vez ahí, buscar el apartado **Privacidad & Seguridad**. Si nos desplazamos hasta la parte inferior del mismo, encontraremos la sección **Certificados**, en la que debemos pulsar en **Ver certificados**. Acto seguido, nos moveremos al apartado **Autoridades** para que así, pulsando en **Importar**, nos permita importar el certificado de una autoridad certificadora.

Una vez seleccionado el certificado a importar, se nos mostrará el siguiente mensaje:

![https3](https://i.ibb.co/GdfmkmX/Captura-de-pantalla-de-2020-11-17-20-31-23.png "HTTPS")

Tendremos que marcar, como se puede apreciar, la casilla **Confiar en esta CA para identificar sitios web.**, de manera que todos los certificados de servidores que hayan sido firmados por dicha autoridad certificadora, serán marcados de confianza y por tanto, no se mostrará la advertencia. Tras ello, podremos apreciar que ha sido correctamente importado:

![https4](https://i.ibb.co/7NsxWf1/Captura-de-pantalla-de-2020-11-17-20-33-36.png "HTTPS")

Posteriormente, recargaremos el navegador para así volver a mostrar la página **alvaro.iesgn.org**, accediendo desde HTTPS:

![https5](https://i.ibb.co/PmQq0G5/Captura-de-pantalla-de-2020-11-17-20-33-51.png "HTTPS")

En ésta ocasión, no se ha mostrado ninguna advertencia de seguridad, y como se puede apreciar al lado de la URL, se muestra un candado cerrado, indicando que la identidad del servidor ha podido ser correctamente validada. Si pulsamos en el mismo, se nos mostrará más información referente:

![https6](https://i.ibb.co/GJxdw6X/Captura-de-pantalla-de-2020-11-17-20-34-09.png "HTTPS")

En el mismo, se muestra mayor información técnica sobre el certificado, así como el nombre de la entidad certificadora, fecha de expiración, tipo de cifrado...

Ya hemos comprobado que con _apache2_ hemos conseguido hacer que HTTPS funcione, así que ahora vamos a hacer la misma prueba con la otra cara de la moneda, con _nginx_.

El primer paso será desinstalar _apache2_, para evitar por tanto posibles conflictos, y tras ello, instalar _nginx_. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@https:~# apt remove apache2 && apt install nginx
{% endhighlight %}

Tras ello, ya podríamos comenzar con la configuración del nuevo servidor web. Dicho servicio, también implementa un VirtualHost por defecto, pero a diferencia de _apache2_, podremos unificar la configuración del VirtualHost correspondiente al puerto 80 y el correspondiente al 443 en un único fichero, que se encuentra ubicado en **/etc/nginx/sites-available/default**. Para modificarlo, ejecutaremos el comando:

{% highlight shell %}
root@https:~# nano /etc/nginx/sites-available/default
{% endhighlight %}

Dentro del mismo, tendremos que crear dos directivas **server**, una para el VirtualHost accesible en el puerto 80 HTTP, dentro del cuál únicamente asignaremos el ServerName correspondiente y una redirección permanente para así forzar HTTPS. En la otra, para el VirtualHost accesible en el puerto 443 HTTPS, tendremos que configurar las siguientes directivas:

* **server_name**: Indicaremos el nombre de dominio a través del cuál accederemos al servidor.
* **ssl**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **ssl_certificate**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/ssl/certs/alvaro.crt**.
* **ssl_certificate_key**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/ssl/private/alvaro.key**.

De manera que el resultado final sería:

{% highlight shell %}
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        server_name alvaro.iesgn.org;

        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;

        ssl    on;
        ssl_certificate    /etc/ssl/certs/alvaro.crt;
        ssl_certificate_key    /etc/ssl/private/alvaro.key;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name alvaro.iesgn.org;

        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

Para que los cambios surtan efecto, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@https:~# systemctl reload nginx
{% endhighlight %}

Tras ello, ya estará todo listo para acceder a **alvaro.iesgn.org** desde el navegador. El resultado sería el siguiente:

![https7](https://i.ibb.co/PmQq0G5/Captura-de-pantalla-de-2020-11-17-20-33-51.png "HTTPS")

Una vez más, la redirección ha funcionado correctamente (me he asegurado de limpiar la caché para que la anterior redirección configurada en _apache2_ no influyese en el resultado), además de haber podido verificar la firma en el certificado del servidor, haciendo uso del certificado de la _CA_ previamente importado, como se puede apreciar:

![https8](https://i.ibb.co/6D4z6Ls/Captura-de-pantalla-de-2020-11-18-08-32-08.png "HTTPS")

Como se puede apreciar, la configuración para el uso de HTTPS en _apache2_ y _nginx_ no es para nada complicada. Ahora, vamos a invertir los papeles.

### Tarea 2: Crear una Autoridad Certificadora

En ésta ocasión, el papel de autoridad certificadora lo vamos a desarrollar nosotros, de manera que podremos firmar el certificado para que nuestro compañero pueda implementar HTTPS en su servidor.

El primer paso será generar un directorio en el que ubicaremos nuestra entidad certificadora, con la finalidad de mantener en todo momento una organización. Para ésta ocasión, he decidido que el nombre del directorio será **CA/**. A su vez, hay que generar varios subdirectorios dentro del mismo:

* **certsdb**: En dicho directorio se almacenarán los certificados firmados.
* **certreqs**: En dicho directorio se almacenarán los ficheros de solicitud de firma de certificados (_CSR_).
* **crl**: En dicho directorio se almacenará la lista de certificados que han sido revocados (_CRL_).
* **private**: En dicho directorio se almacenará la clave privada de la autoridad certificadora.

Para generar dichos directorios, haré uso del comando:

{% highlight shell %}
root@https:~# mkdir -p CA/{certsdb,certreqs,crl,private}
{% endhighlight %}

Donde:

* **-p**: Indicamos que cree el directorio padre en caso de no existir.

Una vez generados los directorios, nos moveremos dentro del directorio padre **CA/** ejecutando para ello el comando:

{% highlight shell %}
root@https:~# cd CA/
{% endhighlight %}

Cuando nos encontremos dentro del mismo, listaremos el contenido de forma gráfica para así poder apreciar que todo se ha generado correctamente, haciendo para ello uso de `tree`:

{% highlight shell %}
root@https:~/CA# tree
.
├── certreqs
├── certsdb
├── crl
└── private

4 directories, 0 files
{% endhighlight %}

Efectivamente, los 4 directorios han sido correctamente generados, así que antes de proceder es recomendable cambiar los permisos a **700** al directorio **private/**, ya que contendrá información sensible y nos interesa que únicamente el propietario tenga acceso a la misma. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@https:~/CA# chmod 700 ./private
{% endhighlight %}

Además, necesitaremos en el directorio actual un fichero que actuará como base de datos para los certificados existentes, de nombre **index.txt**, que generaremos haciendo uso del comando:

{% highlight shell %}
root@https:~/CA# touch index.txt
{% endhighlight %}

Lo más probable es que queramos usar un fichero de configuración de _openssl_ que no sea el de nuestra máquina, para así tratar que la autoridad certificadora esté en todo momento lo más aislada posible de la misma, así que haremos una copia de nuestro fichero de configuración nativo y lo adaptaremos a nuestras necesidades.

Dicho fichero se encontraba en mi caso en **/usr/lib/ssl/openssl.cnf**, aunque también podría haber estado en **/etc/openssl.cnf** o en **/usr/share/ssl/openssl.cnf**, así que para copiarlo al directorio actual, ejecutaremos el comando:

{% highlight shell %}
root@https:~/CA# cp /usr/lib/ssl/openssl.cnf ./
{% endhighlight %}

Una vez que lo hayamos copiado, procederemos a hacer uso de `nano` para modificarlo:

{% highlight shell %}
root@https:~/CA# nano openssl.cnf
{% endhighlight %}

Dentro del mismo, tendremos que realizar las oportunas adaptaciones para que haga uso de los directorios anteriormente generados, así como indicar determinada información básica sobre la autoridad certificadora, que nuestro compañero debe conocer, pues le será solicitada a la hora de la generación del fichero de solicitud de firma de certificado. En éste caso, las modificaciones realizadas han sido las siguientes:

{% highlight shell %}
dir             = /root/CA
certs           = $dir/certsdb
new_certs_dir   = $certs

countryName_default             = ES
stateOrProvinceName_default     = Sevilla
localityName_default            = Dos Hermanas
0.organizationName_default      = AlvaroVaca Corp
organizationalUnitName_default  = Informatica

#challengePassword              = A challenge password
#challengePassword_min          = 4
#challengePassword_max          = 20
#unstructuredName               = An optional company name
{% endhighlight %}

A pesar de hacer mención sobre aquellas modificaciones llevadas a cabo, considero que es necesario mostrar el fichero al completo por si quedase algún tipo de duda. Dada su gran extensión, en lugar de ponerlo en éste artículo, [aquí](https://pastebin.com/wnMdAWQ6) puedes encontrarlo al completo.

Tras ello, ya tendremos todo listo para generar nuestro par de claves y un fichero de solicitud de firma de certificado que posteriormente nos autofirmaremos, ejecutando para ello el comando:

{% highlight shell %}
root@https:~/CA# openssl req -new -newkey rsa:2048 -keyout private/cakey.pem -out careq.pem -config ./openssl.cnf
Generating a RSA private key
.........................................................................................................................+++++
................................+++++
writing new private key to 'private/cakey.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [ES]:
State or Province Name (full name) [Sevilla]:
Locality Name (eg, city) [Dos Hermanas]:
Organization Name (eg, company) [AlvaroVaca Corp]:
Organizational Unit Name (eg, section) [Informatica]:
Common Name (e.g. server FQDN or YOUR name) []:alvaro.debian
Email Address []:avacaferreras@gmail.com
{% endhighlight %}

Donde:

* **-new**: Indica que se genere un nuevo par de claves.
* **-newkey**: Especificamos el tamaño y tipo de nuestro par de claves. En éste caso, **RSA 2048-bit**.
* **-keyout**: Especificamos dónde se va a almacenar la clave privada. En éste caso, dentro del directorio **private/**, con nombre **cakey.pem**.
* **-out**: Especificamos dónde se va a almacenar la solicitud de firma de certificado. En éste caso, en el directorio actual, con nombre **careq.pem**.
* **-config**: Especificamos a _openssl_ que utilice el fichero de configuración modificado, no el nativo, con nombre **openssl.cnf**.

Como se puede apreciar, se nos ha solicitado una frase de paso para proteger la clave privada que vamos a generar, así como determinada información básica para el fichero de solicitud de firma de certificado, que debe coincidir con la que anteriormente hemos introducido en el fichero de configuración de _openssl_. Los dos últimos campos no son genéricos, por lo que debemos rellenarlos según el caso.

Tras ello, ya podremos proceder a autofirmarnos el certificado, que es el que posteriormente tendremos que facilitar a los clientes para que incluyan en su lista de certificados de _CA_, para que así no se les muestre la advertencia al utilizar HTTPS. Como he mencionado anteriormente, en una situación real, éste proceso lo suelen llevar a cabo empresas con determinada reputación cuyo certificado se encuentra importado por defecto en los navegadores. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@https:~/CA# openssl ca -create_serial -out cacert.pem -days 365 -keyfile private/cakey.pem -selfsign -extensions v3_ca -config ./openssl.cnf -infiles careq.pem
Using configuration from ./openssl.cnf
Enter pass phrase for private/cakey.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            02:55:9c:2f:50:5d:bb:45:f5:f4:ec:ea:ff:62:40:8a:e2:c3:d2:25
        Validity
            Not Before: Nov 17 17:28:26 2020 GMT
            Not After : Nov 17 17:28:26 2021 GMT
        Subject:
            countryName               = ES
            stateOrProvinceName       = Sevilla
            organizationName          = AlvaroVaca Corp
            organizationalUnitName    = Informatica
            commonName                = alvaro.debian
            emailAddress              = avacaferreras@gmail.com
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                2E:BF:BE:36:58:1E:B5:46:AF:76:0B:FA:90:40:3C:76:0C:0F:72:D3
            X509v3 Authority Key Identifier: 
                keyid:2E:BF:BE:36:58:1E:B5:46:AF:76:0B:FA:90:40:3C:76:0C:0F:72:D3

            X509v3 Basic Constraints: critical
                CA:TRUE
Certificate is to be certified until Nov 17 17:28:26 2021 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
{% endhighlight %}

Donde:

* **-create_serial**: Indicamos que genere un serial de 128 bits para comenzar. Gracias a dicha aleatoriedad, si tuviésemos que volver a empezar, no sobreescribiríamos los ya existentes. Es muy importante.
* **-out**: Especificamos dónde se va a almacenar el certificado firmado. En éste caso, en el directorio actual, con nombre **cacert.pem**.
* **-days**: Especificamos la validez en días del certificado firmado. En este caso, **365** días.
* **-keyfile**: Indicamos la clave privada a usar para firmar dicho certificado. En este caso, la generada en el paso anterior, **private/cakey.pem**.
* **-selfsign**: Indicamos a _openssl_ que vamos a autofirmarnos el certificado.
* **-extensions**: Indicamos a _openssl_ la sección del fichero de configuración en el que se encuentran las extensiones a usar. En éste caso, **v3_ca**.
* **-config**: Especificamos a _openssl_ que utilice el fichero de configuración modificado, no el nativo, con nombre **openssl.cnf**.
* **-infiles**: Indicamos qué queremos firmar, en este caso, el _CSR_ para nuestra nueva autoridad certificadora creado en el paso anterior, con nombre **careq.pem**.

Como se puede apreciar en la salida del comando, se nos ha pedido la frase de paso previamente configurada, para así asegurarnos que aunque la clave privada llegase a malas manos, no puedan realizar firmas fraudulentas. Además, antes de firmar el certificado, se nos ha mostrado toda la información referente al mismo, y se nos ha pedido confirmación.

Para verificar que el certificado de la autoridad certificadora se encuentra contenido en el directorio actual, listaremos el contenido del mismo haciendo uso del comando:

{% highlight shell %}
root@https:~/CA# ls -l
total 52
-rw-r--r-- 1 root root  4658 Nov 17 17:28 cacert.pem
-rw-r--r-- 1 root root  1090 Nov 17 17:27 careq.pem
drwxr-xr-x 2 root root  4096 Nov 17 17:21 certreqs
drwxr-xr-x 2 root root  4096 Nov 17 17:28 certsdb
drwxr-xr-x 2 root root  4096 Nov 17 17:21 crl
-rw-r--r-- 1 root root   170 Nov 17 17:28 index.txt
-rw-r--r-- 1 root root    21 Nov 17 17:28 index.txt.attr
-rw-r--r-- 1 root root     0 Nov 17 17:21 index.txt.old
-rw-r--r-- 1 root root 11153 Nov 17 17:27 openssl.cnf
drwx------ 2 root root  4096 Nov 17 17:27 private
-rw-r--r-- 1 root root    41 Nov 17 17:28 serial
{% endhighlight %}

Efectivamente, existe un fichero **cacert.pem** que es resultado de firmar el fichero de solicitud de firma de certificado **careq.pem**.

Ya está todo listo para proceder a firmar el certificado para el servidor de mi compañero, así que he situado su fichero de solicitud de firma de certificado dentro de **certreqs/**, que es donde deben ubicarse. Vamos a verificarlo listando el contenido de dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@https:~/CA# ls -l certreqs/
total 4
-rw-r--r-- 1 root root 1801 Nov 17 18:24 javierpzh.csr
{% endhighlight %}

Efectivamente, el fichero de solicitud de firma de certificado de nombre **javierpzh.csr** se encuentra contenido en dicho directorio, así que podremos proceder a firmarlo, haciendo uso del comando:

{% highlight shell %}
root@https:~/CA# openssl ca -config openssl.cnf -out certsdb/javierpzh.crt -infiles certreqs/javierpzh.csr 
Using configuration from openssl.cnf
Enter pass phrase for /root/CA/private/cakey.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            02:55:9c:2f:50:5d:bb:45:f5:f4:ec:ea:ff:62:40:8a:e2:c3:d2:29
        Validity
            Not Before: Nov 17 18:25:11 2020 GMT
            Not After : Nov 17 18:25:11 2021 GMT
        Subject:
            countryName               = ES
            stateOrProvinceName       = Sevilla
            organizationName          = AlvaroVaca Corp
            organizationalUnitName    = Informatica
            commonName                = javierpzh.iesgn.org
            emailAddress              = javierperezhidalgo01@gmail.com
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                9E:75:EF:84:A2:C7:92:94:00:ED:B3:94:FC:64:E7:FC:5E:C4:8A:82
            X509v3 Authority Key Identifier: 
                keyid:2E:BF:BE:36:58:1E:B5:46:AF:76:0B:FA:90:40:3C:76:0C:0F:72:D3

Certificate is to be certified until Nov 17 18:25:11 2021 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
{% endhighlight %}

Donde:

* **-config**: Especificamos a _openssl_ que utilice el fichero de configuración modificado, no el nativo, con nombre **openssl.cnf**.
* **-out**: Especificamos dónde se va a almacenar el certificado firmado. En éste caso, dentro del directorio **certsdb/**, con nombre **javierpzh.crt**.
* **-infiles**: Indicamos qué queremos firmar, en este caso, el _CSR_ de nuestro compañero, situado en **certreqs/** con nombre **javierpzh.csr**.

Como se puede apreciar en la salida del comando, se nos ha pedido la frase de paso previamente configurada, para así asegurarnos que aunque la clave privada llegase a malas manos, no puedan realizar firmas fraudulentas. Además, antes de firmar el certificado, se nos ha mostrado toda la información referente al mismo, y se nos ha pedido confirmación.

En un principio, el certificado ya se encuentra firmado dentro de **certsdb/**, así que para verificarlo, vamos a listar el contenido de dicho directorio, ejecutando para ello el comando:

{% highlight shell %}
root@https:~/CA# ls -l certsdb/
total 48
-rw-r--r-- 1 root root 4658 Nov 17 17:28 02559C2F505DBB45F5F4ECEAFF62408AE2C3D225.pem
-rw-r--r-- 1 root root 6284 Nov 17 18:25 02559C2F505DBB45F5F4ECEAFF62408AE2C3D229.pem
-rw-r--r-- 1 root root 6284 Nov 17 18:25 javierpzh.crt
{% endhighlight %}

Efectivamente, existen un total de 3 ficheros dentro del mismo: uno correspondiente al propio certificado de la autoridad certificadora, y dos correspondientes a nuestro compañero, que son exactamente lo mismo pero con diferentes nombres, uno de ellos con un identificador y el otro, con un nombre común que nosotros hemos establecido para así identificarlo con una mayor facilidad.

Hasta aquí ha llegado nuestra parte, ya que únicamente quedaría hacerle llegar a nuestro compañero el fichero **certsdb/javierpzh.crt**, que es su certificado firmado, junto al propio certificado de la autoridad certificadora, el cuál contiene la clave pública, para así poder verificar dicha firma sobre su certificado, de nombre **cacert.pem**.

Por último, vamos a visualizar el contenido del fichero **index.txt**, que es una especie de base de datos en texto plano con información sobre los certificados firmados por la autoridad certificadora. Para ello, haremos uso de `cat`:

{% highlight shell %}
root@https:~/CA# cat index.txt
V	211117172826Z		02559C2F505DBB45F5F4ECEAFF62408AE2C3D225	unknown	/C=ES/ST=Sevilla/O=AlvaroVaca Corp/OU=Informatica/CN=alvaro.debian/emailAddress=avacaferreras@gmail.com
V	211117182511Z		02559C2F505DBB45F5F4ECEAFF62408AE2C3D229	unknown	/C=ES/ST=Sevilla/O=AlvaroVaca Corp/OU=Informatica/CN=javierpzh.iesgn.org/emailAddress=javierperezhidalgo01@gmail.com
{% endhighlight %}

Dentro del mismo, encontramos información básica correspondiente a los dos certificados firmados: el de la propia autoridad certificadora y el de nuestro compañero, además de su estado de validez actual.