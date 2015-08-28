---
layout: post
title:  "Pratique du réseau M2M SIGFOX avec Sens'it et Docker"
date:   2015-08-26 06:00:00
categories: iot m2m sigfox docker
tags: regular
image: http://webdesobjets.fr/wp-content/uploads/2015/03/sensit-sigfox.jpg?898452
legend: L'objet connecté Sens'it
---
**SIGFOX** est un opérateur de télécommunication français exploitant un réseau **M2M** (*Machine to Machine*) à vocation mondiale et dédié à l'[*Internet des Objets*](https://fr.wikipedia.org/wiki/Internet_des_objets).

Cette société a eu la courtoisie de me faire parvenir un exemplaire du [**Sens'it**](https://www.sensit.io/), un objet connecté intégrant plusieurs capteurs (**température**, **mouvement**, **son**) ainsi qu'un **bouton poussoir**.

[//]: # (<center>![SensIt, le goodie connecté](/assets/article_images/sensit.png)</center>)

Développé par [**AXIBLE technologies**](http://www.axible-connects-for-you.com/), partenaire de *SIGFOX*, *Sens'it* est utilisable sans connaissances informatiques pour, par exemple :

- effectuer un relevé de température régulier
- être notifié par SMS ou email à chaque détection de mouvement

Une [API REST](https://api.sensit.io/v1/) permet fort heureusement d'étendre le champ des possibles. On verra dans cet article comment l'utiliser pour obtenir en un clic les disponibilités de stations [**Vélib**](http://www.velib.paris/) à proximité de son lieu de travail ou d'habitation :
<center><iframe src="https://vid.me/e/n45c?tools=1" width="480" height="480" frameborder="0" allowfullscreen webkitallowfullscreen mozallowfullscreen scrolling="no"></iframe></center>

Le service est ***dockerisé*** et déployé sur un *Raspberry Pi model B*.

## Le réseau SIGFOX en bref
J'ai eu l'occasion d'expérimenter dans le cadre de projets *IoT* plusieurs technologies de connexion à un réseau sans fil dont :

- le microcontrôleur avec capacités WiFi [**ESP8266**](http://espressif.com/en/products/esp8266/) (*Espressif*), bon marché mais assez énergivore
- le transceiver [**nRF24l01+**](https://www.nordicsemi.com/eng/Products/2.4GHz-RF/nRF24L01P) (*Nordic*), très bon marché et très économe en énergie, mais pas trivial à utiliser
- le *dongle* 3G **Huawei e173**, relativement cher et très énergivore

Par rapport à d'autres solutions de connexion à un réseau sans fil, en particuler le réseau GSM, le réseau *SIGFOX* présente un certain nombre d'avantages comparatifs :

- une couverture internationale
- un faible coût d'utilisation (10 à 16 € par an, variable en fonction du volume de données échangées et du nombre de *devices*)
- des modules ou transceivers *SIGFOX* **consommant très peu d'énergie** et bon marché (compter 15 € le *chip*)
- des communications sécurisées

Noter que le réseau impose un certain nombre de contraintes :

- il s'agit d'un réseau bas débit (jusqu'à 12 bytes par message), ce qui n'est pas contraignant pour un réseau M2M
- il n'est pas possible d'émettre plus de 140 messages par jour et par device

> **Note** : pour en savoir beaucoup plus sur le sujet, on lira avec profit [les articles](http://www.disk91.com/category/technology/sigfox/) de [Paul Pinault](https://twitter.com/disk_91) (aka *Disk91*)


## Un peu de hardware

Je n'ai pas résisté à la tentation d'ouvrir le boitier :
<center>![A l'intérieur du SensIt](/assets/article_images/sensit-inside.png)</center>

En zoomant, on remarquera en particulier :

- à gauche : le microcontrôlleur PIC **PIC16F1718** de *Microchip*
- à droite : le transceiver **CC1120** de *Texas Instrument* opérant sur la bande de fréquence utilisée par *SIGFOX* (868 MHz en Europe).
<center>![Zoom](/assets/article_images/sensit-zoom.png)</center>

[Anthonny Quérouil](https://twitter.com/anthonny_q) a [écrit un billet](http://anthonnyquerouil.fr/2015/08/24/Sensit-mon-petit-objet-connecte.html) très complet de présentation et d'utilisation du produit ainsi que de l'API REST.

## Un exemple de service avec Sens'it

A titre personnel, je trouve que le bouton poussoir est une des fonctions les plus intéressantes de cet objet. 

Etant parisien et utilisateur du service [**Vélib**](http://www.velib.paris/), je trouve l'application officielle peu pratique pour consulter les disponibilités.<br />
Quoi de plus simple que de cliquer sur un objet connecté et de savoir instantanément combien de vélos sont disponibles dans son quartier ?<br />
Voici une démo de l'application en action :
<center><iframe src="https://vid.me/e/n45c?tools=1" width="480" height="480" frameborder="0" allowfullscreen webkitallowfullscreen mozallowfullscreen scrolling="no"></iframe></center>

Le service est développé sur une *stack* Python ([**Flask**](http://flask.pocoo.org/)) et utilise la technique du [*web scraping*](https://fr.wikipedia.org/wiki/Web_scraping) pour extraire les disponibilités sur le site [*velib.paris.fr*](http://www.velib.paris/). <br />
A cet effet, j'ai utilisé l'excellente librairie [**BeautifulSoup**](http://www.crummy.com/software/BeautifulSoup/).

Les notifications sont acheminées via [**Pushbullet**](https://www.pushbullet.com/), dont l'application est disponible sur une multitude de plateformes (*iOS*, *Android*, *Mac*, *Windows*, *Chrome*, etc.) :
<center>![Notifications Vélib](/assets/article_images/velib.png)</center>

Le code du service est disponible sur [*GitHub*](https://github.com/ksahnine/sensit-sigfox/tree/master/apps/velib-callback).

## Déploiement du service
Le service est packagé sous la forme d'une image **Docker** *ARM* (`ksahnine/rpi-sensit-velib`), ce qui simplifiera grandement son déploiement.

Il existe plusieurs solutions pour déployer un hôte Docker sur Raspberry Pi (via *ArchLinux* par exemple).<br />
Personnellement, j'utilise [l'image de **Hypriot**](http://blog.hypriot.com/downloads/), une distribution allégée de type *Debian* intégrant Docker 1.6, très rapide à mettre en oeuvre et fonctionnant sur n'importe quel modèle de Raspberry Pi.

- commencez par flasher l'image sur une carte SD (ex. sous OS X) :
{% highlight sh %}
$ diskutil unmountDisk /dev/disk1
$ sudo dd if=hypriot-rpi-20150416-201537.img of=/dev/rdisk1 bs=1m
{% endhighlight %}
- se connecter au Raspberry Pi via ssh (`root`/`hypriot`)
- récupérer le fichier de configuration modèle de l'application :
{% highlight sh %}
$ wget https://github.com/ksahnine/sensit-sigfox/raw/master/apps/velib-callback/conf/config.yml
{% endhighlight %}
- personnaliser le fichier de configuration `~/config.yml` en renseignant [le jeton d'accès de votre compte **Pushbullet**](https://www.pushbullet.com/#settings/account) ainsi que l'adresse de votre lieu de travail ou d'habitation :
{% highlight yaml %}
# Pushbullet configuration
pushbullet:
    api: KcY0dyPlqiu0Dp1mrH7RI30AP8O83ZYa

# Address
location: 29 rue d'Astorg, Paris
{% endhighlight %}
- lancer l'application avec un simple `docker run` :
{% highlight sh %}
$ docker run -d -p 5000:5000 -v ~/config.yml:/app/conf/config.yml ksahnine/rpi-sensit-velib
{% endhighlight %}
- se connecter à [l'interface d'administration](https://www.sensit.io/) de **Sens'It** et configurer l'URL de *callback* du bouton :
<center>![Notification Sens'it](/assets/article_images/sensit-config.png)</center>
- double cliquer sur le *Sens'it* pour recevoir une notification

> **Remarque** : Le `docker pull` initial est long sur un RPi model B. Néanmoins, la taille de l'image de base utilisée peut être largement optimisée.

## Eléments d'architecture
Le schéma ci-dessous décrit l'ensemble de la chaîne de liaison :

<center>![Chaîne de liaison](/assets/article_images/sensit-archi.png)</center>

- le double click du *Sens'it* donne lieu à l'émission d'un message via le réseau *SIGFOX*
- *callback* des serveurs d'*AXIBLE* vers le service hébergé sur le Raspberry Pi
- récupération des données depuis le site *velib.paris*
- envoi de la notification via *Pushbullet*

## Quelques remarques

J'ai synthétisé [quelques remarques et observations](https://github.com/ksahnine/sensit-sigfox/wiki/Remarques-tests-API-Sens'it) suite à mes expérimentations.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
