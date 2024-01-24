# REST

REST, ou representational state transfer, est une technique est un style d'architecture logicielle qui a été créé pour guider la conception et le développement de l'architecture du World Wide Web.

En termes simples, chaque entité d'un système peut être identifiée comme une **ressource** qui possède sa propre adresse unique, son URI (uniform resource locator).

On parle parfois de **endpoints**

Par exemple, un utilisateur spécifique sur une plateforme peut avoir l'URI suivant :

```
/user/12345
```

Dans une implémentation HTTP de REST, nous pouvons :
- **créer** un nouvel utilisateur à l'aide d'une méthode `PUT`.
- **lire** les données associées à cet utilisateur via une méthode `GET`.
- **mettre à jour** les données via une méthode `PATCH`.
- **supprimer** les données via une méthode `DELETE`.

(Cette gamme d'opérations est communément appelée **CRUD** et est largement utilisée dans de nombreuses API).

Les implémentations REST présentent généralement les caractéristiques suivantes :
- architecture client-serveur 
- apatridie : toutes les demandes sont indépendantes les unes des autres et peuvent être comprises de manière isolée
- interface uniforme : respect des normes et conventions du secteur pour la création, la lecture, la mise à jour et la suppression de données, ainsi que pour l'authentification des utilisateurs, etc.