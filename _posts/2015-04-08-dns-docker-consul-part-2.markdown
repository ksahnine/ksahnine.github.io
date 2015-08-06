---
layout: post
title:  "Système de découverte de services par DNS : Docker et Consul - 2/2"
date:   2015-04-08 06:00:00
categories: docker consul microservice
tags: regular
image: /assets/article_images/917687222-maersk-line-eurogate-container-terminal-bremerhaven-discharging-unloading-gantry-crane.jpg
legend: Vue nocturne du terminal de Brehmerhaven, Allemagne
---
Dans mon [article précédent](/docker/consul/microservice/2015/03/30/dns-docker-consul-part-1.html), dont la lecture est un préalable, j’avais décrit les principes de la **découverte de services par DNS** comme technique de communication entre conteneurs Docker et ce, indépendamment de leur localisation physique sur un serveur hôte.<br />
Lors de sa [mise en oeuvre pratique](/docker/consul/microservice/2015/03/30/dns-docker-consul-part-1.html) avec un registre **Consul**, nous avions vu en particulier que l’enregistrement manuel des services via l’[**API Agent**](https://www.consul.io/docs/agent/http/agent.html#agent_service_register) pouvait être fastidieuse.

Dans ce second volet de la série, nous mettrons l’accent sur une technique d’enregistrement à chaud de services dockerisés grâce à [**Registrator**](https://github.com/gliderlabs/registrator), développé par [Jeff Lindsay](http://progrium.com/blog/) et disponible [sous la fome d’une image Docker](https://registry.hub.docker.com/u/gliderlabs/registrator/).

## Etude de cas
Le schéma ci-dessous décrit l’architecture d’un cluster constitué de 3 noeuds hébergeant chacun un **agent Consul** et un **hôte Docker**. <br />
Sur chacun des hôtes Docker sont déployés :

- un conteneur **Registrator**
- un conteneur hébergeant un **web service** HTTP (respectivement `svc1`, `svc2` et `svc3`)

![Architecture Docker/Consul/Registrator](http://blog.inovia-conseil.fr/wp-content/uploads/2015/04/consul-cluster-registrator.png
)

Pour accompagner la lecture de cet article, le cluster décrit ci-dessus est provisionnable localement via une configuration **Vagrant** que je mets à disposition sur *Github*, et dont la mise en oeuvre est des plus simples :
{% highlight sh %}
git clone https://github.com/ksahnine/vagrant-config.git
cd vagrant-config/consul-cluster-registrator
vagrant up
{% endhighlight %}
Repartez de cette configuration si vous souhaitez reproduire les exemples de l’article.

> **Note :** Le temps de construction et de démarrage du cluster en partant de zéro est long. Compter plusieurs (longues) minutes.

## Fonctionnement
Lors de l’initialisation du cluster, chaque noeud démarre :

- un agent *Consul* (en mode serveur sur `node1`) : l’**API HTTP Agent** est alors accessible sur le port `8500` de chaque agent
- un conteneur *Registrator* :
  - à l’écoute des évènements Docker `start`, `die`, `stop` et `kill` des conteneurs co-localisés
  - connecté à l’agent *Consul* de l’hôte Docker sur lequel il est déployé
  <br />Ainsi, le démarrage du conteneur *Registrator* sur `node1` (adresse IP `172.20.20.10`) s’effectue comme suit :
{% highlight sh %}
docker run -d -v /var/run/docker.sock:/tmp/docker.sock -h node1 gliderlabs/registrator consul://172.20.20.10:8500
{% endhighlight %}

A ce stade, *Registrator* est prêt à intercepter les évènements de tout conteneur susceptible d’être créé ou détruit, à l’issue de quoi il met à jour en conséquence le registre *Consul*.

> **Note :** le fonctionnement d’un cluster Consul est décrit dans la [première partie de cet article](/docker/consul/microservice/2015/03/30/dns-docker-consul-part-1.html).

## Un cas pratique d'enregistrement de service à chaud
Le cluster étant opérationnel, commençons par déployer le service `svc4` sur le serveur `node3` :
{% highlight sh %}
vagrant ssh node3
vagrant@node3:~$ docker run -td -e "SERVICE_NAME=svc4" -p 8084:8080 --name svc4 ksahnine/dummy-http svc4
{% endhighlight %}
On notera que :

- la **variable d’environnement** `SERVICE_NAME=svc4` permet d’attribuer explicitement le nom du service dans le registre. Dans notre exemple, le service dockerisé sera accessible via le FQDN `svc4.service.consul`
- *Registrator* enregistre le service à l’interception de l’évènement `start` via l’API [`/v1/agent/service/register`](https://www.consul.io/docs/agent/http/agent.html#agent_service_register)
- *Registrator* désenregistre le service à l’interception des évènements `stop`, `kill` et `die`, via l’API [`/v1/agent/service/deregister`](https://www.consul.io/docs/agent/http/agent.html#agent_service_deregister)

En interrogeant Consul via l’API DNS, on constate que le nouveau service est correctement enregistré et déployé sur le noeud 3 :
{% highlight sh %}
vagrant@node2:~$ dig @172.20.20.10 svc4.service.consul SRV

;; QUESTION SECTION:
;svc4.service.consul.		IN	SRV

;; ANSWER SECTION:
svc4.service.consul.	0	IN	SRV	1 1 8084 node3.node.dc1.consul.

;; ADDITIONAL SECTION:
node3.node.dc1.consul.	0	IN	A	172.20.20.30
{% endhighlight %}

Enfin, appelons ce service depuis un conteneur Docker issu de l’image `tutum/curl` déployé sur le serveur `node2` :
{% highlight sh %}
vagrant ssh node2
vagrant@node2:~$ docker run -ti --dns 172.20.20.10 --dns-search service.consul tutum/curl
root@afc268967ce9:/# curl http://svc4:8084/
Hi! I'm [svc4] service and my Docker container's IP is [172.17.0.4]
{% endhighlight %}
Arrêtons le conteneur `svc4` :
{% highlight sh %}
vagrant ssh node3
vagrant@node3:~$ docker kill svc4
{% endhighlight %}
On constate que le service `svc4.service.consul` a été désenregistré à chaud :
{% highlight sh %}
vagrant@node3:~$ dig +short @172.20.20.10 svc4.service.consul
vagrant@node3:~$
{% endhighlight %}

Voilà. Le registre *Consul* est mis à jour dynamiquement et reflète parfaitement le plan de déploiement des services dockerisés, ce qui est éminemment utile dans une **architecture de microservices**.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
