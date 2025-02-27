# Préparations pour la production

Nous voulons préparer notre code pour des déploiements prêts pour la production.

## CORS

Les navigateurs modernes renforcent la sécurité en empêchant, par défaut, les requêtes adressées à des domaines différents de celui de la page web actuelle.

Le cas typique de sécurité est le suivant : quelqu'un partage une image sur Facebook, mais l'image est stockée sur Amazon. Le navigateur détectera que la demande est envoyée à un domaine différent de la page web et la bloquera. Cela nous protège généralement des requêtes malveillantes adressées à des domaines qui n'ont rien à voir avec la page que nous visitons.

Dans certains cas, cependant, nous voulons l'autoriser. Pour ce faire, nous utilisons le système CORS (cross-origin resource sharing). 

CORS est mis en œuvre simplement à l'aide d'en-têtes HTTP.

Le navigateur vérifie si un serveur autorise les requêtes provenant d'un domaine différent du sien. Pour ce faire, il utilise une requête *preflight*, une requête qui ne s'exécute pas complètement, mais juste assez pour obtenir les en-têtes de la réponse (et donc la politique CORS).

Le serveur doit analyser l'origine de la demande de contrôle en amont et définir l'en-tête approprié.

En l'absence d'en-têtes CORS ou d'en-têtes limitant les domaines à ceux autres que la page en cours, le navigateur signale une erreur et les demandes ultérieures sont rejetées.

La validation CORS est donc appliquée côté navigateur, mais la configuration est effectuée côté serveur !

Notez également que la validation CORS n'est effectuée que par les navigateurs, et non par d'autres clients tels que CURL ou Postman. Votre API peut fonctionner avec ces deux derniers clients, mais être rejetée par un navigateur.

Enfin, le serveur ne signale jamais d'erreurs CORS. Tout ce que fait le serveur, c'est ajouter les en-têtes appropriés en fonction de sa configuration.

For our API, we wish to simply allow cross-origin requests. 

```bash
npm install cors
npm install --save-dev @types/cors
```

Dans `src/server.ts` :

```ts
import Cors from 'cors';
...
// Créer l'objet Express
const app = Express();

// Configurer CORS
app.use(Cors())
...
```

The default cors configuration allows all origins, according to the [documentation](https://expressjs.com/en/resources/middleware/cors.html) : 

```json
{
  "origin": "*",
  "methods": "GET,HEAD,PUT,PATCH,POST,DELETE",
  "preflightContinue": false,
  "optionsSuccessStatus": 204
}
```

Notez que d'autres services d'hébergement tels qu'Amazon, Google Cloud, etc. vous demanderont de définir une politique CORS afin d'autoriser les requêtes d'origine croisée sur les baquets de stockage.


## Information sur l'API

Il est pratique d'avoir un endpoint non sécurisé qui renvoie simplement des informations sur l'API. Lorsque nous déployons notre API, nous voulons simplement savoir si le processus est vivant.

Ajoutons un contrôleur qui renvoie simplement des informations sur l'état du processus de l'API, dans `src/controllers/InfoController.ts` :


```ts
import { DB } from '@orm/DB';
import { IORMTableCount } from '@orm/interfaces/IORM';
import { RowDataPacket } from 'mysql2';
import { hostname, platform, type } from 'os';
import { Get, Route } from 'tsoa';

interface IInfo {
  /**
   * Nom de l'API
   */
  title: string;
  /**
   * Le nom d'hôte sur lequel l'API tourne
   */
  host: string;
  /**
   * Le type de OS 
   */
  platform: string;
  /**
   * Le OS 
   */
  type: string;
  /**
   * Le statut de l'OS
   */
  database: {
    state: 'connected'|'disconnected';
    error?: string;
  }
}

@Route("/info")
export class InfoController {

  /**
   * Récupérer une page d'utilisateurs.
   */
  @Get()
  public async getInfo(   
  ): Promise<IInfo> {    
    const info: IInfo = {
      title: "Code Samples API",
      host: hostname(),
      platform: platform(),
      type: type(),
      database: {
        state: 'disconnected'
      }
    }

    try  {
      const db = DB.Connection;
      await db.query<IORMTableCount[] & RowDataPacket[]>(`select count(*) as total from user`);
      info.database.state = 'connected';
    } catch (err: any) {
      info.database.error = err?.message || 'Database could not be contacted';
    }

    return info;
  }
}
```

Le contrôleur ne donne pas seulement des informations sur l'API, mais nous indique également si la base de données est connectée et accessible ou non.

## Remanier notre point d'entrée

Nous voulons mieux contrôler le démarrage et l'arrêt de notre serveur. Lorsqu'un signal d'arrêt est reçu du système d'exploitation (ou du conteneur), nous voulons attendre que les requêtes en cours se terminent et fermer notre connexion à la base de données de manière élégante.

Refactorisons le démarrage et l'arrêt de notre serveur dans un autre fichier, appelé `src/server_manager.ts` :

```ts
import { DefaultErrorHandler } from "@error/error-handler.middleware";
import { Log } from '@logging/Log';
import { requestLogMiddleware } from "@logging/log.middleware";
import { initGraphQL } from "@routes/graphql.route";
import { json } from "body-parser";
import Express, { NextFunction, Request, Response } from "express";
import { createServer, Server } from "http";
import swaggerUi from 'swagger-ui-express';
import { RegisterRoutes } from './routes/routes';
import Cors from 'cors';


export const StartServer = async () => {
  // Récupérer le port des variables d'environnement ou préciser une valeur par défaut
  const PORT = process.env.PORT || 5050;

  // Créer l'objet Express
  const app = Express();
  const httpServer = createServer(app);

  // Configurer CORS
  app.use(Cors())

  // L'appli parse le corps du message entrant comme du json
  app.use(json());

  // Utiliser un middleware pour créer des logs
  app.use(requestLogMiddleware('req'));

  RegisterRoutes(app);

  // Créer un endpoint GET
  app.get('/info', 
    (request: Request, response: Response, next: NextFunction) => {
      response.send("<h1>Hello world!</h1>");
    }
  );

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

  // Ajouter un handler pour les erreurs
  app.use(DefaultErrorHandler);

  // Lancer le serveur
  return new Promise<Server>(
    (resolve) => {
      httpServer.listen(PORT, () => {
        Log(`API Listening on port ${PORT}`)
        resolve(httpServer);
      })     
    }
  ); 
}

export const StopServer = async (server: Server|undefined) => {
  if (!server) { return; }
  return new Promise<void>(
    (resolve, reject) => {
      server.close(
        (err) => {
          if (err) {
            reject(err);            
          } else {
            resolve();
          }
        }
      )
    }
  );  
}
```

La même logique est présente, mais nous avons simplement créé deux fonctions qui démarrent et arrêtent le serveur.
Notez également que cela sera utile pour les tests d'intégration, afin de s'assurer que notre serveur est arrêté après chaque test.

Nous simplifions ensuite notre fichier `src/server.ts` pour qu'il utilise ces fonctions et réponde aux signaux du système d'exploitation :

```ts
import { Log } from "@logging/Log";
import { DB } from "@orm/DB";
import { StartServer, StopServer } from "server_manager";

StartServer().then(
  (server) => {
    
    const shutdown = async () => {
      Log("Stopping server...");
      await StopServer(server);
      Log("Closing DB connections...");
      await DB.Close();
      Log("Ready to quit.");
    }    

    // For nodemon restarts
    process.once('SIGUSR2', async function () {
      await shutdown();
      process.kill(process.pid, 'SIGUSR2');
    });

    // For app termination
    process.on('SIGINT', async function () {
      await shutdown();
      process.exit(0);
    });

    // For Heroku app termination
    process.on('SIGTERM',  async function () {
      await shutdown();
      process.exit(0);
    });
  }
);
```