# Logs

> Vous trouverez le projet fonctionnel de ce chapitre [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/007-deploy-monitor)


Nous allons préparer notre API pour déploiement en lui donnant une façon de créer des journaux pertinents.

D'abord, nous allons installer des modules qui permettent de créer des journaux en format JSON, et permette de extraire des informations concernant les requêtes auprès de notre API:

```bash
# Formater des logs comme on veut
npm install winston

# Intercepter des requêtes express et extraire des informations
npm install morgan
npm install @types/morgan
```

## Un outil de Log normalisé

On crée un outil pour créer des journaux standardisé qui permet d'envoyer nos messages personnalisés bien formatés (dans `src/utility/Logging/Log.ts`):

```ts
import { LogTag } from './LogTag';

import { createLogger, format, transports } from 'winston';

const logger = createLogger({
  level: 'info',
  format: (process.env.NODE_ENV === 'prod' ? format.json() : format.simple()),
  transports: [
    new transports.Console({})
  ]
});

interface ILogMeta {
  details?: any;
  tag: LogTag
}

const constructMeta = (tag: LogTag, data?: unknown): ILogMeta  => {
  const meta: ILogMeta = {
    tag
  };
  if (data) { meta.details = data; }
  return meta;
}

const _Log = (level: 'info'|'warn'|'error', tag: LogTag, message: string, data?: unknown) => {
  logger.log(level, message, constructMeta(tag, data));
}


export const LogWithTag = (tag: LogTag, message: string, data?: unknown) => {
  _Log('info', tag, message, data);
}
export const Log = (message: string, data?: unknown) => {
  LogWithTag('exec', message, data)
}

export const LogWarnWithTag = (tag: LogTag, message: string, data?: unknown) => {
  _Log('warn', tag, message, data);
}
export const LogWarn = (message: string, data?: unknown) => {
  LogWarnWithTag('exec', message, data)
}

export const LogErrorWithTag = (tag: LogTag, message: string, data?: unknown) => {
  _Log('error', tag, message, data);
}
export const LogError = (message: string, data?: unknown) => {
  LogErrorWithTag('exec', message, data)
}
```

On aura juste à emettre, n'importe où dans le code :

```
  ...
  Log("Un message");
  LogError("Il y a eu un message");
  ...
```

Le message sera redirigé où on veut, sur le format qu'on veut, selon l'environnement. Ici, on précise que si on est en développement, on utilise simplement un format text, mais en production (`NODE_ENV=prod`) en utilise un format JSON qu'on peut mettre dans une base de données pour analyse.

## Journaux des requêtes

On crée aussi un **middleware** qui va émettre un log pour chaque requête géré par express (dans `src/utility/Logging/log.middleware.ts`):
 :

```ts
import morgan from 'morgan';
import { LogTag } from './LogTag';


/**
 * Un middleware qui crée un des logs des requêtes.
 * Si la variable d'environnement ENV == 'prod', le format JSON sera utilisé, sinon, un text classique
 * @param tag 
 * @returns 
 */
export const requestLogMiddleware = (tag: LogTag) => {
  return morgan(
    (tokens, req, res) => {
      if (process.env.NODE_ENV === 'prod') {
        return JSON.stringify({
          'tag': tag,
          'remote-address': tokens['remote-addr'](req, res),
          'remote-user': '',    // TODO
          'time': tokens['date'](req, res, 'iso'),
          'method': tokens['method'](req, res),
          'url': tokens['url'](req, res),
          'http-version': tokens['http-version'](req, res),
          'status-code': tokens['status'](req, res),
          'content-length': tokens['res'](req, res, 'content-length'),
          'referrer': tokens['referrer'](req, res),
          'user-agent': tokens['user-agent'](req, res),
          'response-time': tokens['response-time'](req, res),          
        });
      } else {
        return [
          tokens['date'](req, res, 'iso'),
          tokens['status'](req, res),
          tokens['method'](req, res),
          tokens['url'](req, res),
          tokens['res'](req, res, 'content-length'),
          tokens['response-time'](req, res)
        ].join(' ');
      }
    }
  );
}
```

Notez ici on change l'information selon l'environnement aussi. On récupère un certain quantité d'information à chaque requête. 

Enfin on va attacher ce middleware à notre serveur (`src/server.ts`) :

```ts
import Express, { json } from "express";
import swaggerUi from "swagger-ui-express";
import { DefaultErrorHandler } from "./middleware/error-handler.middleware";
import { RegisterRoutes } from './routes/routes';
import { requestLogMiddleware } from "./utility/Logging/log.middleware";


// Récupérer le port des variables d'environnement ou préciser une valeur par défaut
const PORT = process.env.PORT || 5050;

// Créer l'objet Express
const app = Express();

// L'appli parse le corps du message entrant comme du json
app.use(json());

// Utiliser un middleware pour créer des logs
app.use(requestLogMiddleware('req'));

RegisterRoutes(app);

// Ajouter une route qui sert la documentation swagger
app.use(Express.static("public"));
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
app.listen(PORT,
  () => {
    console.info("API Listening on port " + PORT);
  }
);

```

Essayer de lancer l'API et exécuter quelques requêtes. Vous verrez les journaux en format text. Essayer d'utiliser l'environnement de prod :

```bash
NODE_ENV=prod npm run server
```

Vous verrez les logs en format JSON.

