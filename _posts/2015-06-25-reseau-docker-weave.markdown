---
layout: post
title:  "Construire un réseau de conteneurs Docker avec Weave"
date:   2015-06-25 06:00:00
categories: docker architecture microservice
tags: regular 
image: /assets/article_images/tissu-nappe.jpg
legend: Pièce tissée composée de laines mohair
---
Dans un précédent billet, j’avais évoqué la difficulté de construire un **réseau de conteneurs Docker** répartis sur plusieurs machines hôtes, l’interconnexion de 2 conteneurs n’étant possible qu’au sein d’un même hôte Docker.<br />
L’autre limitation est la **difficulté d’accéder, via un port unique**, à 2 ou plusieurs conteneurs co-localisés dans le même hôte (par exemple, 2 serveurs web dockerisés dans le même hôte et écoutant tous deux sur le port 80).<br />
Ces lacunes sont comblées par des solutions diverses, en compétition les unes avec les autres, les plus connues étant :

- [**Weave**](http://weave.works/), objet de ce billet
- **SocketPlane**, racheté par Docker pour former le futur [**Docker Network**](http://blog.docker.com/2015/06/networking-receives-an-upgrade/) (*libnetwork*)
- [**Open vSwitch**](http://openvswitch.org/)

> **Note :** Avant d’entrer dans le vif du sujet, rappelons deux annonces importantes de l’édition DockerCon 2015, la conférence annuelle organisée par Docker :
>
> - [**Docker Network**](http://blog.docker.com/2015/06/networking-receives-an-upgrade/), annoncée pour la version 1.7, est issu en partie de l’intégration de *SocketPlane*
> - [**Docker Plugin**](http://blog.docker.com/2015/06/extending-docker-with-plugins/) est un mécanisme permettant d’étendre les fonctionnalités de *Docker Engine* sous la forme de modules d’extension
> *Docker Network* a été conçu selon le principe *"Batteries included, but removable"* ouvrant la voie au développement de drivers réseau par des sociétés tierces sous la forme de plugins, ce qui est [*déjà le cas de Weave*](https://github.com/weaveworks/docker-plugin/).<br />
> Avec *Docker Network*, nous disposerons donc d’une **interface standardisée de gestion d’un réseau de conteneurs**, à l’image de *Docker Machine* pour le provisionnement d’hôtes Docker. J’y reviendrai dans un prochain billet. Fin de la longue parenthèse.

Dans ce billet, nous explorerons en détail le fonctionnement et la mise en oeuvre de Weave à travers un cas pratique.
## De l'intérêt de Weave
**Weave** permet d’interconnecter des conteneurs répartis sur plusieurs hôtes Docker, indépendamment de leur localisation physique, afin de constituer un **réseau virtuel de conteneurs**.<br />
La disponibilité d’un mécanisme de **découverte de service par DNS** parachève un dispositif adapté à la construction d’une **architecture de microservices**.
## Fonctionnement
Un réseau Weave est constitué de **routeurs virtuels** installés sur chacun des hôtes Docker et connectés entre eux en pair à pair.<br />
Les routeurs sont en fait des conteneurs Docker maintenant entre eux une connexion TCP pour échanger des informations sur la topologie du réseau ainsi que des connexions UDP pour acheminer le trafic inter-conteneur.<br />
Lorsque qu’un conteneur rejoint le réseau Weave, il est relié au routeur à travers une interface réseau virtuelle (bridge `weave`).
## Etude de cas
Le schéma ci-dessous décrit l’architecture constituée de 2 serveurs hébergeant chacun un hôte Docker et un routeur Weave.
<center>![Architecture](http://blog.inovia-conseil.fr/wp-content/uploads/2015/06/weave-usecase.png)</center>

Deux services dockerisés (`svc1`, `svc2`) sont déployés sur l’hôte `orion.local` (sous Ubuntu), tandis que le service `svc3` est déployé sur l’hôte `macbook.local` (sous *Boot2Docker*).<br />
Les services dockerisés sont issus de l’image `ksahnine/dummy-http`, un service REST HTTP de test écoutant par défaut sur le port `8080` et utilisable comme suit :

- création du conteneur :
<pre>
$ docker run -td -p 8080:8080 ksahnine/dummy-http svcN
</pre>
- appel du service dockerisé :
<pre>
$ curl http://host_ip:8080/
Hi! I'm [svcN] service and my Docker container's IP is [container_ip]
</pre>
Notez sur le schéma le **plan d’adressage** des conteneurs dans le réseau weave (`10.0.1.x/24`). Les adresses IP sont attribuées à la création des conteneurs.

## Installation
Weave est distribué sous la forme d’images Docker, dont la disponibilité est évidemment un préalable.<br />
L’utilisation des composants dockerisés de Weave (routeur, DNS, proxy) est encapsulé par un script shell faisant office d’interface CLI (`weave`) installé sur chaque hôte Docker.

### Sous Ubuntu / CentOS
Le mode opératoire d’installation est le suivant :
{% highlight sh %}
sudo wget -O /usr/local/bin/weave https://github.com/weaveworks/weave/releases/download/latest_release/weave
sudo chmod a+x /usr/local/bin/weave
{% endhighlight %}

### Sous Boot2Docker (Windows / MacOS)
Le mode opératoire d’installation est le suivant :
{% highlight sh %}
$ wget https://github.com/weaveworks/weave/releases/download/latest_release/weave
$ boot2docker up
$ boot2docker ssh "cat > /usr/local/bin/weave" < weave
$ boot2docker ssh "chmod a+x /usr/local/bin/weave"
{% endhighlight %}

## Démarrage des routeurs Weave
Démarrer le routeur Weave sur chacun des noeuds (`orion.local` et `macbook.local`) via la commande `weave launch` :
{% highlight sh %}
$ sudo weave launch
{% endhighlight %}
Pour relier les 2 routeurs, connectons-nous sur l’un des noeuds, par exemple `orion.local`, et utilisons la commande `weave connect` suivie de l’adresse IP publique (sur le réseau physique) du second noeud :
{% highlight sh %}
orion.local:~$ sudo weave connect 192.168.0.3
{% endhighlight %}
> **Note :** La commande `weave status` permet de connaître l’état du routeur.

## Déploiement des services dockerisés sur le réseau Weave
La commande `weave run` permet de démarrer un conteneur et de le relier au réseau Weave. Elle est immédiatement suivie de l’adresse IP assignée sur le réseau virtuel :

- sur `orion.local` :
<pre>
orion.local:~$ sudo weave run 10.0.1.1/24 ksahnine/dummy-http svc1
orion.local:~$ sudo weave run 10.0.1.2/24 ksahnine/dummy-http svc2
</pre>
- sur `macbook.local` :
<pre>
macbook.local:~$ sudo weave run 10.0.1.3/24 ksahnine/dummy-http svc3
</pre>

> **Note :** La commande `weave run` démarre le conteneur en mode détaché (en tâche de fond). Par ailleurs, les ports exposés par le conteneur sont par défaut accessibles à l’extérieur du conteneur.

## Accéder aux services dockerisés
A ce stade, le réseau de conteneurs est opérationnel. Voyons comment invoquer les services dockerisés de l’hôte `orion.local` depuis un conteneur localisé sur `macbook.local`.<br />
Pour la démonstration, créons un nouveau conteneur issu de l’image Docker `tutum/curl` de l’hébergeur [Tutum](https://www.tutum.co/) sur `macbook.local` et relié au réseau Weave :
{% highlight sh %}
macbook.local:~$ sudo weave run 10.0.1.4/24 -ti tutum/curl /bin/bash
04d37c6f7781a7046c646999eae43a69d5d51c2114b2d87cf8e10ab033721e63
{% endhighlight %}
Accédons à la console du conteneur :
{% highlight sh %}
macbook.local:~$ sudo docker attach 04d37c6f7781
{% endhighlight %}
> **Note :** Penser à taper sur la touche `Enter` pour avoir le prompt.

Appelons les 3 services via cURL :
{% highlight sh %}
root@04d37c6f7781:/# curl 10.0.1.1:8080
Hi! I'm [svc1] service and my Docker container's IP is [172.17.0.6]
root@04d37c6f7781:/# curl 10.0.1.2:8080
Hi! I'm [svc2] service and my Docker container's IP is [172.17.0.7]
root@04d37c6f7781:/# curl 10.0.1.3:8080
Hi! I'm [svc3] service and my Docker container's IP is [172.17.0.20]
{% endhighlight %}
Plusieurs remarques :

- notez que les 2 services `svc1` et `svc2`, co-localisés sur le même hôte Docker, sont accessibles à travers le même port ! (`8080`)
- l’adresse IP retournée par le service (`172.17.0.x`) est celle de l’interface veth reliée au bridge `docker0`. Chaque conteneur a donc 2 interfaces réseaux :
  - l’une reliée au sous-réseau créé par Docker et partagé entre l’hôte et ses conteneurs
  - l’autre reliée au réseau virtuel Weave :
{% highlight sh %}
root@04d37c6f7781:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:15
          inet addr:172.17.0.21  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:15/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

ethwe     Link encap:Ethernet  HWaddr 32:0f:05:02:d7:1e
          inet addr:10.0.1.4  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::300f:5ff:fe02:d71e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:65535  Metric:1
          RX packets:31 errors:0 dropped:0 overruns:0 frame:0
          TX packets:32 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2674 (2.6 KB)  TX bytes:2391 (2.3 KB)
{% endhighlight %}

  - il devrait être possible de remplacer le bridge `docker0` par le bridge weave via l’option `--bridge` du démon Docker, mais je n’ai pas encore testé. Voir la documentation de Docker : [Advanced networking](https://docs.docker.com/articles/networking/#bridge-building).

## Simplifier le déploiement avec Weave Proxy
**Weave Proxy** est un conteneur Docker interceptant toutes les commandes émises par le client Docker vers le démon Docker.<br />
Il permet d’utiliser directement la commande `docker run` pour **enregistrer automatiquement** un conteneur sur le réseau Weave ou dans un DNS comme on le verra dans le paragraphe suivant.<br />
Sur chaque noeud :

- lancer le routeur et le proxy Weave :
<pre>
\# weave launch && weave launch-proxy
</pre>

> **Note :** le cas échéant, utiliser la commande `weave reset` pour réinitialiser le routeur Weave.

- configurer le client Docker pour le faire pointer vers Weave Proxy :
<pre>
\# \`weave proxy-env\`
</pre>
ce qui est équivalent à :
<pre>
\# export DOCKER_HOST=tcp://127.0.0.1:12375
</pre>

Créons un conteneur et assignons lui une adresse IP spécifique sur le réseau Weave :
{% highlight sh %}
docker run --rm -e WEAVE_CIDR=10.0.1.4/24 -ti tutum/curl /bin/bash
{% endhighlight %}
Le support d’[IPAM](https://en.wikipedia.org/wiki/IP_address_management) permet d’assigner automatique d’une adresse IP au conteneur. Il suffit d’omettre la variable d’environnement `WEAVE_CIDR`, soit :
{% highlight sh %}
docker run --rm -ti tutum/curl /bin/bash
{% endhighlight %}
La plage d’adresse IP peut être redéfinie au lancement du routeur via l’option `-iprange`, par ex: `weave launch -iprange 10.2.3.0/24`.

## Découverte de services par DNS
En combinant le routeur Weave, le proxy Weave et le DNS Weave, il est aisé de mettre en oeuvre une **solution de découverte de service** totalement dynamique.<br />
Pour ce faire, il suffit d’activer les 3 composants Weave sur chaque noeud :

- lancer le routeur, le DNS et le proxy Weave :
<pre>
\# weave launch && weave launch-dns && weave launch-proxy
</pre>
- configurer le client Docker pour le faire pointer vers le proxy Weave :
<pre>
\# \`weave proxy-env\`
</pre>

Déployons le service `svc1` :
{% highlight sh %}
orion.local:~# docker run -d --name svc1 ksahnine/dummy-http svc1
{% endhighlight %}
> **Note :** le paramètre `--name` est utilisé pour enregistrer le service dans le DNS. Ainsi, son nom pleinement qualifié sera `svc1.weave.local`.

Créons un nouveau conteneur issu de l’image Docker `tutum/curl` de l’hébergeur [Tutum](https://www.tutum.co/) sur `macbook.local` et relié au réseau Weave :
{% highlight sh %}
orion.local:~# docker run --rm -ti tutum/curl /bin/bash
{% endhighlight %}
Enfin, vérifions que le service est accessible sous son nom pleinement qualifié (`svc1.weave.local`) :
{% highlight sh %}
root@8f73b6902ee2:/# curl svc1.weave.local:8080
Hi! I'm [svc1] service and my Docker container's IP is [172.17.0.21]
{% endhighlight %}
Noter que l’architecture est **scalable horizontalement** en déployant le service `svc1` sur plusieurs hôtes Docker et ce, sans répartiteur de charge ou reverse-proxy.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
