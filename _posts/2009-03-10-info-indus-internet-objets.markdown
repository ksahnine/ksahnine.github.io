---
layout: post
title:  "L'informatique industrielle, laboratoire oublié de l'Internet des Objets ?"
date:   2009-03-10 06:00:00
categories: prospective
tags: regular
image: /assets/couverture/automate-indus.jpg
legend: Automates progammables industriels utilisé pour le monitoring en industrie pharmaceutique.
---
Médiatisé par Jacques Attali, en particulier dans son [rapport éponyme](http://lesrapports.ladocumentationfrancaise.fr/BRP/084000041/0000.pdf), l’**Internet des objets** est annoncé comme le prochain cycle d’innovations perturbatrices, c’est à dire susceptibles de remettre en cause les ordres technologiques et économiques établis. Le fameux [frigo intelligent connecté](http://www.orangecone.com/archives/2008/01/the_fridge_comp.html) à l’Internet constitue, depuis un moment déjà, le marronnier préféré des journalistes de la presse technologique grand public.

Pourtant, même si le rapport Attali met surtout l’accent sur l’utilisation de puces **RFID** comme moyen de connecter les objets du monde réel à l’Internet, nous pouvons considérer que l’Internet des objets est déjà à l’œuvre, en particulier dans le marché de la télésurveillance. Mais c’est surtout dans le monde de l’Informatique Industrielle que le champ d’application est le plus significatif. Dans cet univers d’exigence et de pragmatisme, on ne parle pas d’Internet des objets mais d’architecture [**M2M**](http://en.wikipedia.org/wiki/Machine_to_Machine), acronyme de *Machine to Machine*, dont la vocation est de permettre la collecte d’informations, voire l’asservissement d’équipements physiques, sans intervention humaine. En pratique, une architecture M2M utilise une infrastructure de type réseau cellulaire.

## Cinterion, un acteur représentatif de l'état de l'art

**Siemens** est un acteur majeur de l’informatique industrielle, connu et respecté notamment pour ses gammes d’[automates programmables industriels](http://www.automation.siemens.com/simatic/portal/index_77.htm).

Lobbyiste du modèle M2M, Siemens s’est positionné sur ce segment à travers son ancienne filiale, Siemens Wireless Module, devenue une entité autonome et indépendante du groupe depuis Juin 2008 ([Cinterion](http://www.cinterion.com/)).

L’attrait des solutions de Cinterion tient à la souplesse de la plateforme matérielle et logicielle.

L’unité matérielle est le **module**, un dispositif informatique complet embarquant un processeur, plusieurs ports d’entrée-sortie (série,  USB, etc.), un modem GSM/GPRS et surtout une machine virtuelle Java.

<center><img src="{{site.url}}/assets/article_images/tc65.jpg" style="display: block; margin: auto;" /></center>

Le module GPRS **TC65** fait figure de module étalon. Plus récent et sophistiqué, le module **XT75** est tri-bande (GSM/GPRS/EDGE) et incorpore une puce GPS, ce qui en fait un dispositif idéal pour développer un système de suivi de véhicules en temps réel.

<center><img src="{{site.url}}/assets/article_images/xt75_xt65.jpg" style="display: block; margin: auto;" /></center>

Cinterion commercialise également des terminaux prêts à l’emploi, comme le **TC65T**, un boîtier embarquant le module TC65, un adapteur secteur et un logement pour la carte SIM.

Le fait que les dispositifs matériels de Cinterion embarquent une machine virtuelle Java laissent imaginer tout le potentiel des applications embarquées. La configuration supportée est la CLDC 1.1. Une CLDC (*Configuration Limited Device Configuration*) définit les caractéristiques de la machine virtuelle embarquée. En version 1.1, elle se distingue de la plateforme Java SE par de très nombreuses simplifications (pas de gestion d’exceptions asynchrones, surcharge du class loader système interdit, une api très limitée (9 classes dans <code>java.lang.util</code> !), etc.).

En dépit de ces limitations, les champs d’applications sont considérables :

- Géolocalisation (suivi d’une flotte de véhicule en temps réel)
- Sécurité (localisation d’un véhicule volé par envoi de MMS)
- Télémétrie (consultation en temps réel du taux de remplissage d’une cuve de stockage)

## Perspectives

Nous pensons que le paradigme M2M est appelé à se généraliser sous l’effet combiné d’un effort de normalisation des plateformes d’exécution embarquées (en particulier celles centrées sur la plateforme Java ME) et de chute des coûts d’accès aux réseaux mobiles (GPRS, EDGE, etc.).<br />
Il existe en particulier un terreau favorable au développement de systèmes de géolocalisation en temps réel à très bas coût.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
