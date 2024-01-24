# Projet NodeJS

Nous sommes prêts à initialiser notre projet NodeJS, et ajouter typescript.

## Initialiser un nouveau projet

Nous commençons des nouveaux projets node en initialisant le projet. Dans le terminal de VSCode :

```shell
npm init
```

Vous répondez à toutes les questions. À l'issue de cette instruction est le fichier crucial au projet : `package.json`


```json
{
  "name": "server",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}

```

## `package.json` et `npm`

Le fichier `package.json` est l'indexe de votre projet. Il contient plusieurs informations utiles pour votre projet :

* Son nom, sa version etc.
* `scripts`: La liste de commandes possibles avec `npm run ...`.&#x20;
* `dependencies`: Les librairies tierces venant des dépôts node à inclure dans notre projet. C'est la liste essentielle pour tourner notre projet en production.
* `devDependencies`: Les librairies supplémentaires pour compléter notre environnement de dev. Typiquement, ce sont les librairies de test, ou des types à inclure (si on utilise TypeScript)

Les entrées `dependencies` et `devDependencies` sont automatiquement gérés par la commande `npm`. En effet `npm` est le **node package manager** et on l'utilise pour installer, mettre à jour, ou enlever les librairies.

Par exemple, nous allons installer Express :

```shell
npm install express
```

On voit que `package.json` s'est mis à jour automatiquement :

```json
{
  // ...
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

On voit apparaître le dossier `node_modules` qui contient toutes les librairies tierces de notre projet.

D'autres commandes de `npm` sont :

```shell
# Installer ou réinstaller tous les dépendances
npm install

# Enlever une librairie, ex. express
npm remove express

# Récupérer des l'information d'une librairie ex. express
npm view express

# Récupérer les versions d'une librairie
npm view express versions

# Installer une version précise
npm install express@4.9.5

# Lister les librairies plus à jour
npm outdated

# Mettre à jours les librairie installées
npm update

```

Les packages déployés par npm évoluent en utilisant un système de versioning :

```
[version majeur].[version mineur].[version patch]
```

* **Patch** : sont les corrections de bug, et ne changent pas le comportement ou compatibilité de la librairie
* **Version mineure** : Normalement rétrocompatible dans la même version mineure, mais peut-être avec des refactoring ou des modifications (normalement ajouts) plus importants
* **Version majeure** : Risque de casser votre projet, il y aura des **breaking changes**, modifications qui risquent de casser votre code, par exemple, changer l'API, enlever les fonctions obsolètes, etc.

Dans `package.json` nous pouvons exprimer les limites de `npm update`, en utilisant les symboles suivants :

<figure><img src="../.gitbook/assets/wheelbarrel-with-tilde-caret-white-bg-w1000.jpg" alt=""><figcaption></figcaption></figure>

[Source de l'image](https://bytearcher.com/goodies/semantic-versioning-cheatsheet/)

Dans `package.json` si on voit :

* un tilde `~` devant la version : on va fixer la version mineure, mais autoriser les mises à jour de la version patch
* un caret `^` devant la version : on va fixer la version majeure, mais autoriser les mises à jour de la version mineure
* aucun symbole devant la version : on ne va jamais mettre à jour la version

On peut aussi préciser la version majeure, et laisser `npm` choisir la version mineure et patch automatiquement :

```
# Install la toute dernière version de express@3.*.*
npm install express@^3.2

# Installer la toute dernière patch de express@4.18.*
npm install express@~4.18
```

## Mon premier script avec NodeJS

Nous allons créer un nouveau fichier `src/hello.js` :

```js
console.log("Hello world");
```

Il est facile d'exécuter notre premier script avec la commande `node`:

```bash
node src/hello.js 
# Hello world !
```

{% hint style="success" %}
Pour info, `console.log` va émettre vers le `stdout`, et `console.error` vers le `stderr`. Vous pouvez donc intégrer un script node dans votre chaîne de commandes.
{% endhint %}

### Exécuter comme un script shell

On peut utiliser un _shebang_ qui permet de rendre notre script exécuter comme n'importe quel script :

```js
#!/usr/bin/env node

console.log("Hello world !");
```

Et, en rendant notre fichier exécutable, on pourrait l'invoquer comme un script shell:

```bash
# Rendre le fichier exécutable
chmod +x src/hello.js 

# Lancer le script (à noter, sans "node" devant car le shebang précise l'interprète à utiliser!)
./src/hello.js 
Hello world !
```

### Exécuter avec npm

La version plus classique de lancer un script node est en utilisant `package.json`, notamment le tableau `scripts` :

{% code title="package.json" %}
```json
{
  // ...
  "scripts": {
    "hello": "node ./src/hello.js",
  },
}
```
{% endcode %}

On lance notre script avec :

```bash
npm run hello
```

Le champ `scripts` est très utile pour créer et configurer plusieurs façons de lancer notre code :

* Lancer le serveur en mode développement
* Lancer le serveur en mode production
* Lancer un script d'utilité qui, par exemple, agit de façon ponctuelle sur votre base de données
* Lancer des tests
* Lancer des outils de validation de votre code (linter, orthographe, etc)

Par exemple, on aimerait que notre code se lance automatiquement dès qu'on apporte une modification. Pour cela, on va utiliser le package `nodemon` qui surveille notre base de code et relance le script dès qu'une modification est détectée :

```shell
# Attention, nodemon est utilisé exclusivement pour nous aider en développement, donc on l'inclut seulement dans les devDependencies en précisant --save-dev
npm install nodemon --save-dev
```

L'option `--save-dev` précise que cette dépendance n'est utile que pour l'environnement de développement. Justement, si on regarde `package.json`, on voit qu'il y a une nouvelle section qui s'appelle `devDependencies` :

```json
  ...
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
  ...
```

Ces dépendances sont ignorées lors de la mise en production. L'idée est d'optimiser l'artéfact de production et exclure tous les modules qui ne sont pas utiles ou potentiellement risqués.

On modifie notre `package.json` afin d'utiliser `nodemon` au lieu de node :


#### **package.json**

```json
{
  // ...
  "scripts": {
    "hello": "nodemon ./src/hello.js",
  },
}
```

Relancez votre script :

```shell
npm run hello
```

Si on modifie `src/hello.js`, on voit que `nodemon` relance le script à chaque fois qu'on le sauvegarde. :fire: Pratique ! :fire:

À noter : Si nodemon ne relance pas votre script après une modification, il faut ajouter l'option `-L` pour activer le **legacy polling**.

