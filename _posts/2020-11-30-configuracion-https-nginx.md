---
layout: post
title:  "Configuración de HTTPS en Nginx"
banner: "/assets/images/banners/https.jpg"
date:   2020-11-30 17:03:00 +0200
categories: seguridad vps
---
Este es el tercer artículo referente a la configuración del VPS, una máquina virtual contratada en OVH que cuenta con un direccionamiento público **51.210.109.246**. Además, se ha contratado una zona DNS en la que nombraremos dicha máquina junto a los servicios que despleguemos, en el dominio **iesgn19.es**.

La intención es la de ir desarrollando a lo largo del curso una serie de _posts_ en los que se detallen las configuraciones llevadas a cabo en dicho VPS, así como el mantenimiento de las mismas. En el día de hoy, vamos a configurar el protocolo HTTPS en los dos sitios web actualmente desplegados en dicha máquina.

Se recomienda realizar una previa lectura de los _post_ [Migración de aplicaciones web PHP](https://www.alvarovf.com/aplicaciones/vps/2020/11/19/migracion-aplicaciones-php.html) y [Certificados digitales y HTTPS](https://www.alvarovf.com/seguridad/2020/11/21/certificados-digitales-https.html), pues en este artículo se tratarán con menor profundidad u obviarán algunos aspectos vistos con anterioridad.

Para lograr el objetivo anteriormente mencionado, haremos uso de _Secure Sockets Layer_ (**SSL**), un protocolo que proporciona la posibilidad de cifrar el contenido que transmitimos en cualquier aplicación basada en **TCP** (_HTTP_, _SMTP_...), es decir, ofrece seguridad en la capa de transporte.

Dicho protocolo se utiliza entre navegadores de Internet y servidores, e incluso entre servidores, es por ello que en aquellos sitios web que tienen dicha característica correctamente configurada, accedemos a través de **https://**.

Los servidores web tienen un certificado emitido por alguna autoridad de certificación de confianza. El certificado contiene la clave pública del servidor y es enviado al cliente en el momento de la petición.

Dicho certificado sirve para ponerse de acuerdo con el cliente de forma asimétrica de la clave de cifrado que van a utilizar para la comunicación, ya que sería muy costoso computacionalmente mantener una comunicación utilizando cifrado asimétrico. El cliente elegirá y cifrará dicha clave simétrica utilizando la clave pública del servidor. Una vez cifrada, se la enviará de vuelta al servidor, siendo éste el único capaz de descifrar dicha clave, pues es quién posee la clave privada asociada. A partir de éste momento, toda la sesión está cifrada simétricamente.

Además del cifrado de datos, otro de los usos de dicho certificado es el de autenticar al servidor, garantizando al usuario que aquel servidor al que se conecta es quién dice ser. Para ello, los navegadores web incluyen una lista de claves públicas de autoridades de certificación (_CA_) de confianza, de manera que gracias a la misma, se puede comprobar la firma del certificado enviado por el servidor. En caso de que no se pueda verificar dicha firma por parte de la _CA_, el navegador mostrará una advertencia al usuario, que podrá continuar bajo su propia responsabilidad, aunque la conexión seguirá siendo cifrada.

Ahora es cuando llega el gran problema. Como acabamos de mencionar, necesitamos que una autoridad certificadora de confianza (la suficiente como para que los navegadores depositen la clave pública de dicha entidad en su lista de claves con confianza, de manera que la advertencia no sea mostrada al usuario, pues la firma se habrá podido comprobar correctamente), ¿pero quién nos va a firmar un certificado de forma gratuita?

Basado en ésta última premisa, surge **[Let's Encrypt](https://letsencrypt.org/)**, una autoridad de certificación libre y gratuita impulsada por la **Fundación Linux**, que permite generar certificados SSL **gratuitos y automáticos** para nuestros sitios web. Su objetivo es el de promover que el tráfico de Internet sea seguro. Al ser gratuito y su método de validación bastante "simple", la confianza que ofrece dicho certificado a nuestros clientes es muy limitada (como se puede suponer, una entidad bancaria no va a hacer uso de dicha autoridad certificadora, pero a nosotros nos servirá para salir del paso).

El funcionamiento de _Let's Encrypt_ es bastante sencillo a la vez que complejo, es decir, para un usuario normal y corriente, el procedimiento a seguir es totalmente trivial y en pocos segundos podrá tener su certificado SSL firmado y funcionando, mientras que internamente, se están llevando a cabo de forma automática varios trámites, que generalmente son totalmente ajenos al mismo. Es por ello que decimos que ésta autoridad certificadora hace uso del protocolo **ACME** (_Automatic Certificate Management Environment_), consistente en dos pasos claramente diferenciados, los cuales llevará a cabo un agente llamado **[Certbot](https://certbot.eff.org/)**:

- **Validación del dominio**: La primera vez que el _software_ del agente que previamente hemos instalado (_Certbot_) se comunica con _Let's Encrypt_, genera un nuevo par de claves. Tras ello, tendremos que demostrar a la autoridad certificadora que realmente somos administradores del dominio para el que queremos generar el certificado SSL, ya que si no, cualquier persona podría obtener certificados para dominios y máquinas que no administra. Para llevar a cabo dicha validación, contamos con dos posibilidades (retos):

    - **HTTP-01 challenger**: Consiste en colocar un fichero con un determinado contenido en una ruta concreta del servidor, que posteriormente _Let's Encrypt_ podrá verificar (es necesario que el nombre de dominio esté correctamente configurado en la zona DNS). De una forma más interna, _Let's Encrypt_ envía un _token_ a _Certbot_, y pedirá que lo incluya en un fichero determinado, en una ruta accesible en el servidor web. Además de dicho _token_, se guarda la firma del mismo realizada con la clave privada del agente. En caso de que la autoridad certificadora sea capaz de acceder al puerto 80, obtener dicho fichero y validar la firma, podrá asegurar que somos los administradores del dominio. Únicamente permite crear certificados para un registro concreto de la zona DNS. Es el que nosotros utilizaremos.

    - **DNS-01 challenger**: Consiste en crear un registro en la zona DNS con una determinada información. A diferencia del método de validación anterior, podemos crear certificados **_wildcard_**, es decir, un certificado válido para __*.iesgn19.es__, ya que habrá podido comprobar que administramos la zona DNS en su totalidad, mientras que el método anterior únicamente comprobaba que somos administradores de uno de los registros existentes en la zona DNS.

- **Solicitud del certificado**: Una vez que hemos completado la validación del dominio, _Certbot_ generará un fichero de solicitud de firma de certificado (**.csr**) que será firmado por el agente y posteriormente enviado a _Let's Encrypt_ para que emita un certificado para dicho dominio. Una vez que la autoridad certificadora lo recibe, verifica la firma previamente realizada, haciendo uso para ello de la clave pública del agente. En caso de que todo haya ido bien, emitirá el correspondiente certificado.

Como se puede apreciar, y gracias a _Certbot_, el proceso que acabo de explicar que a primera vista puede parecer algo tedioso, se lleva a cabo en cuestión de segundos. Dicho agente, hace uso además de varios _[plugins](https://certbot.eff.org/docs/using.html#getting-certificates-and-choosing-plugins)_ que nos permiten llevar la automatización un paso más allá, configurando incluso el servidor web para que haga uso de dicho certificado obtenido, de nombres **nginx** o **apache2**, dependendiendo del caso. Sin embargo, en el otro lado de la balanza, tenemos aquellos _plugins_ que únicamente llevan a cabo el proceso de obtención del certificado, pero no modifican la configuración del servidor web, siendo dicha parte, tarea del administrador, de nombres **webroot** o **standalone**.

De todos los plugins mencionados, únicamente **standalone** requiere parar el servidor web en caso de tenerlo activo, ya que _Certbot_ hospeda su propio servidor web en el puerto 80 para llevar a cabo la validación, que en éste caso no es algo influyente, pero en otras situaciones, puede llegar a serlo. La principal finalidad de dicho _plugin_ es la de poder generar un certificado incluso para una máquina que no tenga un servidor web instalado.

En mi opinión, ya ha sido suficiente explicación teórica, así que vamos a proceder con la parte práctica, en la que vamos a conseguir que nuestros dos sitios web hagan uso de HTTPS. El primer paso consistirá en instalar dicho agente que nos permitirá automatizar el proceso, **_Certbot_**, ejecutando para ello el comando (con permisos de administrador, haciendo uso del comando `sudo su -`):

{% highlight shell %}
root@vps:~# apt install certbot
{% endhighlight %}

Es importante tener claro en todo momento qué es lo que queremos hacer y la infraestructura de la que disponemos. En mi caso, quiero configurar HTTPS para los sitios **www.iesgn19.es** y **portal.iesgn19.es**, que han sido previamente definidos en la zona DNS y existe un servidor web _nginx_ activo con dichos ServerName configurados.

Para esta ocasión, vamos a hacer uso del plugin **standalone**, para que el nivel de automatización no sea total y podamos toquetear un poco por nosotros mismos, modificando los ficheros de configuración del servidor web que utilizamos. Como anteriormente he mencionado, vamos a hacer uso de dicho _plugin_ ya que no tengo inconveniente en parar el servidor web durante unos segundos, pero en caso de haberlo, habría que hacer uso de **webroot**, o incluso de **nginx**, si quisiésemos un nivel de automatización aún mayor.

Como es lógico, lo primero que haremos será dejar libre el puerto 80 para que sea usado por el servidor web que hospeda automáticamente _Certbot_, parando para ello nuestro propio servidor web **_nginx_**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl stop nginx
{% endhighlight %}

Una vez se encuentre libre el puerto 80, podremos empezar con el proceso. La gran característica de [_Certbot_](https://certbot.eff.org/) es que nos permite seleccionar en su página web qué tipo de servidor web vamos a utilizar y en qué sistema operativo lo estamos alojando, para mostrarnos así una serie de instrucciones bastante detalladas sobre el procedimiento a seguir, aunque como anteriormente he mencionado, es bastante trivial.

Vamos a comenzar por generar el certificado para **www.iesgn19.es**, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# certbot certonly --standalone -d www.iesgn19.es
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator standalone, Installer None
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel):
{% endhighlight %}

Donde:

* **certonly**: Indica que únicamente queremos generar el certificado, de manera que no se configurará automáticamente el servidor web.
* **--standalone**: Indicamos el plugin que queremos utilizar durante el proceso. En éste caso, **standalone**.
* **-d**: Indicamos el nombre de dominio para el que queremos obtener el certificado, que como anteriormente he mencionado, es necesario que esté creado el correspondiente registro en la zona DNS apuntando a la máquina actual. En éste caso, **www.iesgn19.es**.

Lo primero que nos pedirá será un correo electrónico para que la autoridad certificadora tenga un método para contactar con nosotros e informarnos sobre determinados aspectos, así que simplemente lo introduciremos y continuaremos:

{% highlight shell %}
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): avacaferreras@gmail.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel:
{% endhighlight %}

Tras ello, nos pedirán que leamos los términos y condiciones del servicio, así que "tras leerlos", los aceptaremos (**A**) y continuaremos con el proceso:

{% highlight shell %}
(A)gree/(C)ancel: A

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
{% endhighlight %}

Por último, nos preguntarán si tienen nuestro permiso para compartir nuestro correo electrónico para enviarnos campañas, noticias... en mi caso, dije que no (**N**) y tras continuar, comenzó la generación de mi certificado para **www.iesgn19.es**:

{% highlight shell %}
(Y)es/(N)o: N
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for www.iesgn19.es
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/www.iesgn19.es/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/www.iesgn19.es/privkey.pem
   Your cert will expire on 2021-02-22. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
{% endhighlight %}

El proceso no duró más de 5 segundos, y mi certificado ya se encuentra generado dentro de **/etc/letsencrypt/live/www.iesgn19.es/**. Posteriormente explicaremos qué es cada uno de los ficheros que se han ubicado en dicho directorio, no sin antes repetir el mismo proceso para el otro sitio web, **portal.iesgn19.es**:

{% highlight shell %}
root@vps:~# certbot certonly --standalone -d portal.iesgn19.es
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator standalone, Installer None
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for portal.iesgn19.es
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/portal.iesgn19.es/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/portal.iesgn19.es/privkey.pem
   Your cert will expire on 2021-02-22. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
{% endhighlight %}

Listo, el nuevo certificado se ha generado en ésta ocasión dentro de **/etc/letsencrypt/live/portal.iesgn19.es/**, de manera que ya disponemos de ambos certificados, por lo que volveremos a arrancar nuestro servidor web nativo para que así vuelva a estar operativo, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# systemctl start nginx
{% endhighlight %}

Nuestro servidor web _nginx_ vuelve a estar operativo, así que volviendo al tema de los certificados, listaremos de forma gráfica y recursiva el contenido del directorio padre, en el que se encuentran los subdirectorios de todos los certificados, para así verificar que han sido correctamente generados, haciendo para ello uso del comando `tree`:

{% highlight shell %}
root@vps:~# tree /etc/letsencrypt/live/
/etc/letsencrypt/live/
├── portal.iesgn19.es
│   ├── cert.pem -> ../../archive/portal.iesgn19.es/cert1.pem
│   ├── chain.pem -> ../../archive/portal.iesgn19.es/chain1.pem
│   ├── fullchain.pem -> ../../archive/portal.iesgn19.es/fullchain1.pem
│   ├── privkey.pem -> ../../archive/portal.iesgn19.es/privkey1.pem
│   └── README
├── README
└── www.iesgn19.es
    ├── cert.pem -> ../../archive/www.iesgn19.es/cert1.pem
    ├── chain.pem -> ../../archive/www.iesgn19.es/chain1.pem
    ├── fullchain.pem -> ../../archive/www.iesgn19.es/fullchain1.pem
    ├── privkey.pem -> ../../archive/www.iesgn19.es/privkey1.pem
    └── README

2 directories, 11 files
{% endhighlight %}

Como se puede apreciar, para cada uno de los sitios web se han generado varios ficheros, que procedo a explicar a continuación:

* **cert.pem**: Es el certificado como tal, que en consecuencia, contiene nuestra clave pública.
* **chain.pem**: Contiene el certificado intermedio, es decir, el certificado de _Let's Encrypt_ (que contiene su clave pública), asociado a la clave privada con la que han firmado nuestro certificado. Es necesario para que los clientes puedan comprobar la firma en nuestro certificado, especialmente en aquellas aplicaciones que no sean navegadores (pues ellos suelen contener ya dicha clave pública almacenada), como puede ser la aplicación de escritorio de _Nextcloud_.
* **fullchain.pem**: Es la concatenación de los ficheros **cert.pem** y **chain.pem** en un único fichero. En la mayoría de ocasiones especificaremos dicho fichero como certificado a servir, de manera que todo lo necesario será enviado en un único fichero.
* **privkey.pem**: Es nuestra clave privada asociada al certificado existente, que debemos mantener asegurada, pues es utilizada a la hora de la negociación de la clave de cifrado simétrico con el cliente.

Una vez explicados todos los ficheros, podríamos proceder con la configuración del servidor web para que utilice HTTPS, no sin antes hacer uso de un comando que proporciona _Certbot_ para ver los certificados existentes y su validez restante. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@vps:~# certbot certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: portal.iesgn19.es
    Domains: portal.iesgn19.es
    Expiry Date: 2021-02-22 11:46:11+00:00 (VALID: 84 days)
    Certificate Path: /etc/letsencrypt/live/portal.iesgn19.es/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/portal.iesgn19.es/privkey.pem
  Certificate Name: www.iesgn19.es
    Domains: www.iesgn19.es
    Expiry Date: 2021-02-22 11:45:20+00:00 (VALID: 84 days)
    Certificate Path: /etc/letsencrypt/live/www.iesgn19.es/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/www.iesgn19.es/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
{% endhighlight %}

Como se puede apreciar, nos ha mostrado los dos certificados existentes, junto al dominio que han validado y la ruta de los ficheros correspondientes al certificado y la clave privada. Por último, se muestra también la fecha de expiración de los mismos, que aunque ahí aparezca que son 84 días, en realidad son 3 meses (**90 días**), ya que los certificados los generé hace casi una semana.

Muy probablemente os estareís preguntando, ¿y cuando expiren dichos certificados, tendré que repetir el mismo procedimiento? La respuesta es un rotundo **NO**, ya que _Certbot_ es tan inteligente que ha programado una tarea **cron** para renovarlos automáticamente sin interacción del usuario, y que así nos podamos despreocupar totalmente. Dicha tarea la podremos encontrar dentro del directorio **/etc/cron.d/** con el nombre **certbot**, así que visualizaremos su contenido haciendo uso de `cat`:

{% highlight shell %}
root@vps:~# cat /etc/cron.d/certbot
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew
{% endhighlight %}

Como se puede apreciar en la última línea, existe una tarea que cada 12 horas comprueba si al certificado le quedan menos de 30 días de validez, gracias al comando **certbot -q renew**, de manera que en caso de así ser, lo renovará de forma silenciosa, sin generar ningún tipo de salida, gracias a la opción **-q**.

Tras ello, ya podríamos comenzar con la configuración del servidor web, moviéndonos para ello al directorio en el que se encuentran contenidos los ficheros de configuración de los VirtualHost, ejecutando para ello el comando:

{% highlight shell %}
root@vps:~# cd /etc/nginx/sites-available/
{% endhighlight %}

Una vez dentro del mismo, listaremos el contenido existente para así comprobar los VirtualHost que tenemos:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# ls
default  drupal  iesgn19
{% endhighlight %}

De los tres ficheros existentes, únicamente nos interesan:

* **drupal**: VirtualHost correspondiente al sitio web **portal.iesgn19.es**.
* **iesgn19**: VirtualHost correspondiente al sitio web **www.iesgn19.es**.

Comenzaremos por modificar el fichero **iesgn19**, por lo que haremos uso del comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# nano iesgn19
{% endhighlight %}

Dentro del mismo, tendremos que crear dos directivas **server**, una para el VirtualHost accesible en el puerto 80 HTTP, dentro del cuál únicamente asignaremos el ServerName correspondiente y una redirección permanente para así forzar HTTPS. En la otra, para el VirtualHost accesible en el puerto 443 HTTPS, tendremos que configurar las siguientes directivas:

* **server_name**: Indicaremos el nombre de dominio a través del cuál accederemos al servidor.
* **ssl**: Activa el motor SSL, necesario para hacer uso de HTTPS, por lo que su valor debe ser **on**.
* **ssl_certificate**: Indicamos la ruta del certificado del servidor firmado por la _CA_. En este caso, **/etc/letsencrypt/live/www.iesgn19.es/fullchain.pem**.
* **ssl_certificate_key**: Indicamos la ruta de la clave privada asociada al certificado del servidor. En este caso, **/etc/letsencrypt/live/www.iesgn19.es/privkey.pem**.

Como es lógico, toda aquella configuración previamente existente en el bloque del VirtualHost accesible en el puerto 80 HTTP, pasará ahora al bloque del VirtualHost accesible en el puerto 443 HTTPS, de manera que el resultado final sería:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        server_name www.iesgn19.es;

        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        ssl    on;
        ssl_certificate    /etc/letsencrypt/live/www.iesgn19.es/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/www.iesgn19.es/privkey.pem;

        root /srv/iesgn19;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name www.iesgn19.es;

        rewrite ^/$ /principal redirect;

        location / {
                try_files $uri $uri/ =404;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        location /.well-known {
            rewrite ^/\.well-known/host-meta\.json  /cloud/public.php?service=host-meta-json    last;
            rewrite ^/\.well-known/host-meta        /cloud/public.php?service=host-meta         last;
            rewrite ^/\.well-known/webfinger        /cloud/public.php?service=webfinger         last;
            rewrite ^/\.well-known/nodeinfo         /cloud/public.php?service=nodeinfo          last;

            location = /.well-known/carddav   { return 301 /cloud/remote.php/dav/; }
            location = /.well-known/caldav    { return 301 /cloud/remote.php/dav/; }

            try_files $uri $uri/ =404;
        }

        location ^~ /cloud {
            client_max_body_size 512M;
            fastcgi_buffers 64 4K;

            gzip on;
            gzip_vary on;
            gzip_comp_level 4;
            gzip_min_length 256;
            gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
            gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

            add_header Referrer-Policy                      "no-referrer"   always;
            add_header X-Content-Type-Options               "nosniff"       always;
            add_header X-Download-Options                   "noopen"        always;
            add_header X-Frame-Options                      "SAMEORIGIN"    always;
            add_header X-Permitted-Cross-Domain-Policies    "none"          always;
            add_header X-Robots-Tag                         "none"          always;
            add_header X-XSS-Protection                     "1; mode=block" always;

            fastcgi_hide_header X-Powered-By;

            index index.php index.html /cloud/index.php$request_uri;

            expires 1m;

            location = /cloud {
                if ( $http_user_agent ~ ^DavClnt ) {
                    return 302 /cloud/remote.php/webdav/$is_args$args;
                }
            }

            location ~ ^/cloud/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)    { return 404; }
            location ~ ^/cloud/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

            location ~ \.php(?:$|/) {
                fastcgi_split_path_info ^(.+?\.php)(/.*)$;
                set $path_info $fastcgi_path_info;

                try_files $fastcgi_script_name =404;

                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $path_info;
                fastcgi_param HTTPS on;

                fastcgi_param modHeadersAvailable true;
                fastcgi_param front_controller_active true;
                fastcgi_pass unix:/run/php/php7.3-fpm.sock;

                fastcgi_intercept_errors on;
                fastcgi_request_buffering off;
            }

            location ~ \.(?:css|js|svg|gif)$ {
                try_files $uri /cloud/index.php$request_uri;
                expires 6M;
                access_log off;
            }

            location ~ \.woff2?$ {
                try_files $uri /cloud/index.php$request_uri;
                expires 7d;
                access_log off;
            }

            location /cloud {
                try_files $uri $uri/ /cloud/index.php$request_uri;
            }
        }
}
{% endhighlight %}

Tras ello, guardaremos los cambios y repetiremos el mismo procedimiento para el fichero **drupal**, correspondiente al sitio web **portal.iesgn19.es**, estableciendo ahora la ruta para el certificado en **/etc/letsencrypt/live/portal.iesgn19.es/fullchain.pem** y la ruta para la clave privada en **/etc/letsencrypt/live/portal.iesgn19.es/privkey.pem**, quedando de la siguiente manera:

{% highlight shell %}
server {
        listen 80;
        listen [::]:80;

        server_name portal.iesgn19.es;

        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        ssl    on;
        ssl_certificate    /etc/letsencrypt/live/portal.iesgn19.es/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/portal.iesgn19.es/privkey.pem;

        root /srv/drupal;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name portal.iesgn19.es;

        location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        location ~* \.(txt|log)$ {
            deny all;
        }

        location ~ \..*/.*\.php$ {
            return 403;
        }

        location ~ ^/sites/.*/private/ {
            return 403;
        }

        location ~ ^/sites/[^/]+/files/.*\.php$ {
            deny all;
        }

        location ~* ^/.well-known/ {
            allow all;
        }

        location ~ (^|/)\. {
            return 403;
        }

        location / {
            try_files $uri /index.php?$query_string;
        }

        location @rewrite {
            rewrite ^ /index.php;
        }

        location ~ /vendor/.*\.php$ {
            deny all;
            return 404;
        }

        location ~* \.(engine|inc|install|make|module|profile|po|sh|.*sql|theme|twig|tpl(\.php)?|xtmpl|yml)(~|\.sw[op]|\.bak|\.orig|\.save)?$|/(\.(?!well-known).*|Entries.*|Repository|Root|Tag|Template|composer\.(json|lock)|web\.config)$|/#.*#$|\.php(~|\.sw[op]|\.bak|\.orig|\.save)$ {
            deny all;
            return 404;
        }

        location ~ '\.php$|^/update.php' {
            fastcgi_split_path_info ^(.+?\.php)(|/.*)$;

            try_files $fastcgi_script_name =404;

            include fastcgi_params;

            fastcgi_param HTTP_PROXY "";
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_param QUERY_STRING $query_string;
            fastcgi_intercept_errors on;

            fastcgi_pass unix:/run/php/php7.3-fpm.sock;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            try_files $uri @rewrite;
            expires max;
            log_not_found off;
        }

        location ~ ^/sites/.*/files/styles/ {
            try_files $uri @rewrite;
        }

        location ~ ^(/[a-z\-]+)?/system/files/ {
            try_files $uri /index.php?$query_string;
        }

        if ($request_uri ~* "^(.*/)index\.php/(.*)") {
            return 307 $1$2;
        }
}
{% endhighlight %}

Una vez más, guardaremos los cambios, y para que surtan efecto, tendremos que volver a cargar la configuración del servicio _nginx_, ejecutando para ello el comando:

{% highlight shell %}
root@vps:/etc/nginx/sites-available# systemctl reload nginx
{% endhighlight %}

Tras ello, ya estará todo listo para acceder a **www.iesgn19.es** y a **portal.iesgn19.es** desde el navegador, anteponiendo **http://**, para así verificar que la redirección al VirtualHost existente en el puerto 443 HTTPS se lleva a cabo correctamente:

![https1](https://i.ibb.co/f0XyXbf/principal.jpg "www.iesgn19.es")

![https2](https://i.ibb.co/r704MZY/drupal.jpg "portal.iesgn19.es")

Efectivamente, las redirecciones para forzar el uso de HTTPS se han llevado a cabo correctamente y no se ha mostrado ninguna advertencia de seguridad, por lo que podemos asegurar que el navegador ha podido comprobar la firma de la autoridad certificadora, de manera que la conexión se encuentra ahora cifrada y nuestros clientes podrán tener ese plus de seguridad y confianza a la hora de navegar en nuestro sitio web. Por curiosidad, pulsaremos en los correspondientes candados para mostrar los detalles de los certificados:

![https3](https://i.ibb.co/Qr1VfCF/principal2.jpg "www.iesgn19.es")

![https4](https://i.ibb.co/3mXqwGr/drupal2.jpg "portal.iesgn19.es")

Como se puede apreciar, se han mostrado algunos detalles básicos sobre los mismos, así como el dominio para el que han sido emitidos, la autoridad certificadora y la fecha de validez.

La configuración del servidor web para hacer uso de HTTPS ha llegado hasta aquí, pero antes de finalizar, me gustaría comprobar si la aplicación de escritorio de _Nextcloud_ configurada en el anterior artículo sigue funcionando, así que simplemente accederé a la misma y veré la información que se muestra:

![nextcloud](https://i.ibb.co/34MVhV5/nextcloud.jpg "Nextcloud")

Como se puede apreciar, la redirección desde **http://www.iesgn19.es** a **https://www.iesgn19.es** ha sido efectiva una vez más y la aplicación de escritorio de _Nextcloud_ ha sido capaz de seguirla, estando ahora configurada para ofrecernos el servicio a través de HTTPS.