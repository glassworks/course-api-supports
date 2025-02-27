# Sécurité : implémentation avec JWT

Nous utiliserons des JWT (jetons web JSON) pour mettre en œuvre notre protocole d'identification et d'autorisation.

Un JWT est un document signé numériquement qui contient :
- un objet JSON non chiffré contenant certains champs standardisés ainsi que des champs personnalisés
- la version chiffrée de l'objet JSON ci-dessus, chiffrée à l'aide de la clé privée de l'émetteur. Cela crée une signature numérique qui prouve que l'émetteur est la seule personne à pouvoir créer ce JWT.

Les différentes parties sont sérialisées dans un string en base64.

Lors de la connexion, notre API créera un JWT et le signera à l'aide de notre clé privée secrète. Nous le transmettons à l'utilisateur.

Nous attendons de l'utilisateur qu'il nous envoie ce JWT à chaque demande. Comme nous l'avons signé, nous pouvons prouver l'authenticité de la clé transmise en essayant de la décoder à l'aide de la clé publique correspondante. Si le résultat est identique à la partie JSON non chiffrée, nous avons prouvé qu'il s'agit bien d'un JWT que nous avons émis. Dans le cas contraire, la clé est fausse et nous rejetons la demande.

La partie JSON du JWT contient un champ d'expiration, ce qui signifie que le JWT n'est valable que pour une période spécifique. Il devra être réédité régulièrement.

{% hint style="warning" %}

All fields in a JWT are plain-text and can be read by anyone! Never send sensitive information via a JWT!

{% endhint %}

{% hint style="warning" %}

Étant donné que le JWT est envoyé avec chaque demande, faites attention à la quantité d'informations que vous y mettez ! En général, vous n'incluez que l'identifiant de l'utilisateur et éventuellement son rôle.

{% endhint %}


## Identity flow

Mettons en place un flux d'identité sans mot de passe. 
- Un utilisateur s'identifie à l'aide de son adresse électronique. Nous aurons besoin d'un point de terminaison tel que `POST /auth/login` pour cela
- Ce point de terminaison vérifie la base de données pour cet utilisateur, et si l'utilisateur existe, crée un JWT temporaire qui est envoyé à l'adresse email de l'utilisateur. Le JWT doit expirer après un court laps de temps. De plus, il s'agit d'un JWT spécial qui ne peut pas être utilisé pour accéder à des routes protégées dans notre API !
- L'utilisateur consulte son adresse électronique. Il y trouve un lien qui le dirige vers une route d'autorisation (`GET /auth/authorize?jwt={{JWT}}`) qui validera le JWT. Si le JWT est validé et n'a pas expiré, la route renverra un nouveau JWT, cette fois-ci un JWT qui PEUT être utilisé pour autoriser un utilisateur dans notre API (jeton d'accès)


## Clés asymétriques

Notre API va émettre et signer des JWT. Pour signer un JWT, nous avons besoin d'une paire de clés cryptographiques asymétriques.

Naviguez dans le dossier `config/signing` (créez le dossier s'il n'existe pas). C'est là que nous conserverons nos identifiants de signature.

Utilisez les commandes suivantes pour générer une paire de clés à utiliser pour signer les JWT :

```
ssh-keygen -t rsa -b 2048 -m PEM -f signing.key
openssl rsa -in signing.key -pubout -outform PEM -out signing.pub
```

Il existe maintenant une paire de clés asymétriques :
- une clé publique `signing.pub`
- une clé privée `signing.key` : cette clé doit être protégée !

## Outil JWT

Ensuite, créons un outil qui créera et validera les JWTs. Tout d'abord, vous aurez besoin de la bibliothèque `jsonwebtoken`: 

```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```


Ensuite, dans `src/utility/JWT/JWT.ts` :


```ts
import { ApiError } from '@error/ApiError';
import { ErrorCode } from '@error/ErrorCode';
import { readFileSync } from 'fs';
import { JwtPayload, sign, SignOptions, TokenExpiredError, verify, VerifyOptions } from 'jsonwebtoken';
import { join } from 'path';

export class JWT {

  private static PRIVATE_KEY: string;
  private static PUBLIC_KEY: string;

  constructor() {
    if (!JWT.PRIVATE_KEY) {
      JWT.PRIVATE_KEY = readFileSync(process.env.PRIVATE_KEY_FILE || join('config', 'signing', 'signing.key'), 'ascii')
    }

    if (!JWT.PUBLIC_KEY) {
      JWT.PUBLIC_KEY = readFileSync(process.env.PUBLIC_KEY_FILE || join('config', 'signing', 'signing.pub'), 'ascii')
    }
  }

  async create<T extends object>(payload: T, options: SignOptions) {
    return new Promise<string>(
      (resolve, reject) => {
        sign(payload, JWT.PRIVATE_KEY, Object.assign(options, { algorithm: 'RS256' }), (err: any, encoded) => {
          if (err) {
            reject(err);
            return;
          }
          resolve(encoded!);
        });
      }
    )
  }

  async decodeAndVerify<T extends JwtPayload>(token: string, options: VerifyOptions) {
    return new Promise<T>(
      (resolve, reject) => {
        verify(token, JWT.PUBLIC_KEY, Object.assign(options, {
          algorithms: ['RS256']
        }), (err: any, decoded) => {
          if (err) {
            if (err instanceof TokenExpiredError) {
              reject(new ApiError(ErrorCode.Unauthorized, 'token/expired', 'Token expired'))
            } else {              
              reject(new ApiError(ErrorCode.Unauthorized, 'token/invalid', 'Token invalid'))
            }
            return;
          }
          resolve(decoded as T);
        })
      }
    )
  }
}
```

There are multiple points to note: 
- we first use environnement variables for recovering our signing keys, falling back to local stored ones on the disk. This allows for providing a unique set for use only in production
- Our key uses RS256 encryption. You may use other, stronger encryption algorithms such as ES256, but you must generate signing keys that correspond.
- We throw 401 errors if the JWT could not be validated, but providing more information about whether the token was expired  (`token/expired`) or just invalid (`token/invalid`).

## Mailing

Pour notre flux d'identité, nous avons besoin d'un moyen d'envoyer un e-mail à notre utilisateur contenant le JWT.

De nos jours, je vous conseille d'utiliser un service d'envoi tiers qui s'occupe de tous les problèmes de délivrabilité que vous pouvez rencontrer. 

[Mailjet](https://www.mailjet.com), par exemple, fournit ce type de service à l'aide d'une API. 

Je vous laisse créer votre compte gratuit. Je vous laisse créer votre compte gratuit et commencer les premières étapes. Vous aurez besoin d'obtenir vos clés API afin d'envoyer un email via leurs API.


```bash
npm install node-mailjet
```

@todo !

## Controleur d'identification

Nous pouvons maintenant écrire notre contrôleur qui implémente les routes dont nous avons parlé précédemment.

Tout d'abord, nous allons définir un type pour la charge utile de notre jeton d'accès, dans `src/utility/auth/IAccessToken.ts` :

```ts
export interface IAccessToken {
  userId: number
}
```

Nous créons ensuite notre contrôleur, dans `src/controllers/AuthController.ts` :

```ts
import { ApiError } from "@error/ApiError";
import { ErrorCode } from "@error/ErrorCode";
import { IUserRO } from "@model/types/IUser";
import { ORM } from "@orm/ORM";
import { Body, Get, Post, Query, Route } from 'tsoa';
import { IAccessToken } from "utility/auth/IAccessToken";
import { Emailer } from "utility/email/Emailer";
import { JWT } from "utility/JWT/JWT";
import { JWT_ACCESS_AUD, JWT_EMAIL_LINK_AUD, JWT_ISSUER } from "utility/JWT/JWTConstants";

@Route("/auth")
export class AuthController {
  
  @Post("/login")
  public async sendMagicLink(  
    @Body() body: {
      /**
       * Identifiant de l'utilisateur.
       */
      email: string;
    }
  ): Promise<{ ok: boolean}> {    
    // Vérifier si on a un utilisateur avec l'adresse email dans notre base
    const user = await ORM.Read<IUserRO>({
      table: 'user',
      idKey: 'email',
      idValue: body.email,
      columns: ['userId', 'email']
    });

   
    // Create the new JWT
    const jwt = new JWT();
    const encoded = await jwt.create({
      userId: user.userId,
    }, {
      expiresIn: '30 minutes',
      audience: JWT_EMAIL_LINK_AUD,
      issuer: JWT_ISSUER
    }) as string;
    
    const emailer = new Emailer();

    const link = (process.env.FRONT_URL || 'http://localhost:' + (process.env.PORT || 5050)) + '/auth/authorize?jwt=' + encodeURIComponent(encoded);
    await emailer.sendMagicLink(body.email, link, 'Mon service');

    return {
      ok: true
    };
  }

  @Get("/authorize")
  public async authorizeFromLink(  
    @Query() jwt: string
  ): Promise<{ 
    access: string;
    redirectTo: string;
    message: string;
  }> {    
        
    const helper = new JWT();
    const decoded = await helper.decodeAndVerify(jwt, {
      issuer: JWT_ISSUER,
      audience: JWT_EMAIL_LINK_AUD,
    });

    if (!decoded.userId) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/invalid-authorize-link-token', "userId was not found in the payload for token");
    }

    // Vérifier que l'utilisateur existe toujours
    const user = await ORM.Read<IUserRO>({
      table: 'user',
      idKey: 'userId',
      idValue: decoded.userId,
      columns: ['userId']
    });

    let payload: IAccessToken = {
      userId: user.userId
      /** @todo: Ajouter des rôle(s) ici ! */
    };    

    const access = await helper.create(payload, {
      expiresIn: '12 hours',
      issuer: JWT_ISSUER,
      audience: JWT_ACCESS_AUD,
    }) as string;

    return {
      access: access,
      redirectTo: 'https://lien.vers.mon.front',
      message: 'Normalement ce endpoint va demander au navigateur de rediriger vers votre site ou ressource'
    };
  }

}
```

N'oubliez pas d'exécuter le suivant pour mettre à jour les routes :

```bash
npm run compile
```

Testez vos itinéraires avec Postman ! Obtenez-vous un JWT ?

Lorsque vous obtenez un JWT, vous pouvez le déboguer sur [jwt.io](https://jwt.io)


## Sécuriser les endpoints

Au lieu de créer un middleware *ad-hoc* comme on a fait avant, `tsoa` préconise un emplacement fixe pour la logique de note notre sécurisation. Cet emplacement est défini dans `tsoa.json`, notamment la ligne `authenticationModule`

```json
{
  ...
  "routes": {
    "routesDir": "./src/routes",
    "authenticationModule": "./src/utility/auth/authentication.middleware.ts"
  },
  ...
}
```

Il faut donc ajouter un fichier qui s'occupe de l'autorisation, dans `/src/utility/auth/authentication.middleware.ts` :

```ts
import { ApiError } from '@error/ApiError';
import { ErrorCode } from '@error/ErrorCode';
import { Request } from 'express';
import { JWT } from 'utility/JWT/JWT';
import { JWT_ACCESS_AUD, JWT_ISSUER } from 'utility/JWT/JWTConstants';
import { IAccessToken } from './IAccessToken';

export async function expressAuthentication(
  request: Request,
  securityName: string,
  scopes?: string[]
): Promise<IAccessToken|null> {

  if (securityName === 'jwt') {
    const authheader = request.headers.authorization || '';
    if (!authheader.startsWith('Bearer ')) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/missing-header', 'Missing authorization header with Bearer token');
    }

    const token = authheader.split('Bearer ')[1];

    const jwt = new JWT();
    let decoded = await jwt.decodeAndVerify<IAccessToken>(token, {
      issuer: JWT_ISSUER,
      audience: JWT_ACCESS_AUD,
    });
    
    return decoded;
  }

  return null;
}
```

Vous remarquerez que `tsoa` permet de définir différentes stratégies d'autorisation, et contient aussi la logique pour les *scopes* (qu'on n'utilise pas dans notre exemple).

Ensuite, pour protéger nos routes, il suffit d'ajouter le décorateur `@Security` devant une classe ou méthode :

```ts
@Route("/user")
@Security('jwt')
export class UserController {

  /**
   * Récupérer une page d'utilisateurs.
   */
  @Get()
  public async getUsers(
```

Après une recompilation (`npm run compile`), la route est désormais sécurisée par notre authentification pas JWT. 

Essayez la route `GET /user` après avoir fait le changement ci-dessus. Quel type d'erreur obtenez-vous ? Pouvez-vous configurer postman pour utiliser votre nouveau jeton ?

Vous trouverez de plus amples informations sur la configuration des rôles à l'adresse suivante : [Authentication with tsoa](https://tsoa-community.github.io/docs/authentication.html)

## Renouvellement du jeton d'accès

Notre jeton d'accès a également une date d'expiration. Celle-ci est généralement assez courte, de l'ordre de 5 minutes. 

Pourquoi si peu de temps ? Si quelqu'un vole le jeton, ou si nous supprimons l'accès à cet utilisateur, nous aurons limité les dommages possibles à une courte période de temps.

Mais après 5 minutes, que se passe-t-il ?

Nous devons renouveler le jeton, mais si le jeton original est expiré, comment faire ?

Nous utilisons un jeton de rafraîchissement (**refresh token**). Il s'agit d'un jeton spécial qui ne peut être utilisé que sur un seul endpoint de notre API `POST /auth/renew` afin d'obtenir un nouveau jeton d'accès.

Le refresh-token est émis en même temps que le jeton d'accès, mais n'est pas transmis à chaque demande (il est donc plus privé, moins de risque d'être volé). Son délai d'expiration est plus long (généralement 1 semaine). 

Le endpoint `/auth/renew` valide le refresh-token, valide (par une requête auprès de la base de données) que l'utilisateur est valide, extrait (par une requête auprès de la base de données) les rôles mis à jour de l'utilisateur, et réémet le jeton d'accès (ainsi qu'un nouveau refresh token au passage).

Remarque : un refresh-token ne peut pas autoriser un utilisateur sur un autre point de terminaison de l'API !

Je vous laisse vous exercer à la mise en œuvre de ce refresh-token.