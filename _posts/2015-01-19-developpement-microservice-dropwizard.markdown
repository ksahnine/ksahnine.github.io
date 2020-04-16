---
layout: post
title:  "Le développement de microservices avec Dropwizard"
date:   2015-01-19 06:00:00
categories: architecture microservice dropwizard
tags: regular
image: /assets/article_images/Drop-Wizard-Game.jpg
legend: Teo, le héro magicien du jeu d'arcade Drop Wizard
---
Après avoir présenté **Dropwizard** dans le précédent billet de la série, dont la lecture est un préalable, place aux travaux pratiques.

Cet article est consacré au développement d’un service de consultation temps-réel des horaires du métro parisien avec **Dropwizard**.

L’ensemble du projet est [disponible sur GitHub](https://github.com/ksahnine/trafic-ratp-dropwizard).

L’interface *RESTful* du service est la suivante :<br />
<u>*Requête* :</u>
<pre>
GET /trafic-ratp/metro/**:numero_ligne**/**:nom_station**/**:sens**
</pre>
où :

- `numero_ligne` est le numéro de la ligne du métro
- `nom_station` est le nom de la station de métro pour laquelle on souhaite connaître les prochains passages
- `sens` a 2 valeurs possibles : `A` pour *Aller* ou `R` pour *Retour*

A titre d’exemple, la requête `GET /trafic-ratp/metro/14/pyramides/A` restitue la liste des prochains passages de rames de métro de la station *Pyramides*, sur la ligne *14* en direction de *Saint-Lazare*.

<u>*Réponse* :</u><br />
Le format de la réponse, au format **JSON**, contient une liste des prochains passages d’une rame dont voici un exemple :
{% highlight json %}
[
  {
    "direction": "Saint-Lazare",
    "attente": "1 mn"
  },
  {
    "direction": "Saint-Lazare",
    "attente": "4 mn"
  },
  {
    "direction": "Saint-Lazare",
    "attente": "7 mn"
  }
]
{% endhighlight %}

## Structure de l'application
### Architecture
Les différents composants constituant l’application sont représentés dans le schéma ci-dessous :

<center><img src="{{site.url}}/assets/article_images/architecture-app.png" style="display: block; margin: auto;" /></center>

###Configuration de l’application
Le fichier de configuration de l’application `trafic-ratp.yml` est décrit au format **YAML**. On externalise en particulier le paramètre applicatif `urlRatp`, l’URL du portail de la RATP.
{% gist a96a3f5a44e39b9c34c7 %}

Le fichier de configuration pourrait être réduit au minimum. Il est évidemment possible de surcharger la [centaine de paramètres](https://dropwizard.github.io/dropwizard/manual/configuration.html) par défaut, dont le port d’écoute du conteneur *Jetty* par exemple (port `8080` par défaut).

La classe `TraficRatpConfiguration` est le gestionnaire de configuration de l’application. C’est par ce biais que l’on accède aux paramètres applicatifs.
{% gist 1b478e5ccf87453fc8dd %}

### Objet métier `Rame`
L’objet métier `Rame.java` modélise l’état du passage d’une rame de métro. Il possède deux propriétés : la direction et la durée d’attente.
{% gist 372c92038462887b47e4 %}

### Ressource `TraficRatpResource`
La ressource (au sens REST du terme) implémente le point de terminaison du service JAX-RS. La récupération des informations sur le trafic consiste à interroger le portail [ratp.fr](http://www.ratp.fr/) puis à extraire les informations utiles en utilisant la technique du [*web scraping*](http://fr.wikipedia.org/wiki/Web_scraping). La librairie [**jsoup**](http://jsoup.org/) est utilisée à cet effet.
{% gist 1f9f2c0839a46d19026b %}

On récupère les 3 paramètres `ligne`, `station` et `sens` dans le nom de la ressource.
{% highlight java %}
@Path("/metro/{ligne: [0-9]{1,2}[b]{0,1}}/{station}/{sens: [A|R]}")
{% endhighlight %}
Noter que l’on a choisit de poser des contraintes en utilisant des expressions régulières au niveau de l’annotation `@Path`. Le paramètre `sens` par exemple n’a que 2 valeurs possibles : `A` ou `R`.

Si l’expression régulière est fausse, le code statut `HTTP 404` est retourné, ce qui est logique puisqu’on considère que la ressource n’existerait pas dans ce cas.

Malheureusement, la validation des paramètres de méthode par annotation (`@Valid`) ne fonctionne pas avec *Dropwizard* 0.7.x (qui intègre *Jersey* 1.x). Il faudra attendre une version compatible avec *Jersey* 2.x).

### Exception mapper `IOExceptionMapper`
Cette classe permet de traiter l’exception `IOException`, lorsqu’elle est levée pendant l’interrogation du portail [ratp.fr](http://www.ratp.fr/), sous la forme d’une réponse `HTTP 500` avec une charge utile au format **JSON** contenant le message d’erreur technique.
{% gist 28a8a7a423ca8990365d %}

### Health checker `RatpHealthChecker`
Le test de disponibilité d’une ressource vitale de l’application consiste à s’assurer de la disponibilité du portail RATP (le test, basique, n’a qu’une valeur pédagogique).
{% gist b36f6a56c78f162d0113 %}

Le *"bilan de santé"* de l’application est consultable par simple interrogation de la ressource `GET /healthcheck?pretty=true` dont voici la réponse lorsque le système est opérationnel :
{% highlight json %}
{
  "Dispo site RATP" : {
    "healthy" : true
  },
  "deadlocks" : {
    "healthy" : true
  }
}
{% endhighlight %}

### Classe application `TraficRatpApplication`
Il s’agit de la classe `main` assemblant le tout, responsable du démarrage de l’application (enregistrement des *ressources*, *exception mappers*, *configuration*, *health checkers*).
{% gist c380ea1236e8b89db73e %}

## Construction et exécution
Récupérer le [code source du projet](https://github.com/ksahnine/trafic-ratp-dropwizard) sur GitHub :
{% highlight sh %}
git clone https://github.com/ksahnine/trafic-ratp-dropwizard.git
{% endhighlight %}
puis construire le projet *Maven* (fabrique l’unité de déploiement `trafic-ratp-1.0.0-SNAPSHOT.jar`) :
{% highlight sh %}
cd trafic-ratp-dropwizard
mvn package
{% endhighlight %}
Pour démarrer le service, exécuter la commande :
{% highlight sh %}
java -jar target/trafic-ratp-1.0.0-SNAPSHOT.jar server trafic-ratp.yml
{% endhighlight %}
On utilisera [**cURL**](http://curl.haxx.se/) en association avec l’utilitaire [`jq`](http://stedolan.github.io/jq/) pour requêter le service et formater la sortie standard.

- prochains passages du métro ligne *14* / Station *Pyramides* / Direction *Saint-Lazare* :
{% highlight sh %}
curl -s http://localhost:8080/trafic-ratp/metro/14/pyramides/A | jq "."
{% endhighlight %}
- prochains passages du métro ligne *14* / Station *Pyramides* / Direction *Olympiades* :
{% highlight sh %}
curl -s http://localhost:8080/trafic-ratp/metro/14/pyramides/R | jq "."
{% endhighlight %}

Le [prochain billet](/microservice/docker/dropwizard/2015/01/21/dockeriser-microservice.html) de la série sera consacré au déploiement d’un microservice Dropwizard via [**Docker**](https://www.docker.com/).

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
