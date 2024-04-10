# Le SGBDR


On veut maintenant établir une connexion à une base de données, et commencer à ajouter et supprimer les lignes au travers de notre API.

## Dev Container et `docker-compose.yml`

On peut directement inclure une instance d'un SGBDR à notre dev-container grâce à docker :

{% code title="docker-compose.dev.yml" lineNumbers="true" %}
```yml
  ...

  dbms:
    image: mariadb
    restart: always
    container_name: my_sql_database
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
* Pour plus facilement identifier notre container dans Docker, on précise un nom avec `container_name`

Dans le terminal VSCode, vous pouvez connecter à votre SGBDR avec :

```sh
mycli -h dbms -u root
```

Ou bien, d'un terminal en dehors de VSCode (si vous êtes sur un serveur par exemple): 

```sh
docker exec -it my_sql_database mariadb -u root -p
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

## Intégration NodeJS

Nous utilisons la librairie [`mysql2`](https://www.npmjs.com/package/mysql2) pour communiquer avec notre base de données.

```bash
npm install mysql2
```

Normalement, notre API va ouvrir une connexion unique auprès du SGBDR pour chaque requête en cours. Ceci peut être lourd et chronophage, donc le créateur de la librairie a prévu les **connection pools**. C'est-à-dire, on va essayer de réutiliser les connexions déjà ouvertes.

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

## Opérations CRUD

Voici un exemple d'un set de **endpoints** pour la gestion de l'utilisateur (code source entière disponible [ici](https://github.com/glassworks/course-api-supports/tree/main/exemples/mysql)).

{% code title="routes/user.route.ts" lineNumbers="true" %}
```ts
import { NextFunction, Request, Response, Router } from "express";
import { ResultSetHeader, RowDataPacket } from "mysql2";
import { DB } from "../utility/DB";

const router = Router({ mergeParams: true });

// Créer un utilisateur
router.put('/',
  async (request: Request, response: Response, next: NextFunction) => {
    console.log(request.body);
    
    try {
      const db = DB.Connection;

      const result = await db.query<ResultSetHeader>(
        'insert into user set ?',
        request.body
      );

      response.send({
        id: result[0].insertId
      });
    } catch (err: any) {
      next(err.message)
    }

  }
)

// Lire les utilisateurs
router.get('/',
  async (request: Request, response: Response, next: NextFunction) => {
    
    const db = DB.Connection;

    const limit = parseInt(request.query.limit as string) || 10;
    const offset = (parseInt(request.query.page as string) || 0) * limit;

    try {
      const data = await db.query("select userId, email, familyName, givenName from user limit ? offset ?", [limit, offset]);
      const count = await db.query<{count: number}[] & RowDataPacket[]>("select count(*) as count from user");

      response.send({
        total: count[0][0].count,
        rows: data[0]
      });
    } catch (err: any) {
      next(err.message);
    }

  }
)

// Lire un utilisateur avec ID userId
router.get('/:userId',
  async (request: Request, response: Response, next: NextFunction) => {
    console.log(`Le userId est: ${request.params.userId}`);

    const db = DB.Connection;
    try {
      const data = await db.query<RowDataPacket[]>('select userId, familyName, givenName, email from user where userId = ?', [ request.params.userId ]);

      response.send(data[0][0]);
    } catch (err: any) {
      next(err.message);
    }

   
  }
);

// Mettre à jour un utilisteur
router.patch('/:userId',
  async (request: Request, response: Response, next: NextFunction) => {
    console.log(`Le userId est: ${request.params.userId}`);
    
    try {
      const db = DB.Connection;

      const result = await db.query<ResultSetHeader>(
        'update user set ? where userId = ?',
        [ request.body, request.params.userId ]
      );

      response.send({
        id: result[0].insertId
      });
    } catch (err: any) {
      next(err.message)
    }

    
  }
);

// Supprimer un utilisateur
router.delete('/:userId',
  async (request: Request, response: Response, next: NextFunction) => {
    console.log(`Le userId est: ${request.params.userId}`);
    
    
    try {
      const db = DB.Connection;

      const result = await db.query<ResultSetHeader>(
        'delete from user where userId = ?',
        [ request.params.userId ]
      );

      response.send({
        id: result[0].insertId
      });
    } catch (err: any) {
      next(err.message)
    }

  }
)

export const ROUTES_USER = router;
```
{% endcode %}

Il y a plusieurs choses à noter, comme discuté dans les sections suivantes.


## Le formatage des requêtes SQL

Comme toutes les librairies, on évite l'injection SQL en utilisant des fonctionnalités pour échapper les données :

```ts
// On met les ?, puis on passe comme 2ème paramètre un tableau des données à injecter, dans l'ordre
const data = await db.query<RowDataPacket[]>("select userId, familyName, givenName, email from user limit ? offset ?", [limit, offset]);      
```

Pour les inserts, il est pratique de passer plutôt un objet, et laisser la librairie formuler la requête :

```ts
const user = {
  email: "kevin@nguni.fr",
  familyName: "Glass",
  givenName: "Kevin",
}
const data = await db.query<OkPacket>("insert into user set ?", user);
```

## Accrocher les routes à la hiérarchie

On compose notre application principale par les routes qu'on vient de créer :

{% code title="server.ts" lineNumbers="true" %}
```ts
import { json } from "body-parser";
import Express from "express";
import { join } from 'path';
import { ROUTES_USER } from "./routes/user.route";

// Récupérer le port des variables d'environnement ou préciser une valeur par défaut
const PORT = process.env.PORT || 5050;

// Créer l'objet Express
const app = Express();

// Ajouter un 'middleware' lit du json dans le body
app.use(json());

// Accrocher les routes 'user' à l'api qui se trouvent
// dans routes/user.route.ts
app.use('/user', ROUTES_USER)

// Server des fichiers statiques
app.use('/public', Express.static(join('assets')));


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

## Middleware

Il arrive que l'on veuille répéter une certaine logique dans un certain nombre routes différents.

Par exemple, l'authentification d'un utilisateur avant l'exécution d'une opération.

Pour ce faire, nous utilisons un "middleware", une fonction intermédiaire qui est appelée avant le endpoint final.

Ajoutez le suivant au debut de notre fichier `user.route.ts` :

{% code title="routes/user.route.ts" lineNumbers="true" %}
```ts
router.use(
  (request: Request, response: Response, next: NextFunction) => {
    console.log("this is a middleware");

    const auth = request.headers.authorization;
    console.log(auth);

    if (!auth || auth !== 'Bearer 12345') {
      next("Unidentified user!");
      return;
    }


    next();
  }
)
```

Cette fonction sera appelé systématiquement avant **tous les endpoints** de ce router !

Middlewares doivent obligatoirement signaler quand ils ont finis :

- en invoquant la fonction `next()`, sans paramètre quand tout s'est bien passé
- en invoquant la fonction `next('message')`, avec un paramètre, quand il y a une erreur. Le paramètre  contient de l'information sur l'erreur 

Si on oublie d'appeler `next()` notre serveur va bloquer dans cette fonction et ne jamais avancer.

{% hint style="success" %}
Mettez en commentaire votre middleware pour le moment, après l'avoir essayé. Pour l'instant, nous ne l'utiliserons pas.
{% endhint %}

## Exercice (facultatif) : Erreurs

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
import { StructuredErrors } from './StructuredErrors';


export class ApiError extends Error {
  constructor(public httpCode: ErrorCode, public structuredError: StructuredErrors, public errMessage: string, public errDetails?: any) {
    super(errMessage)
  }

  get json() {
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

Utilisez le `DefaultErrorHandler` comme middleware sur votre serveur :

```ts
app.use(DefaultErrorHandler);
```

</details>



