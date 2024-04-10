# Docker

Aujourd'hui, on n'a qu'un environnement de développement, dans lequel on lance notre API manuellement.

On va s'occuper d'un **build** de notre API et son lancement comme un service dans notre `docker-compose.stage.yml`.

### Transpilation

A noter : effectuez ces manipulations sur votre machine locale !

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

Une fois prêt, faites :

```sh
git add .
git commit -m "Ajouté le build docker"
git push
```

Connectez vous à mon serveur, naviguez dans le dossier de votre projet et faites :

```sh
git pull
```
Les nouveaux fichiers sont apparus !

### Environnement de production

Nous sommes prêts maintenant à créer un `docker-compose.yml` qui lance notre serveur en production. Créez ce fichier sur le serveur à la racine de votre projet : 

Pour ouvrir un éditeur texte, tapez dans le terminal du serveur :

```sh
nano docker-compose.yml
```

Ensuite, collez le suivant :

```yml
version: '3.9'

services:
  api_NOMPRENOM:
    build: 
      context: ./
      dockerfile: ./Dockerfile
    container_name: NOMPRENOM
    ports:
      - "REMPLACER_PORT:5050"
    environment:
      - NODE_ENV=prod
      - PORT=5050      
    networks:
      - api-network-NOMPRENOM
    restart: always
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"
    

networks:
  api-network-NOMPRENOM:
    driver: bridge    
```

Chaque étudiant dans la classe va écouter sur un port différents. Il faut donc choisir une valeur aléatoire entre 20000 et 30000, et remplacer les valeurs suivants :

- REMPLACER_PORT &rarr; remplacer ce texte par votre port unique
- NOMPRENOM &rarr; remplacer ce texte par nom et prénom, tout collé, sans accents, tout en miniscule.

Par exemple, pour moi :

```yaml
version: '3.9'

services:
  api_glasskevin:
    build: 
      context: ./
      dockerfile: ./Dockerfile
    container_name: glasskevin
    ports:
      - "6000:5050"
    environment:
      - NODE_ENV=prod
      - PORT=5050      
    networks:
      - api-network-glasskevin
    restart: always
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"
    

networks:
  api-network-glasskevin:
    driver: bridge
```

Sauvegardez le fichier en tapant `Ctrl+O`, puis `Entrer`. Quitter avec `Ctrl+X`.

On est prêt à lancer notre serveur :

```bash
docker compose up -d
```

Tester avec Postman maintenant, en utilisant plutôt votre nom d'hôte, par exemple :

```http
GET https://unixshell.hetic.glassworks.tech:PORT/user
```

Remplacez `PORT` avec le port que vous avez choisie pour votre `docker-compose.yml`. C'est le port *exposé* par docker de votre API !


## Mise à jour de votre déploiement

Pour mettre à jour votre déploiement, vous allez faire le suivant :

1. Modifier votre code sur votre machine locale
2. Avec Git, vous faites `add`, `commit`, `push`
3. Vous vous connectez au serveur distant, et **vous naviguez dans le dossier de votre projet**
4. Vous faites `git pull`
5. Vous recontruisez l'image docker avec `docker compose build`
6. Vous relancez vos services avec `docker compose down` et ensuit `docker compose up -d`

A tout moment, vous pouvez consulter la liste de containers qui tournent :

```bash
docker ps
```

Pour consulter les journaux d'un container :

```bash
docker logs ID_DU_CONTAINER
```

Vous pouvez à tout moment arrêter un container et supprimer :

```bash
docker stop ID_DU_CONTAINER
docker rm ID_DU_CONTAINER
```

