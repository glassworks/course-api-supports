
# Exercice : Autorisation

Ajoutez une première couche d'identification et autorisation à votre API.

* Ajoutez une colonne "password" à la table `user`.
* Ajoutez la possibilité de renseigner un mot de passe et le stocker en format crypté (utilisez la librairie [`bcrypt.js`](https://github.com/dcodeIO/bcrypt.js)). L'idée est qu'on génère un hash avec bcrypt qui est stocké dans la base de données. Quand un utilisateur se connect, on recalcule le hash du mot de passe passé et on le compare avec celui dans notre base. C'est ainsi qu'on se protège si jamais notre base est hacké, car on ne stocke pas des mots de passes en text clair.
* Ajoutez un endpoint `POST /auth/login` qui permet de se connecter en renseignant son adresse e-mail et son mot de passe. La réponse devrait fournir un `JWT` crée avec la librairie [`jsonwebtoken`](https://github.com/auth0/node-jsonwebtoken). Votre JWT doit être signé par une clé privé (`RS256`), comme ça on est sur que le JWT a été crée par vous même (quand la clé et repassé comme jeton de validation). [Voici comment générer la clé privée](https://gist.github.com/ygotthilf/baa58da5c3dd1f69fae9). Il faut ensuite charger cette clé avec `fs` afin de l'utiliser dans la signature du JWT. Est-ce qu'on charge la clé à chaque requête, ou peut-on optimiser notre code ?
* Créez un middleware qui protège les endpoints CRUD pour la table `user` et `repas`. Le middleware doit regarder le header `authorization` qui devrait contenir le text `Bearer <JWT>`. Le middleware doit décrypter le token, le valider, et si invalide, renvoyer une erreur avec code `401`
* Configurer Postman pour récupérer le JWT automatiquement (grâce à des variables et un script) pour les endpoints protégés.
* Ajoutez le endpoint `GET /auth/requestPasswordReset` qui demande d'envoyer un mail de changement de mot de passe. Ce mail doit envoyer un code unique et utilisable une seul fois à l'adresse email précisé par l'utilisateur (vous pouvez créer un compte gratuit de Mailjet et intégrer la librairie [`node-mailjet`](https://github.com/mailjet/mailjet-apiv3-nodejs#readme)). 
* Ajoutez le endpoint `POST /auth/resetPassword` qui prend dans le corps du message le code unique envoyé par email, l'adresse e-mail et le nouveau mot de passe de l'utilisateur, et qui met à jour son mot de passe.


:closed_lock_with_key: :closed_lock_with_key: :closed_lock_with_key: Félicitations, vous avez sécurisé votre api ! :closed_lock_with_key: :closed_lock_with_key: :closed_lock_with_key:

# Exercice 5 : Autorisation magique

Dans vos challenges vous avez pu vous connecter juste en renseignant votre adresse email (sans mot de passe). Un mail avec un code est envoyé à votre boîte mail. Quand on clique dessus on est considéré autorisé.

Mettez en place ce flux avec les contraintes suivantes :
* Le lien dans le mail doit avoir un timeout de 5 minutes, au delà de cette intervalle, il faut retourner une erreur comme quoi il faut redemander un nouveau mail.
* On ne doit pas pouvoir acceder aux autres endpoints dans notre API avec le jeton dans le mail. En effet il faut que le jeton de connexion magique soit uniquement pour générer le jeton d'accès final

<details>

<summary>Solution</summary>

La solution complète se trouve [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/002-magic-link-authorisation).

Notez bien :
- Les endpoints pour créer, envoyer (par email) un lien magique, ainsi que la conversion de ce lien en `access token` : `src/routes/Auth.ts`
- Le middleware utilisé pour valider un `access token` avant une route : `src/middleware/auth.middleware.ts`
- L'insertion de ce middleware devant certaines routes de l'API : `src/server.ts`
- Le fichier Postman pour tester la solution : `src/test`

</details>
