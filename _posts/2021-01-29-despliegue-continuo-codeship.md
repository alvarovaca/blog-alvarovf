---
layout: post
title:  "Despliegue continuo con Jekyll y Codeship"
banner: "/assets/images/banners/despliegue.png"
date:   2021-01-29 12:11:00 +0200
categories: aplicaciones
---
Es muy probable que alguna vez hayas tenido que desplegar las aplicaciones que desarrollas de forma manual, haciéndole una serie de pruebas unitarias para comprobar su correcto funcionamiento antes de ponerlas en producción, tratando de arreglarlos lo antes posible para volver a llevar a cabo dichas pruebas, hasta que finalmente funcione, y pase de desplegarse en un entorno de desarrollo a hacerlo en producción.

Estos pasos anteriormente mencionados consumen en general, una gran cantidad de tiempo, a la vez de ser tediosos, al tener que llevar a cabo un trabajo repetitivo (repetir las mismas pruebas) para tener la certeza de que el _software_ funciona a la perfección. Es por ello que surge la **integración**, **entrega** y **despliegue** continuo:

* **Integración continua (CIntegration)**: Hace énfasis en la ejecución automática de las pruebas unitarias. De esta forma se garantiza que los errores se detecten en una etapa temprana en el desarrollo, con lo cual se facilita la corrección de los mismos. Los desarrolladores realizan _merges_ de forma constante en el repositorio de GitHub, de manera que el equipo de desarrollo estará siempre trabajando con una versión actualizada del proyecto, cada _commit_ será una posible versión y por supuesto, al estar actualizando el repositorio en una forma constante, evitaremos que salgan a relucir cientos de conflictos al realizar el _merge_.

* **Entrega continua (CDelivery)**: Es una extensión de la integración continua, pues permite que nuestro proyecto esté listo para que el usuario final hago uso de él (al realizar un _commit_ se llevan a cabo las pruebas, se compila el código, migraciones...), sin embargo, despliegue a producción no se realiza, pues tendrá que llevarse a cabo de forma manual. La gran ventaja es que nos permite reducir la cantidad de errores al momento de realizar el _deploy_ de la aplicación, puesto que todos los pasos para la liberación fueron ejecutados anteriormente.

* **Despliegue continuo (CDeploy)**: Si en la entrega continua nos quedamos a un solo paso para que el usuario final utilice el producto, con despliegue continuo todo el ciclo de entrega se completa. Al realizar un _commit_ sobre la rama principal, se ejecutarán de forma automática todos los pasos necesarios para el despliegue del proyecto a producción. Esto se traduce en que el usuario final estará utilizando un producto en constante actualización.

![diagrama1](https://i.ibb.co/YW0phc5/Captura3.jpg "Diagrama ciclo")

Para tratar de aclarar estos conceptos, vamos a llevar a cabo un ejemplo práctico que a mi personalmente, me puede llegar a ser bastante útil. El blog en el que se ha publicado este artículo ha sido generado mediante un generador de páginas estáticas muy conocido, de nombre [Jekyll](https://jekyllrb.com/), en el que se escriben los posts en MarkDown (**.md**) y en el momento de subir un nuevo artículo, se vuelve a generar la página en sí (transformando dichos **.md** en **.html**) y se sube donde se considere oportuno. La instalación de dicho generador es bastante sencilla, por lo que vamos a obviarlo en este artículo.

Es un proceso que a pesar de no ser muy tedioso, nos puede servir perfectamente para entender el concepto de **despliegue continuo**, pues conseguiremos ahorrar por tanto unos minutos al día.

Dado que el contenido con el que estamos trabajando no es algo demasiado crítico, no será necesario un proceso de **_debugging_**, por lo que no llevaremos a cabo pruebas sobre el mismo, sin embargo, si en lugar de una página estática se tratase de una aplicación web a la que hemos añadido una nueva funcionalidad, sería totalmente conveniente ejecutar una batería de pruebas sobre el nuevo código desarrollado.

Actualmente, me atrevería a decir que existen cientos de herramientas que nos permiten automatizar dicho trabajo, siendo algunas de las más conocidas [Jenkins](https://www.jenkins.io/), [Travis CI](https://www.travis-ci.com/) y [CircleCI](https://circleci.com/). Sin embargo, tras una pequeña búsqueda, me he topado con [CodeShip](https://www.cloudbees.com/products/codeship), que me ha llamado bastante la atención la gran flexibilidad que nos ofrece y he decidido darle una oportunidad.

Lo primero que necesitaremos será un [repositorio](https://github.com/alvarovaca/blog-alvarovf) en GitHub que contenga, en este caso, todo lo necesario para que nuestro generador de páginas estáticas pueda hacer su trabajo, pues dicho repositorio será clonado por **CodeShip** en el momento de hacer un **_push_** y se llevarán a cabo las acciones que nosotros especifiquemos de forma explícita.

Tras ello, todo estará listo para crear nuestra cuenta en [CodeShip](https://app.codeship.com/registrations/new). Mi recomendación es que por comodidad, dicho registro se realice vinculando la cuenta de GitHub.

![cuenta1](https://i.ibb.co/KK5prFD/Captura-de-pantalla-de-2021-01-30-11-12-51.png "Creación cuenta")

El proceso de creación de la misma es totalmente trivial y su explicación se saldría del objetivo del artículo. Una vez que la cuenta haya sido correctamente generada, tendremos que crear un nuevo proyecto pulsando para ello en "**New Project**". Acto seguido, se nos preguntará el proveedor que queremos utilizar, que en este caso, será **GitHub**:

![proyecto1](https://i.ibb.co/QC1fZdf/Captura-de-pantalla-de-2021-01-30-11-22-12.png "Selección proveedor")

Una vez elegido el proveedor, se nos pedirá vincular el repositorio en cuestión, sin embargo, será necesario instalar la **CodeShip GitHub App** antes de ello, para así otorgar a dicha herramienta los privilegios necesarios sobre el repositorio en cuestión, ya que actualmente no podemos elegir ningún repositorio, tal y como se puede apreciar a continuación:

![proyecto2](https://i.ibb.co/ZBc8WXc/Captura-de-pantalla-de-2021-01-30-11-22-20.png "Selección repositorio")

Para otorgar dichos privilegios, tendremos que pulsar en "**installed the CodeShip GitHub App**", que nos ofrecerá la posibilidad de instalar dicha aplicación para todos nuestros repositorios o para uno en específico, que en mi caso, lo instalaré únicamente para el repositorio de nombre **blog-alvarovf**, ya que es el único para el que voy a utilizar esta herramienta:

![proyecto3](https://i.ibb.co/7jtR5jS/Captura-de-pantalla-de-2021-01-30-12-56-17.png "Selección repositorio")

Tras ello, ya podremos vincular el repositorio elegido de la siguiente forma, al estar correctamente instalada la aplicación necesaria para ello:

![proyecto4](https://i.ibb.co/WDBy5hZ/Captura-de-pantalla-de-2021-01-30-12-56-28.png "Selección repositorio")

Por último, tendremos que elegir tipo de proyecto que queremos desplegar, ofreciéndonos dos opciones, una de pago y otra gratuita. Para lo que nosotros necesitamos y pretendremos hacer, nos sobrará con la opción gratuita (**CodeShip Basic**):

![proyecto5](https://i.ibb.co/ygppdft/Captura-de-pantalla-de-2021-01-30-12-56-35.png "Selección tipo de proyecto")

La creación del proyecto ya ha finalizado, sin embargo, actualmente se encuentra totalmente inservible, ya que no le hemos especificado las acciones a llevar a cabo para cada despliegue. Lo primero que tendremos que añadir son los comandos que se ejecutarán para construir correctamente el entorno de pruebas (**Setup Commands**), es decir, en este caso nos servirá para instalar **Jekyll**.

Es necesario conocer que **Jekyll** está desarrollado en **Ruby**, de manera que con la finalidad de conseguir que nuestro entorno sea lo más similar posible al que va a ser generado, vamos a averiguar la versión que estamos utilizando, para así instalar esa misma versión en el nuevo entorno efímero, ejecutando para ello el comando:

{% highlight shell %}
alvaro@debian:~$ ruby -v
ruby 2.5.5p157 (2019-03-15 revision 67260) [x86_64-linux-gnu]
{% endhighlight %}

Como se puede apreciar, la versión de **Ruby** instalada es la **2.5.5**, de manera que ya podemos proceder a especificar los comandos que se deberán llevar a cabo para la correspondiente generación del entorno de pruebas:

{% highlight shell %}
rvm use 2.5.5 --install             # Instalamos la versión 2.5.5 de Ruby.
gem install bundler jekyll          # Instalamos bundler y Jekyll, necesarios para el correcto funcionamiento.
bundle install                      # Instalamos las dependencias existentes en el Gemfile.
{% endhighlight %}

El resultado final sería:

![proyecto6](https://i.ibb.co/JxMjjsZ/Captura-de-pantalla-de-2021-01-30-12-57-07-copia.png "Comandos iniciales")

Tras ello, tendremos que indicar además los comandos que se tendrán que llevar a cabo en forma de _test_ (en este caso, despliegue) una vez que la creación del entorno haya finalizado. Para ello, tendremos que crear un nuevo **_pipeline_** en la parte inferior, pulsando para ello en "**Add Pipeline**". El nombre que le asignemos no es determinante, por lo que en mi caso he utilizado el nombre "**Despliegue**", quedando de la siguiente forma:

![proyecto7](https://i.ibb.co/VHLLg5r/Captura-de-pantalla-de-2021-01-30-12-57-07.png "Creación pipeline")

Posteriormente, pulsaremos en "**Save**" seguido de "**Save Changes**" para así generar el _pipeline_ y poder escribir los comandos en cuestión en el mismo.

Todavía no lo he mencionado, pero el despliegue del sitio estático lo vamos a llevar a cabo en **Surge**, que nos permitirá _hostear_ de forma gratuita ofreciéndonos además un subdominio propio dentro de **surge.sh** para poder visualizarla en cualquier momento.

En realidad, este blog se encuentra alojado en un servidor virtual privado (VPS), pero como no todo el mundo dispone de uno, he decidido hacerlo de una forma más genérica, utilizando un servicio gratuito al alcance de todos. Quizás en un futuro me anime y haga un _post_ sobre un despliegue continuo en un VPS.

En este caso, los comandos que se deberán llevar a cabo para el despliegue continuo automatizado y que por tanto, tendremos que introducir en el _pipeline que acabamos de generar, serían los siguientes:

{% highlight shell %}
npm install --global surge          # Instalamos Surge, necesario para el despliegue.
bundle exec jekyll build            # Generamos el sitio final con Jekyll, que será desplegado.
surge _site/ alvarovf.surge.sh      # Desplegamos el sitio final generado en _site a Surge, en el subdominio alvarovf.surge.sh.
{% endhighlight %}

El resultado final sería:

![proyecto8](https://i.ibb.co/3TKvxtq/Captura-de-pantalla-de-2021-01-30-12-59-26.png "Comandos despliegue")

Por último, pulsaremos una vez más en "**Save Changes**" para guardar todos los cambios realizados hasta ahora en el proyecto.

Antes de continuar es necesario realizar un pequeño inciso, y es que necesitamos una cuenta en Surge para así poder realizar correctamente nuestro despliegue, para ello, volveré a la máquina anfitriona, siendo necesario haber llevado a cabo previamente la [instalación](https://surge.sh/help/getting-started-with-surge) de dicho _software_, que una vez más, obviaremos dada su gran sencillez.

Como es lógico, sería un gran agujero de seguridad indicar a CodeShip nuestra contraseña de Surge, es por ello que haremos uso de un **_token_** que nos proporcionará Surge y que será suficiente para que CodeShip lleve a cabo el despliegue de manera automatizada. Para ello, tendremos que hacer uso del siguiente comando:

{% highlight shell %}
alvaro@debian:~$ surge token

   Login (or create surge account) by entering email & password.

          email: avacaferreras@gmail.com
       password: 

   XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
{% endhighlight %}

Como se puede apreciar, hemos creado una nueva cuenta en Surge y se nos ha proporcionado un _token_, que como se puede suponer, he censurado por seguridad. Dicho _token_ tendremos que añadirlo como una variable de entorno en nuestro proyecto de CodeShip, desplazándonos al apartado **Environment** y añadiendo dos nuevas variables de entorno:

* **SURGE_LOGIN**: Indicamos la dirección de correo electrónico asociada a nuestra cuenta de Surge.
* **SURGE_TOKEN**: Indicamos el _token_ que nos ha proporcionado Surge.

El resultado final sería:

![proyecto9](https://i.ibb.co/rsX7842/Captura-de-pantalla-de-2021-01-30-13-01-25.png "Variables de entorno")

La configuración de nuestro proyecto en CodeShip ha finalizado, de manera que guardaremos todos los cambios pulsando para ello en "**Save and go to dashboard**", de manera que una vez finalizado el guardado, se nos mostrará el siguiente mensaje:

![proyecto10](https://i.ibb.co/gmkX8gb/Captura-de-pantalla-de-2021-01-30-13-02-24.png "Proyecto finalizado")

En dicho mensaje, se nos ha informado que todo está listo para que llevemos a cabo un _commit_ en nuestro repositorio y así comprobemos el correcto funcionamiento del despliegue continuo. Para ello, he realizado una pequeña modificación en uno de los posts previamente existentes, tal y como se puede apreciar si ejecuto el comando:

{% highlight shell %}
alvaro@debian:~/GitHub/blog-alvarovf$ git status
En la rama master
Tu rama está actualizada con 'origin/master'.

Cambios no rastreados para el commit:
  (usa "git add <archivo>..." para actualizar lo que será confirmado)
  (usa "git checkout -- <archivo>..." para descartar los cambios en el directorio de trabajo)

	modificado:     _posts/2021-01-28-cortafuegos-perimetral-nftables.md

sin cambios agregados al commit (usa "git add" y/o "git commit -a")
{% endhighlight %}

Tras ello, realizaremos un nuevo _commit_ para guardar el estado actual de los ficheros existentes en el repositorio en una nueva versión, haciendo para ello uso del comando:

{% highlight shell %}
alvaro@debian:~/GitHub/blog-alvarovf$ git commit -am "Prueba IC."
[master a2e1996] Prueba IC.
 1 file changed, 2 insertions(+), 2 deletions(-)
{% endhighlight %}

En el **_commit_** hemos establecido el mensaje "**Prueba IC.**", que servirá para diferenciarlo del resto, por lo que siempre es recomendable que sea descriptivo.

Nos queda el último paso, subir dichas modificaciones a nuestro repositorio GitHub, ya que actualmente, se encuentran almacenadas de forma local. Para ello, haremos uso del comando:

{% highlight shell %}
alvaro@debian:~/GitHub/blog-alvarovf$ git push
{% endhighlight %}

Nuestra labor en la máquina de desarrollo ha finalizado, de manera que volveremos a CodeShip para verificar si algo ha cambiado:

![proyecto11](https://i.ibb.co/LtFXF0s/Captura-de-pantalla-de-2021-01-30-13-05-29.png "Despliegue continuo")

Como era de esperar, el proyecto ha detectado el nuevo _commit_ realizado en nuestro repositorio y se encuentra ahora en estado "**RUNNING**", es decir, está generando el entorno de pruebas y ejecutando los comandos especificados con anterioridad. Si pulsamos en dicho _commit_, podremos apreciar en tiempo real cómo se ejecutan todos los comandos y se llevan a cabo las pruebas, en caso de haberlas especificado.

Tras esperar unos minutos (ya que al ser un servicio gratuito, nos ofrecen unos recursos limitados), el despliegue habrá finalizado exitosamente (en caso de haber indicado una batería de pruebas y no haberlas pasado, el despliegue continuo no se llevaría a cabo), tal y como se puede apreciar:

![proyecto12](https://i.ibb.co/svBfgSj/Captura.jpg "Despliegue continuo")

El estado es ahora "**SUCCEEDED**", por lo que tenemos la certeza de que la página ha sido correctamente desplegada, sin embargo, podemos contrastarlo también en nuestro repositorio GitHub:

![proyecto13](https://i.ibb.co/4fspcHx/Captura4.jpg "Despliegue continuo")

Al haberse ejecutado correctamente todos los comandos especificados y no haber existido ningún error, un _tick_ verde ha aparecido en el _commit_ subido a GitHub. Sin embargo, de nada nos sirve ver un _tick_ verde si la página no ha sido correctamente alojada en Surge, así que accederemos a **alvarovf.surge.sh** y comprobaremos el estado de la misma:

![proyecto14](https://i.ibb.co/xDYQNd6/Captura2.jpg "Despliegue continuo")

Efectivamente, la página se encuentra ahora alojada en Surge y cada vez que hagamos un nuevo _commit_ en nuestro repositorio GitHub, se generará un nuevo entorno de pruebas y se desplegará automáticamente el resultado de la generación de la misma, tal y como ha ocurrido en este caso.