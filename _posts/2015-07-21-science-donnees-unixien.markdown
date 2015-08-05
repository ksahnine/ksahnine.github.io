---
layout: post
title:  "La science des données vue d'un unixien"
date:   2015-07-21 06:00:00
categories: datascience cloudcomputing bigdata
tags: regular featured 
image: http://history-computer.com/ModernComputer/Software/images/Dennis-Ritchie-Ken-Thompson-and-PDP11-UNIX-1972.jpg
legend: Ken Thompson (assis) et Dennis Ritchie, créateurs du système UNIX
---
[Baptiste Coulmont](http://coulmont.com/bac/nuage.html), un universitaire spécialiste de la sociologie des prénoms, diffuse chaque année le classement des prénoms de candidats ayant obtenu une mention *Très Bien* au baccalauréat. Comme pour les années précédentes, ses résultats mettent en évidence une forme de déterminisme social où **Apolline**, par exemple, est beaucoup plus susceptible d’obtenir une mention que **Jordan**.

[La médiatisation de ces travaux](http://www.lemonde.fr/bac-lycee/article/2015/07/10/du-prenom-a-la-mention-au-bac-des-determinismes-sociaux-toujours-puissants_4678806_4401499.html) a suscité ma curiosité. En bon Unixien, je me suis adonné à la pratique de la **science des données** (*data science*) en ligne de commande, en prenant pour objet d’étude les résultats du brevet des collèges 2015 (série générale).

Sur le plan de l’analyse sociologique, j’aboutis aux mêmes conclusions que Baptiste Coulmont :

- **Adèle**, **Apolline**, **Alix**, **Louise** (entre autres) sont sur-représentées chez les détenteurs (détentrices devrais-je écrire) de la mention *Très Bien* au brevet des collèges alors que **Steven**, **Alan**, **Dylan**, **Jordan** ou **Bryan** sont les moins bien représentés
- par ailleurs, les filles obtiennent de bien meilleurs résultats scolaires que les garçons

La représentation visuelle du nuage des prénoms est réalisée avec **GNU Plot**, avec le pourcentage des mentions TB en abscisse et le nombre d’élèves en ordonnée (cliquer sur l’image pour l’agrandir).
<center><a href="http://blog.inovia-conseil.fr/wp-content/uploads/2015/07/brevet-mentions-2015.png"><img width="60%" src="http://blog.inovia-conseil.fr/wp-content/uploads/2015/07/brevet-mentions-2015.png" /></a></center>

Dans cet article, je décrirai chacune des étapes m’ayant permis d’établir ce résultat, en usant abondamment de la **ligne de commande**.<br/>A travers cet article, j’espère pouvoir faire la démonstration qu’un Unixien est aussi un Data Scientist qui s’ignore.

L’ensemble du code dont il est fait référence dans l’article est disponible dans [mon dépôt GitHub](https://github.com/ksahnine/datascience-brevet-mentions).

>**Plan de l'article :**

> - [Méthodologie et outils](#1)
> - [Création du cluster EC2](#2)
> - [Le script d’extraction des données](#3)
> - [Distribution des traitements avec GNU Parallel](#4)
> - [Consolidation et visualisation des résultats](#5)

## <a name="1"></a>Méthodologie et outils
Le site [FranceTV info](http://www.francetvinfo.fr/) publie les [résultats du brevet des collèges](http://www.francetvinfo.fr/brevet) par commune, mais il ne s’agit que d’une publication partielle. Néanmoins, le site permet de constituer un échantillon significatif de plus de 250 000 résultats.<br />
La méthode utilisée suit les étapes suivantes :

- **Extraction des données** : comme on peut s’en douter, aucun fichier formaté n’est mis à disposition. J’ai donc utilisé la technique du *harvesting* (ou *web scraping*) consistant à extraire le contenu du site, suivi d’un formatage afin de faciliter son exploitation ultérieure.
<br/>Techniquement, c’est un script développé en **Python** qui est chargé de l’extraction. Il s’appuie sur l’excellente librairie [**BeautifulSoup**](http://www.crummy.com/software/BeautifulSoup/) dont j’ai déjà vanté les qualités dans un précédent article.<br />
Le script d’extraction est conçu pour être **parallélisable**, sans quoi la durée du traitement serait au bas mot d’une journée sur une machine de série. Il n’extrait que les résultats des collèges d’une commune dont l’identifiant est passé en paramètre.
- **Parallélisation des traitements** : les traitements d’extraction et d’analyse des données sont distribués via [**GNU Parallel**](http://www.gnu.org/software/parallel/) sur un **cluster Amazon EC2** constitué de **10 instances** `t2.micro` (1 CPU 3,3 Ghz / 1 Go RAM) sous Ubuntu. La parallélisation est orchestrée depuis un portable sous OS X.
Le schéma ci-dessous décrit l’architecture de l’ensemble :
![Nuage des prénoms](http://blog.inovia-conseil.fr/wp-content/uploads/2015/07/architecture.png)
- **Agrégation des données** : la consolidation des données issue des traitements réalisés par le cluster est effectuée sur un portable (sous OS X) à l’aide des outils de tout bon Unixien (`grep`, `awk`, `sed`, `join`, `paste`, `bc`, `sort`, `uniq`, etc)
- **Visualisation** : pour faciliter l’interprétation des résultats, la visualisation des données est effectuée avec `gnuplot` sur un portable (sous OS X)
## <a name="2"></a>Création du cluster EC2
Il faut disposer d’un compte AWS (Amazon Web Services). Pour information, Amazon propose de tester (quasi) [gratuitement ses services](http://aws.amazon.com/fr/free/) pendant 12 mois à hauteur de 750 heures de consommation mensuelle.

Une fois votre compte créé, installer `awscli`, le client en ligne de commande de l’API AWS ainsi que l’utilitaire `jq`, un véritable couteau suisse de JSON. Ce dernier nous sera très utile car la sortie standard du client awscli est au format JSON :
{% highlight sh %}
pip install awscli
brew install jq
{% endhighlight %}

Configurer votre client `awscli` :
{% highlight sh %}
aws configure
{% endhighlight %}

Plutôt que d’utiliser directement le client `aws`, [je mets à disposition](https://github.com/ksahnine/datascience-brevet-mentions/tree/master/ec2) un ensemble de scripts Shell prêts à l’emploi permettant de mettre en place le cluster EC2 depuis une invite de commande sur un simple portable :

- `aws-ec2-create_key_pair.sh` : créer une paire de clés asymétriques
- `aws-ec2-create_security_group.sh` : créer un groupe de sécurité
- `aws-ec2-run_instances.sh` : créer un nombre prédéfini d’instances EC2
- `aws-ec2-config_ssh.sh` : mettre à jour la config SSH locale pour accéder aux instances EC2
- `aws-ec2-get_public_ips.sh` : afficher la liste des IP publiques de toutes les instances EC2

Pour récupérer les scripts localement, utiliser les commandes :
{% highlight sh %}
git clone https://github.com/ksahnine/datascience-brevet-mentions.git
cd datascience-brevet-mentions/ec2
{% endhighlight %}

Effectuons pas à pas toutes les opérations de construction du cluster depuis un ordinateur de contrôle (le laptop du schéma ci-dessus) :

- créer une paire de clé asymétriques dont le nom est passé en paramètre (`inovia-kp` dans notre cas) : <br/>```
./aws-ec2-create_key_pair.sh inovia-kp
``` <br/>Le script sauvegarde la **clé privée** dans le fichier `~/.ssh/inovia-kp.pem`.
- créer un groupe de sécurité dont le nom est passé en paramètre (`inovia-sg` dans notre cas) :<br />```
./aws-ec2-create_security_group.sh inovia-sg
``` <br />Le script rajoute une **règle d’accès par SSH** mais restreint à l’adresse IP publique de votre ordinateur si vous êtes derrière un routeur.
- créer et démarrer 10 instances EC2 de type `t2.micro` :<br />```
./aws-ec2-run_instances.sh inovia-kp inovia-sg 10```<br />Les noms de clé asymétrique et de groupe de sécurité créés ci-dessus sont passés en paramètre.
- configurer le fichier `~/.ssh/config` pour chacune des instances EC2 :<br />```
./aws-ec2-config_ssh.sh inovia-kp
``` <br />Ce script configure automatiquement le fichier `~/.ssh/config` pour toutes les instances EC2 du cluster, et dont voici un extrait :
<pre>
Host 52.11.189.180
    IdentityFile ~/.ssh/inovia-kp.pem
    User ubuntu
    StrictHostKeyChecking no
</pre>
- pour afficher toutes les adresses IP publiques des instances EC2, exécuter le script `./aws-ec2-get_public_ips.sh`

A ce stade, vous pouvez accéder à n’importe quelle instance EC2 par SSH sans avoir besoin de saisir un mot de passe. Ex : `ssh ubuntu@52.11.189.180`
## <a name="3"></a>Le script d'extraction des données
Développé en Python, il utilise la technique du *harvesting* via la librairie **BeautifulSoup** installable comme suit :

- sous Mac OS X : `sudo pip install beautifulsoup4`
- sous Ubuntu : `sudo apt-get install python-bs4` (ou via `pip`)

Je ne vais pas rentrer dans les détails du développement. Je vous renvoie au [code source](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/extract_brevet.py) pour plus de détails.<br/>
Retenez qu’il fonctionne de la manière suivante :

- la commande `./extract_brevet.py` retourne une liste de triplets (académie, département, commune) dont voici un très court échantillon :
<pre>
bordeaux Pyrenees-Atlantiques bedous
bordeaux Pyrenees-Atlantiques biarritz
bordeaux Pyrenees-Atlantiques bidache
</pre>
- la commande `./extract_brevet.py -a bordeaux -d Pyrenees-Atlantiques -c biarritz` extrait les résultats obtenus par les candidats de la commune de Biarritz (dans l’académie de Bordeaux). <br />Le script produit 2 fichiers (un par série) :
  - pour la série générale : `output/bordeaux/Pyrenees-Atlantiques/biarritz/Serie-Generale.csv`
  - pour la série professionnelle : `output/bordeaux/Pyrenees-Atlantiques/biarritz/Serie-Professionnelle.csv`
<br />Les données extraites sont au format CSV, dont voici un exemple fictif :
<pre>
bordeaux;Pyrenees-Atlantiques;biarritz;ADMIS;MENTION BIEN;Dupont;Lucie
</pre>

## <a name="4"></a>Distribution des traitements avec GNU Parallel
[**GNU Parallel**](http://www.gnu.org/software/parallel/) est un formidable outil en ligne de commande permettant de paralléliser l’exécution de scripts shell sur un système UNIX.
<br />
La parallélisation peut être **verticale**, c’est-à-dire exploitant tous les coeurs du processeur, et/ou **horizontale** en distribuant les traitements sur plusieurs machines, ce qui sera le cas dans cet article.
> **Note** : Il n’est pas nécessaire d’installer GNU Parallel sur les machines distantes. Seul un accès SSH suffit.

### Installation de GNU Parallel
Installons GNU Parallel sur le portable de contrôle :

- sous Mac OS X : `brew install parallel`
- sous Ubuntu : `sudo apt-get install parallel`

### Installation de BeautifulSoup sur les instances EC2
Les instances EC2 sous Ubuntu disposent de l’interpréteur Python mais pas de la librairie BeautifulSoup.<br />
Créons le fichier `machines` contenant toutes les adresses IP publiques des instances EC2 :
{% highlight sh %}
./aws-ec2-get_public_ips.sh > machines
{% endhighlight %}
A l’aide de GNU Parallel, installons *BeautifulSoup* à distance sur l’ensemble des instances EC2 :
{% highlight sh %}
parallel --nonall --slf machines "sudo apt-get install python-bs4"
{% endhighlight %}
[<sub><sup>Source : `parallel-install-bs4.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/parallel-install-bs4.sh)
### Distribution du traitement d’extraction des données
Dans un premier temps, générons le fichier `communes.csv` contenant les triplets (académie, dépa	rtement, commune) pour l’ensemble des académies :
{% highlight sh %}
./extract_brevet.py > communes.csv
{% endhighlight %}
Dans un second temps, lançons les traitements d’extraction répartis sur les instances du cluster par lots de 10 jobs. Ainsi, en rythme de croisière, 100 communes sont traitées simultanément :
{% highlight sh %}
parallel -a communes.csv --colsep ' ' -j 10 --basefile ac_scrap.py \
         --slf machines "./ac_scrap.py -a {1} -d {2} -c {3} 2> {3}.log"
{% endhighlight %}
[<sub><sup>Source : `parallel-extract-data.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/parallel-extract-data.sh)

On notera que :

- le flag `-a` est suivi du fichier contenant les triplets
- le flag `--basefile` est suivi du nom du script téléversé et exécuté sur chacune des machines distantes
- le flag `--colsep` est suivi du séparateur de champ dans le fichier en entrée (le caractère espace dans notre cas).
- le flag `-j` est suivi du nombre de jobs par machine
- le flag `--slf` est suivi du fichier contenant la liste des adresses IP des machines du cluster

### Distribution du traitement statistique
L’extraction étant terminée, mettons au point deux scripts, l’un affichant tous les prénoms des candidats de série générale, l’autre les candidats ayant obtenu une mention très bien :
{% highlight sh %}
$ cat get_prenoms_full.sh
find output -type f -name "Serie-Generale.csv" -exec cat {} \; | cut -d';' -f7

$ cat get_prenoms_meTB.sh
find output -type f -name "Serie-Generale.csv" -exec cat {} \; | grep "TRES BIEN" | cut -d';' -f7
{% endhighlight %}
[<sub><sup>Source : `get_prenoms_full.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/get_prenoms_full.sh) [<sub><sup>Source : `get_prenoms_meTB.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/get_prenoms_meTB.sh)
<br/>Distribuons les traitements sur le cluster (*map*) et comptabilisons les prénoms (*reduce*), triés par ordre alphabétique :

- de tous les candidats :<br /><pre>
parallel --nonall --basefile get_prenoms_full.sh --slf machines \
             --pipe "./get_prenoms_full.sh" | sort | uniq -c | \
             sort -k2 > stats_brevet_full.csv
</pre>
[<sub><sup>Source : `parallel-get_prenoms_full.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/parallel-get_prenoms_full.sh)
- des candidats ayant obtenu une mention TB :<br />{% highlight sh %}
parallel --nonall --basefile get_prenoms_meTB.sh --slf machines \\
             --pipe "./get_prenoms_meTB.sh" | sort | uniq -c | \\
             sort -k2 > stats_brevet_meTB.csv
{% endhighlight %}
[<sub><sup>Source : `parallel-get_prenoms_meTB.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/parallel-get_prenoms_meTB.sh)

Le format des fichiers générés est du type :
{% highlight text %}
[...]
   1 Hormoz
  69 Hortense
   2 Hosanna
   1 Hosni
[...]
{% endhighlight %}
## <a name="4"></a>Consolidation et visualisation des résultats
Utilisons la commande `join` pour assembler les fichiers `stats_brevet_full.csv` et `stats_brevet_meTB.csv` de façon à obtenir, pour chaque prénom, le nombre total d’occurrences et le nombre de mentions TB :
{% highlight sh %}
join -1 2 -2 2 stats_brevet_full.csv stats_brevet_meTB.csv > brevet-mentions-2015.csv
{% endhighlight %}
Le fichier produit ressemble à ceci :
{% highlight text %}
[…]
Hortense 24 69
Houda 1 11
Houdaifa 1 2
Housni 1 2
[…]
{% endhighlight %}
Ainsi, sur 69 candidats prénommés Hortense, 24 ont eu une mention TB.

Nous y sommes presque. Rajoutons pour chaque enregistrement ayant plus de 190 candidats, le pourcentage ayant obtenu une mention TB :
{% highlight sh %}
join -1 2 -2 2 stats_brevet_meTB.csv stats_brevet_full.csv | \
       awk '$3 > 190 { printf("%s %d %d %2.1f\n", $1, $2, $3, ($2/$3)*100) }' > brevet-mentions-2015.csv
{% endhighlight %}
[<sub><sup>Source : `reduce-join_results.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/reduce-join_results.sh)

Voici un extrait du fichier `brevet-mentions-2015.csv où l’on relève que 16,3 % des prénommées Amandine ont obtenu une mention TB :
{% highlight text %}
[…]
Allan 12 228 5,3
Amandine 144 884 16,3
Ambre 79 403 19,6
[…]
{% endhighlight %}

Enfin, visualisons le résultat final avec **GNU Plot** sous la forme d’un nuage de prénoms où le nombre de candidats est représenté en ordonnée et le pourcentage de mention TB en abscisse :
{% highlight sh %}
gnuplot <<EOF
plot "brevet-mentions-2015.csv" u 4:3:1 w labels rotate by 20 font "Helvetica,8"
EOF
{% endhighlight %}
[<sub><sup>Source : `dataviz-results.sh`</sup></sub>](https://github.com/ksahnine/datascience-brevet-mentions/blob/master/app/dataviz-results.sh)

<center><a href="http://blog.inovia-conseil.fr/wp-content/uploads/2015/07/brevet-mentions-2015.png"><img width="60%" src="http://blog.inovia-conseil.fr/wp-content/uploads/2015/07/brevet-mentions-2015.png" /></a></center>


[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
