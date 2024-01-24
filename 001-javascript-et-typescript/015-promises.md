# Promise

Dans les versions anciennes de JS, on se trouvait souvent dans la situation de **callback hell** ... c'est-à-dire, dans un callback, on fait encore une opération asynchrone, et ainsi de suite. Notre code ressemblait au suivant :

```js
var floppy = require('floppy');
floppy.load('disk',function(data) {
  floppy. load ('disk', function (data) {
    floppy. load ('disk', function(data) {
      floppy.load('disk',function(data) {
        floppy. load('disk', function (data) {
          floppy.load('disk',function(data) {
            floppy. load('disk', function (data)  {
              floppy.load('disk',function(data) {
              });
            });
          });
        });
      });
    });
  });
});
```

Afin de contourner ce problème, la notion de **Promise** a été développée au fil des années :

> Une Promise est un objet qui pourrait produire soit une valeur dans le futur, soit une raison pourquoi elle n'aurait pas pu terminer.

Une Promise est essentiellement un **constructor** qui va retourner un `object` qui contient 3 fonctions :

* `then` : une fonction qui prend un _callback_ comme paramètre. Quand la Promise termine (avec succès), le callback passé sera invoqué.
* `catch`: une fonction qui prend un callback comma paramètre. Si jamais il y a une erreur ou une exception, ce callback sera invoqué à la place de celui de `then`
* `finally`: une fonction qui prend un callback comma paramètre. Ce callback sera systématiquement appelé après le `then` ou le `catch`.

En construisant une Promise, nous passons un callback comme paramètre qui prend 2 paramètres :

* la référence à une fonction `resolve`, à invoquer quand notre longue opération est complète
* la référence à une fonction `reject`, à appeler dans le cas d'une erreur

L'exemple suivant démontre la construction d'une Promise :

```js
const p = new Promise(
  (resolve, reject) => {

     try {
      setTimeout(
        () => {
          resolve();
        },
        1000
      )
    } catch (err) {
      reject(err);
    } 

  }
);
```

Ici, on attend 1 seconde avant de résoudre la Promise. Regardez comment on invoque le callback `resolve` seulement dans le callback de l'opération `setTimeout`. Dans le cas d'une erreur, on invoque plutôt le callback `reject`.

Ce code fonctionnera tout seul, mais à priori, on aimerait _enchaîner_ des opérations asynchrones après. On le fait avec la fonction `then` qui nous permet de préciser le callback à invoquer quand la Promise précédent se résout.

```js

// La Promise est un objet, avec 3 fonctions : "then", "catch" et "finally"
p
.then(
  (value) => {
      // Ce callback est exécuté quand "resolve" est appelé dans la Promise

      // Je pourrais retourner une nouvelle Promise pour les enchaîner...
      return new Promise(/* à remplir */);
  }
)
.then(
  () => {

  }
)
.catch(
  (err) => {
    // Si jamais "reject" est appelé, on saute tous les "then" et on vient directement ici pour gérer les erreurs
  }
)

```

Faites attention à retenir cette structure, parce que c'est la fondation pour la suite.

Nous allons essayer de charger un fichier de façon asynchrone avec les Promises :

{% code title="src/promise-loadfile.ts" lineNumbers="true" %}
```js
/* En Typescript on importe les librairies avec "import" (et pas "require")
* Notez comment, en VSCode, on peut récupérer des informations et auto-complète sur les fonctions et objets
* grâce à TypeScript !
*/
import { readFile } from "fs";
import { join } from "path";

const FICHIER1 = join('assets', 'grunter.JPG');
const FICHIER2 = join('assets', 'fichier_qui_nexiste_pas.JPG');
const FICHIER3 = join('tsconfig.json');

// On peut transformer un API qui utilise les callbacks en promise

const loadFileAsync = (path: string): Promise<Buffer> => {
  // Setup le promise avec son callback "resolve" et "rejects"
  return new Promise<Buffer>(
    (resolve, reject) => {
      
      // Appelez la fonction où le 2ème paramètre est un callback traditionnel
      readFile(path, (err, data) => {
        // S'il y a une erreur, on rejette
        if (err) {
          reject(err);
        } else {
          // Sinon on resolve
          resolve(data)
        }
      });
  
    }
  );
}


loadFileAsync(FICHIER1)
  .then(
    (buffer) => {
      console.log("Fichier 1 chargéé !")
      // Faire q.c. avec le buffer
      return loadFileAsync(FICHIER2)
    }
  )
  .then(
    (buffer) => {
      console.log("Fichier 2 chargéé !")
      return loadFileAsync(FICHIER3)
    }
  )
  .then(
    (buffer) => {
      // Tout le média chargé
      // Lancer le jeu !
    }
  )
  .catch(
    (err) => {
      console.error("Oups, il y a eu une erreur !")
    }
  )
```
{% endcode %}

On lance ce code avec `ts-node` :

```bash
ts-node src/promise-loadfile.ts 
# Fichier 1 chargéé !
# Oups, il y a eu une erreur !
```

Notez qu'on ne charge pas le 3ème fichier puisque le 2ème n'existe pas. Le faite qu'une erreur soit rencontré fait que la chaîne de Promise est cassé, on saute directement dans la fonction `catch`.

## Async / await

Vous avez sûrement remarqué que ce n'est pas très propre cet enchaînement de Promises (même si c'est mieux que **callback hell**).

Enchaîner les `then` crée quand même du code chargé de callbacks, donc on a intégré du "syntactic sugar" (syntax sucré) qui permet d'exprimer une suite de Promise plus brièvement.

D'abord, on marque une fonction avec le mot clé `async` :

```js
const myFunc = async () => {

  // Enchainer les promises dans la fonction async
  await AutrePromise();
  await AutrePromise()
  await AutrePromise()

}


// myFunc est un objet de type Promise, et on peut l'utiliser ainsi :

myFunc()
  .then(
    () => {

    }
  )
  .catch(
    () => {

    }
  )


```

Voici notre chargement de fichier modifié avec `async/await`.

{% code title="src/promise-async-await.ts" lineNumbers="true" %}
```js
/* En Typescript on importe les librairies avec "import" (et pas "require")
* Notez comment, en VSCode, on peut récupérer des informations et auto-complète sur les fonctions et objets
* grâce à TypeScript !
*/
import { readFile } from "fs";
import { join } from "path";

const FICHIER1 = join('assets', 'grunter.JPG');
const FICHIER2 = join('assets', 'fichier_qui_nexiste_pas.JPG');
const FICHIER3 = join('tsconfig.json');

// On peut transformer un API qui utilise les callbacks en promise

const loadFileAsync = (path: string): Promise<Buffer> => {
  // Setup le promise avec son callback "resolve" et "rejects"
  return new Promise<Buffer>(
    (resolve, reject) => {
      
      // Appelez la fonction où le 2ème paramètre est un callback traditionnel
      readFile(path, (err, data) => {
        // S'il y a une erreur, on rejette
        if (err) {
          reject(err);
        } else {
          // Sinon on resolve
          resolve(data)
        }
      });
  
    }
  );
}

/* On doit obligatoirement marquer une fonction avec "async" pour indiquer
que la fonction retourne une Promise (c'est implicite). A l'intérieure, on 
n'a juste à utiliser le mot clé "await" devant nos Promises, qui est l'équivalent 
à la clause ".then". 

Si on entoure le tout par try/catch, on retrouve la fonctionnalité de ".catch"
*/

const exec = async() => {
  try {
    const buffer1 = await loadFileAsync(FICHIER1);
    console.log("Fichier 1 chargé !")

    const buffer2 = await loadFileAsync(FICHIER2);
    console.log("Fichier 2 chargé !")

    const buffer3 = await loadFileAsync(FICHIER3);

    // Tout le média chargé
    // Lancer le jeu !
  } catch (err) {
    console.error("Oups, il y a eu une erreur !")
  }

}

exec();

```
{% endcode %}

Il y a 2 choses à retenir :

* En marquant une fonction `async`, on dit que notre fonction retourne un objet de type "Promise". Ceci se voit avec typescript et VSCode.
* Le mot clé `await` fait l'équivalent de ".then", et peut être utilisé sur toutes les Promises.

De plus en plus de librairies offre des alternatifs aux callbacks en utilisant les Promises. Par exemple, la librairie `fs` fournit un alternatif compatible aux Promises :

{% code title="src/promise-async-await.ts" lineNumbers="true" %}
```typescript
// De plus en plus de librairies incluent aussi une version des fonctions
// qui retournent des Promise. Il faut lire la documentation de la librairie en question
import { readFile } from "fs/promises";
import { join } from "path";

const FICHIER1 = join('assets', 'grunter.JPG');
const FICHIER2 = join('assets', 'fichier_qui_nexiste_pas.JPG');
const FICHIER3 = join('tsconfig.json');

const exec = async() => {
  try {
    // Si une fonction retourne un Promise, elle pourrait être utilisé avec await
    const buffer1 = await readFile(FICHIER1);
    console.log("Fichier 1 chargé !")

    const buffer2 = await readFile(FICHIER2);
    console.log("Fichier 2 chargé !")

    const buffer3 = await readFile(FICHIER3);

    // Tout le média chargé
    // Lancer le jeu !
  } catch (err) {
    console.error("Oups, il y a eu une erreur !");
  }

}

exec();
```
{% endcode %}

## Exercice

Enveloppez la fonction "setTimeout" dans une fonction qui s'appelle `wait(seconds: number)`, qui prend comme valeur le nombre de secondes à attendre avant de continuer.

Ensuite, utilisez cette fonction à plusieurs reprises afin rythmer bien l'émission des valeurs sur le stdout (le rythme de la symphonie n° 5 de Beethoven) :

Da - pause (0,33s) - Da - pause (0,33s) - Da - pause (0,33s) - Daaaaaaaaaa - pause (3s) etc.

