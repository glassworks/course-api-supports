# Structurer l'application

Let us Commençons par créer un peu de structure dans notre serveur.
 creating a bit of structure in our server.

Créons quelques repertoires dans `src` :

```
src/
  - routes/
    # Contient nos endpoints
  - controllers/
    # Contient nos controlleurs MVC
  - model/
    # Contient les definitions des données
  - utility/
    - error/
      # Contient les outils pour la bonne gestion d'erreur
    - ORM/
      # Contient les outils pour accéder à la base de données

  - server.ts   
    # Le point d'entrée de notre API    
```

## Raccourcis Typescript

Notre code va souvent importer des classes et des fonctions des dossiers `utility` et `model`. Nous pouvons demander à Typescript de fournir un raccourci que nous pouvons utiliser lors de l'importation à partir de ces dossiers. Modifions `tsconfig.json` :

```json
{
  "compilerOptions": {
    ...
    "baseUrl": "src",                                  /* Specify the base directory to resolve non-relative module names. */
    "paths": {
      "@model/*": ["model/*"],
      "@orm/*": ["utility/ORM/*"],      
      "@error/*": ["utility/error/*"],      
      "@routes/*": ["routes/*"],
      "@controllers/*": ["controllers/*"],
      "@logging/*": ["utility/logging/*"]
    },       
    ...
  }
}
```

Pour utiliser ces chemins avec TypeScript, nous devons installer un paquetage capable de les interpréter : 

```bash
npm install --save-dev tsconfig-paths
```

... et modifiez notre script npm pour utiliser ce module (`package.json`) :

```json
{
  "name": "api",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "server": "nodemon --watch 'src' -r tsconfig-paths/register src/server.ts"
  },
  ...
}
```

## Gestion d'erreurs

Essayez de rentrer des mauvaises informations via Postman :

* Créer un utilisateur doublon
* Essayer de passer un champ qui n'est pas une colonne dans la base
* Passer du texte dans le query param de la requête index (pour limit et offset)
* Afficher, mettre à jour, ou supprimer un utilisateur qui n'existe pas

Pour l'instant, on reçoit un message moche (pas en json) et pas très parlant dans Postman.

Idéalement, pour notre api JSON, nous voulons que les erreurs soient renvoyées en JSON également, et avec des informations qui nous sont utiles !

* Renvoyer plutôt du json
* Meilleure gestion des codes HTTP :  
  * `400` : la requête est mauvaise (erreur dans les données entrantes)
  * `401` : non autorisé
  * `403` : ressource interdit  
  * `404` : élément pas trouvé
  * `500` : erreur interne (dernier recours)
* Imposer une deuxiume couche de code qui explique en plus de détail ce qui ne va pas :  
  * `params-invalid`
  * `connection-error`
  * `auth/unknown-email`


D'abord, on crée une classe qui dérive de la classe générique d'erreur de Javascript, dans `src/utility/error/ApiError.ts` :

```ts
import { ErrorCode } from './ErrorCode';
import { IApiError } from './IApiError';

export class ApiError extends Error {
  constructor(public httpCode: ErrorCode, public structuredError: string, public errMessage: string, public errDetails?: any) {
    super();
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

Cette classe contient notamment une fonction permettant d'exporter l'erreur en format JSON selon l'interface `IApiError` (dans `src/utility/error/IApiError.ts`) : 

```ts
import { ErrorCode } from './ErrorCode';

export interface IApiError {
  code: ErrorCode,
  structured: string,
  message?: string,
  details?: any,
}
```

Le `code` est une énumération des différentes possibilités, dans `src/utility/error/ErrorCode.ts` :

```ts
// Les numéros de d'erreur standard de HTTP
export enum ErrorCode {
  BadRequest = 400,
  Unauthorized = 401,    
  Forbidden = 403,
  NotFound = 404,  
  TooManyRequests = 429,
  InternalError = 500
}
```

Ensuite, nous allons rédiger un **middleware** qui prend 4 paramètres pour qu'Express l'utilise pour gérer des erreurs (dans `src/utility/error/error-handler.middleware.ts`) :


```ts
import { NextFunction, Request, Response } from 'express';
import { ApiError } from './ApiError';
import { ErrorCode } from './ErrorCode';


export const DefaultErrorHandler = async (error: any, req: Request, res: Response, next: NextFunction) => {
  
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

Utilisez le `DefaultErrorHandler` comme middleware sur votre serveur. Ajoutez la ligne suivante dans `server.ts `:

```ts
import { DefaultErrorHandler } from "@error/error-handler.middleware";

...
// Créer l'objet Express
const app = Express();
...

// Ajouter un handler pour les erreurs
app.use(DefaultErrorHandler);

// Lancer le serveur
app.listen(PORT,
  () => {
    console.info("API Listening on port " + PORT);
  }
);
```

## Un ORM basique

Pour cette démonstration, l'accès à la base de données se fera manuellement afin d'exposer spécifiquement les communications qui ont lieu, et sans abstraire les données derrière des interfaces spécifiques.

Si vous souhaitez utiliser un ORM standard, de nombreuses options existent, telles que TypeORM, Prisma, Sequelize, ... 

### Le schema

Pour notre ORM simple, nous allons spécifier le schéma de la base de données et tous les types associés dans `src/model` :

```
src/model/
  - schema/
    # Contient les fichiers SQL qui permettent d'initialiser le schéma de la base de données
    - ddl.sql
    - init.sql
  - types
    # Contient les définitions Typescript qui corréspondent aux tables de la base de données
  - DbTable.ts
    # Contient une énumération des noms des tables dans la base de données
   
```

Nous centralisons les schemas SQL de notre base de données. En premier, un SQL qui permet de créer une base de données et donner accès à un utilisateur (`src/model/schema/init.sql`):

```sql
/*
Script de création de la base de données `school`.
*/
create database IF NOT EXISTS school;

/* Créer l'utilisateur API */
create user IF NOT EXISTS 'api-dev'@'%.%.%.%' identified by 'api-dev-password';
grant select, update, insert, delete on school.* to 'api-dev'@'%.%.%.%';
flush privileges;
```

Ensuite, le DDL (Data Definition Language) qui implémente notre schema dans MariaDB  (`src/model/schema/ddl.sql`)::

```sql

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
```

Au fur et à mesure que vous ajouterez des tables à votre base de données, vous retournerez ce fichier et le mettrez à jour.

Pour jouer avec Typescript, nous allons également créer des définitions de type pour chaque table de la base de données, par exemple, pour la table `user` (dans `src/model/types/IUser.ts`) :

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

export type IUserCreate = Omit<IUser, 'userId'>;

export type IUserUpdate = Partial<IUserCreate>;
```

Cela signifie qu'une fois que les données sont extraites de la base de données, elles seront correctement typées et que Typescript validera au moment de la compilation les fautes de frappe que nous aurions pu commettre.

Enfin, nous voulons créer une énumération des tables disponibles dans la base de données (dans `src/model/DbTable.ts`) :


```ts
export type DbTable = 
  'user'
;
```

Chaque fois que nous ajouterons une table, nous ajouterons son nom ici. Cela nous permet d'éviter les fautes de frappe.

## Le ORM

Nous allons mettre en œuvre un ensemble d'opérations CRUD simples qui sont sans danger pour les types. Voici un ensemble d'interfaces Typescript définissant les différentes requêtes et réponses de notre ORM (`src/utility/ORM/interfaces/IORM.ts`) :

```ts
import { DbTable } from "@model/DbTable";

/// --- CREATE --- ///

/**
 * Requête pour la création d'une ligne
 */
export interface IORMCreateRequest<T> {
  table: DbTable;
  body: T;
}

/**
 * Réponse à une operation d'insertion d'une ligne
 */
export interface IORMCreateResponse {
  /**
   * ID de la ligne créée
   */
  id: number;
}


/// --- READ --- ///

/**
 * Paramètres modifiant une requête (pour la pagination)
 */
export interface IORMIndexQueryParams {
  page?: string;
  limit?: string;  
}

/**
 * Block de conditions sur la requête aupres de la ba de données
 */
export type IORMReadWhere = Record<string, string|number>;


/*
 * Requête pour récupérer un ensemble de lignes
 */
export interface IORMIndexRequest {
  /**
   * La table de la base de données à interroger
   */  
  table: DbTable; 
  /**
   * Un tableau de colonnes à retourner
   */
  columns: string[];
  /**
   * Comment filtrer les lignes
   */  
  where?: IORMReadWhere;
  /**
   * Comment gérer la pagination
   */
  query?: IORMIndexQueryParams;
}

/**
 * Réspone à une opération de lecture de plusieurs lignes.
 */
export interface IORMIndexResponse<T> {
  page: number;
  limit: number;
  total: number;
  rows: T[];
}

export interface IORMReadRequest {
  table: DbTable;
  /**
   * Le nom de la colonne qui contient la clé primaire ou ID de la ligne demandée
   */
  idKey: string;
  /**
   * La valeur de l'ID
   */
  idValue: number|string;
  columns: string[];
}

/**
 * Structure retourné par MySQL quand on fait une requête de type `count(*)` 
 */
export interface IORMTableCount {
  total: number;
}

/// --- UPDATE --- ///

export interface IORMUpdateRequest<T> {  
  table: DbTable;
  /**
   * Le nom de la colonne qui contient la clé primaire ou ID de la ligne demandée
   */
  idKey: string;
  /**
   * La valeur de l'ID
   */
  idValue: number|string;

  /**
   * La mise à jour à effectuer
   */
  body: T;
}

/**
 * Réponse à une operation de mise à jour
 */
export interface IORMUpdateResponse {
  id: number|string;
  rows: number;
}

/// --- DELETE --- ///

export interface IORMDeleteRequest {
  table: DbTable, 
  /**
   * Le nom de la colonne qui contient la clé primaire ou ID de la ligne demandée
   */
  idKey: string, 
  /**
   * La valeur de l'ID
   */
  idValue: number|string
}

/**
 * Réponse à une operation de suppression d'une ligne
 */
export interface IORMDeleteResponse {
  id: number|string;
  rows: number;
}
```

Nous fournissons l'implémentation simple suivante dans `src/utility/ORM/ORM.ts` :


```ts
import { ApiError } from "@error/ApiError";
import { ErrorCode } from "@error/ErrorCode";
import { DbTable } from "@model/DbTable";
import { ResultSetHeader, RowDataPacket } from "mysql2";
import { DB } from "./DB";
import { IORMCreateResponse, IORMIndexRequest, IORMIndexResponse, IORMCreateRequest, IORMTableCount, IORMUpdateResponse, IORMReadRequest, IORMUpdateRequest } from "./interfaces/IORM";


/**
 * Class qui fournit des fonctions utilitaires pour les opérations ICRUD.
 */
export class ORM {

  /**
   * Ajoute une ligne dans la base de données
   * @param options 
   * @returns ICreateResponse
   */
  public static async Create<T>(options: IORMCreateRequest<T>): Promise<IORMCreateResponse> {
    const db = DB.Connection;
    const data = await db.query<ResultSetHeader>(`insert into ${options.table} set ?`, options.body);

    return {
      id: data[0].insertId
    }   
  }

  /**
   * Récupérer une page de lignes d'une table, en précisant les colonnes souhaitées   
   */
  public static async Index<T>(options: IORMIndexRequest) : Promise<IORMIndexResponse<T>> {

    const db = DB.Connection;      
    // On suppose que le params query sont en format string, et potentiellement
    // non-numérique, ou corrompu
    const page = parseInt(options.query?.page || "0") || 0;
    const limit = parseInt(options.query?.limit || "10") || 0;
    
    const offset = page * limit;

    // D'abord, récupérer le nombre total
    let whereClause = '';
    let whereValues: any[] = [];
    if (options.where) {
      const whereList: string[] = [];
      Object.entries(options.where).forEach(
        ([key, value]) => {
          whereList.push(key + ' = ?');
          whereValues.push(value);
        }
      )
      whereClause = 'where  ' + whereList.join(' and ');
    }

    // console.log(mysql.format(`select count(*) as total from ${table} ${whereClause}`, whereValues))

    const count = await db.query<IORMTableCount[] & RowDataPacket[]>(`select count(*) as total from ${options.table} ${whereClause}`, whereValues);      

    // Récupérer les lignes
    const sqlBase = `select ${options.columns.join(',')} from ${options.table} ${whereClause} limit ? offset ?`;
    const data = await db.query<T[] & RowDataPacket[]>(sqlBase, [...whereValues, limit, offset].filter(e => e !== undefined));

    // Construire la réponse
    const res: IORMIndexResponse<T> = {
      page,
      limit,
      total: count[0][0].total,
      rows: data[0]
    }

    return res;
  }

  /**
   * Récupérer une ligne dans la base de données, étant donné son identifiant
   * @param options 
   * @returns 
   * @throws ApiError si aucune ligne n'est trouvée
   */
  public static async Read<T>(options: IORMReadRequest): Promise<T> {
    const db = DB.Connection;
    const data = await db.query<T[] & RowDataPacket[]>(`select ${options.columns.join(',')} from ${options.table} where ${options.idKey} = ?`, [options.idValue]);      

    if (data[0].length > 0) {
      return data[0][0];
    } else {
      throw new ApiError(ErrorCode.BadRequest, 'sql/not-found', `Could not read row with ${options.idKey} = ${options.idValue}`);
    }
  }



  public static async Update<T>(options: IORMUpdateRequest<T>): Promise<IORMUpdateResponse> {
    const db = DB.Connection;

    const data = await db.query<ResultSetHeader>(`update ${options.table} set ? where ${options.idKey} = ?`, [options.body, options.idValue]);

    return {
      id: options.idValue,
      rows: data[0].affectedRows
    } 
  }
  

  public static async Delete(options: {
    table: DbTable, 
    idKey: string, 
    idValue: number|string
  }): Promise<IORMUpdateResponse> {
    const db = DB.Connection;
    const data = await db.query<ResultSetHeader>(`delete from ${options.table} where ${options.idKey} = ?`, [options.idValue]);      

    return {
      id: options.idValue,
      rows: data[0].affectedRows
    }  
  }

}
```