---
layout: post
title:  "Le cloud computing, nouvel eldorado"
date:   2008-12-18 06:00:00
categories: architecture
tags: regular
image: /assets/couverture/cumulus.jpeg
legend: Cumulus, nuages de beau temps.
---
## Effet de mode ou tendance de fond ?
La dernière version du **cycle des tendances technologiques** publiée par le cabinet **Gartner** en Juillet 2008 annonce l’émergence du **cloud computing** dans le paysage informatique de demain.

<center><img src="https://blog.armstrong.space/wp-content/uploads/2008/08/gartner-hype-cycle2008.jpg" style="display: block; margin: auto;" /></center>

Les crises financières puis économiques pourraient stimuler la montée en puissance de ce nouveau paradigme d’infrastructure notamment grâce aux économies de coût d’exploitation qu’il promet.

Derrière ce concept se cachent plusieurs modèles. Aussi, il n’est pas étonnant que les grands acteurs de l’informatique, éditeurs et acteurs du Web, se positionnent sur ce marché naissant en privilégiant un modèle compatible avec leurs propres intérêts.

## Les modèles d'architecture
Il existe deux principaux modèles d’architecture de cloud computing, habituellement désignés sous les acronymes suivants :

- **IaaS** (*Infrastructure As A Service*)
- **PaaS** (*Platform As A Service*)

### Le modèle IaaS
Il s’agit du modèle le moins intrusif puisqu’il consiste à fournir un environnement d’exécution virtualisé - et bien souvent distribué sur une grille (grid computing) - indépendant de la technologie sous-jacente à l’application.

Ce modèle présente deux avantages significatifs :

- il garantit une certaine indépendance du client vis-à-vis d’un fournisseur *IaaS* (la réversibilité)
- son caractère peu intrusif permet le déploiement d’applications existantes avec un minimum d’ajustement

[**Elastic Compute Cloud**](http://aws.amazon.com/ec2/) (EC2), porté par **Amazon**, est l’offre qui incarne le mieux ce modèle. <br />
Initialement développée pour un usage interne, Amazon en a fait une offre commerciale à part entière, ce qui s’inscrit vraissemblablement dans une stratégie de diversification de ses activités.

### Le modèle PaaS
Il s’agit d’un modèle beaucoup plus intrusif puisque le fournisseur PaaS impose un cadre de développement et un environnement d’exécution aux applications distribuables sur son infrastructure, cadre de développement caractérisé par :

- un (ou plusieurs) langage(s) de développement
- une API spécifiquement conçue par le fournisseur

En outre, le risque pour un client d’être prisonnier de son fournisseur est réel, car le coût de sortie d’une infrastructure PaaS peut l’en dissuader.<br />
[**Azure Services Platform**](http://www.microsoft.com/azure/default.mspx), l’offre récemment dévoilée par **Microsoft**, incarne assez bien le modèle, finalement tout à fait compatible avec les intérêts stratégiques de l’éditeur de Redmond. A ce titre, il est intéressant de constater que [**Richard Stallman**](http://fr.wikipedia.org/wiki/Richard_Stallman), figure historique du logiciel libre et père de la licence GPL, est très hostile au cloud computing, mais pour des raisons qui tiennent plus à la propriété des données personnelles qu’à des considérations technologiques.

Autre géant à avoir opté pour ce modèle : **Google**, avec son offre [**Google App Engine**](http://code.google.com/appengine/) dont le seul langage de développement supporté à ce jour est Python. Le support prochain de Java a toutefois été annoncé.<br />
Les intérêts stratégiques de Google pourraient être guidés par une volonté de diversification des sources de revenus. En effet, la firme est encore largement dépendante du marché publicitaire, un marché traditionnellement exposé aux situations de crises économiques.<br />
Par ailleurs, sa rivalité avec Microsoft le pousse à aller très loin dans la promotion du modèle [**SaaS**](http://fr.wikipedia.org/wiki/SaaS) (*Software As A Service*) à bas coût voire gratuit, notamment dans le champ des applications bureautiques, et poursuivre ainsi une logique de dumping pouvant asphyxier Microsoft. L’infrastructure Google App Engine, dont la vocation est d’héberger des applications SaaS, en serait le corollaire indispensable.

## Quels modèles pour quelles entreprises ?
Les providers *PaaS* pourraient intéresser les structures agiles, PME ou startup, pour lesquelles l’Internet est un canal de distribution majeur. Le modèle PaaS est en effet une solution d’hébergement, appropriée pour les applications web, souple, peu chère avec une garantie de haute disponibilité.

Les grandes entreprises, dont le patrimoine applicatif est important, trouverons dans le modèle IaaS une solution agile, économique et peu intrusive, avec la garantie d’une disponibilité à la demande. <br />
Toutefois, les questions très sensibles portant sur la sécurité et la confidentialité des données réparties dans une infrastructure partagée par nature, constituent des freins à l’adoption. Mais dans un contexte de crise économique, l’arbitrage par les coûts plutôt que par le veto sécuritaire, pourrait faire tomber les dernières résistances.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
