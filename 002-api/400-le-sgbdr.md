# Le SGBDR

## Intégration du SGBDR

On veut maintenant établir une connexion à une base de données, et commencer à ajouter et supprimer les lignes au travers de notre API.

### Dev Container et `docker-compose.yml`

On peut directement inclure une instance d'un SGBDR à notre dev-container grâce à docker :

{% code title="docker-compose.dev.yml" lineNumbers="true" %}
```yml
  ...

  dbms:
    image: mariadb
    restart: always
    ports:
      - "3309:3306"
    environment: 
      - MYSQL_ALLOW_EMPTY_PASSWORD=false
      - MYSQL_ROOT_PASSWORD=rootpassword
    command: [
      "--character-set-server=utf8mb4",
      "--collation-server=utf8mb4_unicode_ci",
    ]
    volumes:
      - ./dbms/dbms-data:/var/lib/mysql
    networks:
      - api-network
```
{% endcode %}

Il faut faire le suivant :

* Ajouter les lignes dessus à `docker-compose.dev.yml` sous l'option `services`.
* Ajouter le dossier `dbms`
* Rebuilder votre DevContainer dans VSCode.
* Le SGBDR est désormais disponible.

Docker gère automatiquement les connexions et ports pour nous :

* Le nom d'hôte est le nom du service (`dbms`)
* Le port est automatiquement mappé par docker, pas besoin de le préciser dans notre code

Vous pouvez connecter à votre SGBDR avec :

```sh
mycli -h dbms -u root
```

Ensuite, vous pouvez créer votre base de données, l'utilisateur pour notre api, et créer les premières tables :

```sql
/*
Script de création de la base de données.
A noter, on utilise uns stratégie avec DROP et IF NOT EXISTS afin de rendre 
notre script réutilisable dans le future, même si la base existe déjà
*/
create database IF NOT EXISTS school;

/* Créer l'utilisateur API */
create user IF NOT EXISTS 'api-dev'@'%.%.%.%' identified by 'apipassword';
grant select, update, insert, delete on school.* to 'api-dev'@'%.%.%.%';
flush privileges;

/* La définition de la schéma */
use school;

/* user */
create table if not exists user (
  userId int auto_increment not null,
  email varchar(256) unique not null, 
  familyName varchar(256), 
  givenName varchar(256), 
  primary key(userId)
);

drop trigger if exists before_insert_user;

create trigger before_insert_user
before insert
on user for each row set new.email = lower(trim(new.email));

/* ... */

```

### Intégration NodeJS

Nous utilisons la librairie [`mysql2`](https://www.npmjs.com/package/mysql2) pour communiquer avec notre base de données.

```bash
npm install mysql2
```

Normalement, notre API va ouvrir une connexion unique auprès du SGBDR pour chaque requête en cours. Ceci peut être lourd et long, donc le créateur de la librairie a prévu les **connection pools**. C'est-à-dire, on va essayer de réutiliser les connexions déjà ouvertes.

Moi, je préfère créer une classe qui enveloppe l'objet principal, pour ne pas répéter du code :

{% code title="utility/DB.ts" lineNumbers="true" %}
```ts
import mysql, { Pool } from 'mysql2/promise';

/** Wrapper de la connexion à la SGBDR.
 * On stock une seule référence à la connexion-pool, et on va systématiquement
 * récupérer cette référence pour nos requêtes.
 */
export class DB {

  // Variable "static": une seule instance pour toutes les instances de la classe DB
  private static POOL: Pool;

  /**
   * Récupérer ou créer la connexion-pool.
   */
  static get Connection(): Pool {
    if (!this.POOL) {
      this.POOL = mysql.createPool({
        host: process.env.DB_HOST || 'dbms',
        user: process.env.DB_USER || 'api-dev',
        database: process.env.DB_DATABASE || 'school',
        password: process.env.DB_PASSWORD || 'apipassword',  
      });
    }

    return this.POOL;
  }

}
```
{% endcode %}

Ici, on crée une variable static, et on initialise notre **pool** avec les coordonnées tirées de l'environnement (ou des valeurs par défaut).

### Opérations CRUD

Voici un exemple d'un set de **endpoints** pour la gestion de l'utilisateur (code source entière disponible [ici](https://dev.glassworks.tech:18081/courses/api/api-supports/-/tree/master/exemples/mysql)).

{% code title="routes/User.ts" lineNumbers="true" %}
```ts
import { NextFunction, Request, Response, Router } from "express";
import { OkPacket, RowDataPacket } from 'mysql2';
import { DB } from '../utility/DB';
import { ICreateResponse } from '../types/ICreateResponse';
import { IIndexQuery, IIndexResponse, ITableCount } from '../types/IIndexQuery';
import { IUser, IUserRO } from '../model/IUser';

const routerIndex = Router({ mergeParams: true });

routerIndex.get<{}, IIndexResponse<IUserRO>, {}, IIndexQuery>('/',
  async (request, response, next: NextFunction) => {

    try {

      const db = DB.Connection;
      
      // On suppose que le params query sont en format string, et potentiellement
      // non-numérique, ou corrompu
      const page = parseInt(request.query.page || "0") || 0;
      const limit = parseInt(request.query.limit || "10") || 0;
      const offset = page * limit;

      // D'abord, récupérer le nombre total
      const count = await db.query<ITableCount[] & RowDataPacket[]>("select count(*) as total from user");      

      // Récupérer les lignes
      const data = await db.query<IUserRO[] & RowDataPacket[]>("select userId, familyName, givenName, email from user limit ? offset ?", [limit, offset]);      

      // Construire la réponse
      const res: IIndexResponse<IUserRO> = {
        page,
        limit,
        total: count[0][0].total,
        rows: data[0]
      }

      response.json(res);

    } catch (err: any) {
      next(err);
    }

  }
);


routerIndex.post<{}, ICreateResponse, IUser>('/',
  async (request, response, next: NextFunction) => {

    try {
      const user = request.body;

      // ATTENTION ! Et si les données dans user ne sont pas valables ?
      // - colonnes qui n'existent pas ?
      // - données pas en bon format ?

      const db = DB.Connection;
      const data = await db.query<OkPacket>("insert into user set ?", user);

      response.json({ 
        id: data[0].insertId
      });

    } catch (err: any) {
      next(err);
    }

  }
);

// Regroupé

const routerUser = Router({ mergeParams: true });
routerUser.use(routerIndex);

export const ROUTES_USER = routerUser;
```
{% endcode %}

Il y a plusieurs choses à noter, comme discuté dans les sections suivantes.

#### L'utilisation des types

Une force de Typescript est le fait de pouvoir contrôler les types du début à la fin, même avec les **génériques**.

En effet, Express est écrit avec les **génériques**. Nous pouvons donc préciser la structure des types qui entre dans le corps du message, dans le params query, et à renvoyer dans la réponse :

```ts
// la fonction get avec les génériques, précisés entre les balises <...> :
// 1 - les *params*
// 2 - la structure de la réponse
// 3 - la structure du corps (body)
// 4 - la structure des params query
routerIndex.get<{}, IIndexResponse<IUserRO>, {}, IIndexQuery>('/',
```

Quand on va récupérer `request.query`, par exemple, l'objet typescript retourné va conformer à la structure précisée dans le 4ème paramètre. Il serait plus difficile à faire des bêtises !

Comment on précise un type ? Voici quelques exemples.

D'abord, nous allons reproduire le schéma de nos données en Typescript. Voici, par exemple, le schéma de notre table `User` représenté en Typescript :

{% code title="model/IUser.ts" lineNumbers="true" %}
```ts
// Définition d'un structure IUser
// A noter, le ? veut dire que le champ est optionnel

export interface IUser {
  userId: number;
  familyName?: string;
  givenName?: string;
  email: string;
}

// Outils de manipulation des types :
// https://www.typescriptlang.org/docs/handbook/utility-types.html
// Ici, on rend tous les champs "lecture seul". Typescript ne va pas autoriser l'affectation des champs
export type IUserRO = Readonly<IUser>;
```
{% endcode %}

Nous allons préciser aussi des types retournés par nos interrogations avec la base SQL.

La première interrogation est du type **indexe**, où on cherche plusieurs lignes avec pagination. Nous spécifiions la requête et la réponse :

{% code title="types/IIndexQuery.ts" lineNumbers="true" %}
```ts
/**
 * Paramètres de la requête de type **indexe**, notamment pour la pagination.
 */
export interface IIndexQuery {
  page?: string;
  limit?: string;  
}

/* Ici, on utilise un générique, précisé par <T>
Ca veut dire qu'on va passer un autre type comme paramètre, qui sera utilisé à sa place
ex. const res : IIndexResponse<IUser> = {
  rows: [] // <-- Ici on ne peut juste affecter les structures de type IUser  
}
*/
export interface IIndexResponse<T> {
  page: number;
  limit: number;
  total: number;
  rows: T[];
}

/**
 * Structure retourné par MySQL quand on fait une requête de type `count(*)` 
 */
export interface ITableCount {
  total: number;
}
```
{% endcode %}

Ensuite, pour la requête de **création** d'une ligne :


{% code title="types/ICreateResponse.ts" lineNumbers="true" %}
```ts
export interface ICreateResponse {
  id: number;
}
```
{% endcode %}



#### Le formatage des requêtes SQL

Comme toutes les librairies, on évite l'injection SQL en utilisant des fonctionnalités pour échapper les données :

```ts
// On met les ?, puis on passe comme 2ème paramètre un tableau des données à injecter, dans l'ordre
const data = await db.query<IUserRO[] & RowDataPacket[]>("select userId, familyName, givenName, email from user limit ? offset ?", [limit, offset]);      
```

Pour les inserts, il est pratique de passer plutôt un objet, et laisser la librairie formuler la requête :

```ts
const user: IUser = {
  email: "kevin@nguni.fr",
  familyName: "Glass",
  givenName: "Kevin",
}
const data = await db.query<OkPacket>("insert into user set ?", user);
```

### Accrocher les routes à la hiérarchie

On compose notre application principale par les routes qu'on vient de créer :

{% code title="server.ts" lineNumbers="true" %}
```ts
import Express, { json } from "express";
import { ROUTES_USER } from "./routes/User";

// Récupérer le port des variables d'environnement ou préciser une valeur par défaut
const PORT = process.env.PORT || 5050;

// Créer l'objet Express
const app = Express();

// L'appli parse le corps du message entrant comme du json
app.use(json());

// Accrocher les routes CRUD de l'utilisateur à la hierarchie
app.use('/user', ROUTES_USER);

// Lancer le serveur
app.listen(PORT,
  () => {
    console.info("API Listening on port " + PORT);
  }
);
```
{% endcode %}

Notez bien la ligne `app.use('/user', ROUTES_USER);`.

{% hint style="success" %}
Si vous ne l'avez pas encore fait, ajouter la ligne `"forwardPorts": [ 5050 ]` à votre fichier `.devcontainer/devcontainer.json`. Ceci permet à votre serveur d'être accessible en dehors de votre container VSCode.

Testez les deux endpoints avec PostMan :
* `GET http://localhost:5050/user`
* `POST http://localhost:5050/user`
{% endhint %}

## Transactions

Regardez la documentation pour la librairie `mysql2`. Vous allez remarquer des fonctions suivantes :

- `beginTransaction()`
- `commit()`
- `rollback()`

Ce sont des fonctions qui émettent simplement des commandes équivalentes en SQL auprès de notre SGBDR.

Vous pouvez donc facilement gérer la cohérence de vos opérations d'écriture en utilisant ces fonctions.

```ts
  
  const db = DB.Connection;   
  
  await db.beginTransaction();
  try {

    // Effectuer vos opérations

    // Essayer de les commit
    await db.commit();
  } catch {
    // S'il y a eu une exception, rollback
    await db.rollback();
  }

```

## Exercice 1 : CRUD

Complétez les autres fonctions CRUD pour un utilisateur :

* `GET /user/:userId` : récupérer juste la ligne de l'utilisateur en format `IUser`
* `PUT /user/:userId` : mettre à jour une ligne précise
* `DELETE /user/:userId` : supprimer l'utilisateur

Créez une suite de tests dans Postman afin de valider votre API.

<details>

<summary>Solution</summary>

La solution entière se trouve [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/001-basic-crud-routes-express).

On commence par créer un **Router** dans express qui gère la sous-route `/:userId` :

```ts
const routerSingle = Router({ mergeParams: true });

// Router user existe déjà, on ajoute la sous-route
routerUser.use('/:userId', routerSingle);
```


Ensuite, gérons les différentes méthodes.

La méthode `GET /:userId` : 

```ts
routerSingle.get<{ userId: string }, IUserRO, {}>('',
  async (request, response, next: NextFunction) => {
    try {
      // ATTENTION ! Valider que le userId est valable ?
      const userId = request.params.userId;

      const db = DB.Connection;
      // Récupérer les lignes
      const data = await db.query<IUserRO[] & RowDataPacket[]>("select userId, familyName, givenName, email from user where userId = ?", [userId]);      

      // ATTENTION ! Que faire si le nombre de lignes est zéro ?
      const res = data[0][0];
        
      response.json(res);

    } catch (err: any) {
      next(err);
    }
  }
);
```

La méthode `PUT /:userId` : 

```ts
routerSingle.put<{ userId: string }, IUpdateResponse, IUser>('',
  async (request, response, next: NextFunction) => {
    try {
      // ATTENTION ! Valider que le userId est valable ?
      const userId = request.params.userId;
      const body = request.body;

      const db = DB.Connection;
      // Récupérer les lignes
      const data = await db.query<OkPacket>(`update user set ? where userId = ?`, [body, userId]);

      // Construire la réponse
      const res = {
        id: userId,
        rows: data[0].affectedRows
      }
        
      response.json(res);

    } catch (err: any) {
      next(err);
    }
  }
);
```

La méthode `DELETE /:userId` : 

```ts
routerSingle.delete<{ userId: string }, IDeleteResponse, {}>('',
  async (request, response, next: NextFunction) => {
    try {
      // ATTENTION ! Valider que le userId est valable ?
      const userId = request.params.userId;
      const db = DB.Connection;

      // Récupérer les lignes
      const data = await db.query<OkPacket>(`delete from user where userId = ?`, [ userId ]);      

      // Construire la réponse
      const res = {
        id: userId,
        rows: data[0].affectedRows
      }
        
      response.json(res);

    } catch (err: any) {
      next(err);
    }
  }
);
```


</details>

## Exercice 2 : Erreurs

Essayez de rentrer des mauvaises informations via Postman :

* Créer un utilisateur doublon
* Essayer de passer un champ qui n'est pas une colonne dans la base
* Passer du texte dans le query param de la requête index (pour limit et offset)
* Afficher, mettre à jour, ou supprimer un utilisateur qui n'existe pas

Pour l'instant, on reçoit un message moche (pas en json) et pas très parlant dans Postman.

Ajoutez un middleware qui gère les erreurs :

* Renvoyez plutôt du json
* Avoir au moins le champ suivants :
  * `code` : le code HTTP approprié
    * `400` : la requête est mauvaise (erreur dans les données entrantes)
    * `401` : pas autorisé
    * `404` : élément pas trouvé
    * `500` : erreur interne (dernier recours)
  * `structured` : un champ plus libre, mais plus structuré qui permettrait de localiser l'erreur coté front. Exemple :
    * `params-invalid`
    * `connection-error`
    * `auth/unknown-email`
  * `message`: un message humain décrivant l'erreur
  * `data`: (optionnel) les données supplémentaires concernant l'erreur

Astuce : il faut dire à express d'utiliser votre handler d'erreur avec `app.use(...)`. Votre handler doit avoir 4 paramètres dans le callback, le premier étant l'objet d'erreur.

<details>

<summary>Solution</summary>

La solution entière se trouve [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/001-basic-crud-routes-express).

D'abord, on crée une classe qui dérive de la classe générique de Javascript : `Error` 

```ts
import { ErrorCode } from './ErrorCode';
import { IApiError } from './IApiError';
import { StructuredErrors } from './StructuredErrors';


export class ApiError {
  constructor(public httpCode: ErrorCode, public structuredError: StructuredErrors, public errMessage: string, public errDetails?: any) {
  }

  get json(): IApiError {
    return {
      code: this.httpCode,
      structured: this.structuredError,
      message: this.errMessage,
      details: this.errDetails
    }
  }
}

```

Cette classe contient notamment une fonction permettant d'exporter l'erreur en format JSON selon la description de ce problème. Les `code` et `structured` sont les énumérations des différentes possibilités :

```ts
// Les numéros de d'erreur standard de HTTP
export enum ErrorCode {
  NotFound = 404,
  Unauthorized = 403,
  BadRequest = 400,
  TooManyRequests = 429,
  InternalError = 500
}
```

```ts
// Les types d'erreur connus par notre API, permettant au consommateur de plus facile comprend ce qui s'est passé
export type StructuredErrors = 
  // SQL
  'sql/failed' |  
  'sql/not-found' |

  // Crud
  'validation/failed' | 
    
  // Authorization
  'auth/unknown-email' |


  // Default
  'internal/unknown'
;
```

Ensuite, nous allons rédiger un **handler** (middleware) qui prend 4 paramètres pour qu'Express l'utilise pour gérer des erreurs :


```ts
import { NextFunction, Request, Response } from 'express';
import { ApiError } from '../utility/Error/ApiError';
import { ErrorCode } from '../utility/Error/ErrorCode';


export const DefaultErrorHandler = async (error: any, req: Request, res: Response, next: NextFunction) => {

  console.log(error);
  console.log(error.constructor.name);

  let err = new ApiError(ErrorCode.InternalError, 'internal/unknown', 'An unknown internal error occurred');
    
  if (!!error) {
    if (error instanceof ApiError) {
      err = error;
    } 
    else if (!!error.sql) {
      // Ceci est une erreur envoyé par la base de données. On va supposer une erreur de la part de l'utilisateur
      // A faire : il est peut-être recommandé d'avoir un handler dédié aux erreurs SQL pour mieux trier celles qui sont de notre faute, et celles la faute de l'utilisateur.
      err = new ApiError(ErrorCode.BadRequest, 'sql/failed', error.message, {
        sqlState: error.sqlState,
        sqlCode: error.code
      });      
      // A noter : on ne renvoie pas le SQL pour ne pas divulger les informations secrets
    } else {
      if (error.message) {
        err.errMessage = error.message;
      }
    }
  }
  console.log(err.json);

  res.status(err.httpCode).json(err.json);    
}
```

Notez les paramètres de notre `DefaultErrorHandler`. On accepte comme premier paramètre une erreur inconnue. Ensuite, on construit l'erreur formatée à l'aide de notre classe `ApiError`. Enfin, on renvoie une réponse avec le code HTTP et le json représentant l'erreur.


</details>

## Exercice 3 : Remanier

Essayez d'ajouter une autre table à votre base, e.g. `repas` (vous pouvez vous inspirer de l'application nutrition).

Ajoutez des endpoints CRUD pour cette table.

Essayez au maximum de remanier votre code. Est-ce qu'il y a des éléments qu'on peut réutiliser pour les opérations CRUD classiques ?

