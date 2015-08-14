---
layout: post
title:  "Analyse du hashtag #TelAvivSurSeine"
titleColor: "#fff"
date:   2015-08-14 06:00:00
categories: datascience unix bigdata
tags: regular
image: https://i.embed.ly/1/display/resize?key=1e6a1a1efdb011df84894040444cdc60&url=http%3A%2F%2Fpbs.twimg.com%2Fmedia%2FBsnms29CYAADqYv.jpg
legend: PEACE
---
L'organisation de l'opération *"Tel Aviv Sur Seine"* a suscité une [importante polémique](http://www.lemonde.fr/societe/article/2015/08/09/tel-aviv-sur-seine-la-mairie-de-paris-ne-renonce-pas-malgre-la-polemique_4718346_3224.html) née d'abord au sein des **réseaux sociaux** (*Twitter* en particulier) avant de prendre une tournure **politique** puis **médiatique**.

Le souflet est déjà retombé, mais cette polémique est une leçon pour l'avenir car elle révèle la puissance des réseaux sociaux comme moyen de **peser sur l'agenda médiatique** et l'incapacité des médias traditionnels à **distinguer un mouvement spontané d'une opération militante**.

[Nicolas Vanderbiest](https://twitter.com/nico_vanderb), assistant universitaire à l’université catholique de Louvain et spécialiste des réseaux sociaux, est l'auteur [d'un article](http://rue89.nouvelobs.com/2015/08/11/telavivsurseine-dun-tweet-a-bfmtv-lastroturfing-mode-demploi-260700) fort instructif décrivant l'exploitation du hashtag `#TelAvivSurSeine` par un nombre restreint de militants très engagés, et le manque de clairvoyance des médias face à l'[**astroturfing**](https://fr.wikipedia.org/wiki/Astroturfing), une technique de manipulation de l'opinion.

La démonstration de Nicolas souffre néanmoins d'un petit défaut, celui de s'appuyer sur des outils propriétaires et payants ([*Visibrain Focus*](http://www.visibrain.com/fr/)).

Je n'ai évidemment pas la prétention de faire aussi bien que *Visibrain* mais il est possible de s'adonner soi-même à l'**exploration de données** en étant suffisamment habile sous **UNIX** et relativement à l'aise avec le langage **Python**.<br />
Je me suis prêté à cet exercice par **simple curiosité intellectuelle**, et certainement pas pour des considérations politiques ou militantes.

J'en ai tiré les faits suivants :

- la mobilisation sur *Twitter* contre l'évènement `#TelAvivSurSeine` n'est pas assis sur un mouvement massif, spontané et populaire
- elle est d'abord le fait d'une poignée de comptes militants jouissant d'une certaine audience et liés les uns aux autres par des affinités politiques ou identitaires
- le compte du *Parti de Gauche* et de militants adhérents à ce parti a joué un rôle important dans la médiatisation de la polémique, à l'instar de comptes de personnalités connues et pro évènement
- la campagne BDS du *10 Août 2015* contre l'évènement a été efficace et coordonnée
- sur toutes les périodes étudiées, les clients Twitter les plus utilisés sont, dans l'ordre, le client *iPhone*, le client *Android* puis le client *Web* **sauf** le *10 Août 2015* autour de 19h où c'est le client *Web* (navigateur) qui arrive en tête. Cette anormalité pourrait s'expliquer par une utilisation de comptes fictifs durant cette période, et destinés à *faire du bruit*.

Je présenterai dans cet article les résultats obtenus en évitant les digressions techniques car l'article serait trop long.<br />
Dans mon prochain billet, j'aborderai à nouveau ce sujet mais sous un angle beaucoup plus technique, à l'occasion duquel je publierai sur [mon compte *GitHub*](https://github.com/ksahnine) l'ensemble du code source m'ayant permis d'obtenir ces résultats.

### Collecte des données
Le script de collecte consiste à rechercher l'ensemble des tweets portant le hashtag `#TelAvivSurSeine`. Il s'appuie sur `tweepy`, un excellent client de l'API Twitter pour l'écosystème Python.<br />
J'ai pu collecter `76 698` tweets au format **JSON** sur une période allant du *03 Août 2015 10h49* au *12 Août 2015 08h03*, stockés dans des fichiers horodatés (`2015-08-*DD*_tweets.json`) et représantant plus de 800 Mo de données brutes.<br />
Chaque tweet collecté au format JSON contient une masse très importante d'informations que nous exploiterons avec l'utilitaire `jq`, l'équivalent de la commande `grep` adaptée au format JSON.<br />
La structure d'un tweet est décrite sur le [portail de dev de Twitter](https://dev.twitter.com/overview/api/tweets).

### Eléments statistiques sur l'ensemble de la période

Commençons par comptabiliser le nombre total de tweets ainsi que les comptes ayant tweeté :
{% highlight sh %}
~$ wc -l *.json
   76698 total
~$ jq '.user.screen_name' *.json | sort -u | wc -l
   16666
{% endhighlight %}

En 2 commandes on établit qu'il y a eu `76 698` tweets pour `16 666` comptes, soit une moyenne de **4,6 tweets par compte**. C'est un premier signe suggérant l'utilisation d'une technique de manipulation d'opinion, l'[astroturfing](https://fr.wikipedia.org/wiki/Astroturfing).

Voyons comment sont répartis les tweets par type de support :

{% highlight sh %}
$ cat *.json | jq ".source" | sed 's/"\(<.*>\)\(.*\)\(<.*>\)"/\2/' | sort | uniq -c | sort -rn | head -3
23210 Twitter for iPhone
21341 Twitter for Android
19954 Twitter Web Client
{% endhighlight %}

Nous en aurons besoin plus tard. Relevons au passage que l'activité est majoritairement mobile.

### Courbe des tweets et RT
Il est aisé de produire en une commande la série temporelle du nombre de tweets par tranche horaire. La seule petite difficulté consiste à convertir l'heure UTC du tweet en heure locale :
{% highlight sh %}
~$ cat *.json | jq ".created_at" | xargs -I ? date -j -f "%a %h %d %H:%M:%S %z %Y" "?" "+%Y-%m-%d_%H:00:00" | sort | uniq -c | sed -e 's/^ *//;s/ /,/' | awk -F"," '{ print $2 "," $1}' > data.csv
{% endhighlight %}
La représentation graphique est réalisée avec **GNU Plot** :
![Timeseries du hashtag #TelAvivSurSeine](/assets/article_images/TelAv_timeseries.png)
Ce qui interpelle de prime abord, c'est le **pic brutal** du *10 Août* vers *19h*.<br />

On peut mesurer l'influence d'un compte en comptabilisant le nombre de retweets.<br />
Etablissons le classement des comptes les plus retweetés le *10 Août* entre *18h00* et *19h59* (heure locale) :

{% highlight sh %}
~$ cat 2015-08-10_tweets.json | jq '.|select(.created_at|match("Mon Aug 10 1[6-7].*"))|.retweeted_status.user.screen_name' | grep -v "null" | sort | uniq -c | sort -rn | head -3
 337 "Campagnebds"
 114 "LePG"
 100 "Thalwen"
{% endhighlight %}
On constate que le compte `@Campagnebds` ressort très largement en tête.<br />
Ce pic peut donc être parfaitement corrélé avec l'annonce d'un [*tweet storm*](https://twitter.com/Campagnebds/status/630378884563906560) sur le compte de la campagne [BDS (*Boycott, Désinvestissement et Sanctions*)](https://fr.wikipedia.org/wiki/Boycott,_d%C3%A9sinvestissement_et_sanctions) :
<center>![](/assets/article_images/bds.png)</center>

A noter la présence en seconde position du compte du *Parti de Gauche* : la polémique prend largement une **tournure politique**. On ne reverra plus jamais le compte du PdG à une telle position dans le classement des RT.

Revenons au graphique. On ne constate pas de mouvement significatif entre le *03 Août*, date du premier tweet, et le *08 Août* peu avant midi.<br />
L'emballement commence le *08 Août* à partir de midi avec une pointe vers *14h* heure locale.<br />

De la même façon, identifions les 5 comptes les plus retweeté le *08 Août* entre *12h00* et *14h59* (heure locale)

{% highlight sh %}
~$ cat 2015-08-08_tweets.json | jq '.|select(.created_at|match("Sat Aug 08 1[0-2].*"))|.retweeted_status.user.screen_name' | grep -v "null" | sort | uniq -c | sort -rn | head -5
 214 "pauldraszen"
 213 "KarimaB_"
 155 "MadjidFalastine"
 131 "keyzer_usual"
 124 "Alkanz"
{% endhighlight %}

Effectuons la même opération mais en rajoutant le nombre d'abonnés à ces comptes et en formatant le résultat avec l'utilitaire `csvlook` :

{% highlight sh %}
~$ (echo "Nb_rtweet Nb_Followers Compte"; cat 2015-08-08_tweets.json | jq --raw-output '[.|select(.created_at|match("Sat Aug 08 1[0-2].*"))|.retweeted_status.user.followers_count,.retweeted_status.user.screen_name] | @csv' | tr ',' ' ' | sed '/^ *$/d;/^,$/d' | sort -k2 | uniq -cf1 | sort -k1rn | sed -e 's/^ *//' | head -5 ) | csvlook -d' '
|------------+--------------+------------------|
|  Nb_rtweet | Nb_Followers | Compte           |
|------------+--------------+------------------|
|  214       | 1199         | pauldraszen      |
|  213       | 13046        | KarimaB_         |
|  155       | 2748         | MadjidFalastine  |
|  131       | 7165         | keyzer_usual     |
|  124       | 32831        | Alkanz           |
|------------+--------------+------------------|
{% endhighlight %}

Tous ces comptes sont connus pour être très engagés dans la promotion de la cause palestinienne.<br />
Il me semble évident qu'ils ont été à l'avant garde de la montée du hashtag (premiers pics du graphique).

Si on élargit la recherche à l'ensemble de la journée, on retrouve d'autres comptes à forte audience, tous critiques à l'égard de l'évènement :

{% highlight sh %}
|------------+--------------+------------------|
|  Nb_rtweet | Nb_Followers | Compte           |
|------------+--------------+------------------|
|  625       | 12594        | ybenderbal       |
|  544       | 13045        | KarimaB_         |
|  477       | 1199         | pauldraszen      |
|  424       | 2748         | MadjidFalastine  |
|  374       | 22067        | Linformatrice    |
|  324       | 3047         | oxymorus         |
|  313       | 32831        | Alkanz           |
|  269       | 8426         | DionJack2        |
|  268       | 7265         | s_assbague       |
|  257       | 12765        | LeBjrTristesse   |
|------------+--------------+------------------|
{% endhighlight %}

Sur la seule journée du *08*, les retweets de ces 10 comptes représentent **près d'un quart** du nombre total de tweets (`3 875` sur `16 658`).

### Courbe des tweets et RT par canal
Nous avions vu plus haut que les principaux supports utilisés sont en moyenne, par ordre d'utilisation, l'*iPhone*, un terminal *Android* et un *navigateur web* non mobile.
La distribution des tweets par canal est intéressante :
![Courbe des tweets via iPhone](/assets/article_images/iphone.png)

![Courbe des tweets via Android](/assets/article_images/android.png)

![Courbe des tweets via un navigateur Web](/assets/article_images/web.png)

Les profils d'activité pour les canaux *iPhone*, *Android* et *web* sont assez proches.<br />
En superposant les courbes, l'activité *web* est toujours substantiellement en deçà des supports *iPhone* et *Android* **sauf durant le pic du *10 Août* vers 19h00**.

Cette sur-activité sur ce canal est peut-être induite par des comptes fictifs, mais il n'y a pas de certitude.

### Classement des *hashtags*
En comptabilisant les hashtags associés à `#telavivsurseine`, on peut dégager les sentiments dominants. <br />
La commande ci-dessous permets d'extraire les 20 mots croisillons les plus populaires :
{% highlight sh %}
~$ (echo "Nb,Hashtag"; cat *.json | jq ".entities.hashtags[].text" | tr '[A-Z]' '[a-z]' | tr '[àâäéèêëîïôöùûü]' '[aaaeeeeiioouuu]' | sort | uniq -c | sed -e 's/^ *//;s/ /,/' | sort -rn | head -20 ) | csvlook
|--------+-------------------------|
|  Nb    | Hashtag                 |
|--------+-------------------------|
|  77070 | telavivsurseine         |
|  5834  | apartheidsurseine       |
|  2880  | israel                  |
|  2767  | gaza                    |
|  2406  | paris                   |
|  1868  | againsttelavivsurseine  |
|  1860  | parisplages             |
|  1668  | contretelavivsurseine   |
|  1666  | telaviv                 |
|  1608  | gazasurseine            |
|  1606  | parislovestlv           |
|  1351  | palestine               |
|  843   | freepalestine           |
|  822   | bds                     |
|  787   | parisplage              |
|  621   | france                  |
|  515   | icc4israel              |
|  462   | hidalgo                 |
|  434   | apartheid               |
|  372   | boycottisrael           |
|--------+-------------------------|
{% endhighlight %}

### Classement des comptes les plus mentionnés
La commande ci-dessous permet d'établir les 20 comptes les plus mentionnés, ce qui permet de qualifier leur **réputation** :
{% highlight sh %}
~$ (echo "Nb,Compte"; cat *.json | jq ".entities.user_mentions[].screen_name" | sort | uniq -c | sed -e 's/^ *//;s/ /,/' | sort -rn | head -20 ) | csvlook
|-------+------------------|
|  Nb   | Compte           |
|-------+------------------|
|  8859 | Anne_Hidalgo     |
|  1772 | ybenderbal       |
|  1500 | BrunoJulliard    |
|  1380 | Paris            |
|  1317 | KarimaB_         |
|  1137 | ericcoquerel     |
|  1134 | Campagnebds      |
|  1130 | Linformatrice    |
|  1119 | Simonnet2        |
|  936  | jeremo12         |
|  909  | MadjidFalastine  |
|  884  | Alkanz           |
|  845  | Barakacity       |
|  826  | pauldraszen      |
|  808  | s_assbague       |
|  803  | oxymorus         |
|  748  | YASMINAREBEL     |
|  611  | keyzer_usual     |
|  611  | PKlugman         |
|  588  | itele            |
|-------+------------------|
{% endhighlight %}
On y trouve pèle mèle :

- des politiques
- des activistes déjà identifiés plus haut
- des comptes média généraliste et communautaires

### Classement des 20 comptes ayant été le plus retweetés

Le nombre de retweets est un indicateur **d'influence**. On retrouve dans ce classement les comptes des principaux protagonistes, où dominent les comptes d'obédience pro palestiniennes :
 
{% highlight sh %}
|--------+-----------------+---------------------------------|
|  Nb RT | Compte          | Date Creation                   |
|--------+-----------------+---------------------------------|
|  1759  | ybenderbal      | Sun Mar 11 22:51:26 +0000 2012  |
|  1085  | KarimaB_        | Sat Mar 07 14:36:25 +0000 2009  |
|  1054  | Linformatrice   | Thu Aug 04 02:18:52 +0000 2011  |
|  1045  | Campagnebds     | Tue Dec 15 20:33:31 +0000 2009  |
|  916   | jeremo12        | Sun Feb 26 22:54:54 +0000 2012  |
|  856   | MadjidFalastine | Sun May 10 17:44:54 +0000 2009  |
|  825   | Barakacity      | Sun Oct 30 17:08:53 +0000 2011  |
|  809   | pauldraszen     | Wed Apr 07 22:04:56 +0000 2010  |
|  795   | oxymorus        | Sat Jul 03 04:27:09 +0000 2010  |
|  738   | s_assbague      | Sun Jun 09 18:04:58 +0000 2013  |
|  717   | Alkanz          | Sat Jul 21 18:45:04 +0000 2007  |
|  648   | YASMINAREBEL    | Sat Jan 15 14:26:25 +0000 2011  |
|  611   | keyzer_usual    | Thu Jan 17 22:19:09 +0000 2013  |
|  472   | EladRatson      | Sun Feb 22 17:16:34 +0000 2009  |
|  468   | opFethi         | Sat Apr 13 22:53:43 +0000 2013  |
|  464   | mllesosso       | Mon Dec 22 10:15:10 +0000 2008  |
|  442   | pierrebleue3d   | Sat Nov 12 22:53:42 +0000 2011  |
|  441   | julienbahloul   | Sat Jan 09 08:40:16 +0000 2010  |
|  421   | Thalwen         | Sun Jul 15 17:09:19 +0000 2012  |
|  419   | ZohraBitan      | Thu Sep 23 16:13:46 +0000 2010  |
|--------+-----------------+---------------------------------|
{% endhighlight %}

### Classement des 20 comptes ayant le plus twitté
Ce tableau est intéressant :

{% highlight sh %}
|------------+--------------+-----------------+------------+---------------------------------|
|  Nb_tweets | Nb_Followers | Compte          | ID         | Date_creation                   |
|------------+--------------+-----------------+------------+---------------------------------|
|  3030      | 36           | Dajjalsystem    | 3126834263 | Sun Mar 29 16:24:44 +0000 2015  |
|  483       | 840          | ahenjar         | 2390165938 | Fri Mar 07 06:02:44 +0000 2014  |
|  437       | 1919         | amilcarhanniba1 | 2414559685 | Thu Mar 27 15:37:54 +0000 2014  |
|  357       | 560          | AgatheNRV       | 21016128   | Mon Feb 16 19:22:40 +0000 2009  |
|  328       | 13           | NataliaGGarcia5 | 3409050532 | Sat Aug 08 18:47:23 +0000 2015  |
|  323       | 926          | LILTIKKIB       | 407732116  | Tue Nov 08 13:22:29 +0000 2011  |
|  322       | 130          | yanisssou       | 117523080  | Thu Feb 25 20:42:53 +0000 2010  |
|  312       | 369          | mouadibun       | 2402247031 | Fri Mar 21 22:37:25 +0000 2014  |
|  279       | 89           | Mrelguevara     | 266072395  | Mon Mar 14 16:11:00 +0000 2011  |
|  260       | 4502         | blabalade       | 244170872  | Fri Jan 28 17:37:02 +0000 2011  |
|  235       | 1002         | fredussart      | 2864427501 | Thu Nov 06 20:46:09 +0000 2014  |
|  228       | 539          | RafikBiff       | 282921586  | Sat Apr 16 06:24:28 +0000 2011  |
|  217       | 1344         | opFethi         | 1350411432 | Sat Apr 13 22:53:43 +0000 2013  |
|  198       | 2206         | BenLevyCohen    | 194045521  | Thu Sep 23 09:02:08 +0000 2010  |
|  193       | 4938         | FinDesTemps     | 736880372  | Sat Aug 04 15:03:52 +0000 2012  |
|  188       | 13           | GozoGazi        | 3243703193 | Sat May 09 17:18:15 +0000 2015  |
|  179       | 1194         | Mars1300        | 573709142  | Mon May 07 15:00:56 +0000 2012  |
|  178       | 4257         | Aevalillith     | 242492168  | Mon Jan 24 22:30:44 +0000 2011  |
|  155       | 216          | Giiorgio100     | 2271358198 | Thu Jan 09 18:47:01 +0000 2014  |
|  151       | 637          | julievalles     | 618927738  | Tue Jun 26 07:52:22 +0000 2012  |
|------------+--------------+-----------------+------------+---------------------------------|
{% endhighlight %}

- il n'y a aucun compte connu ou réputé influent
- la majorité des comptes possède relativement peu de followers mais ils sont particulièrement volubiles
- en particulier, le compte `Dajjalsystem` intrigue par le nombre de tweets et RT (`3 030`) au regard de son très faible nombre d'abonnés (`36`)

Le qualificatif de *"compte pour crier"* utilisée par Nicolas dans son article prend là tout son sens.

L'API Twitter `followers/ids` est très utile pour reconstituer le réseau de relations de ces petits comptes avec les comptes d'activistes patentés.

Dans leur immense majorité, les comptes les plus volubiles sont liés à l'un des comptes ouvertement pro palestiniens :

{% highlight text %}
Dajjalsystem		 Linformatrice, Campagnebds, Alkanz, ybenderbal
ahenjar				KarimaB_, YASMINAREBEL, MadjidFalastine, Linformatrice, Campagnebds, Alkanz, pauldraszen, s_assbague
amilcarhanniba1		Campagnebds, ybenderbal, Barakacity
AgatheNRV			YASMINAREBEL, MadjidFalastine, Campagnebds, Alkanz, ybenderbal, pauldraszen, s_assbague
NataliaGGarcia5		Campagnebds
LILTIKKIB			KarimaB_, keyzer_usual, Linformatrice, oxymorus, Campagnebds, Alkanz, ybenderbal, Barakacity, s_assbague
yanisssou			KarimaB_, MadjidFalastine, Linformatrice, Campagnebds, Alkanz, ybenderbal, Barakacity, pauldraszen, s_assbague
mouadibun			YASMINAREBEL, keyzer_usual, MadjidFalastine, Linformatrice, oxymorus, Campagnebds, Alkanz, Barakacity, pauldraszen, s_assbague
Mrelguevara
blabalade			Campagnebds
fredussart			jeremo12
RafikBiff			KarimaB_, Linformatrice, Campagnebds, Alkanz, ybenderbal, Barakacity
opFethi				KarimaB_, keyzer_usual, MadjidFalastine, Linformatrice, Alkanz, ybenderbal
BenLevyCohen		jeremo12
FinDesTemps	
GozoGazi			KarimaB_, oxymorus
Mars1300	
Aevalillith			KarimaB_, MadjidFalastine, Linformatrice, Campagnebds, Alkanz, ybenderbal, Barakacity, pauldraszen, s_assbague
Giiorgio100			jeremo12
julievalles			KarimaB_, MadjidFalastine, ybenderbal, pauldraszen, s_assbague
{% endhighlight %}

En définitive, cette polémique est une **tempête dans un verre d'eau** qui n'aurait jamais dû sortir des réseaux sociaux ni des cercles militants.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
