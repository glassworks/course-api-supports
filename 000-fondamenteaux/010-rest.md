# REST

REST, ou representational state transfer, est une technique est un style d'architecture logicielle qui a été créé pour guider la conception et le développement de l'architecture du World Wide Web.

En termes simples, chaque entité d'un système peut être identifiée comme une **ressource** qui possède sa propre adresse unique, son URI (**uniform resource identifier**).

On parle parfois de **endpoints**

Par exemple, un utilisateur spécifique sur une plateforme peut avoir l'URI suivant :

```
/user/12345
```

Il est important de noter que cet URI identifie l'utilisateur `12345` **de manière unique**. Nous ne devrions jamais recevoir d'autres utilisateurs à cette adresse. En suivant les notions de REST, en utilisant ce même URI, nous pouvons alors effectuer un certain nombre d'opérations différentes :
- **créer** un nouvel utilisateur à l'aide d'une méthode `PUT`.
```
PUT /user/12345  HTTP/1.1
Content-Type: application/json
Accept: application/json 

{
  "givenName": "Kevin",
  "familyName": "Glass"
}
```
- **lire** les données associées à cet utilisateur via une méthode `GET`.
```
GET /user/12345  HTTP/1.1
Content-Type: application/json
Accept: application/json 
```
- **mettre à jour** les données via une méthode `PATCH`.
```
PATCH /user/12345  HTTP/1.1
Content-Type: application/json
Accept: application/json 

{
  "givenName": "Robert",  
}
```
- **supprimer** les données via une méthode `DELETE`.
```
DELETE /user/12345  HTTP/1.1
Content-Type: application/json
Accept: application/json 
```

Cette gamme d'opérations est communément appelée **CRUD** et est largement utilisée dans de nombreuses API: 
- **C**reate
- **R**ead
- **U**pdate
- **D**elete.

Que vous écriviez une API REST, que vous utilisiez des alternatives telles que GraphQL, ou même que vous écriviez une interface dans votre logiciel, vous pouvez utiliser la notion de **CRUD** pour guider votre conception du cycle de vie de vos entités.


Les implémentations REST présentent généralement les caractéristiques suivantes :
- architecture **client-serveur**
- apatridie : toutes les demandes sont indépendantes les unes des autres et peuvent être comprises de manière isolée
- interface uniforme : respect des normes et conventions du secteur pour la création, la lecture, la mise à jour et la suppression de données, ainsi que pour l'authentification des utilisateurs, etc.