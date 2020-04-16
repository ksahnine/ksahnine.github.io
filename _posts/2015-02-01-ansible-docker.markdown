---
layout: post
title:  "Provisionner des microservices avec Ansible et Docker"
titleColor: "#fffacd"
date:   2015-02-01 06:00:00
categories: microservice ansible docker
tags: regular
image: /assets/article_images/ansible.png
legend: Ansible, machine de communication supraluminique imaginée par l'auteure de Science Fiction Ursula Le Guin
---
C’est bien connu, la fainéantise est la plus grande qualité d’un administrateur système.<br />
Si ce dernier est en outre un inconditionnel de [**Puppet**](http://puppetlabs.com/), [**Chef**](https://www.chef.io/chef/) ou [**Ansible**](http://www.ansible.com/), c’est qu’il élève l’indolence au rang de valeur cardinale.<br />
C’est dans cet état d’esprit que nous allons voir comment provisionner depuis un ordinateur portable et avec très peu d’efforts, une plateforme d’hébergement de microservices à l’aide de **Docker** et **Ansible**.

## Ansible : présentation et fonctionnement
[**Ansible**](http://www.ansible.com/) est un outil automatisant la configuration et la gestion d’un parc de machines.<br />
Ecrit en Python et doté de [nombreux modules](http://docs.ansible.com/modules_by_category.html) extensibles, il permet de réaliser à distance des tâches aussi diverses que l’installation et la configuration d’une application, le montage d’un système de fichiers ou le déploiement sur le cloud EC2 d’Amazon.<br />
**Aucun agent n’est installé sur les machines à administrer**, les seuls pré-requis étant la disponibilité de **SSH** et **Python**.

Un **module Ansible** est une unité de traitement réutilisable écrite en Python et effectuant une tâche précise. Par exemple, le module [`shell`](http://docs.ansible.com/shell_module.html) permet d’exécuter une commande quelconque sur une machine distante.<br />
Lorsque qu’une tâche Ansible est lancée depuis le poste de contrôle, Ansible :

- transfère via SSH le ou les modules dans le répertoire temporaire `~/.ansible/tmp` du serveur distant
- exécute via SSH le ou les modules sur le serveur distant
- analyse la sortie standard du module (au format JSON)

Si, comme on le verra dans notre cas d’utilisation, les clés publiques SSH sont déployées sur les serveurs distants, Ansible peut ouvrir une connexion SSH sans avoir besoin de mot de passe.

## Exercice pratique
Dans de précédents articles d’*Inovia Blog*, nous avions d’une part développé un service REST de consultation des horaires du métro avec *Dropwizard* et d’autre part dockerisé celui-ci (l’image `ksahnine/ratp-rest-api:1.0` est [disponible sur mon repository public sur *Docker Hub*](https://registry.hub.docker.com/u/ksahnine/ratp-rest-api/)).<br />
Nous allons repartir de cet exemple pour construire l’infrastructure hébergeant ce service.<br />
Toutes les tâches de construction et de configuration de l’infrastructure, décrite dans le schéma ci-dessous, sont coordonnées par Ansible et exécutées depuis un ordinateur portable (`laptop.local`).

<center><img src="{{site.url}}/assets/article_images/ansible-docker-architecture.png" style="display: block; margin: auto;" /></center>

L’infrastructure technique est constituées de 2 serveurs Linux :

- `pegase.loca`l : un serveur sous *Arch Linux* hébergeant un répartiteur de charge (*nginx* configuré en loadbalancer)
- `orion.local` : un serveur sous *Debian* hébergeant 2 instances du service dockerisé sur 2 ports différents (`8080` et `8081`).

Voici les tâches à réaliser sur chacun des noeuds :

- côté front (`pegase.local`) :
  - installation de *nginx* via le gestionnaire de paquets `pacman` (on est sur un serveur *Arch Linux*). <br />Noter que *nginx* aurait très bien pu être déployé dans un conteneur Docker.
  - déploiement de la configuration *nginx* pour répartir la charge sur les 2 instances du service
  - redémarrage du serveur *nginx*
- côté back (`orion.local`) :
  - installation de Docker, via le gestionnaire de paquets aptitude (`apt-get`)
  - provisionnement de 2 conteneurs Docker avec l’image du service de consultation des horaires du métro (image `ksahnine/ratp-rest-api:1.0` disponible sur *Docker Hub*)

Réaliser toutes ces tâches manuellement serait fastidieux et potentiellement source d’erreur. Avec Ansible, **toutes ces tâches peuvent être réalisées en une seule commande** et sans risque d’erreur.<br />
Toutes les opérations qui vont suivre sont à réaliser sur la machine de contrôle, un simple portable dans notre cas (`laptop.local`).

## Mise en oeuvre
### Installation de Ansible
Le mode opératoire d’installation le plus simple s’effectue via `brew` (sous OS X) :
{% highlight sh %}
brew install ansible
{% endhighlight %}
ou `pip` :
{% highlight sh %}
sudo pip install ansible
{% endhighlight %}

### Création et déploiement des clés SSH
Pour permettre à Ansible d’ouvrir une connexion SSH sans utiliser de mot de passe, nous allons mettre en oeuvre une **authentification par clé SSH**.<br />
Commençons par générer une paire de clés RSA2 sans *passphrase* à l’aide de la commande `ssh-keygen` :
{% highlight sh %}
ssh-keygen -t rsa -f ~/.ssh/id_rsa_galaxy -N "" -C "Galaxy"
{% endhighlight %}
La clé privée (fichier `~/.ssh/id_rsa_galaxy`) reste évidemment sur le poste client (`laptop.local`) et sera utilisée par Ansible.<br />
En revanche, la clé publique (fichier `~/.ssh/id_rsa_galaxy.pub`) doit être copiée sur les 2 serveurs à administrer (`orion.local` et `pegase.local`).<br />
L’utilitaire `ssh-copy-id` permet de copier une clé publique dans le fichier `~/.ssh/authorized_keys` des comptes d’exécution des serveurs distants :
{% highlight sh %}
ssh-copy-id -i ~/.ssh/id_galaxy.pub markab@pegase.local
ssh-copy-id -i ~/.ssh/id_galaxy.pub betelgeuse@orion.local{% endhighlight %}
Sous OSX, vous pouvez installer l’utilitaire `ssh-copy-id` comme suit :
{% highlight sh %}
curl -L https://raw.githubusercontent.com/beautifulcode/ssh-copy-id-for-OSX/master/install.sh | sh
{% endhighlight %}

### Le fichier d’inventaire
Ce fichier au [format INI](http://fr.wikipedia.org/wiki/Fichier_INI) contient l’inventaire des machines orchestrées par Ansible (fichier `/etc/ansible/hosts` par défaut), avec 2 serveurs dans notre cas :
{% gist 15527b36a4470dde98ce %}

Les comptes d’exécution des serveurs distants sont définis dans la variable Ansible `ansible_ssh_user`.<br />
Par ailleurs, la localisation de la clé privée utilisée pour l’ouverture des connexions SSH est définie dans la variable Ansible `ansible_ssh_private_key_file`.<br />
Enfin, on remarquera l’utilisation d’une variable personnalisée `env` que l’on exploitera plus loin.

### Le fichier Playbook : un Makefile de l’infrastructure
Les tâches de construction et de configuration de l’infrastructure sont décrites dans un langage très simple utilisant le format **YAML** : le **playbook**.<br />
D’une certaine façon, **le playbook est le Makefile de construction d’une infrastructure IT**. On pourrait également dire que le *playbook* est à Ansible ce que le *Dockerfile* est à Docker, mais au fond la paternité de cette idée revient au bon vieux *Makefile* !<br />
Le fichier `infra_services_playbook.yml` ci-dessous décrit l’ensemble des tâches à réaliser pour notre environnement :
{% gist bb0872a005e25b675774 %}

Quelques observations sur la façon dont nous configurons *nginx* dans le fichier *playbook* :

{% highlight text %}
- name: Configuration loadbalancer
  template: src=conf/loadbalancer-nginx-{{env}}.conf dest=/etc/nginx/nginx.conf
{% endhighlight %}

On remarquera que :

- le fichier de configuration *Nginx* à déployer (`conf/loadbalancer-nginx-prod.conf` dont le contenu figure ci-après) est stocké sur le poste de contrôle (`laptop.local`).
{% gist b7e947a8f6e73d51a3d0 %}

- Le fichier de destination (`/etc/nginx/nginx.conf`) correspond au système de fichier du serveur distant (`pegase.local`).
- le nom du fichier source est variabilisé dans le *playbook* (variable `env` déréférencée `{\{ env }}`)
- la variable `env` est renseignée dans le fichier d’inventaire `hosts.ini` (`env=prod`)

### Construction de l’infrastructure
La commande `ansible-playbook` permet d’exécuter un *playbook*, et donc l’ensemble des tâches de construction de l’infrastructure. Il prend en paramètre le fichier d’inventaire (`-i hosts.ini`) ainsi que le fichier *playbook* `infra_services_playbook.yml` décrit plus haut :
{% highlight sh %}
ansible-playbook -i hosts.ini infra_services_playbook.yml
{% endhighlight %}
Voici le compte-rendu d’exécution du playbook affiché sur la sortie standard :
{% highlight sh %}
PLAY [** Frontend **] *********************************************************

TASK: [Installation nginx] ****************************************************
changed: [pegase.local]

TASK: [Configuration loadbalancer] ********************************************
changed: [pegase.local]

TASK: [Redemarrage nginx] *****************************************************
changed: [pegase.local]

PLAY [** Backend **] **********************************************************

TASK: [Installation Docker] **************************************************
changed: [orion.local]

TASK: [Arret des containers] **************************************************
changed: [orion.local]

TASK: [Provisionnement du service (instance 1)] *******************************
changed: [orion.local]

TASK: [Provisionnement du service (instance 2)] *******************************
changed: [orion.local]

PLAY RECAP ********************************************************************
orion.local                : ok=4    changed=4    unreachable=0    failed=0
pegase.local               : ok=3    changed=3    unreachable=0    failed=0
{% endhighlight %}

Et voilà !

A noter que l’on pourrait élégamment utiliser le [module Ansible docker](http://docs.ansible.com/docker_module.html), en installant préalablement le module Python [*docker-py*](https://github.com/docker/docker-py) dont il dépend.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
