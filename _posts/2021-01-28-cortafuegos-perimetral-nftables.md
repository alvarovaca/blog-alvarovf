---
layout: post
title:  "Cortafuegos perimetral con nftables"
banner: "/assets/images/banners/cortafuegos.jpg"
date:   2021-01-28 13:21:00 +0200
categories: seguridad openstack
---
## Configuración inicial.

### 1. Instalación.

{% highlight shell %}
apt update && apt upgrade && apt install nftables
systemctl start nftables
systemctl enable nftables
{% endhighlight %}

### 2. La política por defecto que vamos a configurar en nuestro cortafuegos será de tipo DROP.

{% highlight shell %}
nft chain inet filter input { policy drop \; }
nft chain inet filter forward { policy drop \; }
nft chain inet filter output { policy drop \; }
{% endhighlight %}

### 3. Configura de manera adecuada las reglas NAT para que todas las máquinas de nuestra red tenga acceso al exterior.

{% highlight shell %}
nft add table nat

nft add chain nat postrouting { type nat hook postrouting priority 100 \; }

nft add rule ip nat postrouting oifname "eth0" ip saddr 10.0.1.0/24 counter snat to 10.0.0.7
nft add rule ip nat postrouting oifname "eth0" ip saddr 10.0.2.0/24 counter snat to 10.0.0.7
{% endhighlight %}

### 4. Configura de manera adecuada las reglas NAT para que los servicios expuestos al exterior sean accesibles.

{% highlight shell %}
nft add chain nat prerouting { type nat hook prerouting priority 0 \; }

nft add rule ip nat prerouting iifname "eth0" udp dport 53 counter dnat to 10.0.1.7
nft add rule ip nat prerouting iifname "eth0" tcp dport 80 counter dnat to 10.0.2.6
nft add rule ip nat prerouting iifname "eth0" tcp dport 443 counter dnat to 10.0.2.6
{% endhighlight %}

## ping

### 1. Todas las máquinas de las dos redes pueden hacer ping entre ellas.

#### LAN a DMZ

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" ip daddr 10.0.2.0/24 oifname "eth2" icmp type echo-request counter accept
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" ip daddr 10.0.1.0/24 oifname "eth1" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ ping 10.0.2.6
PING 10.0.2.6 (10.0.2.6) 56(84) bytes of data.
64 bytes from 10.0.2.6: icmp_seq=1 ttl=63 time=2.12 ms
64 bytes from 10.0.2.6: icmp_seq=2 ttl=63 time=1.75 ms
64 bytes from 10.0.2.6: icmp_seq=3 ttl=63 time=1.67 ms
^C
--- 10.0.2.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.672/1.847/2.120/0.198 ms
{% endhighlight %}

#### DMZ a LAN

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" ip daddr 10.0.1.0/24 oifname "eth1" icmp type echo-request counter accept
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" ip daddr 10.0.2.0/24 oifname "eth2" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ ping 10.0.1.7
PING 10.0.1.7 (10.0.1.7) 56(84) bytes of data.
64 bytes from 10.0.1.7: icmp_seq=1 ttl=63 time=2.75 ms
64 bytes from 10.0.1.7: icmp_seq=2 ttl=63 time=1.73 ms
64 bytes from 10.0.1.7: icmp_seq=3 ttl=63 time=1.36 ms
^C
--- 10.0.1.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.362/1.947/2.749/0.586 ms
{% endhighlight %}

#### Dulcinea a LAN y DMZ

{% highlight shell %}
nft add rule inet filter output ip daddr 10.0.1.0/24 oifname "eth1" icmp type echo-request counter accept
nft add rule inet filter input ip saddr 10.0.1.0/24 iifname "eth1" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ping 10.0.1.7
PING 10.0.1.7 (10.0.1.7) 56(84) bytes of data.
64 bytes from 10.0.1.7: icmp_seq=1 ttl=64 time=2.37 ms
64 bytes from 10.0.1.7: icmp_seq=2 ttl=64 time=1.25 ms
64 bytes from 10.0.1.7: icmp_seq=3 ttl=64 time=0.989 ms
^C
--- 10.0.1.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.989/1.537/2.368/0.597 ms
{% endhighlight %}

{% highlight shell %}
nft add rule inet filter output ip daddr 10.0.2.0/24 oifname "eth2" icmp type echo-request counter accept
nft add rule inet filter input ip saddr 10.0.2.0/24 iifname "eth2" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ping 10.0.2.6
PING 10.0.2.6 (10.0.2.6) 56(84) bytes of data.
64 bytes from 10.0.2.6: icmp_seq=1 ttl=64 time=1.38 ms
64 bytes from 10.0.2.6: icmp_seq=2 ttl=64 time=0.626 ms
64 bytes from 10.0.2.6: icmp_seq=3 ttl=64 time=0.844 ms
^C
--- 10.0.2.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 22ms
rtt min/avg/max/mdev = 0.626/0.948/1.375/0.315 ms
{% endhighlight %}

### 2. Todas las máquinas pueden hacer ping a una máquina del exterior.

#### LAN

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" oifname "eth0" icmp type echo-request counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth0" oifname "eth1" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=57.8 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=69.6 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=47.0 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 47.043/58.160/69.630/9.226 ms
{% endhighlight %}

#### DMZ

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" oifname "eth0" icmp type echo-request counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth0" oifname "eth2" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=49.0 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=211 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=95.6 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 49.002/118.417/210.640/67.930 ms
{% endhighlight %}

#### Dulcinea

{% highlight shell %}
nft add rule inet filter output oifname "eth0" icmp type echo-request counter accept
nft add rule inet filter input iifname "eth0" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=55 time=58.2 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=55 time=40.7 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=55 time=51.2 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 40.698/50.011/58.171/7.182 ms
{% endhighlight %}

### 3. Desde el exterior se puede hacer ping a Dulcinea.

{% highlight shell %}
nft add rule inet filter input iifname "eth0" icmp type echo-request counter accept
nft add rule inet filter output oifname "eth0" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ ping 172.22.200.134
PING 172.22.200.134 (172.22.200.134) 56(84) bytes of data.
64 bytes from 172.22.200.134: icmp_seq=1 ttl=63 time=145 ms
64 bytes from 172.22.200.134: icmp_seq=2 ttl=63 time=169 ms
64 bytes from 172.22.200.134: icmp_seq=3 ttl=63 time=167 ms
^C
--- 172.22.200.134 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 145.275/160.544/169.215/10.835 ms
{% endhighlight %}

### 4. A Dulcinea se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT).

#### LAN

{% highlight shell %}
nft add rule inet filter input ip saddr 10.0.1.0/24 iifname "eth1" icmp type echo-request counter reject
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ ping 10.0.1.9
PING 10.0.1.9 (10.0.1.9) 56(84) bytes of data.
^C
--- 10.0.1.9 ping statistics ---
9 packets transmitted, 0 received, 100% packet loss, time 197ms

ip saddr 10.0.1.0/24 iifname "eth1" icmp type echo-request counter packets 9 bytes 756 reject
{% endhighlight %}

#### DMZ

{% highlight shell %}
nft add rule inet filter input ip saddr 10.0.2.0/24 iifname "eth2" icmp type echo-request counter accept
nft add rule inet filter output ip daddr 10.0.2.0/24 oifname "eth2" icmp type echo-reply counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ ping 10.0.2.12
PING 10.0.2.12 (10.0.2.12) 56(84) bytes of data.
64 bytes from 10.0.2.12: icmp_seq=1 ttl=64 time=0.554 ms
64 bytes from 10.0.2.12: icmp_seq=2 ttl=64 time=0.569 ms
64 bytes from 10.0.2.12: icmp_seq=3 ttl=64 time=0.658 ms
^C
--- 10.0.2.12 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 80ms
rtt min/avg/max/mdev = 0.554/0.593/0.658/0.053 ms
{% endhighlight %}

## ssh

### 1. Podemos acceder por ssh a todas las máquinas.

#### LAN

{% highlight shell %}
nft add rule inet filter output ip daddr 10.0.1.0/24 oifname "eth1" tcp dport 22 ct state new,established counter accept
nft add rule inet filter input ip saddr 10.0.1.0/24 iifname "eth1" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ssh debian@10.0.1.7
Linux freston 4.19.0-13-cloud-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
No mail.
Last login: Fri Jan 29 08:46:05 2021
{% endhighlight %}

#### DMZ

{% highlight shell %}
nft add rule inet filter output ip daddr 10.0.2.0/24 oifname "eth2" tcp dport 22 ct state new,established counter accept
nft add rule inet filter input ip saddr 10.0.2.0/24 iifname "eth2" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ssh centos@10.0.2.6
Last login: Fri Jan 29 08:46:30 2021
{% endhighlight %}

#### Dulcinea

{% highlight shell %}
nft add rule inet filter input ip saddr 172.22.0.0/15 iifname "eth0" tcp dport 22 ct state new,established counter accept
nft add rule inet filter output ip daddr 172.22.0.0/15 oifname "eth0" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ ssh debian@172.22.200.134
Linux dulcinea 4.19.0-13-cloud-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jan 29 08:37:36 2021 from 172.22.1.24
{% endhighlight %}

### 2. Todas las máquinas pueden hacer ssh a máquinas del exterior.

#### LAN

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" oifname "eth0" tcp dport 22 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth0" oifname "eth1" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ ssh debian@51.210.109.246
debian@51.210.109.246: Permission denied (publickey).
{% endhighlight %}

#### DMZ

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" oifname "eth0" tcp dport 22 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth0" oifname "eth2" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ ssh debian@51.210.109.246
debian@51.210.109.246: Permission denied (publickey).
{% endhighlight %}

#### Dulcinea

{% highlight shell %}
nft add rule inet filter output oifname "eth0" tcp dport 22 ct state new,established counter accept
nft add rule inet filter input iifname "eth0" tcp sport 22 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# ssh debian@51.210.109.246
Linux vps 4.19.0-13-cloud-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
No mail.
Last login: Thu Jan 28 09:56:47 2021 from 80.59.1.152
{% endhighlight %}

## DNS

### 1. El único DNS que pueden usar los equipos de las dos redes es Freston, no pueden utilizar un DNS externo.

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" ip daddr 10.0.1.7 oifname "eth1" udp dport 53 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth1" ip saddr 10.0.1.7 oifname "eth2" udp sport 53 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
ubuntu@sancho:~$ dig www.alvaro.gonzalonazareno.org
...
;; ANSWER SECTION:
www.alvaro.gonzalonazareno.org.	86400 IN CNAME	quijote.alvaro.gonzalonazareno.org.
quijote.alvaro.gonzalonazareno.org. 7199 IN A	10.0.2.6
...
;; Query time: 3 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Fri Jan 29 09:24:15 CET 2021
;; MSG SIZE  rcvd: 97

ubuntu@sancho:~$ dig @8.8.8.8 google.es

; <<>> DiG 9.16.1-Ubuntu <<>> @8.8.8.8 google.es
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached

[centos@quijote ~]$ dig www.alvaro.gonzalonazareno.org
...
;; ANSWER SECTION:
www.alvaro.gonzalonazareno.org.	86400 IN CNAME	quijote.alvaro.gonzalonazareno.org.
quijote.alvaro.gonzalonazareno.org. 86400 IN A	10.0.2.6
...
;; Query time: 2 msec
;; SERVER: 10.0.1.7#53(10.0.1.7)
;; WHEN: Fri Jan 29 09:27:55 CET 2021
;; MSG SIZE  rcvd: 163

[centos@quijote ~]$ dig @8.8.8.8 google.es

; <<>> DiG 9.11.20-RedHat-9.11.20-5.el8 <<>> @8.8.8.8 google.es
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
{% endhighlight %}

### 2. Dulcinea puede usar cualquier servidor DNS.

{% highlight shell %}
nft add rule inet filter output udp dport 53 ct state new,established counter accept
nft add rule inet filter input udp sport 53 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# dig www.alvaro.gonzalonazareno.org
...
;; ANSWER SECTION:
www.alvaro.gonzalonazareno.org.	86400 IN CNAME	quijote.alvaro.gonzalonazareno.org.
quijote.alvaro.gonzalonazareno.org. 86400 IN A	10.0.2.6
...
;; Query time: 3 msec
;; SERVER: 10.0.1.7#53(10.0.1.7)
;; WHEN: Fri Jan 29 09:31:41 CET 2021
;; MSG SIZE  rcvd: 163

root@dulcinea:~# dig @8.8.8.8 google.es
...
;; ANSWER SECTION:
google.es.		299	IN	A	172.217.168.163
...
;; Query time: 124 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Jan 29 09:32:07 CET 2021
;; MSG SIZE  rcvd: 54
{% endhighlight %}

### 3. Tenemos que permitir consultas DNS desde el exterior a Freston, para que, por ejemplo, papion-dns pueda preguntar.

{% highlight shell %}
nft add rule inet filter forward ip daddr 10.0.1.7 iifname "eth0" oifname "eth1" udp dport 53 ct state new,established counter accept
nft add rule inet filter forward ip saddr 10.0.1.7 iifname "eth1" oifname "eth0" udp sport 53 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ dig www.alvaro.gonzalonazareno.org
...
;; ANSWER SECTION:
www.alvaro.gonzalonazareno.org.	2242 IN	CNAME	dulcinea.alvaro.gonzalonazareno.org.
dulcinea.alvaro.gonzalonazareno.org. 2242 IN A	172.22.200.134
...
;; Query time: 1 msec
;; SERVER: 192.168.202.2#53(192.168.202.2)
;; WHEN: vie ene 29 09:38:38 CET 2021
;; MSG SIZE  rcvd: 140
{% endhighlight %}

### 4. Tenemos que permitir consultas DNS al exterior a Freston, para que pueda hacer las preguntas recursivas.

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.7 iifname "eth1" oifname "eth0" udp dport 53 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.7 iifname "eth0" oifname "eth1" udp sport 53 ct state established counter accept

nft add rule inet filter forward ip saddr 10.0.1.7 iifname "eth1" oifname "eth0" tcp dport 53 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.7 iifname "eth0" oifname "eth1" tcp sport 53 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ dig www.facebook.es
...
;; ANSWER SECTION:
www.facebook.es.	7200	IN	CNAME	www.facebook.com.
www.facebook.com.	3600	IN	CNAME	star-mini.c10r.facebook.com.
star-mini.c10r.facebook.com. 60	IN	A	31.13.83.36
...
;; Query time: 4289 msec
;; SERVER: 10.0.1.7#53(10.0.1.7)
;; WHEN: Fri Jan 29 09:48:10 CET 2021
;; MSG SIZE  rcvd: 390
{% endhighlight %}

## Base de datos

### 1. A la base de datos de Sancho sólo pueden acceder las máquinas de la DMZ y la LAN.

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" ip daddr 10.0.1.4 oifname "eth1" tcp dport 3306 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth1" ip saddr 10.0.1.4 oifname "eth2" tcp sport 3306 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ mysql -u quijote -p -h bd.alvaro.gonzalonazareno.org
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 40
Server version: 10.3.25-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

debian@freston:~$ mysql -u quijote -p -h bd.alvaro.gonzalonazareno.org
Enter password: 
ERROR 1130 (HY000): Host '10.0.1.7' is not allowed to connect to this MariaDB server

root@dulcinea:~# mysql -u quijote -p -h bd.alvaro.gonzalonazareno.org
Enter password: 
ERROR 2002 (HY000): Can't connect to MySQL server on 'bd.alvaro.gonzalonazareno.org' (115)
{% endhighlight %}

## Web

### 1. Las páginas web de Quijote (80, 443) pueden ser accedidas desde todas las máquinas de nuestra red y desde el exterior.

#### LAN

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" ip daddr 10.0.2.6 oifname "eth2" tcp dport 80 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth2" ip saddr 10.0.2.6 oifname "eth1" tcp sport 80 ct state established counter accept

nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" ip daddr 10.0.2.6 oifname "eth2" tcp dport 443 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth2" ip saddr 10.0.2.6 oifname "eth1" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ curl https://www.alvaro.gonzalonazareno.org
<h2>Álvaro Vaca Ferreras</h2>
{% endhighlight %}

#### DMZ

{% highlight shell %}
[centos@quijote ~]$ curl https://www.alvaro.gonzalonazareno.org
<h2>Álvaro Vaca Ferreras</h2>
{% endhighlight %}

#### Dulcinea

{% highlight shell %}
nft add rule inet filter output ip daddr 10.0.2.6 oifname "eth2" tcp dport 80 ct state new,established counter accept
nft add rule inet filter input ip saddr 10.0.2.6 iifname "eth2" tcp sport 80 ct state established counter accept

nft add rule inet filter output ip daddr 10.0.2.6 oifname "eth2" tcp dport 443 ct state new,established counter accept
nft add rule inet filter input ip saddr 10.0.2.6 iifname "eth2" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# curl https://www.alvaro.gonzalonazareno.org
<h2>Álvaro Vaca Ferreras</h2>
{% endhighlight %}

{% highlight shell %}
nft add rule inet filter forward ip daddr 10.0.2.6 iifname "eth0" oifname "eth2" tcp dport 80 ct state new,established counter accept
nft add rule inet filter forward ip saddr 10.0.2.6 iifname "eth2" oifname "eth0" tcp sport 80 ct state established counter accept

nft add rule inet filter forward ip daddr 10.0.2.6 iifname "eth0" oifname "eth2" tcp dport 443 ct state new,established counter accept
nft add rule inet filter forward ip saddr 10.0.2.6 iifname "eth2" oifname "eth0" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
alvaro@debian:~$ curl https://www.alvaro.gonzalonazareno.org
<h2>Álvaro Vaca Ferreras</h2>
{% endhighlight %}

## Extra

### 1. Permitimos que todas las máquinas puedan acceder a los puertos 80 y 443 del exterior (necesario para las actualizaciones).

#### LAN

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" oifname "eth0" tcp dport 80 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth0" oifname "eth1" tcp sport 80 ct state established counter accept

nft add rule inet filter forward ip saddr 10.0.1.0/24 iifname "eth1" oifname "eth0" tcp dport 443 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.1.0/24 iifname "eth0" oifname "eth1" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
debian@freston:~$ sudo apt update
Hit:1 https://ftp.cica.es/debian buster InRelease
Get:2 https://ftp.cica.es/debian buster-updates InRelease [51.9 kB]
Hit:3 https://ftp.cica.es/debian-security buster/updates InRelease
Fetched 51.9 kB in 1s (55.1 kB/s)            
Reading package lists... Done
Building dependency tree       
Reading state information... Done
1 package can be upgraded. Run 'apt list --upgradable' to see it.
{% endhighlight %}

#### DMZ

{% highlight shell %}
nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" oifname "eth0" tcp dport 80 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth0" oifname "eth2" tcp sport 80 ct state established counter accept

nft add rule inet filter forward ip saddr 10.0.2.0/24 iifname "eth2" oifname "eth0" tcp dport 443 ct state new,established counter accept
nft add rule inet filter forward ip daddr 10.0.2.0/24 iifname "eth0" oifname "eth2" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
[centos@quijote ~]$ sudo dnf update
Last metadata expiration check: 1:31:35 ago on Fri 29 Jan 2021 08:23:51 AM CET.
Dependencies resolved.
Nothing to do.
Complete!
{% endhighlight %}

#### Dulcinea

{% highlight shell %}
nft add rule inet filter output oifname "eth0" tcp dport 80 ct state new,established counter accept
nft add rule inet filter input iifname "eth0" tcp sport 80 ct state established counter accept

nft add rule inet filter output oifname "eth0" tcp dport 443 ct state new,established counter accept
nft add rule inet filter input iifname "eth0" tcp sport 443 ct state established counter accept
{% endhighlight %}

{% highlight shell %}
root@dulcinea:~# apt update
Hit:1 http://deb.debian.org/debian buster InRelease
Get:2 http://security.debian.org/debian-security buster/updates InRelease [65.4 kB]
Get:3 http://deb.debian.org/debian buster-updates InRelease [51.9 kB]
Get:4 http://security.debian.org/debian-security buster/updates/main amd64 Packages [270 kB]
Fetched 387 kB in 1s (396 kB/s)   
Reading package lists... Done
Building dependency tree       
Reading state information... Done
1 package can be upgraded. Run 'apt list --upgradable' to see it.
{% endhighlight %}

### 2. Almacenamos las reglas en un fichero que se importará automáticamente tras un reinicio, consiguiendo que perduren.

{% highlight shell %}
nft list ruleset > /etc/nftables.conf
{% endhighlight %}
