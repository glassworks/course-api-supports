# Validation des données

Aujourd'hui avec la programmation réactive, on a tendance à faire beaucoup de validation des données dans le client (le navigateur, par exemple) :

* est-ce qu'une adresse email est correctement formatée ?
* est-ce qu'une valeur numérique est passée (au lieu d'une chaîne de caractères) ?
* est-ce que **toutes** les données attendues sont envoyées ?
* est-ce qu'il y a des champs supplémentaires dans les requêtes dont on ne veut pas ?
* 
{% hint style="warning" %}
On ne fait **jamais** confiance au client ! Même si c'est un client développé en interne.

Pourquoi ? Puisque le client tourne sur la machine de l'utilisateur, où on ne contrôle pas l'environnement d'exécution ni l'utilisateur même (est-ce qu'il aurait reverse-ingénieur notre client, par exemple ?)
{% endhint %}

On peut dépendre de notre schéma de base de données et des contraintes pour faire une partie de notre travail :

* des attributs `NOT NULL`
* des types sur les attributs
* des contraints sur les attributs (`unique`)

Mais, il n'est pas toujours possible d'exprimer tout ce qu'on veut via ce schéma. D'autant plus, parfois, on attend des valeurs qui n'iront pas directement dans la base de données. Idéalement, on pourrait valider leur présence et type quand même.

## Validation avec AJV

Une librairie (parmi plusieurs) sur NodeJS de validation des données s'appelle [`ajv`](https://www.npmjs.com/package/ajv) qui permet d'utiliser Typescript pour créer des schémas de validation.

```bash
npm install ajv
```

### Types

On commence par renforcer nos types Typescript de nos modèles :

```ts
// Définition d'un structure IUser
// A noter, le ? veut dire que le champ est optionnel

export interface IUser {
  userId: number;
  familyName?: string;
  givenName?: string;
  email: string;
}

export type IUserRO = Readonly<IUser>;

export type IUserCreate = Omit<IUser, 'userId'>;

export type IUserUpdate = Partial<IUserCreate>;
```

Notez qu'ici, on crée 2 types de plus :

* Un type pour la **création** d'un utilisateur avec les champs obligatoires (`email`), mais en excluant (via l'opérateur `Omit`) le `userId` (car le SGBDR va s'occuper de sa création)
* Un type pour la **update**, ou tous les champs sont optionnels

Ensuite, avec l'aide de AJV, on peut créer les structures de validation :

```ts
import Ajv, { JSONSchemaType } from "ajv";
import { IUserCreate, IUserUpdate } from "./IUser";

const UserCreateSchema : JSONSchemaType<IUserCreate> = {
  type: "object",
  properties: {
    familyName: { type: 'string', nullable: true },
    givenName: { type: 'string', nullable: true},
    email: { type: 'string' },  
  },
  required: ["email"],
  additionalProperties: false,
};

const UserUpdateSchema : JSONSchemaType<IUserUpdate> = {
  type: "object",
  properties: {
    familyName: { type: 'string', nullable: true },
    givenName: { type: 'string', nullable: true },
    email: { type: 'string', nullable: true },  
  },  
  additionalProperties: false,
};

const ajv = new Ajv();
export const UserCreateValidator = ajv.compile(UserCreateSchema);
export const UserUpdateValidator = ajv.compile(UserUpdateSchema);
```

Les deux valeurs exportées sont des fonctions qu'on peut utiliser qui vont valider une structure entrante.

On peut, par exemple, utiliser ces validateurs dans notre route `/user`. La route `POST /user` pour la création d'un utilisateur pourrait être le suivant :

```ts
routerIndex.post<{}, ICreateResponse, IUser>('',
  async (request, response, next: NextFunction) => {

    try {
      const user = request.body;

      // Valider la structure de création
      if (!UserCreateValidator(user)) {
        throw new ApiError(ErrorCode.BadRequest, 'validation/failed', 'Data did not pass validation', UserCreateValidator.errors);      
      }

      const result = await Crud.Create(user, 'user');
      
      response.json(result);

    } catch (err: any) {
      next(err);
    }

  }
);
```

Testez vous-même la route en omettant, par exemple, l'adresse e-mail. Vous allez voir qu'un message est émis qui contient toutes les informations concernant les données manquantes ou erronées.

Un exemple complet qui ajoute de la validation sur les endpoints de création et mise à jour d'un `User` se trouve ici : [https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/003-data-validation](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/003-data-validation)

