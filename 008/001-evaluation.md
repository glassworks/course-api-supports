# Evaluation

Nous sommes en train de créer une API pour le système d'hébergement d'un magasin de location de vidéos.

Vous pouvez trouver le schéma de la base de données et des exemples de données ici : [SAKILA](https://dev.glassworks.tech/courses/sgbdr/sgbdr-supports/-/tree/main/src/samples/sakila)

Démontrez vos compétences dans la création d'une API avec la spécification suivante :

- Commencez par implémenter l'API exactement comme documenté dans ces notes, mais en remplaçant la base de données `school` par `sakila`. Notez que vous devrez mettre à jour vos types et schémas en conséquence !
- Une API JSON REST-ful :
  - avec toutes les routes CRUD pour la création d'un film : toutes les routes doivent avoir les bonnes actions HTTP et les URI qui adhèrent au standard REST
  - avec des routes pour lister tous les acteurs d'un film, et pour obtenir un seul acteur d'un film particulier
  - la possibilité de charger une image de couverture de film et de télécharger l'image du film (vous pouvez utiliser mes informations d'identification).  
- Sécurisez votre API REST :
  - les routes ne peuvent être accédées que si elles sont identifiées et authentifiées
  - seuls les utilisateurs « administrateurs » connectés peuvent ajouter, mettre à jour ou supprimer des films. Implémentez le RBAC (contrôle d'accès basé sur les rôles) en utilisant `scopes` dans `tsoa` [Authentification](https://tsoa-community.github.io/docs/authentication.html)
- Fournir une route GraphQL publique qui liste les films et leurs acteurs.

L'API doit être démontrée en démarrant une image Docker devant l'intervenant, et en appelant les différents points de terminaison soit dans un navigateur, soit avec Postman.


{% hint style="danger" %}
Vous ne pouvez pas utiliser de framework tiers comme Nest, Next, Nuxt, Sympfony, etc. Votre projet doit être écrit en Typescript, doit utiliser MariaDB, et doit généralement adhérer aux principes décrits dans ces notes !
{% endhint %}

## Livraison

Vous devez faire la démonstration de votre API en démarrant votre API en tant que conteneur Docker préconstruit sur votre machine locale. L'intervenant vous demandera alors de démontrer chaque fonctionnalité.

La notation est incrémentale, vous pouvez gagner des points progressivement tout au long de la semaine, n'attendez donc pas la dernière minute. Une fois que vous avez réalisé une fonctionnalité qui peut vous rapporter des points, n'hésitez pas à la montrer à l'intervenant pour obtenir vos points !

Vos démonstrations doivent être réalisées avant 17h00 à la fin de cette semaine.

## Notation

La notation suivante sera utilisée :

| Élément | Score |
|--|--|
| Conteneur Docker fonctionnel | 1 |
| `/info` fonctionne, avec une connexion à la base de données | 1 |
| Routes CRUD REST-ful pour les films (+ docs) | 5 |
| Pagination sur la liste des films | 2 |
| Routes REST-ful pour les acteurs de chaque filme | 2 |
| Téléchargement d'images de couvertures de films | 2 |
| GraphQL pour les films et les acteurs | 2 |
| Sécurité et utilisation correct des JWT | 2 |
| Getion du refresh-token | 1 |
| RBAC | 2 |


Tous les points seront évalués en prenant en compte l'ensemble des notions dont ont avait parlé pendant le cours :

- est-ce qu'il y a une documentation claire et disponible (et automatique) ?
- est-ce qu'il y a des logs informatifs et pertinents ?
- est-ce qu'on a optimisé en fonction du temps, poids, utilisations des ressources (RAM, bande-passante, ...)
- est-ce que l'API est "stateless" ?
- ...



