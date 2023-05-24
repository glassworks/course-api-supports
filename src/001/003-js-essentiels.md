# L'essentiel de Javascript

## Variables

Javascript n'oblige pas des types strictes, mais il y a quand même un certain nombre de types de données de base supportés :
* `number` : une valeur numérique
* `string` : une chaîne de caractères
* `boolean` : une valeur boolean (soit `true`, soit `false`)
* `object` : un dictionnaire de tuples, de type clé-valeur
* `array` : un tableau de valeurs de type simple ou object
* `function` : une déclaration de fonction


Le type d'une variable est, par défaut implicite et n'est pas imposé. 


```js

// On déclare une variable qui peut changer de valeur avec `let`
let myvariable = 5;

myvariable = "Hello world";
myvariable = true;

// On peut créer un `object` qui est un dictionnaire de clés et valeurs
// Les valeurs peuvent être des objets, des fonctions, des array etc.
myvariable = {
  a: 123,
  b: "Subtext",
  c: {
    aa: "Une sous valeur",
    bb: 456
  },
  d: function() {
    console.log("Ceci est une fonction")
  }
}

// On peut même affecter une fonction à une variable
myvariable = function(param) {
  console.log("Ceci est une fonction");
  return true;
}

// Les arrays se déclare implicitement en utilisant les crochets
myvariable = [1, 2, 3, 4]


```

Pour optimiser notre code, on utilise le mot clé `const` (au lieu de `let`) pour indiquer les variables qui ne changent pas de valeur :

```js
const myvariable = 5;
```

Nous utilisons des `objets` afin de rassembler un ensemble de valeurs, indexés par leur clé. On peut rappeler ou modifier ces valeurs en les adressant via leur chemin d'accès :

```js
const myvariable = {
  a: 123,
  b: "Subtext",
  c: {
    aa: "Une sous valeur",
    bb: 456
  },
  d: function() {
    console.log("Ceci est une fonction")
  }
}

// Afficher uniquement une valeur étant donné son chemin :
console.log(myvariable.c.aa);
```


## Functions

Javascript est essentiellement un language [fonctionnel](https://fr.wikipedia.org/wiki/Programmation_fonctionnelle) dans lequel la **fonction** est très haut placé.

On define une fonction avec le mot clé `function` :

```js
function testFn(param) {
  console.log("Ceci est une fonction, avec un paramètre : " + param);
}
```

On invoque une fonction en citant son nom suivi par des parenthèses (dans lequel on précise de valeurs à passer à la fonction) :

```js
testFn("abcd");
```

Ceci est normal pour la plupart des language de programmation. 

La différence en JS est qu'une fonction peut-être aussi traité comme une *valeur*, en l'affectant à une variable :

```js
// Une fonction est aussi une valeur, on peut l'affecter à une variable !
const fn = testFn;

// Je peux ensuite invoquer la fonction à travers de cette variable
fn("1234");
```

Cela veut dire que je pourrais définir une fonction, ensuite passer la référence de cette fonction comme paramètre partout pour invocation.

### Comportement dynamic

Le fait de pouvoir *affecter* une fonction à une variable nous permet d'adapter le comportement de notre code en fonction des conditions dynamiques.

```js
const dayFn = function() {
  console.log("Bonjour !");
}
const nightFn = function() {
  console.log("Bonsoir !");
}

const hour = (new Date().getHours());

// Ici, on peut choisir la fonction à utiliser, selon 
// les conditions 
const greetingFn = hour < 17 ? dayFn : nightFn;

// Plus tard, on attend juste le bon message d'accueil, sans raisonner sur le
// pourquoi ne le comment
greetingFn();
```


### Encapsulation 

Intelligemment utilisés, les fonctions nous permettes d'encapsuler différents données et fonctions qui sont liées :

```js
function MyService(name, age) {

  this.age = age;
  this.name = name;

  const privateFn = function() {
    console.log("Ceci est une fonction à l'interne");
  }

  this.publicFn = function() {
    console.log("Ceci est une fonction publique de " + this.name + " qui a " + this.age + "ans");
  }
}

const resA = new MyService("Kevin", 40);
```

On commence par définir la fonction `MyService`, qui va encapsuler le nom et age d'une personne.

Nous invoquons cette fonction à l'aide du mot clé `new`  qui va traiter la fonction comme `constructor`. C'est à dire : un nouvel `object` est crée comme résultat de la fonction. Ensuite on exécute les contenus de la fonction. Par défaut, cet `object` est retourné.

En exécutant la fonction, à chaque fois qu'on cite `this`, on parle de l'objet crée lors de la construction (**l'instance**). Dans l'exemple, on crée une nouvelle clé `age` sur l'objet `this`, et on l'affecte la valeur de `40`. Notez aussi, qu'on déclare une fonction avec la clé `publicFn` qui pourrait être invoqué plus tard, et qui va utiliser les valeurs sur le même objet `this`.

Par exemple :

```js
const resA = new MyService("Kevin", 40);
const resB = new MyService("Sylvie", 32);

console.log(resA.publicFn());
// Ceci est une fonction publique de Kevin qui a 40 ans
console.log(resB.publicFn());
// Ceci est une fonction publique de Sylvie qui a 32 ans
```


## Callbacks

Si on peut affecter une fonction à une variable, on peut également passer une fonction comme paramètre à une autre fonction.

L'utilité de cette technique est le suivant : *exécuter  la fonction `fn`, et je vous passe un référence à une autre fonction à invoquer quand c'est fini*

```js

function outerFn(callback) {
  console.log("Une fonction qui prend une fonction comme paramètre");
  /// ... ici, quelques operations longues

  // Appeler le callback passé comme paramètre
  callback();
}

outerFn(function(param) {
  console.log("Ceci est le callback");
});


const fn = function() {
  console.log("Ceci est un autre callback");
}
outerFn(fn);
```

Essayez cet exemple. Notez qu'on peut modifier l'acheminement du code en passant une fonction différente à chaque fois.

{% hint style="warning" %}

Dans le contexte de l'exemple précédent, pourquoi le code suivant n'est pas valide ?

```js
outerFn(fn());
```

{% endhint %}

## Fonctions fmèches (arrow functions)

Jusqu'a présent nous avons utilisés le mot clé `function` afin de déclarer nos fonctions. 

Il y a une autre manière de déclarer des fonctions qui est parfois plus brève :

```js
const myFn = (name, age) => {
  console.log("Bonjour " + name + ", vous avez " + age + " ans");
}
```

Il y a beaucoup de différences entre une fonction **régulière** et une fonction **anonyme**  (fonction flèche):
- Une fonction flèche ne construit pas d'objet `this`. Le mot clé `this` réfère à celui qui existait déjà (le parent) lors de la création de la fonction
- Une fonction flèche ne peut pas être invoqué avant d'être déclaré. En effet, les effets de *hoisting* ne s'appliquent pas.
- ... (cherchez sur le web les autres différences !)


Les fonctions flèches sont très utiles par contre pour les callbacks. Cela permet de créer facilement une fonction callback pour quoi le context (l'objet `this`) est le même que son emplacement dans le code (moins de confusion).

Nous pouvons reformuler l'exemple précédent des callbacks avec des fonctions flèches : 

```js

function outerFn(callback) {
  console.log("Une fonction qui prend une fonction comme paramètre");
  /// ... ici, quelques operations longues

  // Appeler le callback passé comme paramètre
  callback();
}

outerFn((param) => {
  console.log("Ceci est le callback");
});


const fn = () => {
  console.log("Ceci est un autre callback");
}
outerFn(fn);
```