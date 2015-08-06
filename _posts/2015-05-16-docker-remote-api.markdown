---
layout: post
title:  "Docker Remote API : piloter un hôte Docker à distance"
date:   2015-05-16 06:00:00
categories: docker architecture
tags: regular
image: /assets/article_images/AP-02884_1411028829.jpg
legend: Dockers sur le vieux port de Marseille - Georges POMERAT
---
Parmi toutes les fonctionnalités disponibles, il en est une qui explique pour beaucoup le dynamisme phénoménal de l’écosystème Docker. Il s’agit de [**Docker Remote API**](https://docs.docker.com/reference/api/docker_remote_api/), une **API REST** extrêmement bien conçue exposant toutes les fonctionnalités du moteur Docker et permettant ainsi de piloter un hôte Docker depuis une machine ou une application distante.

Toutes les solutions d’orchestration de conteneurs Docker utilisent directement ou indirectement l’API Docker Remote, raison pour laquelle je vous propose d’y jeter un oeil attentif.

Dans cet article, nous verrons :

- comment fonctionne l’interaction avec le **démon Docker**
- comment activer **Docker Remote API** avec et sans certificats TLS
- comment utiliser **Docker Machine** avec un hôte Docker déjà provisionné sur un réseau local d’entreprise

## La communication avec le démon Docker
Le schéma ci-dessous représente l’architecture type d’un serveur Linux (sous Ubuntu) hébergeant un hôte Docker. Le moteur Docker (serveur) est un démon écoutant par défaut sur un **socket UNIX** (`/var/run/docker.sock`).<br />
Pour mémoire, les sockets UNIX permettent à deux ou plusieurs processus d’échanger des données de façon bi-directionnelle.
<center>![Hôte Docker](http://blog.inovia-conseil.fr/wp-content/uploads/2015/05/docker-engine.png)</center>

Ainsi, le client Docker **installé localement** interagit avec le démon via le socket UNIX `/var/run/docker.sock`. Sans surprise, le protocole d’échange utilisé par l’API Docker est de type **REST** comme nous allons le voir ci-après.

Exécutons par exemple la commande `GET /info`, permettant d’obtenir des informations système de l’hôte Docker.<br />
Pour ce faire, j’utilise l’utilitaire [Netcat](http://netcat.sourceforge.net/) (commande `nc`) afin d’envoyer la requête au socket `/var/run/docker.sock` :
{% highlight sh %}
echo -e “GET /info HTTP/1.0rn” | sudo nc -U /var/run/docker.sock
{% endhighlight %}
Voici un extrait de la réponse obtenue :
{% highlight text %}
HTTP/1.0 200 OK
Content-Type: application/json
Job-Name: info
Date: Tue, 12 May 2015 19:56:14 GMT
Content-Length: 889

{”Containers”:118,[…]”Name”:”orion”,”OperatingSystem”:”Ubuntu 14.04.2 LTS”,”RegistryConfig”:
{”IndexConfigs”:{”docker.io”:{”Mirrors”:null,”Name”:”docker.io”,”Official”:true,”Secure”:true}},
“InsecureRegistryCIDRs”:[”127.0.0.0/8″]},”SwapLimit”:0,
“SystemTime”:”2015-05-12T21:56:14.735623265+02:00″}
{% endhighlight %}
On remarquera que le corps de la réponse est au format **JSON**.

Maintenant, voyons à quoi ressemblerait une combinaison associant :

- un hôte Docker avec activation de *Docker Remote API*
- un client Docker localisé sur une machine distante
<center>![Hôte Docker distant](http://blog.inovia-conseil.fr/wp-content/uploads/2015/05/docker-engine-remote.png)</center>

Voyons comment activer et utiliser *Docker Remote API* avec et sans certificats TLS.

## Activer Docker Remote API sans certificat TLS
Cette configuration n’est pas recommandée en production car la communication entre client et hôte Docker est en clair. Elle est néanmoins fort utile à des fins de test. Voici comment la mettre en oeuvre :

Côté serveur (`orion.local` sous Ubuntu) :

- éditer le fichier `/etc/default/docker` puis renseigner la variable d’environnement `DOCKER_OPTS` comme suit :
<pre>
DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"
</pre>
- redémarrer le démon docker :
<pre>
**ksahnine@orion:~$** sudo service docker restart
</pre>

A ce stade, le démon écoute sur le port TCP `2375` ainsi que sur le socket UNIX `/var/run/docker.sock`.
> **Note :** Le fichier `/etc/default/docker` n’est présent que sous Ubuntu. A défaut, il faudra renseigner et exporter la variable d’environnement `DOCKER_OPTS`.

Côté client (`laptop.local`) :

- pour interroger l’hôte Docker depuis une autre machine, utiliser la commande docker en renseignant l’option `-H` avec l’adresse et le port d’écoute de l’hôte Docker distant.
<br />L’exemple ci-dessous permet d’afficher la liste des conteneurs actifs sur l’hôte Docker distant :
<pre>
ksahnine@laptop:~$ docker -H=192.168.0.29:2375 ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
6faede9c46af        nginx:latest        “nginx -g ‘daemon of   12 minutes ago      Up 12 minutes       80/tcp, 443/tcp     sleepy_lalande
</pre>

> **Note :** On peut aussi renseigner la variable d’environnement `DOCKER_HOST` sur le poste client (`export DOCKER_HOST=192.168.0.29:2375`) et lancer tout simplement la commande `docker ps` sans l’option `-H`

A l’aide de `tcpdump`, interceptons les échanges entre le client Docker et l’hôte Docker distant :
{% highlight sh %}
sudo tcpdump -c 20 -s 0 -i en1 -A host 192.168.0.29 and tcp port 2375
{% endhighlight %}
On remarquera que la requête issue de la commande `docker ps` est la suivante :
{% highlight text %}
GET /v1.18/containers/json HTTP/1.1
Host: 192.168.0.29:2375
User-Agent: Docker-Client/1.6.0
Accept-Encoding: gzip
{% endhighlight %}
On pourrait donc se passer du client Docker et utiliser directement l’utilitaire **cURL**, dont la commande équivalente à `docker ps` serait :
{% highlight sh %}
curl -X GET http://192.168.0.29:2375/v1.18/containers/json
{% endhighlight %}

## Activer Docker Remote API avec certificat TLS
Il nous faut tout d’abord produire 3 certificats :

- le **certificat racine** utilisé comme autorité de certification. Il sera utilisé pour signer les certificats client et serveur.
- le **certificat serveur** installé et utilisé par l’hôte Docker
- le **certificat client** installé et utilisé par le client Docker

Je mets à disposition un [script shell](https://github.com/ksahnine/docker/blob/master/gen-docker-certs.sh) permettant de générer les certificats de test et dont voici le mode opératoire d’utilisation correspondant à notre cas :
<pre>
**ksahnine@orion:~$** wget https://raw.githubusercontent.com/ksahnine/docker/master/gen-docker-certs.sh
**ksahnine@orion:~$** sh gen-docker-certs.sh
- Nom de l’hôte Docker : orion
- Adresse IP de l’hôte Docker : 192.168.0.29
</pre>
Le script produit :

- le certificat **racine** (clé publique `ca.pem`)
- le certificat **client** (clé publique `cert.pem` / clé privée `key.pem`)
- le certificat **serveur** (clé publique `server.pem` / clé privée `server-key.pem`)

Passons à la configuration du serveur et du client Docker.

Côté serveur (`orion.local` sous Ubuntu) :

- copier les fichier `ca.pem`, `server.pem` et `server-key.pem` dans le répertoire `/root/.docker` (ou tout autre répertoire sécurisé)
- éditer le fichier `/etc/default/docker` puis renseigner la variable d’environnement `DOCKER_OPTS` comme suit :
<pre>
DOCKER_OPTS="--tlsverify --tlscacert=/root/.docker/ca.pem --tlscert=/root/.docker/server.pem \\
     --tlskey=/root/.docker/server-key.pem -H=unix:///var/run/docker.sock -H=0.0.0.0:2376"
</pre>
- redémarrer le démon docker :
<pre>
**ksahnine@orion:~$** sudo service docker restart
</pre>
A ce stade, le démon écoute sur le socket UNIX `/var/run/docker.sock` ainsi que sur le port TCP `2376`, port utilisé par convention pour les échanges sécurisés.

Côté client (`laptop.local`) :

- copier les fichiers `ca.pem`, `cert.pem` et `key.pem` dans le répertoire `~/.docker`
- utiliser la commande `docker` avec le flag `--tlsverify` pour passer en mode TLS.
<br />L’exemple ci-dessous permet d’afficher la liste des conteneurs actifs sur l’hôte Docker distant :
<pre>
**ksahnine@laptop:~$** docker –tlsverify -H=192.168.0.29:2376 ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
6faede9c46af        nginx:latest        “nginx -g ‘daemon of   12 minutes ago      Up 12 minutes       80/tcp, 443/tcp     sleepy_lalande
</pre>

> **Note :** Vous avez noté que je n’ai pas passé explicitement les clés dans la ligne de commande. Par défaut, le client docker s’attend à trouver le certificat racine sous `~/.docker/ca.pem`, le certificat client sous `~/.docker/cert.pem` et la clé privée sous `~/.docker/key.pem`.
> Si les noms des clés diffèrent ou s’ils sont localisés dans un autre répertoire, il faudra utiliser respectivement les options `--tlscacert`, `--tlscert` et `--tlskey`.
> Le flag `--tlsverify` peut être omis si la variable d’environnement `DOCKER_TLS_VERIFY` est valorisée à `1`.

## Utilisation avec Docker Machine
Si vous gérez plusieurs hôtes Docker répartis sur un LAN et/ou dans un cloud public, il est beaucoup plus pratique de travailler avec [**Docker Machine**](https://docs.docker.com/machine/) depuis un simple ordinateur portable.<br />
Dans le cas de cet article, un hôte Docker est déjà provisionné sur le serveur Linux `orion.local` (`192.168.0.29`).<br />
Rajoutons l’hôte sous le nom `orionbox` :
<pre>
**ksahnine@laptop:~$** docker-machine create –driver none –url=tcp://192.168.0.29:2376 orionbox
</pre>

La commande `docker-machine ls` le confirme :
<pre>
**ksahnine@laptop:~$** docker-machine ls
NAME       ACTIVE   DRIVER   STATE   URL                       SWARM
orionbox   *        none             tcp://192.168.0.29:2376
</pre>

Copier les certificats client, serveur et racine précédemment générés (fichiers `ca.pem`, `cert.pem`, `key.pem`, `server.pem` et `server-key.pem`) dans le répertoire `~/.docker/machine/machines/orionbox`.

La commande `docker-machine env` affiche les variables d’environnement à positionner pour forcer le client docker à interagir avec une machine particulière, `orionbox` dans notre cas :
{% highlight sh %}
ksahnine@laptop:~$ docker-machine env orionbox
Bad port ‘0′
ERRO[0002] Error running SSH command to get /etc/os-release: exit status 255
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=”/Users/ksahnine/.docker/machine/machines/orionbox”
export DOCKER_HOST=tcp://192.168.0.29:2376
{% endhighlight %}
> **Note :** Remarquez l’erreur d’exécution obtenue en sortie. A ma connaissance, l’utilisation de l’option `--driver none` ne permet pas de configurer d’authentification par clés SSH afin que Docker Machine puisse exécuter des commandes à distance.
> A ce jour, Docker Machine est encore en version bêta (v 0.20). Sauf erreur de ma part, il faudra attendre la publication du driver generic actuellement [en cours de développement](https://github.com/docker/machine/tree/master/drivers/generic). Il devrait permettre de provisionner un hôte Docker sur n’importe quel serveur existant accessible par SSH, un peu à la manière [d’Ansible](http://blog.inovia-conseil.fr/blog.inovia-conseil.fr/?p=180).

> **Mise à jour 29/06/2015** : Le driver `generic` est désormais disponible depuis la [version 0.3 de Docker Machine](https://github.com/docker/machine/releases/tag/v0.3.0).

Passons outre le message d’erreur et positionnons les variables d’environnements comme suit :
{% highlight sh %}
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH="/Users/ksahnine/.docker/machine/machines/orionbox"
export DOCKER_HOST=tcp://192.168.0.29:2376
{% endhighlight %}

ou plus simplement :
{% highlight sh %}
eval $(docker-machine env orionbox)
{% endhighlight %}
C’est terminé. Notre client Docker est configuré pour interagir avec l’hôte docker distant.

{% highlight sh %}
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
