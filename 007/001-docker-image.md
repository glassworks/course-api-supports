# Image Docker

Nous voulons construire notre API, supprimer toutes les dépendances utilisées uniquement pour le développement et compiler le tout dans une image Docker qui peut être déployée localement ou dans le nuage en tant que conteneur.

## Transpiler TS vers JS

Tout d'abord, vous remarquerez que nous pouvons compiler notre projet Typescript existant en Javascript pur :

```bash
tsc
```

Vous noterez la sortie créée dans `build`.

Vous pouvez essayer d'exécuter votre serveur comme n'importe quelle autre application node :


```bash
node build/server.js
```

Mais cela provoquera une erreur concernant nos alias de chemin comme `@controller` ou `@orm`. Nous devons les résoudre dans le cadre de notre construction :

```bash
npm install --save-dev tsc-alias
npm install -g tsc-alias
```

Post-traiter les fichiers construits à l'aide de :


```bash
tsc-alias -p tsconfig.json
```

Et redémarrez le serveur :

```bash
node build/server.js
```

Votre serveur devrait fonctionner normalement. Essayez [http://localhost:5050/info](http://localhost:5050/info) pour tester.

Nous aurons aussi besoin de copier tous les fichiers statiques dans le répertoire `build/public` (comme notre `swagger.json` généré).

Combinons toutes nos étapes de construction en un seul script `npm` dans `package.json`.

Tout d'abord, nous avons besoin d'outils pour nettoyer l'ancien répertoire de construction et copier les fichiers :


```bash
# For deleting directories
npm install --save-dev rimraf
# For copying files  
npm install --save-dev copyfiles
```

Nous pouvons maintenant créer un script pour construire notre API (`package.json`) :

```json
{
  "name": "api",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "server": "nodemon",
    "compile": "tsoa -r tsconfig-paths/register spec-and-routes",
    "clean": "rimraf build",
    "build": "npm run clean && npm run compile && tsc && tsc-alias -p tsconfig.json && copyfiles public/**/* build/"
  },
  ... 
```

Nous pouvons maintenant construire notre projet entier avec :

```bash
npm run build
```


## Dockerfile

Nous voulons maintenant créer une image Docker qui inclut Node 20, notre répertoire de construction et seulement les dépendances nécessaires à l'exécution de notre projet (à l'exception des outils de développement). L'idée est de créer la plus petite image Docker possible pour faire fonctionner notre projet.

Créons un fichier Docker dans `config/docker/Dockerfile.prod` :


```Dockerfile
FROM node:20-alpine AS api-builder
WORKDIR /app
COPY . .
RUN npm install
RUN npm run clean
RUN npm run build

FROM node:20-alpine AS api
WORKDIR /app
COPY --from=api-builder /app/build ./
COPY package* ./
RUN npm install --omit=dev
CMD ["npm", "run", "start-api"]
```

Notez qu'il s'agit d'une construction en deux phases. Nous créons d'abord un conteneur dans lequel nous allons construire notre projet. Ensuite, nous créons une seconde image propre, en copiant seulement le contenu du répertoire `build` dans le premier conteneur. Nous installons aussi seulement les paquets de production de nos dépendances avec `npm install --omit=dev`. Enfin, nous démarrons notre serveur avec le script `npm run start-api`. C'est un script que nous n'avons pas encore ajouté à notre `package.json` :

```json
{
  "name": "api",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "server": "nodemon",
    "compile": "tsoa -r tsconfig-paths/register spec-and-routes",
    "clean": "rimraf build",
    "build": "npm run clean && npm run compile && tsc && tsc-alias -p tsconfig.json && copyfiles public/**/* build/",
    "start-api": "node ./server.js" 
  },
  ... 
```

Enfin, ajoutez le fichier `.dockerignore` à la racine de votre projet pour ignorer tous les fichiers que vous ne voulez pas dans vos images :


```
.data
.data-prod
.devcontainer
.vscode
build
node_modules
nodemon.*
```

Dans un terminal **hors de votre DevContainer**, naviguez jusqu'à la racine de votre projet et exécutez ce qui suit :


```bash
docker buildx build --platform linux/amd64,linux/arm64 --pull -f ./config/docker/Dockerfile.prod -t api .
```

Nous donnons à l'image une balise `api` pour pouvoir y faire référence plus tard.

Cela permettra de construire une image multiplateforme ! Vérifiez que votre image existe dans votre dépôt Docker local :


```bash
docker image ls | grep "api"
```

Vous pouvez également vérifier le contenu de l'image :


```bash
docker run -it api /bin/sh

# Then:
ls -la
```

Vous verrez le contenu de votre image ! Tapez `exit` pour quitter le conteneur.

Pour exécuter votre image en tant que conteneur local (assurez-vous que le serveur est arrêté dans votre DevContainer !) :

```bash
docker run -p 5050:5050 api
```

Essayez de consulter le lien [http://localhost:5050/info](http://localhost:5050/info)

Vous devriez obtenir vos informations, mais la base de données n'est pas connectée. C'est normal, cette image n'a pas accès à notre base de données déployée avec notre Dev Container !

## Compose

Localement, ou sur un serveur distant, nous pouvons vouloir configurer l'ensemble de notre système en utilisant Docker Compose. Le fichier de configuration ressemblerait à ce qui suit :


```yml
services:
  api:
    image: api
    ports:
      - "5050:5050"
    environment:
      - NODE_ENV=prod
      - PORT=5050
      - FRONT_URL=http://127.0.0.1
      - DB_HOST=dbms
      - DB_USER=api-dev
      - DB_PASSWORD=api-dev-password
      - DB_DATABASE=school     
      - AWS_ACCESS_KEY_ID=SCWQKFBV9QSRDRKHJN7G
      - AWS_SECRET_ACCESS_KEY=69554f92-aeea-4995-8be1-5ba2911f9d76
      - STORAGE_REGION=fr-par
      - STORAGE_ENDPOINT=https://s3.fr-par.scw.cloud
      - STORAGE_BUCKET=object-storage-playground
      - MJ_APIKEY=650c4a928026033d089089cc09e870f2
      - MJ_APISECRET=69f7bdf5eaa69818892bb4e3828569cc
      - MJ_EMAIL_FROM=kevin@nguni.fr
      - MJ_EMAIL_NAME=Kevin
    networks:
      - api-prod-network
    restart: always
    labels:
      api_logging: "true"
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"
    
  dbms:
    image: mariadb
    restart: always
    ports:
      - "3309:3306"
    environment: 
      - MYSQL_ALLOW_EMPTY_PASSWORD=false
      - MYSQL_ROOT_PASSWORD=Q8qUnHTp3aVuwDdy
    command: [
      "--character-set-server=utf8mb4",
      "--collation-server=utf8mb4_unicode_ci",
    ]
    volumes:
      - ./.data-prod:/var/lib/mysql
    networks:
      - api-prod-network

networks:
  api-prod-network:
    driver: bridge
    name: api-prod-network
```

Notez l'utilisation de **variables d'environnement**, que notre code mentionne en utilisant `process.env.`. Nous remplaçons effectivement nos variables par d'autres pour l'utilisation en production ! **N'utilisez jamais en production les mêmes secrets qu'en développement !

Vous pouvez démarrer votre environnement en utilisant :


```bash
docker compose -f ./docker-compose.prod.yml up -d
```

Vérifiez vos nouveaux services : 

```bash
docker ps
```

Obtenez l'ID du service `mariadb`.

Initialisez votre base de données avec :

```bash
docker exec -i [ID] mariadb -u root -p[root password] < ./src/model/schema/init.sql
docker exec -i [ID] mariadb -u root -p[root password] < ./src/model/schema/ddl.sql 
```

Essayez de consulter le lien [http://localhost:5050/info](http://localhost:5050/info), vous devriez constater que la base de données est maintenant connectée !

Pour arrêter tous vos services :

```bash
docker compose -f docker-compose.prod.yml down
```
