---
layout: post
title:  "Gestionando el almacenamiento con rclone"
banner: "/assets/images/banners/raid.jpg"
date:   2020-10-02 19:53:00 +0200
categories: seguridad
---
## Tarea 1: Instala rclone en tu equipo.

Lo primero será instalar el paquete rclone en nuestro equipo. Para ello, ejecutaremos el comando (con permisos de administrador, ejecutando el comando `su -`):

{% highlight shell %}
apt install rclone
{% endhighlight %}

Listo. Así de sencillo es tener instalado el paquete.

## Tarea 2: Configura dos proveedores cloud en rclone.

En este caso, he decidido usar **Dropbox** y **Google Drive**, posiblemente los más conocidos, así que vamos a proceder a configurarlos. Para ello, ejecutaremos el comando:

{% highlight shell %}
alvaro@debian:~$ rclone config
2020/10/01 20:09:11 NOTICE: Config file "/home/alvaro/.config/rclone/rclone.conf" not found - using defaults
No remotes found - make a new one
n) New remote
s) Set configuration password
q) Quit config
n/s/q>
{% endhighlight %}

Como se puede apreciar, nos informa de que no ha encontrado ningún fichero de configuración existente con anterioridad, así que ha generado uno nuevo. Nos está solicitando una opción, así que para agregar un nuevo servicio, pulsaremos la letra **n**.

{% highlight shell %}
n/s/q>n
name>
{% endhighlight %}

Tras ello, nos estará pidiendo un **nombre** para identificar el servicio de forma única en rclone. Dado que el primero que voy a configurar es Dropbox, voy a nombrarlo **Dropbox**.

{% highlight shell %}
name> Dropbox
Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / A stackable unification remote, which can appear to merge the contents of several remotes
   \ "union"
 2 / Alias for a existing remote
   \ "alias"
 3 / Amazon Drive
   \ "amazon cloud drive"
 4 / Amazon S3 Compliant Storage Providers (AWS, Ceph, Dreamhost, IBM COS, Minio)
   \ "s3"
 5 / Backblaze B2
   \ "b2"
 6 / Box
   \ "box"
 7 / Cache a remote
   \ "cache"
 8 / Dropbox
   \ "dropbox"
 9 / Encrypt/Decrypt a remote
   \ "crypt"
10 / FTP Connection
   \ "ftp"
11 / Google Cloud Storage (this is not Google Drive)
   \ "google cloud storage"
12 / Google Drive
   \ "drive"
13 / Hubic
   \ "hubic"
14 / JottaCloud
   \ "jottacloud"
15 / Local Disk
   \ "local"
16 / Microsoft Azure Blob Storage
   \ "azureblob"
17 / Microsoft OneDrive
   \ "onedrive"
18 / OpenDrive
   \ "opendrive"
19 / Openstack Swift (Rackspace Cloud Files, Memset Memstore, OVH)
   \ "swift"
20 / Pcloud
   \ "pcloud"
21 / SSH/SFTP Connection
   \ "sftp"
22 / Webdav
   \ "webdav"
23 / Yandex Disk
   \ "yandex"
24 / http Connection
   \ "http"
Storage>
{% endhighlight %}

El siguiente paso será seleccionar el **tipo de servicio** que queremos utilizar. En este caso, tras buscar en la lista, podemos escribir **dropbox** o introducir el número **8**, pues ambos apuntan al mismo servicio.

{% highlight shell %}
Storage> 8
** See help for dropbox backend at: https://rclone.org/dropbox/ **

Dropbox App Client Id
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_id>
{% endhighlight %}

Tras elegir el tipo de servicio, nos devuelve un enlace a la documentación en el que muestra cómo configurar el servicio en cuestión. Además, nos está preguntando por el **client_id** (referente a OAuth), que en nuestro caso lo dejaremos vacío, ya que no vamos a usar ese tipo de autentificación.

{% highlight shell %}
client_id>
Dropbox App Client Secret
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_secret>
{% endhighlight %}

Dado que no vamos a usar OAuth para la autentificación, y por tanto, no hemos especificado ningún **client_id**, tampoco será necesario especificar un **client_secret**. Procedemos sin introducir nada.

{% highlight shell %}
client_secret> 
Edit advanced config? (y/n)
y) Yes
n) No
y/n>
{% endhighlight %}

En este caso, no vamos a utilizar ningún tipo de **configuración avanzada**, pues no será necesario para el uso básico que vamos a darle a rclone, así que introduciremos una **n**.

{% highlight shell %}
y/n> n
Remote config
Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine
y) Yes
n) No
y/n>
{% endhighlight %}

Ahora debemos elegir si queremos usar la **auto-configuración** o vamos a configurarlo para otra máquina (en caso de que estemos en remoto). En este caso, como quiero configurar rclone para el equipo local, tendremos que elegir la opción **y**.

{% highlight shell %}
y/n> y
If your browser doesn't open automatically go to the following link: http://127.0.0.1:53682/auth
Log in and authorize rclone for access
Waiting for code...
{% endhighlight %}

Tras ello, nuestro navegador se debería abrir automáticamente redirigiendo al enlace que nos ha mostrado. En caso de que no sea así (como en mi caso), abriremos de forma manual el enlace que nos muestra, para terminar de configurar el servicio.

![inicio1](https://i.ibb.co/P1kkfST/Captura-de-pantalla-de-2020-10-01-20-17-02.png "Inicio Dropbox")

Lógicamente nos pedirá nuestras credenciales para poder autorizar a la aplicación externa (rclone) a usar nuestra cuenta de Dropbox a través de la API. Introduciremos nuestras credenciales de forma totalmente segura, pues esta información no llega en ningún momento a rclone.

![inicio2](https://i.ibb.co/rbQZjjx/Captura-de-pantalla-de-2020-10-01-20-17-25.png "Inicio Dropbox")

Se nos mostrarán dos botones, uno para **Cancelar** la operación y otro para **Permitir** que rclone tenga acceso a los **archivos y carpetas** de Dropbox, tal y como se muestra en el mensaje.

![inicio3](https://i.ibb.co/tpkp7CG/Captura-de-pantalla-de-2020-10-01-20-17-34.png "Inicio Dropbox")

Tras permitir la operación nos dirá que el proceso ha sido exitoso y ya podemos volver a rclone.

{% highlight shell %}
Got code
--------------------
[Dropbox]
token = {"access_token":"################################################################","token_type":"bearer","expiry":"0001-01-01T00:00:00Z"}
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d>
{% endhighlight %}

La terminal nos ha devuelto un **access_token**, que en este caso he censurado por seguridad y algunos parámetros más. Simplemente, tendremos que aceptar la conexión introduciendo la opción **y**.

{% highlight shell %}
y/e/d> y
Current remotes:

Name                 Type
====                 ====
Dropbox              dropbox

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q>
{% endhighlight %}

Genial. Nuestra primera conexión con un servicio ha sido realizada y ya nos aparece en la lista, pero todavía nos queda por configurar Google Drive. Para ello, volveremos a introducir la opción **n** para configurar un nuevo servicio.

Dado que ya hemos explicado cómo se configura el primer servicio, no es necesario volver a hacerlo, hasta que haya alguna diferencia entre la configuración de ambos.

{% highlight shell %}
e/n/d/r/c/s/q>n
name> Drive
Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
...
12 / Google Drive
   \ "drive"
...
Storage> 12
** See help for drive backend at: https://rclone.org/drive/ **

Google Application Client Id
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_id> 
Google Application Client Secret
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_secret> 
Scope that rclone should use when requesting access from drive.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / Full access all files, excluding Application Data Folder.
   \ "drive"
 2 / Read-only access to file metadata and file contents.
   \ "drive.readonly"
   / Access to files created by rclone only.
 3 | These are visible in the drive website.
   | File authorization is revoked when the user deauthorizes the app.
   \ "drive.file"
   / Allows read and write access to the Application Data folder.
 4 | This is not visible in the drive website.
   \ "drive.appfolder"
   / Allows read-only access to file metadata but
 5 | does not allow any access to read or download file content.
   \ "drive.metadata.readonly"
scope>
{% endhighlight %}

Considero necesario hacer una pausa en este punto ya que es diferente con respecto al proveedor anterior. En este caso, nos da la libertad de elegir qué nivel de **privilegios** queremos darle a rclone, siendo **1** el más prioritario (acceso a todos los ficheros) y **5** el menos prioritario (acceso únicamente a los metadatos). En mi caso, para poder mostrar bien cómo funciona el software, voy a darle todos los privilegios (**1**).

{% highlight shell %}
scope> 1
ID of the root folder
Leave blank normally.
Fill in to access "Computers" folders. (see docs).
Enter a string value. Press Enter for the default ("").
root_folder_id>
{% endhighlight %}

En caso de tener activada en Google Drive la sincronización automática con ordenadores, los ficheros de éstos se encuentran en un apartado distinto, llamado "Computers". Si se quisiese acceder a los mismos, habría que introducir el **root_folder_id**. En mi caso, no tengo dicha funcionalidad activada, así que lo dejaré vacío.

{% highlight shell %}
root_folder_id> 
Service Account Credentials JSON file path 
Leave blank normally.
Needed only if you want use SA instead of interactive login.
Enter a string value. Press Enter for the default ("").
service_account_file>
{% endhighlight %}

En cuanto al **service_account_file**, tampoco es necesario, dado que vamos a hacer uso del login interactivo, por lo que lo dejaré vacío. A partir de ahora, la parte final del proceso es exactamente igual que en el proveedor anterior.

{% highlight shell %}
service_account_file> 
Edit advanced config? (y/n)
y) Yes
n) No
y/n> n
Remote config
Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine or Y didn't work
y) Yes
n) No
y/n> y
If your browser doesn't open automatically go to the following link: http://127.0.0.1:53682/auth
Log in and authorize rclone for access
Waiting for code...
Got code
Configure this as a team drive?
y) Yes
n) No
y/n> n
--------------------
[Drive]
scope = drive
token = {"access_token":"################################################################","token_type":"Bearer","refresh_token":"################################################################","expiry":"2020-10-01T21:28:22.446369344+02:00"}
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y
Current remotes:

Name                 Type
====                 ====
Drive                drive
Dropbox              dropbox

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q>
{% endhighlight %}

Tal y como se puede apreciar en la lista, ambos proveedores se encuentran ya añadidos a rclone, por lo que podremos introducir la opción **q** para salir de la configuración de rclone.

## Tarea 3: Muestra distintos comandos de rclone para gestionar los ficheros de los proveedores cloud.

Antes de empezar, me gustaría dejar clara la estructura de ficheros que tengo en cada proveedor:

{% highlight shell %}
*Drive*
├───20200415_205255.mp4
├───bashrc
├───Criptografía.odt
├───Examen.odt
├───hola.txt
├───How to get started with Drive
├───packages_list
└───Script.py
{% endhighlight %}

{% highlight shell %}
*Dropbox*
├───deberes.txt
├───ComandosWindows.odt
├───ComandosLinux.odt
├───Get Started with Dropbox Paper.url
└───Get Started with Dropbox.pdf
{% endhighlight %}

Las **funcionalidades** de rclone que vamos a tratar son:

* Listar ficheros.
* Copiar ficheros locales a la nube.
* Copiar directorios locales a la nube.
* Copiar ficheros entre proveedores.
* Obtener información sobre la cuota.
* Eliminar ficheros/directorios.
* Copiar stdin a un fichero remoto.
* Sincronizar directorios.
* Vista de árbol (tree).
* Crear directorios remotos.
* Montar directorios remotos en local.

### Listar ficheros

---

La sintaxis de rclone para [listar](https://rclone.org/commands/rclone_ls/) los ficheros/directorios existentes es:

{% highlight shell %}
rclone ls <proveedor>:<ruta>
{% endhighlight %}

En este caso, para que me muestre todo lo que tengo almacenado en el directorio raíz del proveedor, no especificaré nada en la ruta.

#### Drive

{% highlight shell %}
alvaro@debian:~$ rclone ls Drive:
2540369488 20200415_205255.mp4
    17135 Criptografía.odt
    16940 Examen.odt
  3017667 How to get started with Drive
     2604 Script.py
     3688 bashrc
        0 hola.txt
    58606 packages_list
{% endhighlight %}

#### Dropbox

{% highlight shell %}
alvaro@debian:~$ rclone ls Dropbox:
   108229 ComandosLinux.odt
    36476 ComandosWindows.odt
      240 Get Started with Dropbox Paper.url
  1102331 Get Started with Dropbox.pdf
      648 deberes.txt
{% endhighlight %}

Como se puede apreciar, la estructura devuelta por ambos proveedores coincide con la que he mencionado antes de comenzar.

### Copiar fichero local a la nube

---

La sintaxis de rclone para [copiar](https://rclone.org/commands/rclone_copy/) un fichero local a la nube es:

{% highlight shell %}
rclone copy <proveedor>:<ruta> <proveedor>:<ruta>
{% endhighlight %}

Esta sintaxis es genérica, y sirve tanto para copiar un fichero/directorio local a la nube, para copiar un fichero/directorio de un proveedor a otro y para copiar un fichero/directorio que está en la nube a nuestra máquina local.

#### Drive

En este caso, voy a generar un fichero con un texto en su interior para posteriormente subirlo a la nube y verificar si mantiene sus propiedades. Para ello, ejecutaré el comando:

{% highlight shell %}
alvaro@debian:~$ echo "Prueba Drive" > drive.txt
{% endhighlight %}

Tras ello, lo copiaré a la nube, en el directorio raíz (por lo que no hay que especificar ruta):

{% highlight shell %}
alvaro@debian:~$ rclone copy drive.txt Drive:
{% endhighlight %}

Para ver si el archivo se ha generado correctamente y el contenido en su interior es legible, iremos a Google Drive.

![copy1](https://i.ibb.co/NZsFPKP/Captura-de-pantalla-de-2020-10-01-20-37-57.png "rclone copy")

![copy2](https://i.ibb.co/GtYbDFx/Captura-de-pantalla-de-2020-10-01-20-38-01.png "rclone copy")

Efectivamente, así ha sido.

#### Dropbox

En este caso, voy a repetir el mismo ejemplo que en Google Drive. Para ello, ejecutaré el comando:

{% highlight shell %}
alvaro@debian:~$ echo "Prueba Dropbox" > dropbox.txt
{% endhighlight %}

Tras ello, lo copiaré a la nube, en el directorio raíz (por lo que no hay que especificar ruta):

{% highlight shell %}
alvaro@debian:~$ rclone copy dropbox.txt Dropbox:
{% endhighlight %}

Para ver si el archivo se ha generado correctamente y el contenido en su interior es legible, iremos a Dropbox.

![copy3](https://i.ibb.co/r2mzL2J/Captura-de-pantalla-de-2020-10-01-20-37-43.png "rclone copy")

![copy4](https://i.ibb.co/HThtQqd/Captura-de-pantalla-de-2020-10-01-20-37-53.png "rclone copy")

Efectivamente, así ha sido.

### Copiar un directorio local a un directorio en la nube

---

La sintaxis de rclone para [copiar](https://rclone.org/commands/rclone_copy/) un directorio local a otro directorio en la nube es:

{% highlight shell %}
rclone copy <proveedor>:<ruta> <proveedor>:<ruta>
{% endhighlight %}

Como he mencionado en la tarea anterior, ésta sintaxis es genérica.

#### Drive

En este caso, he metido algunas capturas de anteriores prácticas en el directorio **~/Imágenes/**, así que vamos a probar a subir todo el directorio a Google Drive, a una carpeta dentro del mismo que se llame **Capturas/** (esa será la ruta en este caso).

{% highlight shell %}
alvaro@debian:~$ rclone copy ~/Imágenes/ Drive:Capturas/
{% endhighlight %}

Tras ello, iremos a Google Drive a comprobar si el directorio y todo su contenido se ha copiado correctamente en un directorio de nombre _Capturas/_ dentro de Google Drive.

![copy5](https://i.ibb.co/d0nT0s4/Captura-de-pantalla-de-2020-10-01-20-50-06.png "rclone copy")

Efectivamente, todas las capturas han sido subidas (aunque en la captura únicamente se vean las 6 primeras).

#### Dropbox

Llegado este punto, tenía curiosidad por probar algo. Dado que la tarea fue realizada sobre una máquina virtual totalmente virgen y las cuentas tanto de Dropbox como de Google Drive eran nuevas y únicamente usadas para esta práctica, quise probar si sería posible copiar todo el directorio personal (**/home/alvaro**) en Dropbox, dentro de un directorio que se llame **Personal/** (esa será la ruta en este caso).

{% highlight shell %}
alvaro@debian:~$ rclone copy /home/alvaro/ Dropbox:Personal/
{% endhighlight %}

Tras un buen rato esperando y algunos errores debidos a ficheros protegidos, iremos a Dropbox para comprobar si el directorio y el contenido se ha copiado correctamente en un directorio de nombre _Personal/_ dentro de Dropbox.

![copy6](https://i.ibb.co/HzRFJMB/Captura-de-pantalla-de-2020-10-01-20-50-14.png "rclone copy")

Efectivamente, rclone ha sido capaz de copiar la mayoría de los ficheros y directorios dentro de Dropbox. Es importante mencionar que esto es una prueba simplemente y que se ha realizado bajo un escenario totalmente seguro, sin que ningún tipo de información haya podido ser comprometida. No es nada seguro hacer esto con el directorio personal de la máquina común, así que como dicen en la tele, no intentes esto en casa.

### Copiar ficheros entre los dos proveedores cloud

---

La sintaxis de rclone para [copiar](https://rclone.org/commands/rclone_copy/) ficheros entre dos proveedores en la nube es:

{% highlight shell %}
rclone copy <proveedor>:<ruta> <proveedor>:<ruta>
{% endhighlight %}

Como se ha mencionado en las tareas anteriores, ésta sintaxis es genérica.

#### Drive

En este caso, supongamos que necesitamos copiar el directorio _Capturas/_ actualmente existente en Drive a Dropbox, a una carpeta del mismo nombre, es decir, _Capturas/_. En este caso, la ruta del proveedor origen será **Capturas/** y la ruta del proveedor destino, también será **Capturas/**.

{% highlight shell %}
alvaro@debian:~$ rclone copy Drive:Capturas/ Dropbox:Capturas/
{% endhighlight %}

Tras ello, iremos a Dropbox a verificar que el proceso se ha completado correctamente.

![copy7](https://i.ibb.co/cDDf18Y/Captura-de-pantalla-de-2020-10-01-20-56-54.png "rclone copy")

Tal y como era de esperar, todas las capturas se han copiado de un proveedor a otro (aunque en la captura únicamente se vean las 3 primeras).

#### Dropbox

Para el caso de Dropbox, vamos a copiar el archivo _ComandosLinux.odt_ a Drive, al directorio raíz. En este caso, la ruta del proveedor origen será el fichero **ComandosLinux.odt** y para el proveedor destino, no tendremos que especificar ninguna ruta.

{% highlight shell %}
alvaro@debian:~$ rclone copy Dropbox:ComandosLinux.odt Drive:
{% endhighlight %}

Tras ello, iremos a Google Drive a verificar que el proceso se ha completado correctamente.

![copy8](https://i.ibb.co/C8xRR20/Captura-de-pantalla-de-2020-10-01-20-56-57.png "rclone copy")

Tal y como era de esperar, el fichero se ha copiado correctamente de un proveedor a otro.

### Obtener información sobre la cuota

---

La sintaxis de rclone para obtener información sobre la [cuota](https://rclone.org/commands/rclone_about/) de un proveedor es:

{% highlight shell %}
rclone about <proveedor>:
{% endhighlight %}

En caso de poner _--json_ al final, nos lo devolvería en formato JSON.

#### Drive

{% highlight shell %}
alvaro@debian:~$ rclone about Drive:
Total:   15G
Used:    2.374G
Free:    12.209G
Trashed: 182.541k
Other:   426.966M
{% endhighlight %}

En este caso, tenemos un total de **15GB** disponibles para almacenamiento de Google, de los cuales en este caso, quedan disponibles **12.2GB**, pues en los archivos hay un total de **2.37GB**, en la papelera **182KB** y en otros servicios de Google, un total de **426MB**.

Esta funcionalidad de rclone puede ser bastante útil para hacer cierto tipo de scripts, como por ejemplo para que te envíe un correo electrónico cuando te queden menos de X GB libres o que borre ciertos ficheros en caso de que quede poco espacio. Todo es usar la imaginación.

#### Dropbox

{% highlight shell %}
alvaro@debian:~$ rclone about Dropbox:
Total:   2G
Used:    161.051M
Free:    1.843G
{% endhighlight %}

En este caso, tenemos un total de **2GB** disponibles, de los cuales en este caso, quedan disponibles **1.84GB**, pues en los archivos hay un total de **161MB**. Como se puede apreciar, Drive muestra más información al respecto, como por ejemplo el tamaño de los archivos que hay en la papelera.

### Eliminar ficheros/directorios

---

La sintaxis de rclone para [eliminar](https://rclone.org/commands/rclone_delete/) ficheros de un proveedor es:

{% highlight shell %}
rclone delete <proveedor>:<ruta>
{% endhighlight %}

En caso de poner _--min-size (tamaño)_ después de **rclone**, podremos eliminar únicamente aquellos que pesen más del tamaño introducido.

La sintaxis de rclone para [eliminar](https://rclone.org/commands/rclone_purge/) directorios de un proveedor es:

{% highlight shell %}
rclone purge <proveedor>:<ruta>
{% endhighlight %}

#### Drive

En este caso, vamos a probar a eliminar el **fichero** de nombre **20200415_205255.mp4**, pues es un vídeo y queremos hacer limpieza para conseguir más espacio libre. En este caso, tendríamos que usar el comando `delete` sobre el fichero en cuestión, pero en lugar de especificar directamente el nombre, vamos a borrar aquellos ficheros que superen los 500MB de tamaño (dado que no tengo más ficheros de ese tamaño, únicamente va a pillar el vídeo, pero si hubiese más, cogería todos aquellos que superen los 500MB).

{% highlight shell %}
alvaro@debian:~$ rclone --min-size 500M delete Drive:
{% endhighlight %}

Para verificar que la eliminación de dicho fichero se ha llevado a cabo correctamente, vamos a entrar a la Papelera de Google Drive.

![delete](https://i.ibb.co/SJjq13R/Captura-de-pantalla-de-2020-10-01-21-00-23.png "rclone delete")

Como era de esperar, el vídeo ha sido eliminado correctamente.

#### Dropbox

A diferencia del caso anterior, ahora lo que vamos a eliminar es un directorio por completo, por lo que no podemos usar la opción `delete`, ya que lo que haríamos sería borrar los ficheros que se encuentran dentro del mismo, y lo que queremos es eliminarlo en su totalidad. Para ello, haremos uso de la opción `purge`. La ruta del proveedor en este caso sería **Personal/**, puesto que es el directorio que queremos eliminar.

{% highlight shell %}
alvaro@debian:~$ rclone purge Dropbox:Personal/
{% endhighlight %}

Para verificar que la eliminación de dicho directorio se ha llevado a cabo correctamente, vamos a entrar a la Papelera de Dropbox.

![purge](https://i.ibb.co/J7cgCz5/Captura-de-pantalla-de-2020-10-01-21-06-03.png "rclone purge")

Como era de esperar, el directorio ha sido eliminado correctamente.

### Copiar stdin a un fichero remoto

---

La sintaxis de rclone para [plasmar](https://rclone.org/commands/rclone_rcat/) la entrada estándar en un fichero de un proveedor es:

{% highlight shell %}
<stdin> | rclone rcat <proveedor>:<ruta>
{% endhighlight %}

#### Drive

En este caso, imaginemos que quiero subir la salida del comando `lsblk -f` en un fichero de nombre **lsblk.txt** en el proveedor de cloud. Si no existiese esta opción, el procedimiento sería **volcar** la salida de la ejecución del comando a un fichero para posteriormente **subirlo** al cloud. Gracias a esto, esta tarea se convierte en algo mucho más sencillo.

{% highlight shell %}
alvaro@debian:~$ lsblk -f | rclone rcat Drive:lsblk.txt
{% endhighlight %}

Para verificar que dicho fichero se ha generado correctamente y el contenido es el que queremos, voy a ir a Google Drive para verificarlo.

![rcat1](https://i.ibb.co/nmzGhp5/Captura-de-pantalla-de-2020-10-01-21-11-45.png "rclone rcat")

Efectivamente, el fichero se encuentra generado y contiene la salida del comando `lsblk -f`.

#### Dropbox

Del mismo modo que en el ejemplo anterior, vamos a ejecutar otro comando, por ejemplo `ls -l` y vamos a subir la salida de dicho comando a Dropbox, en un fichero de nombre **ls.txt**.

{% highlight shell %}
alvaro@debian:~$ ls -l | rclone rcat Dropbox:ls.txt
{% endhighlight %}

Para verificar que dicho fichero se ha generado correctamente y el contenido es el que queremos, voy a ir a Dropbox para verificarlo.

![rcat2](https://i.ibb.co/k1DGSc2/Captura-de-pantalla-de-2020-10-01-21-12-55.png "rclone rcat")

Efectivamente, el fichero se encuentra generado y contiene la salida del comando `ls -l`.

### Sincronizar directorios

---

La sintaxis de rclone para [sincronizar](https://rclone.org/commands/rclone_sync/) un directorio local y un directorio de un proveedor es:

{% highlight shell %}
rclone sync <directorio> <proveedor>:<ruta>
{% endhighlight %}

Es importante mencionar que donde se van a realizar las modificaciones para llevar a cabo la sincronización es en el directorio remoto, el local se queda intacto. En resumen, lo que hace es igualar ambos directorios pero únicamente teniendo en cuentas las modificaciones (de manera que si queremos sincronizar un directorio muy grande no sea necesario eliminar todo el contenido y volverlo a subir, sino únicamente ver qué modificaciones se han llevado a cabo y copiarlas en el remoto). Es aconsejable usar la opción _-i_ en caso de no estar seguro.

#### Drive

En este caso, supongamos que quiero eliminar las 9 primeras capturas del directorio **Capturas/**, para posteriormente sincronizarlo con el remoto. Para ello, lo primero que tendremos que hacer será eliminar dichos ficheros:

{% highlight shell %}
alvaro@debian:~/Imágenes$ rm [0-9].png
{% endhighlight %}

Tras ello, ya podremos ejecutar el comando para que se lleve a cabo la sincronización con la ruta **Capturas/** del proveedor.

{% highlight shell %}
alvaro@debian:~/Imágenes$ rclone sync ./ Drive:Capturas/
{% endhighlight %}

Para verificar que dichas 9 capturas también se han eliminado del directorio remoto, vamos a acceder al directorio de Google Drive.

![sync1](https://i.ibb.co/KxZ4hmS/Captura-de-pantalla-de-2020-10-01-21-25-22.png "rclone sync")

Efectivamente, ahora la primera captura que se muestra es **10.png**, por lo tanto, las 9 primeras han sido eliminadas.

#### Dropbox

Vamos a repetir la misma operación en Dropbox, pero esta vez, en lugar de eliminar las 9 primeras capturas, vamos a eliminar todas aquellas que se nombren con números. Para ello, lo primero que tendremos que hacer será eliminar dichos ficheros:

{% highlight shell %}
alvaro@debian:~/Imágenes$ rm [0-9]*.png
{% endhighlight %}

Tras ello, ya podremos ejecutar el comando para que se lleve a cabo la sincronización, con la ruta **Capturas/** del proveedor.

{% highlight shell %}
alvaro@debian:~/Imágenes$ rclone sync ./ Dropbox:Capturas/
{% endhighlight %}

Para verificar que dichas capturas también se han eliminado del directorio remoto, vamos a acceder al directorio de Dropbox.

![sync2](https://i.ibb.co/98Y1MKF/Captura-de-pantalla-de-2020-10-01-21-25-27.png "rclone sync")

Efectivamente, todas las capturas nombradas con números han sido eliminadas del directorio remoto.

### Vista de árbol (tree)

---

La sintaxis de rclone para [ver](https://rclone.org/commands/rclone_tree/) en forma de árbol la estructura de ficheros y directorios de un proveedor es:

{% highlight shell %}
rclone tree <proveedor>:<ruta>
{% endhighlight %}

#### Drive

En este caso, voy a listar en forma de árbol absolutamente todo el contenido de mi Google Drive, dejando para ello la ruta vacía, así empezará a listar desde el directorio raíz.

{% highlight shell %}
alvaro@debian:~$ rclone tree Drive:
/
├── Capturas
│   ├── 10.png
│   ├── 11.png
│   ├── 12.png
...
├── ComandosLinux.odt
├── Criptografía.odt
├── Examen.odt
├── How to get started with Drive
├── Script.py
├── bashrc
├── drive.txt
├── hola.txt
├── lsblk.txt
└── packages_list

2 directories, 91 files
{% endhighlight %}

#### Dropbox

Al igual que en el caso anterior, voy a listar todo el contenido de mi Dropbox, dejando para ello la ruta vacía, así empezará a listar desde el directorio raíz.

{% highlight shell %}
alvaro@debian:~$ rclone tree Dropbox:
/
├── Capturas
│   ├── Captura de pantalla de 2020-10-01 20-08-47.png
│   ├── Captura de pantalla de 2020-10-01 20-09-16.png
│   ├── Captura de pantalla de 2020-10-01 20-10-48.png
...
├── ComandosLinux.odt
├── ComandosWindows.odt
├── Get Started with Dropbox Paper.url
├── Get Started with Dropbox.pdf
├── deberes.txt
├── dropbox.txt
└── ls.txt

2 directories, 58 files
{% endhighlight %}

### Crear directorio

---

La sintaxis de rclone para [crear](https://rclone.org/commands/rclone_mkdir/) un directorio vacío en un proveedor es:

{% highlight shell %}
rclone mkdir <proveedor>:<ruta>
{% endhighlight %}

En este caso, vamos a generar un directorio llamado **Pruebasync/** en cada uno de los proveedores, que nos servirán para la siguiente tarea.

#### Drive

{% highlight shell %}
alvaro@debian:~$ rclone mkdir Drive:Pruebasync/
{% endhighlight %}

Para verificar que se ha generado correctamente, accederé a Google Drive.

![mkdir1](https://i.ibb.co/wNP6XZg/Captura-de-pantalla-de-2020-10-01-21-28-13.png "rclone mkdir")

#### Dropbox

{% highlight shell %}
alvaro@debian:~$ rclone mkdir Dropbox:Pruebasync/
{% endhighlight %}

Para verificar que se ha generado correctamente, accederé a Dropbox.

![mkdir2](https://i.ibb.co/3YLYR1b/Captura-de-pantalla-de-2020-10-01-21-28-18.png "rclone mkdir")

Efectivamente, el directorio vacío ha sido correctamente generado en ambos proveedores.

## Tarea 4: Monta en un directorio local de tu ordenador, los ficheros de un proveedor cloud. Comprueba que copiando o borrando ficheros en este directorio se crean o eliminan en el proveedor.

Lo primero que tendremos que hacer será crear dos directorios locales donde montaremos los directorios del proveedor cloud. En este casó, lo haré dentro de **/mnt/**. El comando a ejecutar sería:

{% highlight shell %}
alvaro@debian:~$ mkdir /mnt/{drive,dropbox}
{% endhighlight %}

Tras ello, tendremos que hacer uso de la siguiente sintaxis para [montar](https://rclone.org/commands/rclone_mount/) directorios remotos:

{% highlight shell %}
rclone mount <proveedor>:<ruta> <puntomontaje> &
{% endhighlight %}

Gracias al **&** del final, conseguiremos ejecutar el proceso en background para así no bloquear la terminal actual. En este caso, voy a montar ambos directorios remotos **Pruebasync/** en las correspondientes rutas locales. Para ello, ejecutamos los comandos:

{% highlight shell %}
alvaro@debian:~$ rclone mount Drive:Pruebasync/ /mnt/drive/ &
[1] 11831
alvaro@debian:~$ rclone mount Dropbox:Pruebasync/ /mnt/dropbox/ &
[2] 11853
{% endhighlight %}

Una vez montados los directorios remotos, ya podemos empezar a trabajar con los mismos. Por ejemplo voy a crear ficheros de nombre **1.txt** hasta **10.txt** dentro del directorio **/mnt/drive/** y ficheros de nombre **1.txt** hasta **20.txt** dentro del directorio **/mnt/dropbox/**. Para ello, ejecutaremos los comandos:

{% highlight shell %}
alvaro@debian:~$ touch /mnt/drive/{1..10}.txt
alvaro@debian:~$ touch /mnt/dropbox/{1..20}.txt
{% endhighlight %}

Los ficheros están generados, pero ahora queda ver si se han sincronizado correctamente dentro de dichos directorios remotos. Para ello, vamos a acceder a ambos proveedores para ver si ha sido así.

![mount1](https://i.ibb.co/qk2ck0S/Captura-de-pantalla-de-2020-10-01-21-32-59.png "rclone mount")

![mount2](https://i.ibb.co/JK2ZYjY/Captura-de-pantalla-de-2020-10-01-21-33-09.png "rclone mount")

Efectivamente, en Google Drive hay ficheros desde **1.txt** hasta **10.txt** y en Dropbox hay ficheros desde **1.txt** hasta **20.txt** (aunque en las capturas no se vea al completo).

Por último, voy a eliminar los 5 primeros ficheros del directorio **/mnt/drive/** y todos los ficheros del directorio **/mnt/dropbox/** para mover dentro de este último un fichero que acabo de descargar, para ver si se sincroniza correctamente también. Los comandos a ejecutar serían:

{% highlight shell %}
alvaro@debian:~$ rm /mnt/drive/[1-5].txt
alvaro@debian:~$ rm /mnt/dropbox/*
alvaro@debian:~$ mv /home/alvaro/Descargas/packages_list ./
{% endhighlight %}

De nuevo, volveremos a acceder a los directorios de los proveedores para ver si la sincronización ha sido correcta.

![mount3](https://i.ibb.co/27jpjxV/Captura-de-pantalla-de-2020-10-01-21-36-18.png "rclone mount")

![mount4](https://i.ibb.co/D9y1QtZ/Captura-de-pantalla-de-2020-10-01-21-36-21.png "rclone mount")

De nuevo, la sincronización se ha llevado a cabo correctamente, pues en Drive se han borrado los 5 primeros ficheros y en Dropbox, se han borrado todos, existiendo ahora únicamente el fichero **packages_list**, que he movido manualmente.