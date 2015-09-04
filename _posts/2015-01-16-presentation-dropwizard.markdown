---
layout: post
title:  "Présentation de Dropwizard"
titleColor: "yellow"
date:   2015-01-16 06:00:00
categories: microservice dropwizard java
tags: regular
image: /assets/article_images/micro-service-architecture.png
legend: Monolithique versus microservices
---
[**Dropwizard**](http://dropwizard.io/) est un framework Java léger **adapté au développement rapide de microservices REST** et ne nécessitant pas de serveur d’application comme environnement d’exécution.

Cela dit, au delà du framework, c’est surtout un assemblage habile de composants spécialisés parmi les meilleurs de l’écosystème Java :

- [**Jetty**](http://eclipse.org/jetty/), un serveur HTTP et un moteur de servlet compacts et très performants
- [**Jersey**](https://jersey.java.net/), l’implémentation de référence de la spécification JAX-RS (web services REST)
- [**Jackson**](https://github.com/FasterXML/jackson), une librairie de sérialisation/dé-sérialisation **JSON**
- [**Hibernate Validator**](http://hibernate.org/validator/), l’implémentation de référence de l’API *Bean Validation* (*JSR 303*)
- [**SLF4J**](http://www.slf4j.org/) et [**Logback**](http://www.slf4j.org/) pour la gestion des traces
- [**Metrics**](https://dropwizard.github.io/metrics/) pour le monitoring
- [**jDBI**](http://www.jdbi.org/) pour l’interfaçage rapide à une base de données relationnelle. Cette librairie est de bien plus bas niveau que JPA ou Hibernate et présente peu d’abstraction ce qui rend sa prise en main aisée

On peut considérer ce projet comme une **alternative crédible aux serveurs d’applications Java EE** perçus comme lourds, compliqués et gourmands en ressources. Le champ des applications couvert par **Dropwizard** est en réalité plus vaste que celui des microservices (il est tout a fait possible de développer une IHM web) mais l’aisance avec laquelle on développe et déploie un service REST en fait une solution très adaptée à ce type d’usage (à l’instar de [Spring Boot](http://projects.spring.io/spring-boot/) de [Pivotal](http://www.pivotal.io/) ou [Spark](http://sparkjava.com/) avec lesquels il est en compétition).

Packagée sous la forme d’un *jar* **autonome** contenant toutes ses dépendances, l’unité de déploiement n’a **pas besoin de serveur d’application** pour être exécutée (le conteneur *Jetty* est embarqué dans le *jar*). Avec ses 10 Mo tout au plus (dépendances comprises) l’empreinte mémoire d’une application **Dropwizard** est donc incomparablement plus faible qu’un Web Service SOAP déployé dans un serveur d’application Java EE (jusqu’à plusieurs centaines de Mo).<br />
Conséquemment, le temps de démarrage d’une application **Dropwizard** est de quelques secondes quand il faut parfois plusieurs minutes pour un serveur d’application.

## Struture type d'une application
Le schéma ci-dessous décrit les principaux éléments d’une application **Dropwizard** de type API RESTful :

<center>![Architecture d'une application Dropwizard](/assets/article_images/architecture-generique.png)</center>

Les composants mis en oeuvre sont les suivants :

- **Configuration** : le gestionnaire de configuration permet l’accès aux paramètres externalisés dans le fichier de configuration de l’application, représenté sur le schéma par le fichier `conf.yml` au format YAML
- **Ressource** : la ressource (au sens REST du terme) est l’implémentation d’un service JAX-RS (Jersey).
- **ExceptionMapper** : c’est une fontionnalité de Jersey. Ce composant (facultatif) permet de traiter une exception de manière globale en retournant, lorsqu’elle est levée, une réponse HTTP au format souhaité (JSON le plus souvent) et associée à un code HTTP correspondant à vos besoins.
- **HealthCheck** : facultatives mais fortement conseillées, ce sont dans ces classes que l’on implémente les tests de disponibilité de toutes les ressources nécessaires à la bonne exécution de l’application. Le "bilan de santé" de l’application est consultable par simple interrogation de la ressource `GET /healthcheck?pretty=true`.
- **Application** : c’est la classe `main` assemblant le tout (enregistrement des *ressources*, *exception mappers*, *configuration*, *health checkers*).

## L'écosystème Dropwizard
A ce jour, la dernière version stable est la **0.7.1**. Il est important de noter que cette version est liée à **Jersey 1.8** (JAX-RS 1.1) et n’est pas compatible avec Jersey 2.0 (JAX-RS 2.0). On ne pourra donc malheureusement pas tirer parti, entre autre, des [validateurs de méthode](https://jersey.java.net/documentation/latest/bean-validation.html#d0e9301) de resources REST.<br />
Néanmoins, il existe un [fork de Dropwizard intégrant Jersey 2.7](https://github.com/saadmufti/dropwizard/tree/jersey-2) mais il s’agit d’une branche expérimentale.

> **Note 08 Mars 2015** : Jersey 2 est désormais supporté depuis la [version 0.8](http://www.dropwizard.io/about/release-notes.html#v0-8-0-mar-5-2015) publiée le 05 Mars 2015.

Dropwizard est un framework **modulaire** et **extensible** dont l’étude du module central ([Dropwizard Core](http://dropwizard.io/manual/core.html)) a fait l’objet de cet article.<br />
Parmi les modules officiels intéressants, citons :

- [Dropwizard Views](http://dropwizard.io/manual/views.html) : un module de développement de vues HTML s’appuyant sur un moteur de template (les moteurs [**Freemarker**](http://freemarker.org/) et [**Mustache**](http://mustache.github.io/) sont supportés)
- [Dropwizard Migration](http://dropwizard.io/manual/migrations.html) : un wrapper [**Liquibase**](http://www.liquibase.org/) pour gérer les évolutions d’une base de données
- [Dropwizard Hibernate](http://dropwizard.io/manual/hibernate.html) : un wrapper Hibernate
- [Dropwizard Authentication](http://dropwizard.io/manual/auth.html) : le support des mécanismes d’authentification *HTTP Basic Authentication* et *OAuth*

Il existe également de [nombreuses contributions](http://modules.dropwizard.io/thirdparty/) officieuses. Citons entre autre :

- [Dropwizard LDAP](https://github.com/yammer/dropwizard-auth-ldap) : un module d’authentification LDAP
- [Dropwizard Redis](https://github.com/benjamin-bader/droptools/tree/master/dropwizard-redis) : un client de la [base NoSQL orientée clé/valeur Redis](http://blog.inovia-conseil.fr/?p=124)

Dans le [prochain billet de la série](/architecture/microservice/dropwizard/2015/01/19/developpement-microservice-dropwizard.html), nous verrons comment développer un service de consultation du trafic métro RATP en temps-réel avec Dropwizard.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
