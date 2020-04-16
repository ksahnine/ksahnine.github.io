---
layout: post
title:  "Dockeriser un microservice (Dropwizard + Docker)"
date:   2015-01-21 06:00:00
categories: microservice docker dropwizard
tags: regular
image: /assets/article_images/dsc01010.jpg
legend: L'intérieur d'un conteneur maritime
---
L’association de **Dropwizard** et **Docker** constitue une solution fort intéressante pour une architecture orientée microservices.<br />
En effet, elle répond remarquablement aux besoins :

- d’**isolation** de l’environnement d’exécution d’un service
- de déploiement **rapide** et sans couture
- de **portabilité** et de **scalabilité**
- de compatibilité avec des solutions d’hébergement dans un **Cloud** privé, public ou hybride

Accessoirement, une application *Dropwizard* est packagée sous la forme d’un fichier *jar* **unique** et **autonome** ce qui rendra d’autant plus simple la fabrication d’une image Docker comme on le verra dans cet article.

En pratique, nous verrons comment :

- **dockeriser** le service de consultation des horaires du métro parisien, une application *Dropwizard* qui a fait l’objet du [précédent billet de la série](/architecture/microservice/dropwizard/2015/01/19/developpement-microservice-dropwizard.html) (sa lecture préalable est recommandée) et dont le [code source est disponible](https://github.com/ksahnine/trafic-ratp-dropwizard) sur GitHub
- **créer des environnements d’exécution cloisonnés** en mappant un volume de données du conteneur sur le système de fichier local
- **publier** l’image dans un dépôt privé (situé dans un réseau local) mais également dans le dépôt en ligne *Docker Hub*

## Construction de l'image Docker
J’ai choisi une recette simple en construisant séparément :

- l’application *Dropwizard* matérialisée par le fichier *jar* `trafic-ratp-1.0.0-SNAPSHOT.jar`
- une image Docker dérivée de l’image publique d’OpenJDK 7 (`FROM dockerfile/java:openjdk-7-jdk`) à laquelle on intègre le *jar* de l’application précédemment fabriqué ainsi que son fichier de configuration (`trafic-ratp.yml)

Le fichier `Dockerfile` résultant est le suivant :
{% gist 9f0f24f1d3073d2ceff6 %}

Quelques observations :

- le premier paramètre de l’instruction `ADD` correspond au fichier source à copier dans le conteneur (`target/trafic-ratp-1.0.0-SNAPSHOT.jar`). <br />Le chemin est relatif au répertoire contenant le fichier `Dockerfile` (répertoire racine). Il n’est pas possible d’ajouter un fichier situé à l’extérieur du répertoire racine ou de ses sous-répertoires (par ex. `../dist/20150204/trafic-ratp-1.0.0-SNAPSHOT.jar`)
- le second paramètre de l’instruction `ADD` correspond au fichier de destination dans le système de fichiers du conteneur (`/app/trafic-ratp-1.0.0-SNAPSHOT.jar`)
- le service est exposé par le conteneur sur le port `8080` (instruction `EXPOSE`)

La séquence complète de construction de l’image en repartant du code source sur GitHub se résume aux commandes suivantes :
{% highlight sh %}
git clone https://github.com/ksahnine/trafic-ratp-dropwizard.git
cd trafic-ratp-dropwizard
mvn package
docker build -t ksahnine:trafic-ratp-dropwizard .
{% endhighlight %}

Si l’on ne dispose pas d’environnement de développement, on pourrait tout à fait *dockeriser* l’étape de construction du microservice :

- en partant de l’image de base d’une *Debian* (`FROM debian:wheezy`)
- sur laquelle on installerait *OpenJDK* (`RUN apt-get install -y openjdk-7-jdk`), *git* (`RUN apt-get install -y git`) et *Maven* (`RUN apt-get install -y maven`)
- suivie de la construction de l’application (récupération du code source et compilation)

## Démarrage du conteneur
Il n’existe pas d’hôte Docker natif pour les systèmes d’exploitation autres que Linux. Sous Windows ou Mac OS X, vous devez installer la VM ultra légère [Boot2docker](http://boot2docker.io/), basée sur *Virtual Box* et la distribution [*Tiny Core Linux*](http://tinycorelinux.net/).

<center><img src="{{site.url}}/assets/article_images/b2d.png" style="display: block; margin: auto;" /></center>

Dans cette situation, les conteneurs Docker ne s’exécutent pas directement au dessus de l’OS mais **au sein de la machine virtuelle** *Boot2docker*.

La commande `docker run` permet de lancer un conteneur à partir de l’image précédemment construite :
{% highlight sh %}
docker run -td -p 8080:8080 ksahnine:trafic-ratp-dropwizard
{% endhighlight %}

L’application embarquée dans le conteneur ne sera accessible qu’à travers l’adresse IP et le port **de la VM** (et non `localhost`).<br />
La commande `boot2docker ip` permet de connaître l’adresse IP de la VM :
{% highlight sh %}
ksahnine@inoviabook:~$ boot2docker ip
192.168.59.103
{% endhighlight %}

Le service *dockerisé* est donc interrogeable via **cURL** comme suit :
{% highlight sh %}
curl -s http://192.168.59.103:8080/trafic-ratp/metro/14/pyramides/A
{% endhighlight %}

## Créer des environnements cloisonnés en mappant les volumes
Pour créer 2 environnements d’exécution distincts (appelons les *dev* et *recette*) à partir de la même image, il suffit de mapper le volume `/conf` du conteneur vers le répertoire du système de fichier local contenant le fichier de configuration idoine :

- configuration de *dev* : `~/conf/dev/trafic-ratp-dropwizard.yml`
- configuration de *recette* : `~/conf/rec/trafic-ratp-dropwizard.yml`

Le fichier `Dockerfile` initial a été très légèrement modifié de sorte à ne pas ajouter le fichier de configuration dans l’image :
{% gist 43478d2d052622554361 %}

Ainsi, le démarrage des instances de *dev* et *recette* (ports `9010` et `9020`) s’effectue avec les commandes `docker run` suivantes :

- *dev* : `docker run -t -p 9010:8080 -v ~/conf/dev:/conf --name svc-metro-dev ksahnine:trafic-ratp-dropwizard`
- *recette* : `docker run -t -p 9020:8080 -v ~/conf/rec:/conf --name svc-metro-rec ksahnine:trafic-ratp-dropwizard`

L’option `-v rep_local:rep_conteneur` rend accessible le contenu d’un répertoire local (`rep_local`) depuis le conteneur (`rep_conteneur`).<br />
Si vous travaillez sous Mac OS X, le point de montage du conteneur accède en réalité à la VM *boot2docker* et non au système de fichier du Mac.<br />
[Depuis la version 1.3 de Docker](https://blog.docker.com/2014/10/docker-1-3-signed-images-process-injection-security-options-mac-shared-directories/), le montage de volume s’effectue sans couture mais est **limité uniquement aux sous-répertoires contenus dans le répertoire** `/Users`.<br />
Pour les versions antérieures de Docker il est nécessaire d’installer une version modifiée de *boot2docker* (lire cet [excellent article](https://medium.com/boot2docker-lightweight-linux-for-docker/boot2docker-together-with-virtualbox-guest-additions-da1e3ab2465c)).

## Distribuer une image Docker
### Publier dans un index privé d’images
Il suffit d’une machine sur le réseau local (`192.168.0.29` dans l’exemple ci-après) sur laquelle est installée Docker, puis d’installer et démarrer le registry en exécutant la commande suivante :
{% highlight sh %}
docker run -d -p 5000:5000 registry
{% endhighlight %}
La commande `docker push` permet d’ajouter l’image de notre microservice dockerisé :
{% highlight sh %}
docker tag ksahnine:trafic-ratp-dropwizard 192.168.0.29:5000/ksahnine:trafic-ratp-dropwizard
docker push 192.168.0.29:5000/ksahnine:trafic-ratp-dropwizard
{% endhighlight %}
L’accès à un index non sécurisé (HTTP) doit être explicitement autorisé au démarrage du démon docker (option `--insecure-registry`), sous peine de rencontrer l’erreur suivante :
{% highlight text %}
Error response from daemon: Invalid registry endpoint https://192.168.0.29:5000/v1/[...]
{% endhighlight %}
Avec `boot2docker`, l’option peut être passée via la variable `EXTRA_ARGS` :

- accéder à la VM en mode *shell* :
{% highlight sh %}
boot2docker ssh
{% endhighlight %}
- puis renseigner la variable `EXTRA_ARGS` dans le fichier `/var/lib/boot2docker/profile` :
{% highlight sh %}
EXTRA_ARGS="–insecure-registry 192.168.0.29:5000"
{% endhighlight %}
- enfin sortir du *shell* :
{% highlight sh %}
exit
{% endhighlight %}

A noter que le client `boot2docker` sera à relativement court terme remplacé par la commande `docker machine` toujours en [cours de développement](https://github.com/docker/machine) à ce jour.

Une simple interrogation de l’index via **cURL** confirme que l’image a bien été publiée :
{% highlight sh %}
ksahnine@inoviabook:~$ curl -s http://192.168.0.29:5000/v1/repositories/ksahnine/tags | jq "."
{
  "trafic-ratp-dropwizard": "c1454f0f800331644e43971e56c1e391bf06625a13600fc52160fc1c927e6fa4"
}
{% endhighlight %}
Elle peut ensuite être récupérée via la commande `docker pull` :
{% highlight sh %}
docker pull 192.168.0.29:5000/ksahnine:trafic-ratp-dropwizard
{% endhighlight %}
et enfin exécutée :
{% highlight sh %}
docker run -td -p 8080:8080 192.168.0.29:5000/ksahnine:trafic-ratp-dropwizard
{% endhighlight %}

### Publier sur Docker Hub
[Docker Hub](https://hub.docker.com/account/signup/) est le dépôt central des repositories hébergés chez Docker. L’hébergement d’index publics est gratuit tout comme le premier index privé ([payant au delà](https://registry.hub.docker.com/plans/)).<br />
Mon repository public [`ksahnine/ratp-rest-api`](https://registry.hub.docker.com/u/ksahnine/ratp-rest-api/) contient l’image de notre service dockerisé.<br />
La publication de l’image dans le repository s’effectue comme suit :
{% highlight sh %}
docker tag ksahnine:trafic-ratp-dropwizard ksahnine/ratp-rest-api:1.0
docker push ksahnine/ratp-rest-api:1.0
{% endhighlight %}
La récupération de l’image suivie de la création du conteneur et du démarrage du service depuis le hub s’effectue par cette simple commande :
{% highlight sh %}
docker run -td -p 8080:8080 ksahnine/ratp-rest-api:1.0
{% endhighlight %}

### Sauvegarder et restaurer une image
Plus rustique, la commande `docker save` permet d’exporter une image dans un fichier de *dump* :
{% highlight sh %}
docker save ksahnine:trafic-ratp-dropwizard > trafic-ratp-img.tar
{% endhighlight %}
Réciproquement, la commande `docker load` permet de restaurer une image à partir d’un *dump* :
{% highlight sh %}
docker load ksahnine:trafic-ratp-dropwizard < trafic-ratp-img.tar
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
