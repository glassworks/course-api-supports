# Framework 

Jusqu'à présent, nous avons utilisé Basic Express pour construire notre API REST. Vous avez peut-être remarqué qu'il y a beaucoup de répétitions et que nous réinventons parfois la roue !

C'est pourquoi il existe un certain nombre de Frameworks qui mettent en place une structure de projet, un schéma de routage normalisé, des protocoles de sécurité, etc. C'est parfois une bonne chose, mais c'est aussi parfois trop restrictif !

Ce qui nous manque pour l'instant, c'est une façon harmonisée de structurer notre projet et nos appels REST. De plus, il serait formidable de pouvoir **documenter** notre API afin que les gens puissent l'utiliser !

Il existe bien sûr des frameworks complets que nous n'abordons pas dans notre cours : NestJS, NextJS, Nuxt, Symfony, Ruby On Rails, Java Spring, etc. 

Pour ce cours, nous voulons être aussi neutres que possible par rapport à un Framework, tout en introduisant juste ce qu'il faut pour structurer notre projet. Pourquoi ?
- Pour exposer plus proprement les principes et les choix de conception
- Pour nous donner la marge de manœuvre nécessaire pour mettre en œuvre nous-mêmes certaines techniques
- Pour nous permettre d'apprendre et d'apprécier ce qui se passe dans les coulisses d'un Framework moderne.


Nous utiliserons `tsoa` pour structurer notre projet. Je trouve qu'il ajoute une quantité suffisante de structure à notre projet, tout en laissant assez de flexibilité pour faire ce que nous voulons. `tsoa` est _opinionated_. On est obligé de structurer notre code à leurs normes pour que la documentation soit cohérente. 

* __Orienté objet__ : on précise un objet par endpoint avec des fonctions dedans pour les opérations type CRUD - cet objet est un `Controller` dans le pattern MVC.
* __Décorateurs__ : on utilise les décorateurs de typescript pour apporter de l'information concernant les routes, paramètres etc
* __Routing est opaque__ : au lieu de créer des routes nous même, on laisse `tsoa` construire notre API dans Express selon les décorateurs. C'est le seul point que je n'aime pas, mais l'avantage d'avoir une documentation cohérente fait équilibrer ce désavantage.


## Installer `tsoa`

Installer `tsoa` (TypeScript Open Api) :

```bash
npm install tsoa
```

Dans `tsconfig.json` il faut activer l'usage des décorateurs TypeScript : 

```json
{
  "compilerOptions": {
    ...
    "experimentalDecorators": true,                  
    ...
```

## Créer un controller

On transforme nos routes `/user` sur le format précisé par `tsoa` (dans le fichier `src/controllers/UserController.ts`) :

```ts
import { IUser, IUserCreate, IUserUpdate } from '@model/types/IUser';
import { IORMCreateResponse, IORMDeleteResponse, IORMIndexResponse, IORMUpdateResponse } from '@orm/interfaces/IORM';
import { ORM } from '@orm/ORM';
import { Body, Delete, Get, Patch, Path, Put, Query, Route } from 'tsoa';

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
  ): Promise<IORMIndexResponse<IUser>> {    
    return ORM.Index<IUser>({
      table: 'user',
      columns: READ_COLUMNS,
      query: { page, limit },
    });
  }

  /**
   * Créer un nouvel utilisateur
   */
  @Put()
  public async createUser(
    @Body() body: IUserCreate
  ): Promise<IORMCreateResponse> {
    return ORM.Create<IUserCreate>({
      table: 'user',
      body,
    });
  }

  /**
   * Récupérer une utilisateur avec le ID passé dans le URL
   */
  @Get('{userId}')
  public async readUser(
    @Path() userId: number
  ): Promise<IUser> {
    return ORM.Read<IUser>({
      table: 'user', 
      idKey: 'userId', 
      idValue: userId, 
      columns: READ_COLUMNS
    });
  }

  /**
   * Mettre à jour un utilisateur avec le ID passé dans le URL
   */
  @Patch('{userId}')
  public async updateUser(
    @Path() userId: number,
    @Body() body: IUserUpdate
  ): Promise<IORMUpdateResponse> {
    return ORM.Update<IUserUpdate>({
      table: 'user',
      idKey: 'userId', 
      idValue: userId, 
      body,
    });
  }
  
  /**
   * Supprimer un utilisateur
   */
  @Delete('{userId}')
  public async deleteUser(
    @Path() userId: number,
  ): Promise<IORMDeleteResponse> {
    return ORM.Delete({
      table: 'user', 
      idKey: 'userId', 
      idValue: userId, 
    });
  }
}

```

`tsoa` compile tous nos contrôleurs et les transforme en routes Express pour nous. Le contrôleur ci-dessus, grâce à ses **annotations**, créera ce qui suit : 
- `GET /user`
- `PUT /user`
- `GET /user/:userId`
- `PATCH /user/:userId`
- `DELETE /user/:userId`

Cette compilation se fait dans une étape séparée que nous configurerons dans la section suivante. Mais les annotations `@Route`, `@Get`, `@Query`, etc. sont utilisées par le processus de compilation pour :
- écrire automatiquement les routes Express pour nous
- ajouter une validation automatique aux données entrantes
- éventuellement écrire automatiquement notre documentation à partir des commentaires !

Notre contrôleur remplace le fichier `routes/user.route.ts` que nous avons écrit dans un chapitre précédent. Vous pouvez le supprimer.

## Configurer tsoa

Ensuite, nous allons créer un fichier de configuration pour `tsoa` à `./tsoa.json` :

```json
{
  "entryFile": "src/server.ts",
  "noImplicitAdditionalProperties": "throw-on-extras",
  "controllerPathGlobs": ["src/controllers/**/*Controller.ts"],
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
    "routesDir": "src/routes"
  },
  "compilerOptions": {
    "baseUrl": "src",                                  
    "paths": {
      "@model/*": ["model/*"],
      "@orm/*": ["utility/ORM/*"],      
      "@error/*": ["utility/error/*"],      
      "@routes/*": ["routes/*"],
      "@controllers/*": ["controllers/*"],
      "@logging/*": ["utility/logging/*"]
    }
  }
}
```

* On précise `spec.outputDirectory` pour le fichier avec notre documentation. On le met dans le dossier public qui sera servi comme fichier static
* On précise `routes.routesDir` pour la destination du routeur Express qui est automatiquement générée par `tsoa`
* Enfin, on duplique les `compilerOptions` du fichier `tsconfig.json` pour que nos raccourcis soient pris en compte.

`tsoa` va construire un set de routes automatiquement, ainsi qu'un fichier swagger contenant toute la documentation de notre API. Pour cela, il faudrait lancer un script :

```json
  /* package.json */

  ...
  "scripts": {
    ...
    "compile": "tsoa -r tsconfig-paths/register spec-and-routes"
```

Essayons :

```sh
# terminal vscode

npm run compile

# Les fichiers seront crées :
# - public/swagger.json
# - routes/routes.ts
```

Consultez les nouveaux fichiers créés. On verra de la trace de routes définies dans notre `UserController.ts`

Ensuite, il faut inclure ce code dans notre `server.ts` :


```ts
import { DefaultErrorHandler } from "@error/error-handler.middleware";
import { json } from "body-parser";
import Express, { NextFunction, Request, Response } from "express";
import { join } from 'path';
import { RegisterRoutes } from './routes/routes';


// Récupérer le port des variables d'environnement ou préciser une valeur par défaut
const PORT = process.env.PORT || 5050;

// Créer l'objet Express
const app = Express();

// L'appli parse le corps du message entrant comme du json
app.use(json());

RegisterRoutes(app);

// Créer un endpoint GET
app.get('/helo', 
  (request: Request, response: Response, next: NextFunction) => {
    response.send("<h1>Hello world!</h1>");
  }
);

// Server des fichiers statiques
app.use('/public', Express.static(join('assets')));

// Ajouter un handler pour les erreurs
app.use(DefaultErrorHandler);

// Lancer le serveur
app.listen(PORT,
  () => {
    console.info("API Listening on port " + PORT);
  }
);
```

Essayer de lancer votre API. Vos routes `/user` devront fonctionner correctement. Vous pouvez utiliser Postman grâce à une collection des requêtes [ici](./api.postman_collection.json).

## Validation 

Essayez d'envoyer des données non valides à votre API. Ceci est rejeté par notre API car `tsoa` valide automatiquement les données en se basant sur les types que nous spécifions dans les annotations. C'est très intéressant !

Cependant, nous pouvons souhaiter modifier notre gestionnaire d'erreur pour envoyer des informations utiles à l'utilisateur.


```ts
export const DefaultErrorHandler = async (error: any, req: Request, res: Response, next: NextFunction) => {
  
  let err = new ApiError(ErrorCode.InternalError, 'internal/unknown', 'An unknown internal error occurred');
    
  if (!!error) {
    if (error instanceof ApiError) {
      err = error;
    } 
    /** Ajouter ici **/
    else if (error instanceof ValidateError) {
      err = new ApiError(ErrorCode.BadRequest, 'validation', 'Validation error', {
        fields: error.fields
      });
    } 
    ....

```

Si vous répétez la demande, vous obtiendrez des informations utiles sur ce qui n'a pas fonctionné.

## Automatiser la recompilation de tsoa

Vous avez sûrement remarqué qu'à chaque modification, on est obligé de recompiler les routes de `tsoa`. On peut automatiser tout cela en ajoutant une configuration pour `nodemon` dans `nodemon.json` :

```json
{
  "exec": "npm run compile && ts-node -r tsconfig-paths/register src/server.ts",
  "watch": ["src"],
  "ignore": ["src/routes/routes.ts"],
  "ext": "ts"
}
```

Ici, on lance systématiquement la recompilation des routes avant de lancer le serveur, tout en ignorant le fichier `routes.ts` qui est le fichier généré par `tsoa`.

On peut simplifier alors notre `package.json`, car `nodemon.json` est automatiquement lu par `nodemon` lors de son exécution :

```json
  ...
  "scripts": {
    "server": "nodemon",
    ...
```

## Déploiement

Il y a quelques fichiers de plus à ajouter à notre configuration de déploiement, notamment le dossier `public` qui contient le `swagger.json`.

En bref, il faut copier le dossier `public` dans le dossier `build` après la compilation. Pour cela, on utilise la librairie `copyfiles` :

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


