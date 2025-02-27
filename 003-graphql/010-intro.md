# GraphQL

En 2015, une alternative à la méthode REST, appelée GraphQL, a été proposée pour simplifier et harmoniser la manière dont nous accédons aux données.

Comme vous l'avez vu dans les sections précédentes, nous avons dû déployer beaucoup d'efforts pour typer nos requêtes et nos réponses, et fournir une documentation suffisante pour les différentes URI.

De plus, REST est très lié à HTTP, empruntant les notions d'actions et d'URI.

Enfin, il est parfois difficile de représenter les relations entre les entités à l'aide de REST, ce qui conduit à des URI complexes et parfois à de multiples appels à l'API REST pour obtenir toutes les données requises. 

GraphQL est une alternative à REST qui harmonise le processus demande-réponse. Elle vise à permettre aux utilisateurs de spécifier la structure dans laquelle nous voulons nos données, et les données renvoyées respecteront cette structure. Bien qu'il fonctionne sur HTTP, il n'est pas dépendant des URI ni des actions http.

