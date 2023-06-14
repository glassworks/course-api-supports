# Intégration du SGBDR

On veut maintenant établir une connexion à une base de données, et commencer à ajouter et supprimer les lignes à travers de notre API.

## Dev Container et `docker-compose.yml`

On peut directement inclure une instance d'un SGBDR à notre dev-container grace à docker :

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
      - ./dbms/mariadb.cnf:/etc/mysql/mariadb.cnf
    networks:
      - api-network
```

Il faut faire le suivant :
* Ajouter les lignes dessus à `docker-compose.dev.yml`
* Ajouter le dossier `dbms` 
* Ajouter le fichier `mariadb.cnf` dans le dossier `dbms`. [Trouver un exemple ici](https://docs.glassworks.tech/sgbdr/operations/001-operations-docker#options-de-production-mariadb)
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
Script de création de la base de données de test.
A noter, on utilise uns stratégie avec DROP et IF NOT EXISTS afin de rendre 
notre script réutilisable dans le future, même si la base existe déjà
*/
create database IF NOT EXISTS test;

/* Créer l'utilisateur API */
create user IF NOT EXISTS 'api-dev'@'%.%.%.%' identified by 'api-dev-password';
grant select, update, insert, delete on test.* to 'api-dev'@'%.%.%.%';
flush privileges;

/* La définition de la schéma */
use test;

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

## Integration NodeJS

Nous utilisons la librairie `mysql2` pour communiquer avec notre base de données.

Normalement notre API va ouvrir une connexion unique auprès du SGBDR pour chaque requête en cours. Ceci peut-être lourd et longue, donc le créateur de la librairies à prévu les **connection pools**. C'est à dire, on va essayer de réutiliser les connexions déjà ouvertes.

Moi je préfère créer une classe qui enveloppe l'objet principal, pour ne pas répéter du code :

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
        database: process.env.DB_DATABASE || 'test',
        password: process.env.DB_PASSWORD || 'api-dev-password',  
      });
    }

    return this.POOL;
  }

}
```

Ici, on crée une variable static, et on initialise notre **pool** avec les coordonnées tirés de l'environnement (ou des valeurs par défaut).

## Opérations CRUD

Voici un exemple d'un set de **endpoints** pour la gestion de l'utilisateur (code source entière disponible [ici](https://dev.glassworks.tech:18081/courses/api/api-supports/-/tree/master/exemples/mysql)).

```ts
import { NextFunction, Request, Response, Router } from "express";
import { OkPacket, RowDataPacket } from 'mysql2';
import { DB } from '../../classes/DB';
import { ICreateResponse } from '../types/ICreateResponse';
import { IIndexQuery, IIndexResponse } from '../types/IIndexQuery';
import { ITableCount } from '../types/ITableCount';
import { IUser, IUserRO } from '../types/IUser';



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


Il y a plusieurs choses à noter, comme discuté dans les sections suivantes.

### L'utilisation des types

Une force de Typescript est le fait de pouvoir contrôler les types du début à la fin, même avec les **génériques*. 

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


```ts
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
```


### Le formatage des requêtes SQL

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

# Exercice 1 : CRUD

Complétez les autres fonctions CRUD pour un utilisateur : 
* `GET /user/<userId>` : récupérer juste la ligne de l'utilisateur en format `IUser`
* `PUT /user/<userId>` : mettre à jour une ligne précise
* `DELETE /user/<userId>` : supprimer l'utilisateur


# Exercice 2 : Erreurs

Essayez de rentrer des mauvaises informations via Postman :
* Créer un utilisateur doublon
* Essayer de passer un champ qui n'est pas une colonne dans la base
* Passer du texte dans le query param de la requête index (pour limit et offset)
* Afficher, mettre à jour, ou supprimer un utilisateur qui n'existe pas

Pour l'instant, on reçois un message moche (pas en json) et pas très parlant dans Postman.

Ajoutez un middleware qui gère les erreurs :
* Renvoyez plutôt du json
* Avoir au moins le champs suivants :
  * `code` : le code HTTP approprié
    * `400` : la requête est mauvaise (erreur dans les données entrantes)
    * `401` : pas autorisé
    * `404` : élément pas trouvé
    * `500` : erreur interne (dernier recours)
  * `structured` : un champ plus libre mais plus structuré qui permettrait de localiser l'erreur coté front. Exemple : 
    * `params-invalid` 
    * `connection-error`
    * `auth/unknown-email`
  * `message`: un message humain décrivant l'erreur
  * `data`: (optionnel) les données supplémentaires concernant l'erreur

Astuce : il faut dire à express d'utiliser votre handler d'erreur avec `app.use(...)`. Votre handler doit avoir 4 paramètres dans le callback, le premier étant l'objet d'erreur.

# Exercice 3 : Factoring

Essayez d'ajouter une autre table à votre base, e.g. `repas` (vous pouvez vous inspirer de l'application nutrition).

Ajoutez des endpoints CRUD pour cette table. 

Essayez au maximum de factoriser votre code. Est-ce qu'il y a des éléments qu'on peut réutiliser pour les opération CRUD classiques ?
