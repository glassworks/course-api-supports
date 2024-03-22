# Image Docker

Notre API est maintenant prêt à déployer. Nous allons le déployer grâce à Docker.

Aujourd'hui, on n'a qu'un environnement de développement, dans lequel on lance notre API manuellement.

On va s'occuper d'un **build** de notre API et son lancement comme un service dans notre `docker-compose.yml`.

### Transpilation

Nous aimerions construire une version finale de notre API, déjà transpilée en JS, et avec le minimum de dépendances nécessaires pour exécuter.

Pour cela, nous allons ajouter quelques lignes dans notre `package.json` :

```json
   "scripts" : {
       ...
      "clean": "rimraf build",
      "build": "npm run clean && tsc",
      "start-api": "node ./build/server.js"
   }
```

La commande `tsc` effectue définitivement le build de notre API, et enregistre les fichiers en .js pur dans le dossier `./build` (cet emplacement est précisé dans `tsconfig.json`).

Le script `clean` vide cet emplacement, pour être certain qu'on a un build propre à chaque fois. Il dépend d'une petite librairie `rimraf` qu'il faut installer avec :

```bash
npm install rimraf --save-dev
```

Le script `start-api` sera utilisé après le build, et lancera le javascript résultant du build. Vous remarquerez que l'API commence plus vite avec cette commande (après le build) car la transpilation de Typescript a déjà été faite.

Essayez ces 3 scripts - tout devrait fonctionner en local !

### Construire une image Docker

Ensuite, nous allons créer une image Docker qui permet d'être lancé comme un service. Pour cela, on aura un fichier  `Dockerfile` :

```Dockerfile
FROM node:18-alpine AS api-builder
WORKDIR app
COPY . .
RUN npm install
RUN npm run clean
RUN npm run build

FROM node:18-alpine AS api
WORKDIR app
COPY --from=api-builder /app/build ./build
COPY package* ./
RUN npm install --omit=dev
CMD npm run start-api
```

Ce Dockerfile contient les instructions de build d'une image Docker. Il se compose de deux étapes. Une première étape crée une image dans lequel on fait notre transpilation. Notez qu'on exécute les commandes `clean`  et `run` dans cette phase.

Dans la deuxième phase on copie simplement l'artéfact de build de l'étape précédent (le dossier `build`) dans une nouvelle image, et on n'installe que les dépendances nécessaires pour l'exécution (`npm install --omit-dev`).