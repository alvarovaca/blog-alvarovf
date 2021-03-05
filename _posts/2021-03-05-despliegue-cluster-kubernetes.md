---
layout: post
title:  "Despliegue de un cluster de Kubernetes"
banner: "/assets/images/banners/kubernetes.jpg"
date:   2021-03-05 11:10:00 +0200
categories: hlc
---
Los orquestadores de contenedores, como es el caso de **Kubernetes**, surgen dadas las claras limitaciones existentes en los contenedores, como por ejemplo la dificultad de cambiar entre versiones de una aplicación de una forma rápida, la dificultad de balancear la carga entre múltiples contenedores iguales, la necesidad de conectar contenedores que se ejecuten en diferentes demonios de Docker, la necesidad de actualizar una aplicación sin necesidad de dejar de ofrecer el servicio, la dificultad de mover la carga entre nodos...

Hasta hace varios años, existían tres alternativas para la orquestación, siendo todas ellas de _software_ libre:

* **Docker Swarm**
* **Apache Mesos**
* **Hashicorp Nomad**

Sin embargo, hoy en día se acepta generalmente que el vencedor de dicha batalla ha sido **Kubernetes**, aunque el resto de ellos siguen activos como alternativas más sencillas a **k8s** (_Kubernetes_) o en su propio nicho.

Gracias a este proyecto, vamos a poder gestionar el despliegue de aplicaciones sobre contenedores, automatizando dicho despliegue y haciendo un gran énfasis en la escalabilidad, controlando en todo momento su ciclo de vida. Despliegue rápido de aplicaciones, escalabilidad de las aplicaciones al vuelo, integración de cambios sin interrupciones y posibilidad de limitar los recursos a utilizar son algunas de las características más interesantes de esta tecnología.

Es un proyecto totalmente extensible, gracias a la gran multitud de módulos y _plugins_ que se ponen a nuestra total disposición y a través del cuál gestionaremos un _cluster_ de nodos en los que podremos, como previamente he mencionado, desplegar aplicaciones sobre contenedores.

Dentro de nuestro _cluster_ encontraremos una serie de componentes conocidos como _workers_, que son aquellas máquinas (virtuales o físicas) que ofrecerán la potencia de cómputo necesaria para desplegar nuestras aplicaciones. Todas ellas se encontrarán gestionadas por un nodo superior, conocido como _controller_. Cada uno de los nodos existentes tendrán una serie de características:

* **Direcciones**: _Hostname_, IP externa, IP interna...
* **Condición**: Campo que describe el estado del nodo (listo, sin red, sin disco...).
* **Capacidad**: Describe los recursos disponibles.
* **Info**: Información general (_kernel_, _software_ instalado, versiones...).

Existen una serie de proyectos propios de **k8s** que nos permiten realizar una instalación de **Kubernetes** de una forma muy sencilla, como pueden ser **Minikube**, **Kubeadm** o **k3s**. En este caso, utilizaremos **k3s**, una distribución de **Kubernetes** muy ligera que incluye a su vez todas las herramientas necesarias para gestionar nuestro _cluster_.

Entre las herramientas que se nos proporcionan se incluye **kubectl**, una herramienta de línea de comandos desarrollada en Go para gestionar nuestros _clusters_ de **k8s** de forma centralizada, permitiéndonos además, en caso de ser necesario, interactuar de forma directa con la API utilizando las credenciales del usuario.

Dado que **Kubernetes** es una tecnología que cambia de forma radical el concepto de despliegue de aplicaciones que teníamos hasta ahora, debemos conocer una serie de conceptos antes de empezar a trabajar con el mismo, ya que requiere un cambio de mentalidad para adaptarnos cuanto antes:

* **Pod**: Un _pod_ es la unidad más pequeña de _Kubernetes_, "equivalente" a un contenedor de _Docker_. Los _pods_, al igual que los contenedores de _Docker_, son efímeros, pues cuando se destruyen pierden toda la información que contenían. Si queremos que la información persista, debemos utilizar volúmenes.

* **ReplicaSet**: Un _ReplicaSet_ es un recurso de nivel superior que asegura que siempre se ejecute un número de réplicas de un _pod_ determinado, asegurándonos, por tanto, que un conjunto de _pods_ siempre estén funcionando y disponibles. Se podría resumir en que nos ofrece tolerancia a fallos y escalabilidad dinámica.

* **Deployment**: Un _deployment_ es la unidad de más alto nivel que podemos gestionar en _Kubernetes_, y nos ofrece control de réplicas, escalabilidad de _pods_, actualizaciones continuas, despliegues automáticos y _rollback_ a versiones anteriores. A _grosso modo_, el _deployment_ gestiona los _ReplicaSet_ para que ofrezcan en todo momento el servicio con las características concretas que deseemos.

Para aclarar un poco los conceptos, vamos a apreciar de forma gráfica las diferencias entre el despliegue habitual de una aplicación frente el despliegue de una aplicación haciendo uso de **Kubernetes**:

* **Despliegue habitual**:

![grafico1](https://i.ibb.co/SwSBrwy/normal.jpg "Despliegue habitual")

* **Despliegue Kubernetes**:

![grafico2](https://i.ibb.co/SwSBrwy/normal.jpg "Despliegue Kubernetes")

Como se puede apreciar, en la primera imagen tenemos un conjunto de máquinas (**_Frontend_**) que están expuestas al exterior y la carga se balancea entre ellas a través de un balanceador de carga. Dichas máquinas interactuan con otras máquinas internas (**_Backend_**) para responder las peticiones entrantes.

De otro lado, en el despliegue de Kubernetes, tenemos un componente **Ingress** que es el que se expone al exterior (posteriormente hablamos del mismo), que es el encargado de enviar dichas peticiones al servicio **_Frontend_** (**_NodePort_**) para su correspondiente balanceo entre los _pods_ existentes en el _ReplicaSet_ del _deployment_, que como se puede suponer, el número de _pods_ existente variará, en caso de así configurarlo, según la carga existente. Al ser una aplicación, tendrá que interactuar con los _pods_ existentes en el otro _deployment_ (normalmente bases de datos) para gestionar las peticiones, a través del servicio **_Backend_** (**_ClusterIP_**).

Si todavía no ha quedado claro del todo, no hay problema, ya que dentro de poco vamos a llevar a cabo un ejemplo en el que se entenderá todo a la perfección.

Antes de ello, y una vez entendida la parte teórica, es necesario realizar la configuración inicial de nuestro _cluster_ de **Kubernetes**. Para ello, he generado un pequeño [escenario](https://pastebin.com/wVnS6Dsp) compuesto por las siguientes máquinas:

* **controller**: Máquina conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.142**. Actuará como _controller_ y tiene 3 GB de RAM y 2 _cores_ de CPU.
* **worker1**: Máquina conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.143**. Actuará como _worker_ y tiene 3 GB de RAM y 2 _cores_ de CPU.
* **worker2**: Máquina conectada a mi red doméstica en modo puente (_bridge_), con dirección IP asignada **192.168.1.144**. Actuará como _worker_ y tiene 3 GB de RAM y 2 _cores_ de CPU.

Me encuentro actualmente haciendo uso de la máquina **controller**, de manera que lo primero que haremos será llevar a cabo la instalación del _software_ necesario para gestionar nuestro _cluster_. Para ello, disponemos de dos opciones:

* Utilizar el _script_ de instalación, pensado para instalarlo como un servicio en máquinas que ejecuten _systemd_, pues como servicio que es, se configurará para estar en todo momento funcionando junto a la máquina anfitriona. Además de ello, se nos proporcionarán las herramientas de gestión entre las que se encuentra _kubectl_.
* Descargar la última _release_ del [repositorio](https://github.com/k3s-io/k3s) de GitHub y llevar a cabo la configuración de forma manual.

Como se puede suponer, por comodidad y flexibilidad, la primera de las opciones es la elegida en este caso. Para llevar a cabo la instalación, tendremos que ejecutar el siguiente comando (en caso de no tenerlo, es necesario instalar **curl**):

{% highlight shell %}
root@controller:~# curl -sfL https://get.k3s.io | sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
{% endhighlight %}

En consecuencia del proceso que está actualmente en ejecución, se habrá abierto un _socket TCP/IP_ en el puerto por defecto que utiliza **k3s** (**6443**) que estará escuchando peticiones en todas las interfaces de la máquina, así que para verificarlo haremos uso del comando:

{% highlight shell %}
root@controller:~# netstat -tlnp | egrep '6443'
tcp6       0      0 :::6443                 :::*                    LISTEN      445/k3s server
{% endhighlight %}

Donde:

* **-t**: Filtramos únicamente para las conexiones que utilizan el protocolo TCP.
* **-l**: Filtramos únicamente para los _sockets_ que están actualmente escuchando peticiones (**State = LISTEN**).
* **-n**: Indicamos que muestre las direcciones y puertos de forma numérica, en lugar de intentar traducirlos.
* **-p**: Indicamos que muestre el PID y el nombre del proceso al que pertenece dicho _socket_.

Efectivamente, el proceso está escuchando peticiones tal y como debería en el puerto **6443/TCP**, que será el utilizado para la posterior conexión con los nodos _worker_.

Como resultado de la instalación, algunos binarios se habrán instalado en el directorio **/usr/local/bin/** y estarán ahora disponibles para su uso. Para verificarlo, haremos uso del comando:

{% highlight shell %}
root@controller:~# ls -l /usr/local/bin/
total 44296
lrwxrwxrwx 1 root root        3 Mar  4 07:40 crictl -> k3s
lrwxrwxrwx 1 root root        3 Mar  4 07:40 ctr -> k3s
-rwxr-xr-x 1 root root 45350912 Mar  4 07:40 k3s
-rwxr-xr-x 1 root root     1702 Mar  4 07:40 k3s-killall.sh
-rwxr-xr-x 1 root root     1037 Mar  4 07:40 k3s-uninstall.sh
lrwxrwxrwx 1 root root        3 Mar  4 07:40 kubectl -> k3s
{% endhighlight %}

Como era de esperar, nuevos binarios han sido instalados para su correspondiente uso, entre los que se encuentra **kubectl**, que nos permitirá gestionar mediante línea de comandos y de forma centralizada, nuestro _cluster_. Para comprobar que está funcionando correctamente, vamos a listar todos los nodos existentes en el _cluster_, ejecutando para ello el comando:

{% highlight shell %}
root@controller:~# kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
controller   Ready    control-plane,master   37s   v1.20.4+k3s1
{% endhighlight %}

Actualmente, existe un único nodo en el _cluster_, concretamente el controlador que es la máquina desde la que hemos ejecutado el comando. Sin embargo, esto no es lo que queremos, ya que tenemos otras dos máquinas que actuarán como _workers_ y necesitamos integrarla al mismo.

Por motivos de seguridad, para vincular un nuevo nodo a un _cluster_, necesitamos hacer uso además de la dirección IP del _controller_, de un _token_ único de verificación que podremos encontrar en el fichero de nombre **/var/lib/rancher/k3s/server/node-token**, de manera que visualizaremos su contenido haciendo uso del comando:

{% highlight shell %}
root@controller:~# cat /var/lib/rancher/k3s/server/node-token
K1007e764762d120d437ad0ab8460e3710b2b8fb110f74829caf5b5d5e6ba41e1dc::server:da0b3b9a53d12bb2192394d61a59308a
{% endhighlight %}

Todo está listo para llevar a cabo la instalación de **k3s** en los nodos _workers_, haciendo uso una vez más del método de instalación anterior. Sin embargo, durante la misma, tendremos que indicar de forma obligatoria dos parámetros, para así poder llevar a cabo la vinculación con el nodo _controller_ y hacer que dichas máquinas, actúen, como se puede suponer, como _workers_:

* **K3S_URL**: Indicamos la URL de conexión al _controller_, que se generará siguiendo la sintaxis **https://[IP]:6443**. En este caso, la URL final sería **https://192.168.1.142:6443**.
* **K3S_TOKEN**: Indicamos el _token_ del nodo controlador previamente visualizado.

**Nota**: En caso de que los _hostname_ de las máquinas _worker_ coincidiesen, habría que diferenciarlas utilizando también el parámetro **K3S_NODE_NAME** durante la ejecución del _script_ de instalación.

De esta manera, los comandos a ejecutar para instalar **k3s** en ambos nodos _worker_ y vincularlos al _controller_ serían (en caso de no tenerlo, es necesario instalar **curl**):

{% highlight shell %}
root@worker1:~# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.142:6443 K3S_TOKEN=K1007e764762d120d437ad0ab8460e3710b2b8fb110f74829caf5b5d5e6ba41e1dc::server:da0b3b9a53d12bb2192394d61a59308a sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent

root@worker2:~# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.142:6443 K3S_TOKEN=K1007e764762d120d437ad0ab8460e3710b2b8fb110f74829caf5b5d5e6ba41e1dc::server:da0b3b9a53d12bb2192394d61a59308a sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.20.4+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
{% endhighlight %}

Una vez finalizada la instalación de la distribución de **Kubernetes** en todos los nodos, volveremos al nodo controlador para verificar que la conexión con el resto de máquinas se ha llevado a cabo correctamente y ahora se encuentran a su disposición, haciendo para ello uso del comando:

{% highlight shell %}
root@controller:~# kubectl get nodes
NAME         STATUS   ROLES                  AGE    VERSION
controller   Ready    control-plane,master   12m    v1.20.4+k3s1
worker1      Ready    <none>                 101s   v1.20.4+k3s1
worker2      Ready    <none>                 16s    v1.20.4+k3s1
{% endhighlight %}

Como era de esperar, nuestro _cluster_ de _Kubernetes_ está ahora compuesto por un total de tres nodos: un _controller_ y dos _worker_.

Sin embargo, si lo pensamos, gestionar nuestro _cluster_ desde el nodo controlador puede llegar a ser un tanto incómodo, ya que tenemos que conectarnos al mismo cada vez que queramos llevar a cabo cualquier acción. La solución más sencilla a esto consiste en instalar **kubectl** en la máquina anfitriona, para así gestionar de forma remota el _cluster_.

Dado que dicho paquete no se encuentra actualmente disponible en los repositorios de Debian, procederemos a descargarlo de los repositorios oficiales de _Kubernetes_.

Para añadir dicho repositorio a nuestra máquina, ejecutaremos el comando:

{% highlight shell %}
root@debian:~# echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
{% endhighlight %}

De otro lado, tendremos que añadir también la clave pública GPG de _Google_ a nuestro anillo de claves para así poder verificar la integridad e instalar el paquete de Kubernetes que vamos a descargar, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:~# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
OK
{% endhighlight %}

Tras ello, podremos dar paso a la instalación de _kubectl_, no sin antes actualizar la lista de la paquetería disponible, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# apt update && apt install kubectl
{% endhighlight %}

El paquete **kubectl** ha sido instalado en la máquina, de manera que vamos a verificar la versión del mismo para así comprobar su correcto funcionamiento, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:~# kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4", GitCommit:"e87da0bd6e03ec3fea7933c4b5263d151aafd07c", GitTreeState:"clean", BuildDate:"2021-02-18T16:12:00Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
{% endhighlight %}

Actualmente, nos encontramos haciendo uso de la versión **v1.20.4**, que es la misma que la instalada en los nodos del _cluster_, de manera que vamos a disponer de un soporte completo, al estar compartiendo la misma versión.

Si apreciamos la última línea, hemos obtenido un error de conexión rehusada. Es totalmente normal, ya que _kubectl_ está tratando de acceder a nuestro _cluster_ local, sin embargo, no existe ningún _cluster_ en ejecución de forma local.

Para solucionarlo, necesitaremos un fichero **config** con las credenciales para el acceso al _cluster_ remoto, que podremos encontrar en **/etc/rancher/k3s/k3s.yaml** en el nodo controlador, que debemos ubicar dentro de un directorio de nombre **~/.kube** en la máquina anfitriona, el cuál generaremos ejecutando el comando:

{% highlight shell %}
root@debian:~# mkdir .kube
{% endhighlight %}

Una vez generado, podremos llevar a cabo la copia de dicho fichero desde el nodo controlador al directorio que acabamos de generar haciendo para ello uso del comando:

{% highlight shell %}
root@debian:~# scp root@192.168.1.142:/etc/rancher/k3s/k3s.yaml .kube/config
{% endhighlight %}

Una vez completada la transferencia, visualizaremos el contenido de dicho fichero mediante la ejecución del comando:

{% highlight shell %}
root@debian:~# cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJlRENDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTVRRNE5ETTJNRFV3SGhjTk1qRXdNekEwTURjME1EQTFXaGNOTXpFd016QXlNRGMwTURBMQpXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTVRRNE5ETTJNRFV3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFTMXJNTU1ocWlGL0hDM1ZJWGtTNzlBQTRoZ1FYbkFMVkp3UVVMd1ArK0IKQXA5NFpwWFR0cDZTNnNDVFBxTCs5ZE5CcTBhYW9sSlBDWnN6UWNBWTFYdUJvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVWMzNDhZOHErZzFMOEVxY1VpQTVNCmZCaEt6cDB3Q2dZSUtvWkl6ajBFQXdJRFNRQXdSZ0loQUs4RHp4T0p4M01jOVlkUHdaemhkU1h3Nnk3UlZtbnMKQ09qalExVGZXc2FSQWlFQW5TcUpEbVpnekNTSnZsUndXVVVEc0piQXZZTHBLdUJ3TytPM3QxVGhOR0k9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJrakNDQVRlZ0F3SUJBZ0lJZS84WU50UTZwd0l3Q2dZSUtvWkl6ajBFQXdJd0l6RWhNQjhHQTFVRUF3d1kKYXpOekxXTnNhV1Z1ZEMxallVQXhOakUwT0RRek5qQTFNQjRYRFRJeE1ETXdOREEzTkRBd05Wb1hEVEl5TURNdwpOREEzTkRBd05Wb3dNREVYTUJVR0ExVUVDaE1PYzNsemRHVnRPbTFoYzNSbGNuTXhGVEFUQmdOVkJBTVRESE41CmMzUmxiVHBoWkcxcGJqQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDlBd0VIQTBJQUJNbzZDNTFwR3dBK2pEbGoKdFhYYUI2TkhoNnNmbXpUK2NkZnZZKzlzbGFBYms2Z1VjdFZvdUt4Q3JvRkFjeXN1b0UzM21HRlJkMEJqT2ViTwovZXh6YW42alNEQkdNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBakFmCkJnTlZIU01FR0RBV2dCUUhsZmJ3RTBEN1RlSHVDWnBlTkMvbTVRT3pHVEFLQmdncWhrak9QUVFEQWdOSkFEQkcKQWlFQWxTQ25ZRWJWbzVFYkdnZHpWcEhrbHZTdTNoSGdzSGRyOUgvOXJibmltUUFDSVFETVZCL056aGw4azF5ZApyNFZhQmY0OGJKenRSdWdUWjY2amxsdHB6c3orTEE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlCZHpDQ0FSMmdBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakFqTVNFd0h3WURWUVFEREJock0zTXRZMnhwClpXNTBMV05oUURFMk1UUTRORE0yTURVd0hoY05NakV3TXpBME1EYzBNREExV2hjTk16RXdNekF5TURjME1EQTEKV2pBak1TRXdId1lEVlFRRERCaHJNM010WTJ4cFpXNTBMV05oUURFMk1UUTRORE0yTURVd1dUQVRCZ2NxaGtqTwpQUUlCQmdncWhrak9QUU1CQndOQ0FBUjgwMWVvY3JsQ09FNkZ1WmplQWxPVHFJTDhnM1l5bDltelI3UUdoQUlpCkFnNmovVkpRYzhXb2w1eHRWdUNBeGViODIvUDVSYlBkMHpzdnlVYlFxTk1ObzBJd1FEQU9CZ05WSFE4QkFmOEUKQkFNQ0FxUXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVVCNVgyOEJOQSswM2g3Z21hWGpRdgo1dVVEc3hrd0NnWUlLb1pJemowRUF3SURTQUF3UlFJaEFNVkhMSmIyeDVuUzlMN2FUbEhYZ0V3QUp5U3F5UnNPClRTcDRoUGJoaG9NVkFpQUhlZEEzeEwxcTJ4aTNQaU9vQ2RiZnZzT2d2WCsrTE41QWNBUjYrNW1jNFE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBFQyBQUklWQVRFIEtFWS0tLS0tCk1IY0NBUUVFSU1kd0RNc2ZsMlB0SjI4NDZlcTlVTlVwRk5HTVVwc3N1UmlkNnQ5dEZpUE5vQW9HQ0NxR1NNNDkKQXdFSG9VUURRZ0FFeWpvTG5Xa2JBRDZNT1dPMWRkb0hvMGVIcXgrYk5QNXgxKzlqNzJ5Vm9CdVRxQlJ5MVdpNApyRUt1Z1VCekt5NmdUZmVZWVZGM1FHTTU1czc5N0hOcWZnPT0KLS0tLS1FTkQgRUMgUFJJVkFURSBLRVktLS0tLQo=
{% endhighlight %}

Como se puede apreciar, el mismo contiene algunas credenciales para el acceso remoto al _cluster_, que en mi caso, no me importa mostrar de forma pública ya que es un _cluster_ de ejemplo y que será eliminado una vez finalizada la práctica.

Sin embargo, existe una directiva **server** que no se encuentra correctamente configurada, ya que está apuntando a la dirección de la máquina local (**127.0.0.1**), a pesar de que debería estar apuntando al nodo controlador (**192.168.1.142**). Para solucionarlo, modificaremos el fichero haciendo uso del comando:

{% highlight shell %}
root@debian:~# nano .kube/config
{% endhighlight %}

El resultado final de dicha directiva, tras llevar a cabo la correspondiente modificación debería ser:

{% highlight shell %}
server: https://192.168.1.142:6443
{% endhighlight %}

Toda la configuración en la máquina anfitriona para poder gestionar de forma remota el _cluster_ ha finalizado, de manera que comprobaremos que tenemos acceso al mismo listando una vez más los nodos existentes en dicho _cluster_, ejecutando para ello el comando:

{% highlight shell %}
root@debian:~# kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
controller   Ready    control-plane,master   30m   v1.20.4+k3s1
worker1      Ready    <none>                 19m   v1.20.4+k3s1
worker2      Ready    <none>                 18m   v1.20.4+k3s1
{% endhighlight %}

Efectivamente, así ha sido, de manera que podemos concluir que tenemos acceso remoto a nuestro _cluster_ de **Kubernetes**.

Todo está listo para empezar la parte interesante del artículo, en la que vamos a realizar un despliegue de una aplicación conocida como "**Let's Chat**", utilizando para ello el repositorio de GitHub del IES Gonzalo Nazareno, que nos proporcionará todos los ficheros **.yaml** para definir los _deployment_, los servicios...

En mi caso, me he movido al directorio **/home/alvaro/GitHub/** de la máquina anfitriona haciendo uso de `cd`, pues será donde clonaré dicho repositorio, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub# git clone https://github.com/iesgn/kubernetes-storm.git
Clonando en 'kubernetes-storm'...
remote: Enumerating objects: 288, done.
remote: Counting objects: 100% (288/288), done.
remote: Compressing objects: 100% (213/213), done.
remote: Total 288 (delta 119), reused 224 (delta 60), pack-reused 0
Recibiendo objetos: 100% (288/288), 6.36 MiB | 9.76 MiB/s, listo.
Resolviendo deltas: 100% (119/119), listo.
{% endhighlight %}

Una vez finalizada la clonación, nos moveremos dentro del mismo, concretamente al directorio **unidad3/ejemplos-3.2/ejemplo8/**, pues es donde se encuentra la aplicación en cuestión, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub# cd kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8/
{% endhighlight %}

Para entrar en sintonía, vamos a listar el contenido del directorio actual, para así comprobar los distintos ficheros de los que disponemos, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# ls -l
total 20
-rw-r--r-- 1 root root 247 mar  4 09:12 ingress.yaml
-rw-r--r-- 1 root root 394 mar  4 09:12 letschat-deployment.yaml
-rw-r--r-- 1 root root 177 mar  4 09:12 letschat-srv.yaml
-rw-r--r-- 1 root root 358 mar  4 09:12 mongo-deployment.yaml
-rw-r--r-- 1 root root 149 mar  4 09:12 mongo-srv.yaml
{% endhighlight %}

Como se puede apreciar, tenemos un total de **5** ficheros, los cuales iremos explicando con mayor detenimiento durante el transcurso de la tarea. El primero en el que debemos fijarnos es **mongo-deployment.yaml**, de manera que visualizaremos el contenido del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# cat mongo-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mongo
  template:
    metadata:
      labels:
        name: mongo
    spec:
      containers:
      - name: mongo
        image: mongo
        ports:
          - name: mongo
            containerPort: 27017
{% endhighlight %}

Sin entrar en demasiado detalle, dentro del mismo hemos definido un _deployment_ que generará por consecuencia un _ReplicaSet_ con un único _pod_, por ahora, que ejecutará una imagen **mongo** que ofrecerá su servicio en el puerto **27017**. Gracias al mismo, estamos definiendo la parte **Backend** de nuestra aplicación, con la que posteriormente interactuará el **Frontend** para ofrecer el servicio.

Todo está listo para definir el _deployment_ a partir de dicho fichero (es importante conocer que también es posible hacerlo mediante una instrucción de línea de comandos, aunque no es lo común). Para ello, haremos uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl apply -f mongo-deployment.yaml 
deployment.apps/mongo created
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el _deployment_ ha sido correctamente definido, de manera que vamos a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/mongo   1/1     1            1           51s   mongo        mongo    name=mongo

NAME                               DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/mongo-5c694c878b   1         1         1       50s   mongo        mongo    name=mongo,pod-template-hash=5c694c878b

NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-46ts2   1/1     Running   0          50s   10.42.2.4   worker2   <none>           <none>
{% endhighlight %}

Tal y como he mencionado previamente, el _deployment_ que he definido ha generado a su vez un _ReplicaSet_ asociado que contiene un único _pod_ con la imagen de **mongo**, que como se puede apreciar, se está ejecutando sobre el nodo **worker2**.

Para que empecemos a comprender el potencial de **Kubernetes**, vamos a tratar de eliminar el _pod_ asociado al _ReplicaSet_ que se ha generado, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl delete pod/mongo-5c694c878b-46ts2
pod "mongo-5c694c878b-46ts2" deleted
{% endhighlight %}

En un principio, basándonos en la salida del comando ejecutado, podríamos concluir que el _pod_ ha sido eliminado y que por tanto, se ha dejado de ofrecer el servicio deseado. Vamos a volver a listar los objetos existentes en nuestro _cluster_, ejecutando para ello el comando utilizado con anterioridad:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES   SELECTOR
deployment.apps/mongo      1/1     1            1           3m33s   mongo        mongo    name=mongo

NAME                                  DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES   SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         1       3m32s   mongo        mongo    name=mongo,pod-template-hash=5c694c878b

NAME                            READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Running   0          13s   10.42.2.5   worker2   <none>           <none>
{% endhighlight %}

¿Magia? Podríamos decir que sí. **Kubernetes**, no conforme con nuestra decisión de eliminar el _pod_ asociado, ha generado un nuevo _pod_ de forma inmediata que de nuevo se está ejecutando en el nodo **worker2** (podría haberse ejecutado en el nodo **worker1**, pero ha dado la casualidad), de manera que nuestro servicio vuelve a estar operativo de forma casi inmediata.

Existe un concepto que todavía no hemos introducido, y que nos será necesario para continuar, los **servicios**.

Los **servicios** (_services_) nos permiten acceder a nuestras aplicaciones, ya que proporcionan una capa de abstracción que define un conjunto de _pods_ que implementan un micro-servicio. Nos ofrecen una dirección IP virtual y un nombre que identifica al conjunto de _pods_ que representa, al cuál nos podremos conectar. Dependiendo del tipo de servicio, **NodePort** o **ClusterIP**, permitiremos el acceso desde el exterior desde un puerto aleatorio del _controller_ o únicamente de forma interna desde otros _pods_, respectivamente.

Una vez comprendido el concepto **servicio**, podremos proceder a definir el contenido del fichero **mongo-srv.yaml**, que como ya hemos mencionado, proporcionará una dirección IP virtual para acceder de una forma totalmente abstracta y ajena a nosotros a los _pods_ existentes en el _ReplicaSet_, pues se encargará de "balancear" la carga y enviar las peticiones únicamente a aquellos _pods_ que se encuentren activos, para asegurar que el servicio funcione en todo momento, de manera que visualizaremos el contenido del mismo, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# cat mongo-srv.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: mongo
  selector:
    name: mongo
{% endhighlight %}

Sin entrar en demasiado detalle, dentro del mismo hemos definido un _service_ de tipo **ClusterIP**, es decir, que únicamente será accesible de forma interna por el resto de _pods_ (valor por defecto) y ofrecerá su servicio en el puerto **27017** afectando a aquellas máquinas que están ejecutando **mongo**.

Todo está listo para definir el _service_ a partir de dicho fichero (es importante conocer que también es posible hacerlo mediante una instrucción de línea de comandos, aunque no es lo común). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl apply -f mongo-srv.yaml 
service/mongo created
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el _service_ ha sido correctamente definido, de manera que vamos a listar todos los servicios existentes en nuestro _cluster_ de **Kubernetes**, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
kubernetes   ClusterIP   10.43.0.1     <none>        443/TCP     50m
mongo        ClusterIP   10.43.10.91   <none>        27017/TCP   20s
{% endhighlight %}

Efectivamente, se ha generado un nuevo servicio de nombre **mongo** únicamente accesible a través de la dirección del Cluster IP para los _pods_ internos, pues se trata de un servicio crítico y debemos priorizar en todo momento su seguridad.

Ha llegado la hora de desplegar la aplicación **Let's Chat** como tal, _deployment_ que se encontrará definido en el fichero **letschat-deployment.yaml**, de manera que visualizaremos el contenido del mismo, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# cat letschat-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: letschat
  labels:
    name: letschat
spec:
  replicas: 1 
  selector:
    matchLabels:
      name: letschat
  template:
    metadata:
      labels:
        name: letschat
    spec:
      containers:
      - name: letschat
        image: sdelements/lets-chat
        ports:
          - name: http-server
            containerPort: 8080
{% endhighlight %}

Sin entrar en demasiado detalle, dentro del mismo hemos definido un _deployment_ que generará por consecuencia un _ReplicaSet_ con un único _pod_, por ahora, que ejecutará una imagen **sdelements/lets-chat** que ofrecerá su servicio en el puerto **8080**. Gracias al mismo, estamos definiendo la parte **Frontend** de nuestra aplicación, que interactuará con la parte **Backend** previamente definida para ofrecer el servicio.

Todo está listo para definir el _deployment_ a partir de dicho fichero. Para ello, haremos uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl apply -f letschat-deployment.yaml 
deployment.apps/letschat created
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el _deployment_ ha sido correctamente definido, de manera que vamos a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                 SELECTOR
deployment.apps/mongo      1/1     1            1           2m46s   mongo        mongo                  name=mongo
deployment.apps/letschat   1/1     1            1           8s      letschat     sdelements/lets-chat   name=letschat

NAME                                  DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         1       2m45s   mongo        mongo                  name=mongo,pod-template-hash=5c694c878b
replicaset.apps/letschat-7c66bd64f5   1         1         1       8s      letschat     sdelements/lets-chat   name=letschat,pod-template-hash=7c66bd64f5

NAME                            READY   STATUS    RESTARTS   AGE     IP          NODE      NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Running   0          2m45s   10.42.2.5   worker2   <none>           <none>
pod/letschat-7c66bd64f5-6mt2m   1/1     Running   0          8s      10.42.1.5   worker1   <none>           <none>
{% endhighlight %}

Tal y como he mencionado previamente, el _deployment_ que he definido ha generado a su vez un _ReplicaSet_ asociado que contiene un único _pod_ con la imagen de **sdelements/lets-chat**, que como se puede apreciar, se está ejecutando sobre el nodo **worker1**.

Una vez definida nuestra aplicación, podremos desplegar el servicio existente en el fichero **letschat-srv.yaml**, exponiendo así la parte **Frontend** de nuestra aplicación al exterior para así hacerla accesible, de manera que visualizaremos el contenido del mismo, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# cat letschat-srv.yaml 
apiVersion: v1
kind: Service
metadata:
  name: letschat
spec:
  type: NodePort
  ports:
  - name: http
    port: 8080
    targetPort: http-server
  selector:
    name: letschat
{% endhighlight %}

Sin entrar en demasiado detalle, dentro del mismo hemos definido un _service_ de tipo **NodePort**, es decir, que será accesible de forma externa y redirigirá las peticiones al puerto **8080** de aquellas máquinas que están ejecutando **letschat**.

Todo está listo para definir el _service_ a partir de dicho fichero. Para ello, ejecutaremos el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl apply -f letschat-srv.yaml 
service/letschat created
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el _service_ ha sido correctamente definido, de manera que vamos a listar todos los servicios existentes en nuestro _cluster_ de **Kubernetes**, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP          53m
mongo        ClusterIP   10.43.10.91    <none>        27017/TCP        3m25s
letschat     NodePort    10.43.78.152   <none>        8080:32468/TCP   16s
{% endhighlight %}

Efectivamente, se ha generado un nuevo servicio de nombre **letschat** accesible desde el exterior a través del puerto **32468** de la máquina controladora. Sin embargo, esto no es para nada práctico, pues no es viable hacer a los clientes conectarse de forma manual al puerto 32468 en cada conexión, poniendo al final de la URL '**:32468**'. Para solucionarlo, haremos uso de los **_Ingress controller_**.

Los controladores _Ingress_ o _Ingress controller_ nos permiten utilizar un _proxy_ inverso que por medio de reglas de encaminamiento que obtiene de la API de _Kubernetes_ nos permite el acceso a nuestras aplicaciones por medio de nombres.

![grafico3](https://i.ibb.co/6Rk1jGq/ingress.jpg "Ingress controller")

Una vez comprendido el concepto **_ingress_**, podremos proceder a definir el contenido del fichero **ingress.yaml**, que como ya hemos mencionado, proporcionará un _proxy_ inverso para acceder a través de un nombre a la aplicación que hemos desplegado, proporcionando por tanto una capa de abstracción más, de manera que visualizaremos el contenido del mismo, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# cat ingress.yaml 
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-letschat
spec:
  rules:
  - host: www.letschat.com
    http:
      paths:
      - path: "/"
        backend:
          serviceName: letschat
          servicePort: 8080
{% endhighlight %}

En este caso, la versión de la API que utiliza el fichero se encuentra casi obsoleta, de manera que he decidido actualizar el contenido para adaptarlo, siguiendo la [documentación](https://kubernetes.io/docs/concepts/services-networking/ingress/), a la versión **v1**. Para modificar el fichero ejecutaremos el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# nano ingress.yaml
{% endhighlight %}

El resultado final del fichero, tras llevar a cabo las correspondientes modificaciones sería:

{% highlight shell %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-letschat
spec:
  rules:
  - host: www.letschat.com
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: letschat
            port:
              number: 8080
{% endhighlight %}

Sin entrar en demasiado detalle, dentro del mismo hemos definido un _ingress_ que será accesible a través del nombre **www.letschat.com** afectando a aquellas máquinas que están ejecutando **letschat**, en el puerto **8080**.

Todo está listo para definir el _ingress_ a partir de dicho fichero (es importante conocer que también es posible hacerlo mediante una instrucción de línea de comandos, aunque no es lo común). Para ello, ejecutaremos el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/ingress-letschat created
{% endhighlight %}

Como se puede apreciar en la salida del comando ejecutado, el _ingress_ ha sido correctamente definido, de manera que vamos a listar todos los _ingress_ existentes en nuestro _cluster_ de **Kubernetes**, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get ingress
NAME               CLASS    HOSTS              ADDRESS         PORTS   AGE
ingress-letschat   <none>   www.letschat.com   192.168.1.142   80      70s
{% endhighlight %}

Efectivamente, se ha generado un nuevo _ingress_ de nombre **ingress-letschat** accesible a través del nombre **www.letschat.com** que deberá apuntar a la dirección IP del nodo controlador **192.168.1.142**.

Para hacer que el nombre **www.letschat.com** se resuelva a la dirección IP deseada, modificaremos el fichero de resolución estática de nombres **/etc/hosts**, indicando una nueva entrada para el mismo. Para modificar dicho fichero, ejecutaremos el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# nano /etc/hosts
{% endhighlight %}

El resultado del mismo, tras añadir la nueva correspondencia entre nombre y dirección IP sería:

{% highlight shell %}
127.0.0.1       localhost
127.0.1.1       debian
192.168.1.142   www.letschat.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

Todo está listo para tratar de acceder a **http://www.letschat.com** a través de nuestro navegador, pudiendo apreciar lo siguiente

![letschat1](https://i.ibb.co/64M0x0T/Captura-de-pantalla-de-2021-03-04-09-51-43.png "Let's Chat")

Como era de esperar, la aplicación desplegada en **Kubernetes** se encuentra totalmente operativa y podemos acceder a la misma a través de un nombre de dominio.

Sin embargo, esto no es todo, ya que como bien sabemos, _Kubernetes_ nos ofrece una flexibilidad impresionante sobre el _cluster_ existente, como por ejemplo escalar el número de _pods_ existente en el _deployment_ referente a la aplicación **Let's Chat** para así poder atender un mayor número de peticiones de forma concurrente. En este caso, vamos a escalar a un total de 10 _pods_, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl scale deploy letschat --replicas=10
deployment.apps/letschat scaled
{% endhighlight %}

Una vez escalada la aplicación, volveremos a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
deployment.apps/mongo      1/1     1            1           34m   mongo        mongo                  name=mongo
deployment.apps/letschat   10/10   10           10          31m   letschat     sdelements/lets-chat   name=letschat

NAME                                  DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         1       34m   mongo        mongo                  name=mongo,pod-template-hash=5c694c878b
replicaset.apps/letschat-7c66bd64f5   10        10        10      31m   letschat     sdelements/lets-chat   name=letschat,pod-template-hash=7c66bd64f5

NAME                            READY   STATUS    RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Running   0          30m     10.42.2.5    worker2      <none>           <none>
pod/letschat-7c66bd64f5-wg4cs   1/1     Running   0          8m53s   10.42.0.8    controller   <none>           <none>
pod/letschat-7c66bd64f5-d4ccg   1/1     Running   0          11s     10.42.1.28   worker1      <none>           <none>
pod/letschat-7c66bd64f5-x2s95   1/1     Running   0          11s     10.42.2.25   worker2      <none>           <none>
pod/letschat-7c66bd64f5-68fgp   1/1     Running   0          11s     10.42.2.23   worker2      <none>           <none>
pod/letschat-7c66bd64f5-2mlbn   1/1     Running   0          11s     10.42.2.24   worker2      <none>           <none>
pod/letschat-7c66bd64f5-gfsf5   1/1     Running   0          11s     10.42.2.26   worker2      <none>           <none>
pod/letschat-7c66bd64f5-wtzhq   1/1     Running   0          11s     10.42.1.29   worker1      <none>           <none>
pod/letschat-7c66bd64f5-pvskq   1/1     Running   0          11s     10.42.1.30   worker1      <none>           <none>
pod/letschat-7c66bd64f5-fqjrz   1/1     Running   0          11s     10.42.1.27   worker1      <none>           <none>
pod/letschat-7c66bd64f5-h2zbg   1/1     Running   0          11s     10.42.1.31   worker1      <none>           <none>
{% endhighlight %}

Como se puede apreciar, el número de _pods_ referentes al _ReplicaSet_ **letschat** ha aumentado de **1** a **10**, repartiéndose los mismos entre los dos nodos _worker_ principalmente, aunque cuando aumentamos muy considerablemente la carga en los mismos, puede ocurrir dependiendo de la distribución de **k8s** que algunos _pods_ se ejecuten en el nodo controlador para así desahogarlos un poco. En consecuencia, nuestra aplicación puede soportar ahora una mayor cantidad de tráfico.

Por último, vamos a simular una situación real en la que uno de los nodos _worker_ cayese, para así apreciar como **k8s** es capaz de gestionar la situación levantando ahora todos los _pods_ entre el _worker_ y el _controller_ para así no dejar de ofrecer el servicio en ningún momento.

En mi caso, al estar haciendo uso de un escenario en **vagrant**, bastaría con ejecutar la siguiente instrucción para apagar el nodo **worker2**:

{% highlight shell %}
alvaro@debian:~/vagrant/kubernetes$ vagrant halt worker2
{% endhighlight %}

Si tras ello volvemos a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, podremos apreciar lo siguiente:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
deployment.apps/mongo      0/1     1            0           39m   mongo        mongo                  name=mongo
deployment.apps/letschat   1/10    10           1           37m   letschat     sdelements/lets-chat   name=letschat

NAME                                  DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         0       39m   mongo        mongo                  name=mongo,pod-template-hash=5c694c878b
replicaset.apps/letschat-7c66bd64f5   10        10        1       37m   letschat     sdelements/lets-chat   name=letschat,pod-template-hash=7c66bd64f5

NAME                            READY   STATUS             RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Running            0          36m     10.42.2.5    worker2      <none>           <none>
pod/letschat-7c66bd64f5-x2s95   1/1     Running            0          5m38s   10.42.2.25   worker2      <none>           <none>
pod/letschat-7c66bd64f5-gfsf5   1/1     Running            0          5m38s   10.42.2.26   worker2      <none>           <none>
pod/letschat-7c66bd64f5-68fgp   1/1     Running            0          5m38s   10.42.2.23   worker2      <none>           <none>
pod/letschat-7c66bd64f5-wtzhq   0/1     CrashLoopBackOff   4          5m38s   10.42.1.29   worker1      <none>           <none>
pod/letschat-7c66bd64f5-pvskq   0/1     CrashLoopBackOff   4          5m38s   10.42.1.30   worker1      <none>           <none>
pod/letschat-7c66bd64f5-h2zbg   0/1     CrashLoopBackOff   4          5m38s   10.42.1.31   worker1      <none>           <none>
pod/letschat-7c66bd64f5-d4ccg   0/1     CrashLoopBackOff   4          5m38s   10.42.1.28   worker1      <none>           <none>
pod/letschat-7c66bd64f5-fqjrz   0/1     CrashLoopBackOff   4          5m38s   10.42.1.27   worker1      <none>           <none>
pod/letschat-7c66bd64f5-wg4cs   0/1     CrashLoopBackOff   4          14m     10.42.0.8    controller   <none>           <none>
pod/letschat-7c66bd64f5-w75xr   1/1     Running            5          4m57s   10.42.0.12   controller   <none>           <none>
{% endhighlight %}

Como podemos apreciar, los _pods_ en ejecución en el nodo **worker1** están empezando a fallar, ya que no tienen conexión con el **Backend** que estaba ubicado en el nodo **worker2** que ha caído. Esto es importante tenerlo en cuenta, ya que el _backend_ también debe estar configurado en alta disponibilidad para evitar puntos de fallo únicos como en este caso.

Por defecto, existe un parámetro de nombre **pod-eviction-timeout** que especifica el tiempo que ha de transcurrir hasta que los _pods_ se muevan a otro nodo _worker_ en caso de dejar de funcionar aquel en el que se estaban ejecutando, cuyo valor es de **5 minutos**. Dependiendo de nuestras circunstancias, tendremos que modificar dicho valor para adaptarlo a nuestras necesidades, intentando que no sea muy pequeño ni tampoco muy grande.

Tras haber pasado alrededor de 5 minutos, podremos volver a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, pudiendo apreciar lo siguiente:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
deployment.apps/mongo      1/1     1            1           44m   mongo        mongo                  name=mongo
deployment.apps/letschat   10/10   10           10          41m   letschat     sdelements/lets-chat   name=letschat

NAME                                  DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         1       44m   mongo        mongo                  name=mongo,pod-template-hash=5c694c878b
replicaset.apps/letschat-7c66bd64f5   10        10        10      41m   letschat     sdelements/lets-chat   name=letschat,pod-template-hash=7c66bd64f5

NAME                            READY   STATUS        RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Terminating   0          41m     10.42.2.5    worker2      <none>           <none>
pod/letschat-7c66bd64f5-gfsf5   1/1     Terminating   0          10m     10.42.2.26   worker2      <none>           <none>
pod/letschat-7c66bd64f5-x2s95   1/1     Terminating   0          10m     10.42.2.25   worker2      <none>           <none>
pod/letschat-7c66bd64f5-68fgp   1/1     Terminating   0          10m     10.42.2.23   worker2      <none>           <none>
pod/mongo-5c694c878b-rnnr2      1/1     Running       0          2m51s   10.42.1.33   worker1      <none>           <none>
pod/letschat-7c66bd64f5-9s8dw   1/1     Running       2          2m51s   10.42.1.32   worker1      <none>           <none>
pod/letschat-7c66bd64f5-ljcw6   1/1     Running       2          2m51s   10.42.0.14   controller   <none>           <none>
pod/letschat-7c66bd64f5-tdcpq   1/1     Running       2          2m51s   10.42.0.13   controller   <none>           <none>
pod/letschat-7c66bd64f5-w75xr   1/1     Running       6          9m47s   10.42.0.12   controller   <none>           <none>
pod/letschat-7c66bd64f5-wtzhq   1/1     Running       6          10m     10.42.1.29   worker1      <none>           <none>
pod/letschat-7c66bd64f5-pvskq   1/1     Running       6          10m     10.42.1.30   worker1      <none>           <none>
pod/letschat-7c66bd64f5-d4ccg   1/1     Running       6          10m     10.42.1.28   worker1      <none>           <none>
pod/letschat-7c66bd64f5-h2zbg   1/1     Running       6          10m     10.42.1.31   worker1      <none>           <none>
pod/letschat-7c66bd64f5-fqjrz   1/1     Running       6          10m     10.42.1.27   worker1      <none>           <none>
pod/letschat-7c66bd64f5-wg4cs   1/1     Running       6          19m     10.42.0.8    controller   <none>           <none>
{% endhighlight %}

Como se puede apreciar, el número de _pods_ referentes al _ReplicaSet_ **letschat** sigue siendo **10**, repartiéndose los mismos entre el primer nodo _worker_ y el nodo controlador para así desahogarlo un poco. En consecuencia, nuestra aplicación sigue estando totalmente operativa, tal y como podemos ver:

![letschat1](https://i.ibb.co/64M0x0T/Captura-de-pantalla-de-2021-03-04-09-51-43.png "Let's Chat")

Al igual que podemos escalar nuestro _deployment_ para que tenga un mayor número de _pods_, también podemos desescalarlo para eliminar _pods_ en caso de que el tráfico se haya visto reducido (generalmente esta tarea se suele delegar a un componente para hacerlo de forma automatizada). En este caso, vamos a desescalar a un total de 1 _pod_, haciendo para ello uso del comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl scale deploy letschat --replicas=1
deployment.apps/letschat scaled
{% endhighlight %}

Una vez desescalada la aplicación, volveremos a listar todos los _deployment_, _ReplicaSet_ y _pods_ existentes en nuestro _cluster_ de **Kubernetes**, ejecutando para ello el comando:

{% highlight shell %}
root@debian:/home/alvaro/GitHub/kubernetes-storm/unidad3/ejemplos-3.2/ejemplo8# kubectl get deploy,rs,po -o wide
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
deployment.apps/mongo      1/1     1            1           45m   mongo        mongo                  name=mongo
deployment.apps/letschat   1/1     1            1           43m   letschat     sdelements/lets-chat   name=letschat

NAME                                  DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/mongo-5c694c878b      1         1         1       45m   mongo        mongo                  name=mongo,pod-template-hash=5c694c878b
replicaset.apps/letschat-7c66bd64f5   1         1         1       43m   letschat     sdelements/lets-chat   name=letschat,pod-template-hash=7c66bd64f5

NAME                            READY   STATUS        RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
pod/mongo-5c694c878b-5449c      1/1     Terminating   0          42m    10.42.2.5    worker2      <none>           <none>
pod/letschat-7c66bd64f5-gfsf5   1/1     Terminating   0          11m    10.42.2.26   worker2      <none>           <none>
pod/letschat-7c66bd64f5-x2s95   1/1     Terminating   0          11m    10.42.2.25   worker2      <none>           <none>
pod/letschat-7c66bd64f5-68fgp   1/1     Terminating   0          11m    10.42.2.23   worker2      <none>           <none>
pod/mongo-5c694c878b-rnnr2      1/1     Running       0          4m5s   10.42.1.33   worker1      <none>           <none>
pod/letschat-7c66bd64f5-ljcw6   1/1     Running       2          4m5s   10.42.0.14   controller   <none>           <none>
{% endhighlight %}

Como se puede apreciar, el número de _pods_ referentes al _ReplicaSet_ **letschat** ha desminuido de **10** a **1**. Los otros 4 _pods_ son referentes al nodo **worker2** que todavía no han podido eliminarse ya que la máquina se encuentra inactiva.

Espero que el artículo haya contribuido a la comprensión de esta magnífica tecnología que va a ayudar a que el despliegue de las aplicaciones sea una tarea menos tediosa y podamos gestionar en todo momento de una forma muy sencilla, entre otras cosas, los despliegues y _rollbacks_ entre versiones de la aplicación.