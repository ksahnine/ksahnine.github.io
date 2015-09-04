---
layout: post
title:  "L'émergence des architectures orientées microservices"
date:   2015-01-12 06:00:00
categories: architecture microservice
tags: regular
image: /assets/article_images/Langenberg.jpg
legend: Un monolithe sur le massif des Vosges, Alsace
---
A n’en pas douter, l’année 2015 sera vraisemblablement l’année de la consécration de ce nouveau paradigme d’architecture aux dépens des architectures N-tiers, encore ultra dominantes dans les Systèmes d’Information d’entreprise.<br />
S’agit il d’un nouvel effet de mode stimulé aux fragrances du marketing, une sorte de *remake* des architectures SOA des années 2003/2004 ?<br />
On peut en effet rappeler qu’à cette époque, les architectures SOA étaient le nouveau mantra des DSI et ont fait la fortune des intégrateurs, dans le sillage des grands éditeurs de logiciels (*IBM*, *BEA*, *webMethods*/*Software AG*, *Tibco*, etc.).<br />
Les temps ont changé car **les éditeurs ne sont désormais plus des prescripteurs de solutions**, ce rôle étant désormais de plus en plus dévolu aux grands acteurs du Web.

L’échec relatif des architectures SOA ([*SOA is dead, long live services*](http://apsblog.burtongroup.com/2009/01/soa-is-dead-long-live-services.html)) ne devrait en rien tempérer l’adoption à venir des microservices car il y a au moins une différence de taille :

- très centrées sur les ESB/EAI, les architectures SOA se sont imposées principalement sous l’impulsion des éditeurs logiciels très motivés, il faut l’admettre, par la commercialisation de solutions logicielles intégrées (EAI, ESB, conteneurs de Web Services SOAP).
- les architectures orientées microservices sont une réalité chez de grands acteurs du Web (*Amazon*, *Netflix*, *Airbnb*, [*Gilt*](http://www.gilt.com/), [*Hailo*](https://www.hailoapp.com/fr/), etc.) car elles sont agiles, *cloud compatible* et particulièrement adaptées à un environnement économique mouvant et férocement concurrentiel. L’effervescence actuelle autour de ce modèle en est directement issu.

## Qu'est-ce qu'un microservice ?
Initialement [formalisée par Martin Fowler et James Lewis](http://martinfowler.com/articles/microservices.html), une architecture de microservices est pensée **à rebours d’une architecture monolithique**.<br />
Retenons que :

- la granuarité du service est celle d’une **fonctionnalité élémentaire**, au sens métier du terme, n’impliquant que très peu d’objets métier. Son interface doit être suffisamment simple et expressive pour quiconque souhaitant l’utiliser. Dans la mesure du possible, les services doivent être **faiblement couplés** entre eux.
- un service dispose de son *propre contexte d’exécution* (**un processus par service**), ce qui facilite incidemment sa mise à jour. Il n’y a pas d’a priori sur la pile technologique utilisée (langage, framework) ni même l’architecture applicative
- chaque service gère son **propre espace de stockage**, sans a priori technologique sur la solution de persistence
- il doit pouvoir être construit, testé et déployé de manière totalement **isolée** des autres services
- les services doivent être **inter-opérables**, quel que soit le langage de développement utilisé. On pourra privilégier une API REST/HTTP, bien qu’il n’y ait pas de consensus établi sur ce sujet. Par ailleurs, il n’est pas interdit d’utiliser des protocoles binaires comme [*Protocol Buffers*](https://code.google.com/p/protobuf/), [*Avro*](http://avro.apache.org/) ou encore [*Thrift*](https://thrift.apache.org/) si un besoin de performance l’exige
- un service peut **communiquer avec un ou plusieurs microservices** via une API publique.

## Pourquoi les microservices vont s'imposer
La conjonction de plusieurs facteurs explique l’adoption à venir dans les Systèmes d’Information d’entreprise.<br />
C’est d’une part une réalité chez de nombreux acteurs du Web et dans des domaines d’activité variés (commerce électronique, transport, divertissement).<br />
D’autre part, certaines technologies ou pratiques qui sous-tendent une architecture orientée microservice sont désormais éprouvées et très largement répandues ou en voie de l’être :

- l’approche **REST** et la généralisation du format d’échange de données **JSON**
- l’émergence de frameworks légers adaptés au développement rapide de microservices, comme [*Dropwizard*](http://dropwizard.io/) (auquel j’ai consacré [cet article](/microservice/dropwizard/java/2015/01/16/presentation-dropwizard.html)), [*Spring Boot*](http://projects.spring.io/spring-boot/) et [*Spark*](http://sparkjava.com/) (Java), [*Flask*](http://flask.pocoo.org/) (Python), [*Sinatra*](http://www.sinatrarb.com/) (Ruby) ou [*Vert.x*](http://vertx.io/) (polyglotte)
- la **montée en puissance** [**de technologies de containerisation**](https://linuxcontainers.org/), [Docker](https://www.docker.com/) étant bien parti pour devenir le standard de fait
- le *cloud computing* comme nouveau paradigme d’hébergement (lire à ce sujet mon article publié il y a plus de 5 ans en 2009)

Enfin, les défis de la mutation numérique des entreprises vont contraindre les DSI à *"fluidifier"* le Système d’Information afin de produire, livrer, déployer, faire évoluer les composants du SI de plus en plus rapidement, défis face auxquels les applications monolithiques ne sont pas adaptées.

### Les conditions d’une expérience réussie
Pour exploiter tout le potentiel d’une architecture orientée microservices, il est nécessaire :

- de **faire tomber les barrières** entre les différents acteurs d’un projet (développeurs, DBA, administrateurs système, équipe de recette, équipe d’exploitation, etc.), passer d’un mode d’organisation en silos vers une culture **DevOps**.
- d’automatiser les processus de construction/livraison/supervision dans tous les environnements (du développement à la production) à l’aide d’outils comme [*Jenkins*](http://jenkins-ci.org/), [*Puppet*](http://puppetlabs.com/), [*Chef*](https://www.chef.io/chef/) ou [*Ansible*](http://www.ansible.com/) auquel j’ai [consacré un article](/microservice/ansible/docker/2015/02/01/ansible-docker.html)
- d’**éviter la généralisation massive** de ce modèle d’architecture. Commencer par expérimenter avec deux ou trois services avec une forte valeur ajoutée fonctionnelle.

Par ailleurs, une technologie de [*containerisation*](https://linuxcontainers.org/) comme [Docker](https://www.docker.com/) fournit un environnement idéal pour ce type d’architecture car il répond parfaitement aux besoins:

- d’**isolation** de l’environnement d’exécution d’un service
- de déploiement **rapide** et sans couture
- de **portabilité** et de **scalabilité**
- de compatibilité avec des solutions d’hébergement dans un **Cloud** privé, public ou hybride

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
