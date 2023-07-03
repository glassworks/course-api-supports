# Typescript

Javascript est un language puissant, mais il est très permissif. C'est à dire, si on ne fait pas attention, le risque d'erreur est assez élevé. Considérez l'exemple suivant :

```js
// Affecter des types différentes aux variables est autorisé

let a = "hello";
a = 3;

console.log(a);

// Les objets ne sont pas strictement définis

const response = {
  greeting: "Hello",
  audience: "World",
  number: 123
}

// On peut augmenter un objet sans problème
response.d = 567;

// Remanier le code est plus compliqué ? 
// Ici, on voulais dire `greeting` ? Ou ajouter une nouvelle clé `greetingg` ?

response.greetingg = 5;


// Pas de validation des données avant l’exécution

var request;
const body = request.body;

// ... et si address était null ? Exception !
if (body.address.city === "Paris") {

}
```

Il est gérable pour les petits projects, mes dès qu'on dépasse un certain grandeur, il devient impossible à gérer :

* Remanier le code est compliqué et risqué
* Pas de signe d'erreurs avant de lancer un test
* Manque de rigueur dans les types des données ou la structure des objets

En plus, JS est un language qui évolue constamment, avec des niveaux de support hétérogène sur le marché. Pour écrire du code qui fonctionne partout, on est obligé de viser une version (norme) précise (ECMA 2015, par exemple), qui veut dire qu'un grand nombre de fonctionnalités modernes ne sont pas disponibles.

Avant, on devait utiliser ce qu'on appelle un "transpiler", qui va convertir des fonctions modernes en leur équivalents dans les anciennes versions (souvent des fonctions qui remplace un feature), et tout devenait très compliqué avec des "shims" et des "polyfills" (enduit de rebouchage) pour remplir ces trous.

Chez Microsoft, ils ont développé TypeScript, un superset de JS qui permet d'imposer des types strictes sur JS, et régler une grande partie des problèmes JS **vanilla**.

> Javascript est Typescript, mais Typescript n'est pas Javascript !

Typescript apporte le typage stricte (comme présent dans d'autre languages comme Java, C#, etc) à Javascript. Le compilateur (et éventuellement l'IDE) va signaler :

* L'utilisation des variables non-déclarés
* L'utilisation des types non-autorisées dans le fonctions
* Détecter l'utilisation des fonctions non-existants, ou des erreurs parmi des paramètres
* ... etc

En réalité, Typescript impose une étape de **transpilation** dans laquelle le Typescript est transformé en Javascript vanilla. Pendant cette procédure des erreurs sont détectés et signalées au développeur.

## Activer Typescript sur un projet NodeJS

Installez le package `typescript`, en développement seulement (car notre code final serait du JS transpilé)

```
npm install -g typescript --save-dev
```

Ceci installe typescript comme dependence nécessaire pour transpiler notre projet, et sera nécessaire quand on commence à créer nos images Docker pour production.

Ensuite on initialise typescript sur notre projet :

```
tsc --init
```

Ce script nous crée un fichier `tsconfig.json` qui contient toutes les options pour la compilation de typescript vers javascript, par exemple :

* `target` : la version cible de la compilation javascript. Typescript va automatiquement convertir tout vers cette version là.
* `outDir` : l'endroit où il faut créer les fichier .js

En effet, `outDir` n'est pas précisé par défaut. On va préciser le dossier `build`.

### **tsconfig.json**

```json
  "outDir": "./build",                                   /* Specify an output folder for all emitted files. */
```

On peut ensuite compiler notre code en javascript avec la commande suivante dans l'invite de commandes :

```sh
tsc
```

Dans le dossier `build`, il y aura donc un fichier .js pour chaque fichier .ts dans notre dossier `src`.

{% hint style="warning" %}
Si on lance `tsc` maintenant, on aura un erreur, parce qu'on n'a pas encore de fichiers de type `.ts` dans notre projet.
{% endhint %}

## Mon premier fichier TypeScript

Créez un fichier `.ts` sous `src`, nous allons expérimenter avec des types :

```ts

console.log("This is my first Typescript file");

// A noter, on précise les types par : [string|number|boolean|any ou autres]
const prenom: string = "Kevin";
const nom: string = "Glass";
const age: number = 40;
const etudiant: boolean = false;

// A noter, on précise le type des paramètres aussi
function greet(person: string, date: Date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}

greet(prenom, new Date());

// Définir une "interface" qui est une définition d'un objet
interface IRequest {
  type: 'POST' | 'GET';   // On peut énumérer des valeurs possibles
  orderBy?: string;       // Un point d’interrogation rend le champ optionnel
  limit?: number;
  page?: number;
}

const req: IRequest = {
  type: 'POST',
  orderBy: 'title',
  page: 1
}

// On peut définir explicitement des types
type ActionState = 'pending' | 'busy' | 'done' | 'canceled';

const state: ActionState = 'done';


/* 
On peut définir des types pour les fonctions 
Dans cet example, le callback passe un ActionStates comme paramètre et attend un retour de type IRequest
*/
type CallbackFunction = (state: ActionState) => IRequest;

const SendRequest = (cb: CallbackFunction) => {

  const req = cb('pending');

  // ... envoyez la requête
}

SendRequest(
  (state) => {    
    return {
      type: (state === 'pending' ? 'POST' : 'GET'),
      page: 1
    }
  }
)

```

On transpile ce script avec :

```bash
tsc
```

Regardez dans le répertoire `build`, vous verrez une fichier `myfirsttypescript.js`. Il y a quelques points à remarquer :

* Toutes les définitions et les expressions non-js ont disparues
* Nos commentaires existent toujours

Essayez d'introduire des erreurs suivantes :

* Utiliser une variable sans le déclarer
* Affecter un string à une variable de type numero
* Passer un mauvais paramètre

### _Downleveling_ le `target` dans `tsconfig.json`

Pour voir en action la transpilation, changez la `target` dans `tsconfig.json` à `es3`, et ré-exécutez `tsc`. Vous allez remarquer :

```js
console.log("Hello ".concat(person, ", today is ").concat(date.toDateString(), "!"));
```

Pourquoi ? Parce que les **templates** ne sont pas encore supportés en `es3`, et donc on est obligé de le transpiler autrement.

Remettez la valeur d'avant dans `tsconfig.json`.

### Options de transpilation

A la fin de `tsconfig.json`, il y a une gamme d'options qui rendent de plus en plus stricte notre système de typage.

## Execution automatique avec `ts-node` et `nodemon`

La librairie `typescript` est utile pour la compilation de notre code en JS, mais en développement il est fatiguant de toujours recompiler manuellement notre code.

En plus, on voit que les numéros de ligne dans le JS compilé ne correspondent plus aux lignes dans le `.ts`, qui pourrait perturber aussi les messages d'erreur en exécution (call-stack).

Pendant le développement, nous utilisons un autre outil `ts-node` qui est l'équivalent de `node` sauf on peut exécuter directement les fichiers typescript.

```sh
npm install ts-node --save-dev
sudo npm install -g ts-node
```

Maintenant on peut directement lancer un script .ts :

```
ts-node src/typescript/myfirsttypescript.ts 
```

Ou bien, on peut en créer une entrée dans `package.json` :

```json
 "scripts": {
    ...
    "ts-example": "nodemon --watch 'src' src/myfirsttypescript.ts"
  }
```

On peut donc lancer simplement notre script en utilisant npm :

```sh
npm run ts-example
```
