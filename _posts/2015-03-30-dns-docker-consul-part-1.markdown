---
layout: post
title:  "Système de découverte de services par DNS : Docker et Consul - 1/2"
date:   2015-03-30 06:00:00
categories: docker consul microservice
tags: regular
image: /assets/article_images/bg.jpg
legend: Vue de jour du terminal de Brehmerhaven, Allemagne
---
Comme je l’évoquais dans un précédent billet, il y a tout lieu de penser que les **architectures de microservices** et **polyglottes** constituent un horizon proche des Systèmes d’Information d’Entreprise, quand ce n’est pas déjà une réalité pour de grands acteurs du web (*Netflix*, *Airbnb*).<br />
Dans ce contexte, **Docker** apparait comme une véritable bénédiction tant ses qualités sont en symbiose parfaite avec ce nouveau paradigme. Il répond remarquablement aux besoins :

- d’**isolation** de l’environnement d’exécution d’un service
- de déploiement **rapide** et sans couture
- de **portabilité** et de **scalabilité**
- de compatibilité avec des solutions d’hébergement dans un **Cloud** privé, public ou hybride

Cependant, la communication entre conteneurs Docker répartis sur plusieurs machines hôtes n’est vraiment pas triviale à mettre en oeuvre.<br />
Dans le cas d’une architecture constituée d’un très grand nombre de microservices dockerisés, la difficulté peut conduire au [naufrage](http://fr.wikipedia.org/wiki/Le_Radeau_de_La_M%C3%A9duse)…<br />
Il existe deux approches pour traiter ce problème complexe :

- le **paradigme** [**SDN**](https://fr.wikipedia.org/wiki/Software_Defined_Networking) (Software-Defined Networking), notamment mis en oeuvre par [**Weave**](), [**Open vSwitch**](http://openvswitch.org/) ou [**Socketplane**](https://github.com/socketplane/socketplane), permet de construire un **réseau virtuel de conteneurs** répartis sur plusieurs hôtes Docker. A ce propos, [l’acquisition récente de Socketplane](https://blog.docker.com/2015/03/socketplane-excited-to-be-joining-docker-to-collaborate-with-networking-ecosystem/) par *Docker Inc* semble annoncer l’apparition prochaine d’une sorte “d’API réseau” standardisée dans l’écosystème Docker.
- la **découverte de service** est une technique permettant de trouver une ressource (application, service) sur un réseau. Cette technique n’est pas spécifique au monde Docker bien sûr (songez au DNS) mais elle a fait ses preuves dans cet écosystème grâce à l’adoption grandissante de [**Consul**](https://consul.io/) et [**SkyDNS2**](https://github.com/skynetservices/skydns/) / [**etcd**](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/), des solutions de découverte de service, distribuées et hautement disponibles. On ne résout pas l’intégralité des problèmes de communication inter-conteneurs, mais disons que ce modèle est suffisant dans la plupart des cas d’utilisation.

C’est cette dernière approche que je vous propose d’étudier car elle a le mérite d’être peu intrusive et plus simple à mettre en oeuvre qu’un SDN.<br />
En pratique, nous verrons comment :

- installer et configurer un cluster Consul (sur un environnement provisionnable localement avec **Vagrant**)
- enregistrer des services dockerisés
- communiquer avec un service depuis un conteneur Docker, sans connaitre sa localisation physique (adresse IP ou FQDN)

## Consul : présentation
**Consul** est une solution de découverte de service, distribuée et hautement disponible, développée par la société [Hashicorp](https://www.hashicorp.com/) (dont le fondateur est le créateur de [Vagrant](https://www.vagrantup.com/)).<br />
Par ailleurs, **Consul** est également :

- un datastore distribué de type clé/valeur pour le stockage d’éléments de configuration
- un système de supervision de services (bilan de santé, détection de pannes)

*Consul* est disponible sous [Linux, Windows et Mac OS](https://consul.io/downloads.html) mais seule l’utilisation sous Linux est recommandée en production.<br />
Le schéma ci-dessous décrit l’architecture d’un cluster constitué de 3 noeuds hébergeant chacun un **agent Consul** et un **hôte Docker**. J’ai également représentés sur ce schéma les **services dockerisés** déployés (`svc1`, `svc2` et `svc3`), dont certains sont redondants (`svc1` et `svc2`) :

<center><img src="{{site.url}}/assets/article_images/consul-cluster.png" style="display: block; margin: auto;" /></center>

> **Note :** Je mets à disposition [deux configurations Vagrant](https://github.com/ksahnine/vagrant-config) pour provisionner localement ce cluster et expérimenter vous même les exemples illustrant cet article. Pour les impatients, consultez directement le paragraphe [suivant](#1).

Un **agent** Consul est un composant essentiel chargé d’enregistrer les services, répondre aux requêtes, collecter des informations du cluster etc.<br />
S’exécutant sur chaque noeud du cluster, il peut fonctionner en mode **client** ou **serveur**. Le cluster doit disposer d’au moins un agent serveur. Au delà, un mécanisme d’élection du **leader** du cluster permet d’attribuer ce rôle à un des agents serveur. En production, il est recommandé d’avoir de 3 à 5 agents serveur par cluster pour éviter toute perte de données en cas de panne d’un des serveurs.

Chaque agent supporte 3 protocoles de communication :

- RPC (port par défaut `8400`)
- HTTP (port par défaut `8500`) : une [API RESTful est exposée](https://www.consul.io/docs/agent/http.html)
- DNS (port par défaut `8600)

Les ports `8300`, `8301` et `8302` sont des ports internes réservés à la communication entre agents.

## <a name="1"></a>Configuration Vagrant
Pour accompagner la lecture de cet article, deux configurations du cluster sont provisionnables localement via **Vagrant** :

- Configuration **partielle** : services Dockers `svc1`, `svc2` et `svc3` provisionnés mais pas de cluster Consul :
{% highlight sh %}
git clone https://github.com/ksahnine/vagrant-config.git
cd vagrant-config/consul-cluster-blank
vagrant up
{% endhighlight %}
<br />Repartez de cette configuration si vous souhaiter reproduire les exemples de l’article.
- Configuration complète : services Dockers `svc1`, `svc2` et `svc3` provisionnés, cluster Consul configuré et provisionné :
{% highlight sh %}
git clone https://github.com/ksahnine/vagrant-config.git
cd vagrant-config/consul-cluster-services
vagrant up
{% endhighlight %}

> **Note :** Le temps de construction et de démarrage du cluster en partant de zéro est long. Compter plusieurs minutes.

## Mise en oeuvre du cluster
Commençons par démarrer l’agent du premier noeud en mode serveur sur `node1` dont l’IP privée est `172.20.20.10` :
{% highlight sh %}
vagrant ssh node1
vagrant@node1:~$ consul agent -node node1 -server -bootstrap-expect 1 -data-dir /tmp/consul \
                              -client 172.20.20.10  -advertise 172.20.20.10 &
{% endhighlight %}
L’agent revendique le rôle de leader du cluster :
{% highlight text %}
2015/03/29 13:58:13 [INFO] raft: Node at 172.20.20.10:8300 [Candidate] entering Candidate state
2015/03/29 13:58:13 [INFO] raft: Election won. Tally: 1
2015/03/29 13:58:13 [INFO] raft: Node at 172.20.20.10:8300 [Leader] entering Leader state
{% endhighlight %}

On notera que :

- le flag `-node` permet d’attribuer un nom à l’agent (`node1` dans le cas présent)
- le flag `-server` permet de configurer l’agent en mode serveur
- le flag `-bootstrap-expect 1` est suivi du nombre d’agent serveur devant être actifs avant que le cluster ne soit opérationnel. Dans notre cas, on aurait pu utiliser le flag `-bootstrap` car il n’y a qu’un agent serveur.
- le flag `-client` définit l’**adresse client** de l’agent exposant les interfaces DNS, HTTP et RPC.
- le flag `-advertise` définit l’**adresse cluster** de l’agent, c’est-à-dire l’adresse IP atteignable par les autres agents du cluster.

> **Note :** On peut également externaliser les paramètres de l’agent dans un fichier de configuration au format JSON et activer l’agent via la commande `consul agent -config-file conf/node1.json` :
{% gist e3e6d1fa27c7c08a1ea4 %}

Sur le **deuxième noeud** (`node2` dont l’IP privée est `172.20.20.20`), démarrons l’agent consul en mode client :
{% highlight sh %}
vagrant ssh node2
vagrant@node2:~$ consul agent -node node2 -data-dir /tmp/consul \
         -client 172.20.20.20 -advertise 172.20.20.20 &
{% endhighlight %}

A ce stade, le deuxième agent n’a toujours pas rejoint le cluster. Pour ce faire, utiliser la commande `consul join` suivie de l’adresse IP du leader (`node1`) :

{% highlight sh %}
vagrant@node2:~$ consul join -rpc-addr 172.20.20.20:8400 172.20.20.10
{% endhighlight %}

Procédons de même sur le **troisième noeud** (`node3.local` dont l’IP privée est `10.10.0.3`) mais en une seule commande grâce au flag `-join` :
{% highlight sh %}
vagrant ssh node3
vagrant@node3:~$ consul agent -node node3 -data-dir /tmp/consul \
          -client 172.20.20.30 -advertise 172.20.20.30 -join 172.20.20.10 &
{% endhighlight %}

Pour connaître l’état du cluster Consul, utiliser la commande `consul members` sur n’importe quel noeud :
{% highlight sh %}
vagrant@node3:~$ consul members -rpc-addr 172.20.20.30:8400
Node   Address             Status  Type    Build  Protocol
node3  172.20.20.30:8301   alive   client  0.5.0  2
node2  172.20.20.20:8301   alive   client  0.5.0  2
node1  172.20.20.10:8301   alive   server  0.5.0  2
{% endhighlight %}

> **Note :** si un des agents client venait à tomber, son statut passerait de `alive` à `left`.

Voilà. Le cluster est maintenant opérationnel.

## Enregistrement, désenregistrement et découverte de services par DNS
Chaque agent expose une **API HTTP** permettant d’enregistrer, désenregistrer ou consulter les services qu’il administre.<br />
On peut consulter la définition d’un service (nom, adresse IP, port) via l’API HTTP ou l’API DNS (rappelons que chaque agent fait aussi office de resolver DNS).

### Enregistrement
Enregistrons les 2 instances du service `svc1` des noeuds `node1` et `node2` (port `8081`) :
{% highlight sh %}
node1@vagrant:~$ curl -XPUT 172.20.20.10:8500/v1/agent/service/register -d '{
  "id":"svc1_instance_1",
  "name": "svc1",
  "address": "172.20.20.10",
  "port": 8081,
  "tags": ["api"] }'

node1@vagrant:~$ curl -XPUT 172.20.20.20:8500/v1/agent/service/register -d '{
  "id":"svc1_instance_2",
  "name": "svc1",
  "address": "172.20.20.20",
  "port": 8081,
  "tags": ["api"] }'
{% endhighlight %}
La déclaration manuelle des services peut sembler fastidieuse à bien des égards.<br />
Dans le prochain article de la série, nous mettrons en place une solution *"plug and play"* permettant d’enregistrer automatiquement un service dockerisé au démarrage du conteneur et de désenregistrement de celui-ci lorsque qu’il s’arrête.

### Découverte par DNS
Un service est déclaré dans le DNS Consul sous la forme d’un **enregistrement** [**SRV**](http://fr.wikipedia.org/wiki/Enregistrement_de_service) (ou enregistrement de service) suffixé par `.service.consul`.<br />
Ainsi, les noms pleinement qualifiés des 3 services dockerisés seront :

- `srv1.service.consul`
- `srv2.service.consul`
- `srv3.service.consul`

Interrogeons le DNS Consul, via l’agent sur `node1`/`172.20.20.10`, pour localiser le service `svc1.service.consul` :
{% highlight sh %}
vagrant ssh node2
vagrant@node2:~$ dig @172.20.20.10 -p 8600 svc1.service.consul SRV
{% endhighlight %}
On observe que le service est bien déclaré sur `172.20.20.10:8081` et `172.20.20.20:8081` :

{% highlight text %}
;; ANSWER SECTION:
svc1.service.consul.	0	IN	SRV	1 1 8081 node2.node.dc1.consul.
svc1.service.consul.	0	IN	SRV	1 1 8081 node1.node.dc1.consul.

;; ADDITIONAL SECTION:
node2.node.dc1.consul.	0	IN	A	172.20.20.20
node1.node.dc1.consul.	0	IN	A	172.20.20.10
{% endhighlight %}

On remarquera que l’ordre d’apparition des adresses IP lors de la résolution du nom de service est **aléatoire** :
{% highlight sh %}
vagrant@node2:~$ dig +short @172.20.20.10 -p 8600 svc1.service.consul
172.20.20.20
172.20.20.10
{% endhighlight %}
En requêtant une fraction de seconde plus tard, l’ordre a changé :
{% highlight sh %}
vagrant@node2:~$ dig +short @172.20.20.10 -p 8600 svc1.service.consul
172.20.20.10
172.20.20.20
{% endhighlight %}
Il est ainsi possible de répartir aléatoirement la charge sur l’une ou l’autre instance du service, ouvrant la voie à une API RESTful **scalable horizontalement**.

### Désenregistrement
Le point de terminaison `/v1/agent/service/deregister/` exposé par l’API HTTP de l’agent permet de désenregistrer un service dont il a la charge :
{% highlight sh %}
vagrant@node1:~$ curl -XGET 172.20.20.10:8500/v1/agent/service/deregister/svc1_instance_1
vagrant@node1:~$ curl -XGET 172.20.20.20:8500/v1/agent/service/deregister/svc1_instance_2
{% endhighlight %}

## Communiquer avec un service depuis un conteneur Docker
Voyons comment appeler le service dockerisé `svc1.service.consul` déployé sur `node1` et `node2` depuis un conteneur Docker situé sur `node3`.<br />
Nous utiliserons l’image Docker `tutum/curl` de l’hébergeur [Tutum](https://www.tutum.co/) pour la démonstration :
{% highlight sh %}
vagrant ssh node3
vagrant@node3:~$ docker run -ti --dns 172.20.20.10 --dns-search service.consul tutum/curl
root@200b369020f6:/# curl http://svc1:8081/
Hi! I'm [svc1] service and my Docker container's IP is [172.17.0.2]
root@200b369020f6:/# curl http://svc1:8081/
Hi! I'm [svc1] service and my Docker container's IP is [172.17.0.3]
{% endhighlight %}

- Le flag `--dns` est suivi de l’adresse IP de l’agent Consul du noeud 1 utilisé comme resolver DNS.

> Cela ne fonctionne que si l’agent est configuré pour écouter les requêtes DNS sur le port 53 (le port DNS Consul par défaut étant 8600).

- Le flag `--dns-search` permet de forcer les recherches DNS sur le domaine passé en paramètre.

{% highlight sh %}
{% endhighlight %}

{% highlight sh %}
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
