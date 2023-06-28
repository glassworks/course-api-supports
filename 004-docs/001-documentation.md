# Documentation

Notre API doit être facilement compréhensible pour ses utilisateurs. On pourrait manuellement écrire une documentation, mais :

* il est difficile de le garder à jour, on est souvent faignant, ou en retard
* Une petite modification au code pourrait modifier le comportement documenté

Un standard de documentation qui s'appelle `swagger` et depuis peu [`openapi`](https://www.openapis.org) a crée une norme pour la documentation d'un API. La documentation est en form `.json` et peu être lu est interprété par un autre process. Il y a même des [process qui transforment automatiquement un fichier `swagger` en interface HTML pour notre lecture](https://swagger.io/tools/swagger-ui/). 

Il y a plusieurs façons de documenter un API :

* Définition vers le code : on rédige notre `swagger.yml` ou `swagger.json` manuellement, et puis on fait tourner un processus qui va créer des fonctions/classes/endpoints pour notre architecture cible (nodejs, php, ruby... etc). Le fichier `swagger.*` est la source de vérité de l'application.
    * Personnellement, je n'aime pas cet approche, car le code généré est très répétitif, et parfois pas assez flexible pour ce que je veux faire
    * Parfois on oublie que le fichier swagger est la source de vérité. On ajoute des fonctions, qui seront décroché ou bien supprimer plus tard quand on relance le générateur.
* Code ver la définition : on fait une sorte de bien structurer et documenter notre code (avec des commentaires), et un `swagger.*` fichier est crée de notre code. Notre code devient la source de vérité.
* Manuel : on maintien la doc et l'implémentation indépendamment. Très lourd, et facile à oublier ou de ne pas mettre à jour.

Personnellement je préfère qu'on aie une seule source de vérité : notre code source !

Grâce à Typescript, il y a des projets comme [`tsoa`](https://tsoa-community.github.io/docs/) qui permet d'utiliser la structure Typescript et les commentaires déjà présents dans le code afin d'assembler automatiquement la documentation.

Par contre, `tsoa` est un framework qui est _opinionated_. On est obligé de structurer notre code à leur normes pour que la documentation soit cohérent :

* __Orienté objet__ : on précise un objet par endpoint avec des fonctions dedans pour les opérations type CRUD - cet objet est un `Controller` dans le pattern MVC.
* __Décorateurs__ : on utilise les décorateurs de typescript pour apporter de l'information concernant les routes, paramètres etc
* __Routing est opaque__ : au lieu de créer des routes nous même (comme dans le dernier exemple), on laisse `tsoa` construire notre API dans express selon les décorateurs. C'est le seul point que je n'aime pas, mais l'avantage d'avoir un documentation cohérent équilibre ce désavantage.

Mince ! On est allé un peu trop loin de la remaniage de notre code - il faudrait repasser un modèle orienté objet.


> Vous trouverez le projet fonctionnel de ce chapitre [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/005-docs)

## Installer `tsoa`

Installer `tsoa` (TypeScript Open Api) :

```bash
npm install tsoa
```

Dans `tsconfig.json` il faut activer l'usage des décorateurs TypeScript : 

```json
 "experimentalDecorators": true,                   /* Enable experimental support for TC39 stage 2 draft decorators. */
```

## Un objet et ses décorateurs

On transforme nos routes `/user` sur le format précisé par `tsoa` (dans le fichier `src/routes/UserController.ts`) :

```ts
import { Body, Delete, Get, Path, Post, Put, Query, Route, Security } from 'tsoa';
import { IUser, IUserCreate, IUserUpdate } from '../model/User/IUser';
import { ICreateResponse } from '../types/ICreateResponse';
import { IIndexResponse } from '../types/IIndexQuery';
import { IUpdateResponse } from '../types/IUpdateResponse';
import { Crud } from '../utility/Crud';

const READ_COLUMNS = ['userId', 'familyName', 'givenName', 'email'];

/**
 * Un utilisateur de la plateforme.
 */
@Route("/user")
export class UserController {

  /**
   * Récupérer une page d'utilisateurs.
   */
  @Get()
  public async getUsers(
    /** La page (zéro-index) à récupérer */
    @Query() page?: string,    
    /** Le nombre d'éléments à récupérer (max 50) */
    @Query() limit?: string,    
  ): Promise<IIndexResponse<IUser>> {    
    return Crud.Index<IUser>({ page, limit }, 'user', READ_COLUMNS);
  }

  /**
   * Créer un nouvel utilisateur
   */
  @Post()
  public async createUser(
    @Body() body: IUserCreate
  ): Promise<ICreateResponse> {
    return Crud.Create<IUserCreate>(body, 'user');
  }

  /**
   * Récupérer une utilisateur avec le ID passé dans le URL
   */
  @Get('{userId}')
  public async readUser(
    @Path() userId: number
  ): Promise<IUser> {
    return Crud.Read<IUser>('user', 'userId', userId, READ_COLUMNS);
  }

  /**
   * Mettre à jour un utilisateur avec le ID passé dans le URL
   */
  @Put('{userId}')
  public async updateUser(
    @Path() userId: number,
    @Body() body: IUserUpdate
  ): Promise<IUpdateResponse> {
    return Crud.Update<IUserUpdate>(body, 'user', 'userId', userId);
  }
  
  /**
   * Supprimer un utilisateur
   */
  @Delete('{userId}')
  public async deleteUser(
    @Path() userId: number,
  ): Promise<IUpdateResponse> {
    return Crud.Delete('user', 'userId', userId);
  }

}
```

* Réutilisation de notre class `Crud.ts`
* Décorateurs `@Route`, `@Query`, `@Path`, `@Body`, `@Middlewares`

## Configurer tsoa

Ensuite, nous allons créer un fichier de configuration pour `tsoa` à `./tsoa.json` :

```json
{
  "entryFile": "src/server.ts",
  "noImplicitAdditionalProperties": "throw-on-extras",
  "controllerPathGlobs": ["src/routes/**/*Controller.ts"],
  "spec": {
    "outputDirectory": "./public",
    "specVersion": 3,
    "securityDefinitions": {
      "jwt": {
        "type": "apiKey",
        "name": "jwt",
        "in": "header",
        "authorizationUrl": "http://swagger.io/api/oauth/dialog"
      }
    }
  },
  "routes": {
    "routesDir": "./src/routes"
  }
}
```

* On précise `spec.outputDirectory` pour le fichier avec notre documentation. On le met dans le dossier public qui sera servi comme fichier static
* On précise `routes.routesDir` pour la destination de notre routeur Express.

`tsoa` va construire un set de routes automatiquement, ainsi qu'un fichier swagger contenant toute la documentation de notre API. Pour cela, il faudrait lancer un script :

```json
  /* package.json */

  ...
  "scripts": {
    ...
    "swagger": "tsoa spec-and-routes",
```

Essayons :

```sh
# terminal vscode

npm run swagger

# Les fichiers seront crées :
# - public/swagger.json
# - routes/routes.ts
```

Consultez les nouveaux fichier crées. On verra de la trace de routes définies dans notre `UserController.ts`

Ensuite, il faut inclure ce code dans notre `server.ts` :


```ts
import Express, { json } from "express";
import { DefaultErrorHandler } from "./middleware/error-handler.middleware";
import { RegisterRoutes } from './routes/routes';

// Récupérer le port des variables d'environnement ou préciser une valeur par défaut
const PORT = process.env.PORT || 5050;

// Créer l'objet Express
const app = Express();

// L'appli parse le corps du message entrant comme du json
app.use(json());

RegisterRoutes(app);

// Ajouter un handler pour les erreurs
app.use(DefaultErrorHandler);

// Lancer le serveur
app.listen(PORT,
  () => {
    console.info("API Listening on port " + PORT);
  }
);
```

Essayer de lancer votre API. Vos routes `/user` devront fonctionner correctement.

## Consulter la documentation

Un fichier `swagger.json` et disponible dans le dossier `public`. Ce fichier pourrait être servi, bien formaté, comme une page web grâce à des librairies open source :

```bash
npm install swagger-ui-express
npm install --save-dev @types/swagger-ui-express`
```

Dans notre fichier `server.ts`, on ajoute de lignes pour servir ce contenu :

```ts
  // Servir le contenu static du dossier `public`
  app.use(Express.static("public"));
  // Créer une route qui permet de convertir le .json en format html
  app.use(
    "/docs",
    swaggerUi.serve,
    swaggerUi.setup(undefined, {
      swaggerOptions: {
        url: "/swagger.json",
      },
    })
  );
```


En lançant l'api, le fichier `swagger.json` est disponible à [http://localhost:5050/swagger.json](http://localhost:5050/swagger.json).

Sinon, grâce à `swagger-ui-express` on peut le consulter sous la route `docs` à [http://localhost:5050/docs/](http://localhost:5050/docs/)


{% hint style="success" %}

Notez surtout que les commentaires dans notre code apparaissent maintenant comme de la documentation de notre API. Génial !

{% endhint %}

## Authentication avec `tsoa`

Au lieu de créer un middleware *ad-hoc* comme on a fait avant, `tsoq` préconise un emplacement fixe pour la logique de note notre sécurisation. Cette emplacement est défini dans `./tsoa.json`, notamment la ligne `authenticationModule`

```json
{
  "entryFile": "src/server.ts",
  "noImplicitAdditionalProperties": "throw-on-extras",
  "controllerPathGlobs": ["src/routes/**/*Controller.ts"],
  "spec": {
    "outputDirectory": "./public",
    "specVersion": 3,
    "securityDefinitions": {
      "jwt": {
        "type": "apiKey",
        "name": "jwt",
        "in": "header",
        "authorizationUrl": "http://swagger.io/api/oauth/dialog"
      }
    }
  },
  "routes": {
    "routesDir": "./src/routes",
    "authenticationModule": "./src/auth/authentication.ts"
  }
}
```

Il faut donc convertir notre ancien middleware d'authorization vers ce nouveau fichier :

```ts
import { Request } from 'express';
import { ApiError } from '../utility/Error/ApiError';
import { ErrorCode } from '../utility/Error/ErrorCode';
import { IAccessToken } from '../types/auth/IAccessToken';
import { JWT } from '../utility/JWT';
import { ACCESS_AUD, ISSUER } from '../routes/AuthController';


export async function expressAuthentication(
  request: Request,
  securityName: string,
  scopes?: string[]
): Promise<boolean> {

  if (securityName === 'jwt') {
    const authheader = request.headers.authorization || '';
    if (!authheader.startsWith('Bearer ')) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/missing-header', 'Missing authorization header with Bearer token');
    }

    const token = authheader.split('Bearer ')[1];

    const jwt = new JWT();
    let decoded : IAccessToken|undefined;
    try {
      decoded = await jwt.decode(token, {
        issuer: ISSUER,
        audience: ACCESS_AUD,
      });
      
    } catch (err: any) {
      if (err?.name === "TokenExpiredError") {
        console.log("Token was expired.");
        
        throw new ApiError(ErrorCode.TokenExpired, 'auth/access-token-expired', 'Access token expired. Try renew it with the renew token.');
      }
      console.log(err);
    }
    
    if (!decoded) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/invalid-access-token', "Access token could not be decoded");
    }

    if (!decoded.userId) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/invalid-access-token', "userId was not found in the payload");
    }    

    return true;
  }

  return false;
}
```

Vous remarquerez que `tsoa` permet de définir différentes stratégies d'authorisation, et contient aussi la logique pour les *scopes* (qu'on n'utilise pas dans notre exemple).

Ensuite, pour protéger nos routes, il suffit de simplement ajouter le décorateur `@Security` devant une class ou méthode :

```ts
@Route("/protected/user")
@Security('jwt')
export class ProtectedUserController {

  /**
   * Récupérer une page d'utilisateurs.
   */
  @Get()
  public async getUsers(
```

Après une recompilation (`npm run swagger`), ce la route est désormais sécurisé par notre authentification pas JWT.


## Automatiser la recompilation du swagger

Vous avez sûrement remarqué qu'à chaque modification, on est obligé de recompiler les routes de `tsoa`. On peut automatiser tout cela en ajoutant une configuration pour `nodemon`  dans `nodemon.json` :

```json
{
  "exec": "npm run swagger && ts-node src/server.ts",
  "watch": ["src"],
  "ignore": ["src/routes/routes.ts"],
  "ext": "ts"
}
```

Ici, on lance systématiquement la recompilation des routes avant de lancer le server, toute en ignorant le fichier `routes.ts` qui est le fichier généré par `tsoa`.

On peut simplifier alors notre `package.json`, car `nodemon.json` est automatiquement lu par `nodemon` lors de son exécution :


```json
  ...
  "scripts": {
    "server": "nodemon",
    ...
```


## Déploiement

Il y a quelques fichiers de plus à ajouter à notre configuration de déploiement, notamment le dossier `public` qui contient le `swagger.json`.

En brèf, il faut copier le dossier `public` dans le dossier `build` après la compilation. Pour cela on utilise la librairies `copyfiles` :

```
# outil pour copier les fichiers et dossiers
npm install --save-dev copyfiles
```

On modifier notre `package.json` ainsi :

```json
 /* package.json */

  ...
  "scripts": {
    "api": "nodemon",
    "swagger": "tsoa spec-and-routes",
    /* Supprimer l'ancien build */
    "clean": "rimraf build",
    /* Clean, puis générer les fichiers swagger et routes, puis compiler avec tsc, puis copier le dossier public dans build */
    "build": "npm run clean && npm run swagger && tsc && copyfiles public/**/* build/"
  },

```

Pour compiler une version de déploiement, il faut juste émettre dans le terminal :

```bash
npm run build
```
