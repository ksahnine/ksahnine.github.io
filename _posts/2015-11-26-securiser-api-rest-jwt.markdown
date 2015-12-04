---
layout: post
title:  "Sécuriser une API REST avec JSON Web Token (JWT)"
date:   2015-11-26 06:00:00
categories: rest security jwt
tags: regular
image: /assets/article_images/alan-turing.jpg
legend: Statue d'Alan Turing à Manchester. Alan Turing est aussi un des pères de la cryptographie moderne.
---

Mon [article consacré la spécification **CORS**](http://blog.inovia-conseil.fr/?p=202) a eu un certain succès d’audience. Il me semble opportun de le compléter avec un nouvel article traitant de la sécurisation d’une API REST.
J’ai choisi d’aborder [**JSON Web Token**](http://jwt.io/) (**JWT**, que les anglophones prononcent *jot*), un [standard ouvert](https://tools.ietf.org/html/rfc7519) permettant à deux parties d’échanger de manière sûre des informations encapsulées dans un **jeton signé numériquement**.
En pratique, JWT est utilisé pour mettre en oeuvre des solutions d’**authentification SSO** ou de **sécurisation de web services**.

Bien que le [**protocole OAuth 2**](http://oauth.net/2/) soit très utilisé par des plateformes à forte audience exposant une API web, JWT apparait dans beaucoup de cas d’utilisation comme une alternative intéressante car beaucoup **plus simple** à mettre en oeuvre et **stateless** (ce qui rend la solution scalable).

Après avoir décrit la structure et l’utilisation d’un *“jot“*, nous verrons en pratique comment sécuriser une API REST construite sur **Node.js** et le service de gestion d’identité [**Stormpath**](https://stormpath.com/). Le code source accompagnant cet article est [disponible sur mon dépôt GitHub](https://github.com/ksahnine/jwt-secured-api-node).

# Structure d'un jeton JWT
Un jeton JWT est une chaîne de caractères décomposable en 3 sections séparées par un point (`.`).
<center>![Jeton JWT](/assets/article_images/jeton_jwt.png)</center>

- **En-tête** : c’est un document au format JSON, **encodé en base 64** et contenant des méta-données. Il doit contenir au minimum le type de jeton et l’algorithme de chiffrement utilisé pour le signer numériquement. <br/>Exemple :<br />
{% highlight text %}
{
  "typ": "JWT",
  "alg": "HS256"
}
{% endhighlight %}
- **Charge utile** : cette section est un document au format JSON **encodé en base 64**, contenant des données fonctionnelles minimales que l’on souhaite transmettre au service (ces propriétés sont appelées *claims* ou *revendications* selon la terminologie de la RFC). En pratique, on y fait transiter des informations sur l’identité de l’utilisateur (login, nom complet, rôles, etc.). **Il ne doit pas contenir de données sensibles**. Voici un exemple :<br/>
{% highlight text %}
{
  "iat": 1448383708,
  "exp": 1448383768,
  "name": "compte de test",
  "username": "demo",
  "email": "demo@foobar.com"
}
{% endhighlight %}
- A noter que l’on distingue 3 types de *claims* :
  - *claims* **réservés** : il s’agit de noms [réservés par la spécification](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#RegisteredClaimName) et ne pouvant être utilisés par le développeur. Par exemple, `iat` contient la date de génération du jeton et `exp` sa date d’expiration.
  - *claims* **publics** : il s’agit de noms normalisés dont on encourage l’utilisation (ex. `name`, `email`, `phone_number`). Le registre est maintenu par l’organisation *IANA* et est [consultable sur leur site](http://www.iana.org/assignments/jwt/jwt.xhtml).
  - *claims* **privés** : il s’agit de noms à usage privé pour répondre à des besoins spécifiques à vos applications. Ils ne doivent pas entrer en conflit avec les autres types de *claims*.
- **Signature** : cette zone contient la signature numérique du jeton. La clé privée utilisée pour signer le jeton est stockée côté serveur.

# Fonctionnement et étude de cas
De par son format compact et sa nature stateless (le jeton n’est pas stocké dans une base de données), JWT est très adapté aux transactions HTTP.<br/>
Ainsi, dans la requête d’accès à une ressource protégée, le jeton est véhiculé dans l’en-tête `Authorization` avec le mécanisme d’authentification `Bearer` :
{% highlight text %}
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImRlbW8iLCJmdWxsTmFtZSI6
{% endhighlight %}

Le schéma ci-dessous représente un dialogue entre un client (navigateur ou autre) et une API REST (et CORS compatible) exposant 2 services :
- un service d’authentification : `POST /api/authenticate`
- un service à accès restreint retournant une liste de comptes : `GET /api/restricted/accounts`

<center>![Cinématique JWT](/assets/article_images/cinematique_jwt.png)</center>

- dans un premier temps, le client cherche à accéder à une ressource protégée sans utiliser de jeton. Le service retourne une erreur avec un code **HTTP 401** (*Unauthorized*)
- dans un deuxième temps, le client **s’authentifie**. Le service vérifie que les vecteurs d’accréditation sont corrects, **génère un jeton JWT avec une durée de vie prédéfinie** puis retourne la réponse sous la forme d’un document JSON contenant le jeton (attribut `token`).
- muni de ce jeton, le client accède à nouveau à la ressource protégée en le **propageant dans l’entête** `Authorization`. Le service vérifie que le jeton est effectivement valide puis retourne la liste des comptes.<br/>A noter que si cette requête avait été émise **au delà de la durée d’expiration du jeton**, le service retournerait une erreur **HTTP 401** (jeton invalide).

Le code source de cette API ainsi que les instructions d’installation [sont disponibles sur mon dépôt GitHub](https://github.com/ksahnine/jwt-secured-api-node), les comptes utilisateur étant stockés dans [**Stormpath**](https://stormpath.com/).

## Installation et utilisation de l’API
Le mode opératoire d’installation est le suivant (testé sous *Node.js* 0.12) :
{% highlight sh %}
git clone https://github.com/ksahnine/jwt-secured-api-node.git
cd jwt-secured-api-node
npm install
{% endhighlight %}
Pour démarrer le conteneur de services, exécuter la commande :
{% highlight sh %}
npm start
{% endhighlight %}

Voici quelques cas d’utilisation de l’API via l’utilitaire **cURL** :

- **Cas 1** : accès à une ressource protégée sans jeton :
{% highlight sh %}
$ curl http://localhost:8000/api/restricted/accounts
{
  "status": "error",
  "msg": "Jeton invalide",
  "url": "/api/restricted/accounts"
}
{% endhighlight %}
- **Cas 2** : authentification avec récupération d’un jeton :
{% highlight sh %}
$ curl -X POST --data "username=demo1&password=Demo2015" http://localhost:8000/api/authenticate
{
  "status": "ok",
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImRlbW8xIiwiZn"
}
{% endhighlight %}
- **Cas 3** : accès à une ressource protégée avec un jeton valide :
{% highlight sh %}
$ curl -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImRlbW8xIiwiZn" \
        http://localhost:8000/api/restricted/accounts
[
  {
    "id": 0,
    "name": "Account 0"
  },
  {
    "id": 1,
    "name": "Account 1"
  }
]
{% endhighlight %}

## Description de la pile technique
L’[API](https://github.com/ksahnine/jwt-secured-api-node) est construite au dessus de [**Node.js**](https://nodejs.org/) et s’appuie les modules suivants :
- le framework web [**Express**](https://github.com/strongloop/express) pour l’implémentation des services REST/JSON et la gestion du routage
- le module *node* [**cors**](https://github.com/expressjs/cors) pour le support CORS (compatible avec Express)
- le middleware [**Passport**](http://passportjs.org/) pour la gestion de l’authentification sous *Node.js*, ainsi que le plugin [**passport-stormpath**](https://github.com/stormpath/passport-stormpath) pour l’accès au gestionnaire d’identité **Stormpath**
- le module [**jsonwebtoken**](https://github.com/auth0/node-jsonwebtoken) pour générer et signer numériquement les jetons JWT
- le module *Express* [**express-jwt**](https://github.com/auth0/express-jwt) pour valider les jetons JWT lors de l’accès aux ressources protégées

> **A l’attention du développeur AngularJS** : vous n’êtes pas sans savoir que la spécification CORS interdit d’utiliser conjointement l’en-tête `Access-Control-Allow-Origin: *` et la propriété XHR `withCredentials: true`. Dans cette configuration, il n’est par exemple pas possible de propager un cookie de session. Dans le cas d’espèce, la solution adaptée serait de générer une entête `Access-Control-Allow-Origin` dynamique mais JWT présente une alternative bien plus intéressante. Nous y reviendrons dans un prochain article.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
