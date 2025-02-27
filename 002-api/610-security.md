# Sécurité

Normalement, dans un API fermé, nous allons empêcher l'accès aux utilisateurs qui n'ont pas le droit d'y accéder.

Comme pour la sécurisation d'une installation Linux, il y a plusieurs facettes à la sécurisation d'un API :

* **identité** : Prouvez que vous êtes bien celui que vous prétendez être
* **autorisation** : Ne donnez accès qu'aux ressources auxquelles vous avez droit et pas plus.

## Identité

Avant de se lancer dans la sécurisation d'un API il faut identifier d'abord **QUI** aurait accès à votre API.

Surtout, comment notre API va être capable d'identifier un utilisateur précis, et être 100% certain que c'est bien elle qui fait des requêtes.

Aujourd'hui, il existe plusieurs façons (flux ou **identity flows**) pour établir et vérifier l'identité d'un utilisateur.

Le principe distillé est le suivant :

1. à la première connexion, on demande à l'utilisateur de s'identifier par une valeur unique. Normalement, c'est une adresse e-mail ou nom d'utilisateur qu'il aurait choisi (ou qui lui a été attribué) à la création de son compte
2. Ensuite, l'utilisateur est demandé de **prouver** son identité via une valeur ou secret **seulement à la disposition de cet utilisateur**
3. L'API crée un jeton unique valable uniquement pendant la séance, et on passe ce jeton à l'utilisateur (via un **cookie**, par exemple)
4. Le client de l'utilisateur doit désormais fournir ce jeton avec chaque requête
5. À la réception d'une requête, l'API doit valider le jeton, et rejeter la requête si le jeton n'est pas présent ou n'est pas valable, ou bien si l'utilisateur n'a pas le droit d'accéder à la ressource demandée

### Prouver son identité

Comment l'utilisateur peut prouver son identité ?

* Un mot de passe connu uniquement par l'utilisateur
* Une valeur biométrique physique de l'utilisateur (emprunt digital, iris, identification visage, etc.)

La solution d'un mot de passe est compliquée pour plusieurs raisons :

* Un mot de passe trop simple est facile à deviner grâce aux algorithmes simples (attaque en force brute)
* Un utilisateur a tendance à oublier son mot de passe complexe. Il réutilise donc le même mot de passe partout
* Nous sommes obligés de stocker le mot de passe dans notre base de données.
  * Si on est hacké, on divulgue le mot de passe de nos utilisateurs au hackeur.
  * À ce moment-là, le hackeur aurait potentiellement accès à tous les comptes utilisant le mot de passe partagé

La solution des biométriques est compliquée aussi :

* On a besoin d'un périphérique spécial (lecteur d'emprunt digital)
* Sinon, certaines méthodes ne sont pas totalement fiables (détection faciale) et pourraient engendrer des faux positifs ou des faux négatifs

### Identité par délégation

Une autre stratégie pour prouver l'identité est de dépendre sur un tiers.

Par exemple, si on utilise l'adresse e-mail de la personne, on peut supposer que seulement cette personne aura accès à sa boîte mail.

Pour prouver son identité, on pourrait juste forcer cet utilisateur à prouver qu'il a accès aussi à sa boîte e-mail - en l'envoyant un message à cette adresse.

Normalement, on envoie un émail avec un code ou lien unique. Si l'utilisateur peut nous répéter ce code unique, il prouve qu'il a pu consulter son mail.

Pour nous, cela veut dire qu'on ne stocke plus son mot de passe dans notre base de données ! On dépend du mot de passe utilisé pour protéger son compte email :

* C'est 1 mot de passe en moins à mémoriser pour l'utilisateur
* En revanche, si le mot de passe de sa boîte email est volée, le voleur aura accès à notre service aussi

{% hint style="info" %}
Ce flux pourrait être adapté autrement : par envoi d'un SMS par exemple.
{% endhint %}

Un autre type d'identité par délégation existe aujourd'hui grâce à la norme [OAuth2](https://oauth.net/2/). L'idée de base est qu'un service centralisé s'occupe de l'identification d'un utilisateur, et crée des jetons d'accès utilisables par notre API. On en a tous utilisé :

* connexion avec votre identifiant Google
* connexion avec votre identifiant Facebook
* connexion avec votre identifiant GitHub

En revanche, cela force nos utilisateurs d'avoir un compte chez un tiers avant d'utiliser notre service (parfois pas idéal). Et, aussi, s'il y a une fuite chez un de ces services, notre API sera à risque aussi.

### La double authentification

Pour encore sécuriser nos services aujourd'hui, la tendance est d'aller vers une combinaison des différentes approches :

* Je fournis mon adresse e-mail et mot de passe
* Si le mot de passe est validé, l'API envoie un message avec un code unique à l'adresse email
* l'utilisateur doit répéter le code unique trouvé dans la boîte mail
* le jeton est créé
* ... etc

Même si le premier mot de passe est divulgué, on a une autre couche de protection via le mot de passe de la boîte mail.

## Autorisation

Une fois qu'un utilisateur a été identifié et vérifié, il peut accéder aux ressources. Cependant, tous les utilisateurs ne sont pas identiques. Un étudiant n'aura pas les mêmes droits qu'un administrateur, par exemple.

Nous avons besoin d'un moyen d'accorder ou de refuser l'accès aux routes de notre API en fonction du **rôle** de l'utilisateur. C'est ce que nous appelons le **RBAC**, ou contrôle d'accès basé sur les rôles (role-based access control).

En règle générale, lorsqu'un utilisateur envoie une requête à une route, nous implémentons un logiciel intermédiaire en amont de cette route qui :

1. vérifie si l'utilisateur a été identifié (erreur HTTP `401` si ce n'est pas le cas)
2. extrait le rôle de cet utilisateur
3. vérifie si le rôle a accès à la route demandée (erreur HTTP `403` si ce n'est pas le cas)

Pour l'étape 1, nous demandons à l'utilisateur d'envoyer son jeton d'identification en tant qu'en-tête de la demande (typiquement l'en-tête `Authorization`). Si ce jeton est manquant ou invalide, une erreur est renvoyée.

L'étape 2 peut être réalisée de deux manières différentes :
- Nous recherchons l'utilisateur, puis son rôle dans la base de données. Cela permet d'obtenir des informations actualisées sur ses droits d'accès, mais nécessite de consulter la base de données à chaque demande, ce qui peut s'avérer coûteux.
- Les rôles de l'utilisateur sont encodés directement dans le jeton d'identification (ou d'accès). Les rôles peuvent être directement extraits du jeton sans consultation de la base de données.

Pour l'étape 3, chaque itinéraire doit disposer d'un logiciel intermédiaire qui peut alors comparer les rôles de l'utilisateur avec les rôles requis pour l'itinéraire. Si aucune correspondance n'est trouvée, une erreur est renvoyée (typiquement `403`).