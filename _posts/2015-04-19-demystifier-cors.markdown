---
layout: post
title:  "Démystifier CORS"
date:   2015-04-19 06:00:00
categories: architecture rest cors
tags: regular
image: /assets/article_images/tim-berners-lee.jpg
legend: Tim Berners-Lee, inventeur du World Wide Web et fondateur du W3C
---
J’ai eu souvent l’occasion d’écrire ici que les grands acteurs du Web sont plus que jamais les **nouveaux prescripteurs de solutions** pour le SI.

Même si l’organisme **W3C** agit comme une puissance normative, seules les spécifications validées par l’expérience des grands carrefours d’audience bâtis par *Google*, *Amazon*, *Facebook* ou *Twitter* deviennent des normes effectives. Le feu de l’expérience fait office de contreseing, préalable à l’adoption dans les Systèmes d’Information d’Entreprise.

[**CORS**](http://www.w3.org/TR/cors/) (pour Cross-Origin Resource Sharing) fait partie de ces spécifications largement supportées par les grands du Web et appelées à se généraliser au gré du déploiement grandissant des navigateurs de dernière génération.

## Le futur de JSONP
Depuis des années, les navigateurs restreignent l’accès à des ressources n’appartenant pas au domaine depuis lequel un document a été chargé (règle dite de [**Same-Origin Policy**](http://en.wikipedia.org/wiki/Same-origin_policy)).<br />
Le fait que la balise `<script />` ne soit pas concernée par cette restriction a ouvert la voie à la technique [**JSONP**](http://en.wikipedia.org/wiki/JSONP) (*JSON with padding*), permettant d’émettre des requêtes AJAX cross-domain retournant des données JSON encapsulées dans une fonction de rappel (*callback*) en JavaScript. Il faut admettre cependant que cette technique, bien qu’encore très utilisée, relève surtout de l’astuce.

**CORS** est un mécanisme robuste et [normalisé par le W3C](http://www.w3.org/TR/cors/) permettant à tout navigateur compatible d’effectuer des **requêtes HTTP cross-domain**. Il a valeur de standard industriel et étend le champ des possibilités bien au delà de ce qui est actuellement permis par JSONP.

JSONP est très pratique pour partager des données publiques (comprendre non sensibles).<br />
Néanmoins, cette technique ne supporte pas les méthodes `POST`, `PUT`, `DELETE` (seule la méthode `GET` est supportée). Elle n’est donc à priori pas adaptée pour exposer une API JSON sauf à mettre en place des solutions de contournement (méthode HTTP transmise via un paramètre de requête, contrôles d’accès, etc.).

## Une affaire d'entêtes HTTP
En pratique, le mécanisme CORS est fondé sur l’utilisation d’**entêtes HTTP personnalisés** définis par la [spécification](http://www.w3.org/TR/cors/) :

- [entêtes de requête](http://www.w3.org/TR/cors/#origin-request-header) :
  - `Origin`
  - `Access-Control-Request-Method`
  - `Access-Control-Request-Headers`
- [entêtes de réponse](http://www.w3.org/TR/cors/#access-control-allow-origin-response-header) :
  - `Access-Control-Allow-Origin`
  - `Access-Control-Allow-Credentials`
  - `Access-Control-Expose-Headers`
  - `Access-Control-Max-Age`
  - `Access-Control-Allow-Methods`
  - `Access-Control-Allow-Headers`

Ils permettent au navigateur et au serveur d’en savoir suffisamment l’un sur l’autre afin de déterminer si la requête ou la réponse *cross-domain* doit réussir ou échouer.

Pour fonctionner, le mécanisme nécessite :

- un navigateur supportant cette technologie :
  - Internet Explorer 10+
  - Firefox 3.5+
  - Chrome 3+
  - Safari 4+ 
- une partie serveur capable de produire le ou les entêtes HTTP idoines

> **Note :** IE 8 et 9 ont un support partiel de CORS. Retenir en particulier que :

> - les méthodes HTTP `PUT` et `DELETE` ne sont pas supportées
> - les requêtes à entête HTTP `Content-Type` ne peuvent contenir que la valeur `text/plain`
> - les requêtes émises ne peuvent pas contenir de cookies

## Exemple
Le code ci-dessous permet d’afficher les vignettes des 25 vidéos Youtube les plus populaires :
{% gist 9cfca5492994fbf924bd %}

La page HTML est servie par le serveur `inoviabook.local`, tandis que les données sont issues du serveur `gdata.youtube.com` :
<center><img src="{{site.url}}/assets/article_images/arch-cors.png" style="display: block; margin: auto;" /></center>

La requête AJAX échouerait si le navigateur ne supportait pas CORS. Comment la magie opère-t-elle ? Le mystère se dissipe en analysant la requête / réponse AJAX vers les serveurs de Youtube :

<center><img src="{{site.url}}/assets/article_images/req-1.png" style="display: block; margin: auto;" /></center>

> Le *navigateur* ajoute l’entête `Origin` avant d’émettre la requête cross-domain. Il contient le schéma HTTP, le serveur et le port de provenance (`http://inoviabook.local:8000`) de la page (`youtube.html`) à l’initiative de l’appel CORS.
> Pour des raisons de sécurité, seul le navigateur est responsable de la valorisation de cet entête HTTP.

<center><img src="{{site.url}}/assets/article_images/resp-1.png" style="display: block; margin: auto;" /></center>

> La réponse du **serveur** contient l’entête `Access-Control-Allow-Origin` permettant d’indiquer au client qu’il est bien habilité à obtenir une réponse valide. La requête échouerait si l’entête n’était pas présent ou s’il contenait une valeur **ne concordant pas** avec la valeur de l’entête `Origin`.

Quelques observations :

- Le moyen de plus simple d’avoir un serveur avec support CORS unilatéral serait d’ajouter l’entête `Access-Control-Allow-Origin: *` dans toutes les réponses HTTP.
- La requête de l’exemple ci-dessus échouerait si `Access-Control-Allow-Origin` valait `http://inoviabook.local:80` car le numéro de port ne concorderait pas avec celui de l’entête `Origin` (`8000`).
- La requête de l’exemple ci-dessus échouerait si `Access-Control-Allow-Origin` valait `https://inoviabook.local:8000` car le schéma HTTPS ne concorderait pas avec celui de l’entête `Origin` (`http`).
- La requête fonctionnerait si `Access-Control-Allow-Origin` valait `http://*.local:8000`
car la valeur concorderait avec l’entête `Origin` (`http://inoviabook.local:8000`).

## Les requêtes de contrôle préliminaires (preflight requests)
Un navigateur supportant CORS peut, dans certaines conditions explicitées ci-après, envoyer une **requête de contrôle préliminaire** (*preflight request*) afin de demander la permission au serveur d’envoyer la requête réelle. Si le serveur ne répond pas correctement à la requête préliminaire, le navigateur n’enverra pas la requête réelle et l’objet `XMLHttpRequest` remontera une erreur.

Une requête préliminaire est automatiquement émise par le navigateur dans les 2 cas de figure suivants :

- la requête utilise une méthode HTTP différente de `GET`, `POST` ou `HEAD`
- la requête contient un entête HTTP `Content-Type` dont la valeur est différente de :
  - `text/plain`
  - `multipart/form-data`
  - `application/x-www-form-urlencoded`

Si nous modifions légèrement le code de notre exemple en renseignant l’entête `Content-Type` à `application/json` (rajouter `contentType: 'application/json'` aux settings de la fonction `$.ajax()`), le navigateur envoie la requête préliminaire suivante :
<pre>
**OPTIONS** /feeds/api/standardfeeds/most_popular?v=2&alt=json HTTP/1.1
Host: gdata.youtube.com
origin: http://inoviabook.local:8000
**access-control-request-method:** GET
**access-control-request-headers:** accept, content-type
…
</pre>

> On notera qu’une requête préliminaire utilise la méthode HTTP `OPTIONS`.
> Remarquez l’utilisation des entêtes de requête `access-control-request-method` et `access-control-request-headers`. Par ce biais, le navigateur demande la permission au serveur s’il est bien autorisé à utiliser respectivement la méthode HTTP `GET` et les requêtes HTTP `Accept` et `Content-Type` contenus dans la requête réelle.

La réponse (correcte) du serveur est la suivante :
<pre>
HTTP/1.1 200 OK
X-GData-User-Country: FR
Access-Control-Allow-Origin: http://inoviabook.local:8000
**Access-Control-Allow-Headers:** accept,content-type
**Access-Control-Allow-Methods:** DELETE,GET,HEAD,PATCH,POST,PUT
Content-Type: text/plain
…
</pre>

> On remarquera la présence des entêtes `Access-Control-Allow-Headers` et `Access-Control-Allow-Methods` contenant respectivement la liste des entêtes et méthodes HTTP permises par le serveur.
> Leurs valeurs concordent avec celles des entêtes de requête `Access-Control-Request-Method` et `Access-Control-Request-Headers`. La réponse vaut donc autorisation pour le navigateur d’émettre la requête réelle, celle retournant la liste des 25 vidéos Youtube les plus populaires.

<center><img src="{{site.url}}/assets/article_images/cors-preflight.png" style="display: block; margin: auto;" /></center>

Les requêtes de contrôle peuvent poser des **problèmes de performance** car elles ont pour conséquence de doubler le nombre de *hits* vers le serveur. <br />
Pour limiter le nombre de requêtes préliminaires, le serveur peut forcer le navigateur à mettre en cache la requête en rajoutant l’entête HTTP `Access-Control-Max-Age` dans la réponse.<br />
Voici un exemple pour une mise en cache pendant 1 heure :
{% highlight sh %}
Access-Control-Max-Age: 3600
{% endhighlight %}

## Cookies et authentification
Par défaut, les requêtes cross-domain interdisent la propagation de cookies, de vecteurs d’accréditation ou de certificats SSL client.<br />
Pour lever cette interdiction, il faut opérer quelques modifications côté client et serveur.

- _côté client_ : renseigner la propriété `withCredentials` de l’objet `XMLHttpRequest` à `true`.

> Sous jQuery, modifier les settings de la fonction `$.ajax()` en modifiant la propriété `xhrFields` :
{% highlight json %}
xhrFields: {
    withCredentials: true
}
{% endhighlight %}

- _côté serveur_ : renseigner l’entête HTTP `Access-Control-Allow-Credentials` à `true` :

{% highlight sh %}
Access-Control-Allow-Credentials: true
{% endhighlight %}

## Exposer les entêtes personnalisées des réponses
CORS est assez rigide dans l’utilisation des entêtes HTTP personnalisées.<br />
Imaginons que votre API soit conçue pour retourner le numéro de version détaillée d’un service (ou tout autre méta-donnée utile fonctionnellement) dans l’entête personnalisées `X-App-Version`. <br />
Sachez que le contenu de l’entête **ne pourra pas être lue par le client sans une autorisation explicite du serveur**.<br />
Concrètement, le serveur doit préciser dans chaque réponse la liste des entêtes personnalisées exposées au client via l’entête `Access-Control-Expose-Headers`.<br />
Dans notre cas de figure, le serveur devra retourner :
{% highlight sh %}
Access-Control-Expose-Headers: X-App-Version
{% endhighlight %}

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
