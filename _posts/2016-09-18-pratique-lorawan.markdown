---
layout: post
title:  "Pratique du réseau Lora|WAN avec objenious, platformio et Node-RED"
date:   2016-09-16 06:00:00
categories: iot lora lorawan
tags: regular
image: /assets/couverture/antenne.jpg
legend: Installation d'une antenne passerelle LoraWAN au sommet d'un château d'eau près d'Helsinki
---

La société Objenious, filiale de Bouygues Telecom dédiée à l’Internet des Objets (IoT), a eu la courtoisie de me faire parvenir un kit de développement très complet et permettant d’expérimenter le réseau bas débit LoRa actuellement en cours de déploiement.
Cet article a pour objet de vous familiariser au réseau LoRa à travers le développement d’une boîte aux lettres connectée, notifiant en temps réel dès la réception du courrier.

Ce sera également l’occasion de se familiariser avec PlatformIO, un environnement de plus en plus populaire chez les développeurs de systèmes embarqués, ainsi que Node-RED, un fantastique outil permettant d’élaborer des scénarios d’intégration complexes et dont j’ai déjà vanté les mérites dans un précédent billet.

## Présentation du kit

Remarquablement packagé, le kit de développement d’Objenious contient :

- un microcontrôleur The Airboard, très compact et modulaire, de type Arduino avec un facteur de forme XBee et muni d’une batterie LiPo de 3,7 V rechargeable (capacité de 155 mAh)
- le module LoRaBee, un transceiver LoRa avec un facteur de forme XBee conçu par la société ATIM
- une antenne 868 MHz avec connecteur SMA
- un programmateur FTDI muni d’un câble USB
- un jeu de capteurs et d’actionneurs :
-- un bouton poussoir
-- un thermistor
-- un détecteur filaire d’ouverture de porte
-- un jeu de résistances et de câbles

<center>![Kit Objenious](/assets/article_images/objenious-kit.png)</center>

## Un peu de terminologie

A l’instar de SIGFOX (dont j’ai déjà parlé dans cet article), les réseaux LoRa/LoRaWAN font partis de la famille des réseaux de type LPWAN (Low-Power Wide-Area Network), des réseaux de communication radio à longue portée, bas débit et à faible consommation, très adaptés au déploiement d’un réseau d’objets connectés.
Conçu à l’origine par une start-up française (Cycleo) rachetée par le constructeur de semi-conducteurs Semtech, LoRa est en fait le nom donné à la technologie de modulation (la couche physique dans le modèle OSI) opérant sur la bande de fréquences 868 MHz (en Europe) et 975 Mhz (reste du monde) et permettant un débit théorique compris entre 0,3 et 50 kb/s.

LoRaWAN désigne le protocole mettant en oeuvre un réseau étendu à longue portée (correspond à la couche MAC du modèle OSI) utilisant une typologie en étoile où les noeuds (objets connectés) communiquent avec les applications (backends) via une passerelle LoRa.

L’Alliance LoRa est un consortium de plus de 300 constructeurs, intégrateurs et opérateurs visant à standardiser au niveau mondial les réseaux LPWAN autour du protocole LoRaWAN.

Par rapport à SIGFOX, un réseau LoRaWAN présente plusieurs avantages comparatifs :

- un standard et une technologie ouverte. Il est ainsi tout à fait possible de déployer ses propres passerelles LoRa.
- un débit plus important ouvrant la voie à des mises à jour OTA (Over The Air)
- une technologie indépendante de l’opérateur garantissant un principe de réversibilité

## Eléments d'architecture

L’architecture type que nous allons expérimenter est décrite dans le schéma suivant :

<center>![Architecture](/assets/article_images/lora-objenious-arch.png)</center>

Le dépôt du courrier dans la boîte aux lettres connectée met en oeuvre la chaîne de liaison suivante :

- envoi d’un message via le réseau LoRa de l’opérateur (Bouygues Telecom)
- routage du message par la plateforme d’Objenious vers une instance Node-Red (appel d’un service REST/JSON)
- traitement du message par Node-RED et envoi d’une notification Pushbullet
- réception de la notification Pushbullet sur un smartphone

## Environnement de développement

Etant plutôt adepte de la ligne de commande, j’ai abandonné l’EDI Arduino au profit de PlatformIO, un environnement de développement ouvert (vous avez le choix de l’éditeur de code), supportant un très large éventail de micro-contrôleurs et s’intégrant parfaitement à une chaîne d’intégration continue.
J’ai une préférence pour la variante PlatformIO CLI (command line interface) utilisée dans cet article.

### Installation

PlatformIO CLI est écrit en Python (v 2.7 recommandée) et est installable via pip :

{% highlight sh %}
pip install -U platformio
{% endhighlight %}

### Créer la structure du projet

> **Note** : les impatients peuvent récupérer l’ensemble du projet prêt à l’emploi via les commandes suivantes :
{% highlight sh %}
git clone https://github.com/ksahnine/mailbox-lora.git
cd mailbox-lora/objeniouskit/lora-notifier
{% endhighlight %}

Les commandes suivantes permettent d’initialiser un projet vide.

{% highlight sh %}
mkdir lora-notifier
cd lora-notifier
pio -f init -d . -b fio
{% endhighlight %}

Le paramètre -d indique le répertoire racine du projet. Le micro-contrôleur étant pré-chargé avec le bootloader de l’Arduino Fio, on passera l’option -b fio pour définir le type de plateforme cible.

> **Note** : la commande pio est un alias de la commande platformio.

A ce stade, le répertoire projet ne contient que:

- le fichier de configuration du projet platformio.ini
- 2 sous-répertoires src et lib

PlatformIO gère un registre de librairies tierces installables en ligne de commande et offre une gestion simple des mises à jour des dépendances.
Nous utiliserons les librairies suivantes :

- ArmAPI : l’API utilisée pour piloter le transceiver LoRa (ou tout transceiver ATIM de la gamme ARM - Advanced Radio Modem)
- IsTime, une librairie Arduino implémentant un timer (utilisé pour n’envoyer qu’une seule notification pendant une période donnée)

La commande pio lib search permet de rechercher une librairie dans le registre à partir d’un mot clé :

{% highlight sh %}
$ pio lib search lora
[ ID  ] Name             Compatibility         "Authors": Description
---------------------------------------------------------------------------------------------------
[ 374 ] armapi           arduino, atmelavr, atmelsam, espressif, intel_arc32, microchippic32 [...]

$ pio lib search timer
[ ID  ] Name             Compatibility         "Authors": Description
---------------------------------------------------------------------------------------------------
[1006 ] istime           arduino, atmelavr, atmelsam, espressif8266, intel_arc32, microchippic32 [...]
{% endhighlight %}

Procédons à l’installation des 2 librairies via la commande pio lib install suivie des identifiants des librairies :

{% highlight sh %}
pio lib install 374 1006
{% endhighlight %}

> **Note** : la mise à jour des dépendances s’effectue via la commande pio lib update.
La commande pio lib list affiche la liste des librairies installées. 

Déposer dans le répertoire ./src le code source du programme disponible sur mon dépôt Github.
Le code ne présente aucune difficulté. Noter que :

- la fonction setup()
-- configure le module LoRa (via l’API d’ATIM) et a
-- attache la routine d’interruption tiltStateChange() à chaque changement d’état de l’entrée D3 du micro-contrôleur
- la routine d’interruption tiltStateChange() ignore les changements d’état superflus (par plage de 30 secondes) et envoie un message LoRa
- l’intervalle minimum entre 2 notification est de 30 secondes par défaut (variable interval)

> **Note** : Objenious met à disposition publie sur son compte Github quelques exemples de code prêt à l’emploi.

### Téléverser le programme via une liaison série

Raccordez le micro-contrôleur au programmateur FTDI fourni comme suit :

<center>![Raccordement](/assets/article_images/airboard-ftdi.png)</center>

Mettez sous tension le micro-contrôleur puis lancer la commande ci-dessous pour compiler et téléverser le programme :

{% highlight sh %}
pio run -t upload
{% endhighlight %}

### Raccordement du capteur

Le prototype utilise un capteur d’inclinaison ou de vibration bon marché fixé sur le volet de la boite aux lettre. On pourrait également utiliser un interrupteur REED (à lame souple) et un aimant.

<center>![Raccordement](/assets/article_images/airboard-capteur.png)</center>

## Routage vers un serveur d'application

### Côté Objenious

L’interface d’administration d’Objenious permet de définir des scénarios de traitement des données issues des objets connectés : envoi de mail ou routage vers un serveur tiers.
C’est précisément le dernier type de scénario que nous allons mettre en oeuvre afin de rerouter le message vers votre propre serveur d’application (backend Node-RED dans notre cas).
Naturellement, le backend peut être auto-hébergé sur un serveur Raspberry Pi par exemple. Voir à ce sujet mon article Pratique du réseau M2M SIGFOX avec Sens’it et Docker.

Depuis ce lien de l’interface d’administration, créer un scénario de type Routage, sélectionner le type de capteurs à utiliser et saisir l’URL du point de terminaison du serveur Node-RED.

<center>![Callback](/assets/article_images/callback.png)</center>

Le message est publié par la plateforme d’Objenious vers le serveur d’application tiers sous la forme d’un appel HTTP (méthode POST) avec un contenu au format JSON structuré comme suit :

{% gist 936061ecf6bf664c9f7d9d5c1a1d2de7 %}

Outre les données purement applicative (propriété data), on trouvera les informations suivantes :

- device_id : identifiant de l’objet connecté
- timestamp : horodatage d’émission du message
- deveui : identifiant unique sur 64 bits (DevEUI) défini par le protocole LoRaWAN et attribué par le fabricant du transceiver LoRa. Il est utilisé pour identifier l’objet connecté sur le réseau.
- appeui : identifiant unique sur 64 bits (AppEUI) défini par le protocole LoRaWAN, utilisé pour identifier l’application destinatrice du message (le backend d’Objenious j’imagine)
- lat et lng : les coordonnées de l’objet connecté, relevées au moment de la publication du message. La précision est approximative.

### Côté backend Node-RED

Voici le flux de traitement Node-RED permettant de traiter les messages reroutés par le serveur d’Objenious :

<center>![Node RED](/assets/article_images/objenious-nodered.png)</center>

Il est constitué de l’enchaînement des noeuds suivants :

- noeud Objenious in : listener HTTP/POST interceptant les messages publiés par la platefome d’Objenious :

<center>![Node RED](/assets/article_images/nodered-objenious-in.png)</center>

- noeud Ack : fonction d’acquittement du message publié par Objenious (retourne ACK).
- noeud Objenious out : retourne la réponse HTTP contenant l’acquittement au serveur d’Objenious
- noeud Message à envoyer : fonction construisant le message à envoyer dans la notification Pushbullet
- noeud Notification téléphone : envoi d’une notification Pushbullet à un périphérique (smartphone, ordinateur, …)

Le code source du flux de traitement Node-Red est disponible sur mon dépôt Github (à déposer dans le répertoire .node-red de votre instance, puis à personnaliser)

> Quelques ressources pour approfondir :
> - Guide du développeur LoRa (Source : Orange)
> - LoRaWAN (Source : The Thinks Network)

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
