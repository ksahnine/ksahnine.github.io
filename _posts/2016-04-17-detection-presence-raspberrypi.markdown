---
layout: post
title:  "La détection de présence avec un Raspberry Pi 3 et Node-RED"
date:   2016-04-17 06:00:00
categories: iot raspberrypi
tags: regular
image: /assets/couverture/framboises.jpg
legend: Framboises
---

Comme de nombreux fans, je suis un heureux possesseur d’un [**Raspberry Pi 3**](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/), le dernier né de la fondation Raspberry Pi, intégrant en standard les technologies sans fil _WiFi 802.11 b/g/n_ et _Bluetooth Low Energy 4.1_.
Ce concentré de technologies constitue une plateforme idéale pour une centrale domotique bon marché et un objet d’étude fort intéressant pour quiconque s’intéresse au monde de [**l’Internet des Objets**](https://fr.wikipedia.org/wiki/Internet_des_objets) (_IoT_).

Depuis longtemps déjà, j’avais le projet de réécrire le code de ma solution domotique “maison” avec [**Node-RED**](https://nodered.org/), un fantastique outil permettant d’élaborer des scénarios d’intégration complexes combinant des objets physiques (capteurs, micro-contrôleurs, transceivers, etc.) et des services en ligne.

Dans cet article, j’exposerai une **solution de détection de présence (et d’absence)** via **Bluetooth** fonctionnant aussi bien avec un smartphone ou un tracker Bluetooth bon marché glissé dans un cartable par exemple.
Les cas d’utilisation sont très nombreux. Citons par exemple :

- la réception d’un itinéraire conseillé en transports en commun en sortant de chez soi
- l’envoi d’une notification lorsque ses enfants sont bien rentrés à la maison
- le verrouillage/déverrouillage d’une alarme
- le pilotage d’une prise radio-commandée

**Node-RED** a vu le jour sous les hospices d’_IBM Emerging Technologies Group_ à des fins de prototypage rapide de solutions IoT. Le code source a été publié en licence libre fin 2013.
Il offre un modèle de développement normalisé, orienté message, et une interface graphique permettant de décrire un flux de traitements.

Il existe une [extension Node-RED](https://flows.nodered.org/node/node-red-contrib-noble) fournissant un support Bluetooth (basée sur le module _nodeJS_ [**noble**](https://github.com/sandeepmistry/noble)) mais je n’ai pas trouvé le résultat satisfaisant, l’extension étant beaucoup trop expérimentale.

J’ai opté pour le développement en **Python** d’un service de découverte de périphériques Bluetooth, publiant vers Node-RED les événements d’absence ou de présence dans un **socket TCP** (Node-RED propose en standard un module d’écoute TCP). 
Cette solution fonctionne très bien.
Le code source est disponible sur [mon dépôt Github](https://github.com/ksahnine/rpi-home-automation).

## Mise en oeuvre

Node-RED est disponible par défaut sur les dernières versions de Raspbian Jessie ou installable via le gestionnaire de paquets APT :

{% highlight sh %}
sudo apt-get update
sudo apt-get install nodered
{% endhighlight %}

Procéder ensuite aux étapes suivantes :

- installer les dépendances :
{% highlight sh %}
sudo apt-get install bluetooth python-yaml python-bluetooth
{% endhighlight %}
- recopier la configuration et les flux Node-RED mis en oeuvre dans le cadre de l’article :
{% highlight sh %}
wget -P ~/.node-red https://github.com/ksahnine/rpi-home-automation/blob/master/.node-red/settings.js
wget -P ~/.node-red https://github.com/ksahnine/rpi-home-automation/blob/master/.node-red/flows.json
{% endhighlight %}

> **Attention** : les 2 commandes ci-dessus remplacent votre configuration et vos flux Node-RED ! Faites l’opération manuellement si vous partez d’une configuration existante.

- installer l’application de détection de périphériques :
{% highlight sh %}
git clone https://github.com/ksahnine/rpi-home-automation.git
{% endhighlight %}

- configurer le fichier `rpi-home-automation/ble2tcp/devices.yml`
- démarrer Node-RED puis le programme de détection de périphériques Bluetooth :
{% highlight sh %}
node-red
python rpi-home-automation/ble2tcp/ble2node-red.py -c rpi-home-automation/ble2tcp/devices.yml
{% endhighlight %}

L’interface Node-RED est accessible par défaut sur le port `1880` du Raspberry Pi (ex: `http://rumah.local:1880/`)

## Comment ça marche ?

Les périphériques à surveiller sont définis dans un fichiers de configuration (`devices.yml`) au format YAML :

{% gist 4f2de2bff20d01b10b209cc285348d90 %}

Un scan est effectué périodiquement toutes les 4 secondes.

> **Note** : l’adresse Bluetooth d’un périphérique peut être récupérée via la commande `hcitool scan` sous Debian après avoir installé le paquet `bluetooth` (`sudo apt-get install bluetooth`).

Lorsque l’état d’un périphérique change (présence ou absence), un évènement est publié au format JSON dans un socket TCP :

{% gist fa3e7bfd818176f8a78a591e423b3a78 %}

L’événement est ensuite lu en entrée du flux de traitement Node-RED :

<center><img src="{{site.url}}/assets/article_images/node-red-flow.png" style="display: block; margin: auto;" /></center>

Il est constitué de l’enchaînement de noeuds suivants :

- noeud _BLE to Node TCP scanner_ : implémente un listener TCP écoutant les événements de présence ou d’absence des périphériques surveillés.
<center><img src="{{site.url}}/assets/article_images/node-red-tcp.png" style="display: block; margin: auto;" /></center>
- noeud _json_ : transforme l’événement en objet JSON
- noeud _Présence ?_ : switch orientant le traitement en fonction de la présence ou de l’absence d’un périphérique
<center><img src="{{site.url}}/assets/article_images/node-red-switch.png" style="display: block; margin: auto;" /></center>
- noeud _Arrivé_ : traite l’événement émis lorsqu’un périphérique est proche du foyer
<center><img src="{{site.url}}/assets/article_images/node-red-debug-node.png" style="display: block; margin: auto;" /></center>
- noeud _Parti_ : traite l’événement émis lorsqu’un périphérique quitte le foyer

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
