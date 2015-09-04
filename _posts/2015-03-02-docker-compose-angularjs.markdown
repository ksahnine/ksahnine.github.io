---
layout: post
title:  "De l'intêret de Docker Compose (ex Fig) pour le développeur AngularJS"
date:   2015-03-02 06:00:00
categories: docker angularjs
tags: regular
image: /assets/article_images/figues.jpg
legend: Etal de figues sur un marché de Géorgie, Caucase
---
Il y a plus de 5 ans (en janvier 2010), j’avais écrit un petit billet intitulé [*L’inexorable migration du tiers Présentation*](http://blog.inovia-conseil.fr/?p=91) pour marquer l’émergence des **frameworks MV\* côté client** et ce, à une époque où **JSF** constituait encore un horizon indépassable au sein de nombreuses maîtrises d’oeuvre.<br />
Fort heureusement, les temps ont changé en particulier depuis la déferlante [**AngularJS**](https://angularjs.org/), à telle enseigne que ce type de solution est désormais sur le point de devenir *mainstream*.

Dans la caisse à outil du développeur **AngularJS** (entre autre), on trouvera naturellement le gestionnaire de paquets [**npm**](https://www.npmjs.com/), le gestionnaire de dépendances [**bower**](http://bower.io/) et l’outil d’automatisation [**grunt**](http://gruntjs.com/) (ou [**gulp**](http://gulpjs.com/)), mais on verra surtout dans cet article combien [**Docker Compose**](http://docs.docker.com/compose/) (anciennement [**Fig**](http://www.fig.sh/)) a sa place dans le dispositif, en particulier pour construire un environnement d’exécution calibré pour un projet et de le rendre **opérationnel sur un poste de développement en une seule commande** : `docker-compose up`.

En effet, dans un projet informatique impliquant plusieurs développeurs, la mise au point d’un environnement de développement s’avère être une tâche ardue et chronophage.<br />
Nous allons voir à travers l’étude d’un cas pratique comment **Docker Compose** nous permettra de reconstituer un environnement d’exécution complet et autonome sur le poste du développeur par assemblage d’applications dockerisées.

## Etude de cas
Dans le sillage de ma [série d’articles](/architecture/microservice/2015/01/12/emergence-microservices.html) traitant des architectures de microservices, en particulier celui consacré au développement d’une API RESTful avec **Dropwizard**, j’ai mis à disposition une application *AngularJS* permettant de consulter le trafic du métro parisien et dont le code source est [disponible sur Github](https://github.com/ksahnine/ratp-gui-angularjs) :

<center>![IHM AngularJS](http://blog.inovia-conseil.fr/wp-content/uploads/2015/03/metro-gui.png)</center>

Cette application servira de support à l’article.

Le schéma ci-dessous décrit l’architecture type de l’environnement d’exécution du poste de développement que nous allons mettre au point avec **Docker Compose** :

<center>![Architecture](http://blog.inovia-conseil.fr/wp-content/uploads/2015/03/archi-poste-dev.png)</center>

Elle est constituée :

- d’un frontend **AngularJS** dont on a fait mention ci-dessus
- d’un backend de services REST que j’ai développé avec **Dropwizard**, et dont l’image Docker est [disponible dans mon repository](https://registry.hub.docker.com/u/ksahnine/ratp-rest-api/) public Docker Hub
- d’une base NoSQL orientée document (**CouchDB**)
- d’un serveur web **nginx** servant :
  - les ressources statiques de l’espace de travail du projet AngularJS (code JavaScript, template HTML, code CSS etc.)
  - les ressources dynamiques (JSON) exposées par le backend **Dropwizard** et accessibles par convention via le préfixe `/api`

## Docker Compose, un orchestrateur de conteneurs Docker
Développé par [Orchard](https://www.orchardup.com/), une start-up de 2 personnes (!) rachetée par Docker en Juillet 2014, **Fig** est devenu la solution standard d’orchestration de conteneurs Docker.
**Docker Compose** est le résultat de l’intégration de **Fig** dans l’écosystème Docker.

### Installation
*Docker Compose* est disponible sous la forme d’un module Python installable via l’outil de gestion de paquets PIP.<br />
Le mode opératoire d’installation est des plus simples :
{% highlight sh %}
sudo pip install -U docker-compose
{% endhighlight %}

### Utilisation
La composition de notre infrastructure est décrite dans le fichier `docker-compose.yml` au format YAML et dont voici le contenu :
{% gist 2cfd5a5b4b02265d4be9 %}

Nous retrouvons les 3 composants dockerisés de notre architecture :

- `web` : le conteneur nginx construit à partir de l’image `nginx:1.7.9`.
<br />On notera en particulier l’usage du mapping des volumes pour :
  - redéfinir la racine du conteneur web afin de servir des ressources du système de fichier local issu de la construction du projet via **grunt** : `~/dev/ratp-gui-angularjs/dist`
  - utiliser un fichier de configuration *nginx* calibré pour le projet et disponible localement : `~/dev/ratp-gui-angularjs/conf/nginx-dev.conf`
- `ws` : le backend de service REST construit à partir de l’image `ksahnine/ratp-rest-api:1.0`
- `db` : la base *NoSQL* construite à partir de l’image `fedora/couchdb`

Récupérons le code source du projet **AngularJS** contenant le fichier `docker-compose.yml` à la racine :
{% highlight sh %}
git clone https://github.com/ksahnine/ratp-gui-angularjs.git
cd ratp-gui-angularjs
{% endhighlight %}

Pour construire le projet :

- installer préalablement les dépendances :
{% highlight sh %}
npm install
bower install
{% endhighlight %}
- construire l’application AngularJS via **grunt** :
{% highlight sh %}
grunt all
{% endhighlight %}

Pour créer et démarrer les conteneurs, il suffit de lancer la commande :
{% highlight sh %}
docker-compose up
{% endhighlight %}

Pour créer et démarrer les conteneurs en tâche de fond, rajouter l’option `-d` :
{% highlight sh %}
docker-compose up -d
{% endhighlight %}

Pour afficher la sortie standard des conteneurs, utiliser la commande :
{% highlight sh %}
docker-compose logs
{% endhighlight %}

Pour arrêter les conteneurs, utiliser la commande :
{% highlight sh %}
docker-compose stop
{% endhighlight %}

Pour supprimer les conteneurs, utiliser la commande :
{% highlight sh %}
docker-compose rm
{% endhighlight %}

## A l'attention des utilisateurs de Fig
**Fig** n’est plus maintenu depuis Décembre 2014, date de son remplacement officiel par **Docker Compose**.<br />
Néanmoins au cas où vous l’utiliseriez, voici quelques-uns de mes retours d’expérience :

- sous Mac OS X, j’ai rencontré l’erreur suivante à l’exécution de la commande `fig up` :
{% highlight sh %}
SSL error: hostname ‘192.168.59.103′ doesn’t match ‘boot2docker’
{% endhighlight %}
Le certificat est signé vis-à-vis du nom d’hôte (`boot2docker`) et non de son adresse IP.
<br />On peut contourner le problème :

> -  en rajoutant une entrée dans le fichier `/etc/hosts` :
> {% highlight text %}
192.168.59.103 boot2docker
{% endhighlight %}
> - en modifiant et exportant la variable `DOCKER_HOST` comme suit :
> {% highlight sh %}
export DOCKER_HOST=tcp://boot2docker:2376
{% endhighlight %}
- le fichier de composition `fig.yml` est encore reconnu par Docker Compose (v 1.1.0) mais ce ne sera bientôt plus le cas.
- enfin, **Fig** contient un bug non résolu à ce jour (version 1.0.1 associée à Docker 1.4.0). La modification du paramétrage des volumes dans le fichier `fig.yml` après un `fig up n’est jamais prise en compte (les volumes ne sont pas re-montés).<br />On peut contourner le problème en forçant préalablement la suppression des conteneurs à l’arrêt via la commande :
{% highlight sh %}
fig rm –force
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
