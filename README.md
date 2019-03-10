# I. Mise en place du lab

## Au croisement des routes...

Résumé des commandes pour arriver au résultat ci-dessous:
<img src="https://images.ladepeche.fr/api/v1/images/view/5c3607048fe56f5d53552b91/large/image.jpg" width="300" height="200" />

Après plusieures commandes à base d'`ìp route`, nous arrivons au résultat suivant:

```
[dams@client1 ~]$ ping -c 4 server1
PING server1 (10.2.2.10) 56(84) bytes of data.
64 bytes from server1 (10.2.2.10): icmp_seq=1 ttl=62 time=3.50 ms
64 bytes from server1 (10.2.2.10): icmp_seq=2 ttl=62 time=2.92 ms
64 bytes from server1 (10.2.2.10): icmp_seq=3 ttl=62 time=3.18 ms
64 bytes from server1 (10.2.2.10): icmp_seq=4 ttl=62 time=2.82 ms

--- server1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 2.825/3.108/3.503/0.268 ms
```
`server1` ne peut pas ping vers `router1` car ces deux machines ne se connaissent pas dans /etc/hosts.

## Attrapez donc toutes ces trames!

On effectue un ping `client1` &rarr; `server1` en capturant le trafic sur les deux interfaces de `router2`.

Trafic capturé sur `net2` [ici](Captures/capnet2.pcap).

Du point de vue de `net12`, c'est [par ici!](Captures/capnet12.pcap)

En observant le trafic réseau généré par ce `ping`, on constate que l'échange ARP est différent sur les deux interfaces, chacune gère sa propre table!

# II. NAT et services d'infra

## Mise en place de l'infra:

Afin de s'assurer que les machines puissent résoudre l'hôte `google.com`, il faut rajouter la ligne `nameserver 8.8.8.8` dans le fichier `/etc/resolv.conf`.

Quelques `ip route` plus tard...

```
[dams@router1 ~]$ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- [x] `router1` OK!

```
[dams@client1 ~]$ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- [x] `client1` OK!

```
[dams@router2 ~]$ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- [x] `router2` OK!

Malheureusement, je n'ai pas réussi avec `server1`, mais vu qu'on ne veut pas que le serveur puisse joindre internet, on va dire que c'est bon!

<img src="https://media.giphy.com/media/13Qumr2SLqrl5e/giphy.gif"/>

## Configuration d'un serveur NTP:

Sur `router1`, on modifie le fichier `/etc/chrony.conf`:

```
[dams@router1 ~]$ diff /etc/chrony.conf /etc/chrony.conf.orig
3,6c3,7  
// On remplace les serveurs par défaut par ceux recommandés par le NTP Pool Project.
< server 0.fr.pool.ntp.org
< server 1.fr.pool.ntp.org
< server 2.fr.pool.ntp.org
< server 3.fr.pool.ntp.org
---
> server 0.centos.pool.ntp.org iburst
> server 1.centos.pool.ntp.org iburst
> server 2.centos.pool.ntp.org iburst
> server 3.centos.pool.ntp.org iburst
>

16a18,24   // Ces lignes sont superflues, supprimons les!
> # Enable hardware timestamping on all interfaces that support it.
> #hwtimestamp *
>
> # Increase the minimum number of selectable sources required to adjust
> # the system clock.
> #minsources 2
>

//Enfin, décommentons certaines options!
18c26
< allow 10.2.1.0/24
---
> #allow 192.168.0.0/16
21c29
< local stratum 10
---
> #local stratum 10
24c32
< keyfile /etc/chrony.keys
---
> #keyfile /etc/chrony.keys
30c38
< log measurements statistics tracking
---
> #log measurements statistics tracking
```
