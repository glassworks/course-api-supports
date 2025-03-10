# Documentation

Notre API doit être facilement compréhensible pour ses utilisateurs. On pourrait manuellement écrire une documentation, mais :

* il est difficile de le garder à jour, on est souvent faignant, ou en retard
* Une petite modification au code pourrait modifier le comportement documenté

Un standard de documentation qui s'appelle `swagger` et depuis peu [`openapi`](https://www.openapis.org) a créé une norme pour la documentation d'un API. La documentation est en form `.json` et peu être lue est interprété par un autre process. Il y a même des [process qui transforment automatiquement un fichier `swagger` en interface HTML pour notre lecture](https://swagger.io/tools/swagger-ui/). 

Il y a plusieurs façons de documenter un API :

* **Définition vers le code** : on rédige notre `swagger.yml` ou `swagger.json` manuellement, et puis on fait tourner un processus qui va créer des fonctions/classes/endpoints pour notre architecture cible (nodejs, php, ruby... etc). Le fichier `swagger.*` est la source de vérité de l'application.
    * Personnellement, je n'aime pas cette approche, car le code généré est très répétitif, et parfois pas assez flexible pour ce que je veux faire
    * Parfois on oublie que le fichier swagger est la source de vérité. On ajoute des fonctions, qui seront décrochées ou bien supprimées plus tard quand on relance le générateur.
* **Code ver la définition** : on fait une sorte de bien structurer et documenter notre code (avec des commentaires), et un `swagger.*` fichier est crée de notre code. Notre code devient la source de vérité.
* **Manuel** : on maintient la doc et l'implémentation indépendamment. Très lourd, et facile à oublier ou de ne pas mettre à jour.

Personnellement je préfère qu'on ait une seule source de vérité : notre code source !


Grâce à Typescript, projets comme [`tsoa`](https://tsoa-community.github.io/docs/) permettent d'utiliser la structure Typescript et les commentaires déjà présents dans le code afin d'assembler automatiquement la documentation.

## Consulter la documentation

Un fichier `swagger.json` et disponible dans le dossier `public`. Ce fichier pourrait être servi, bien formaté, comme une page web grâce à des librairies open source :

```bash
npm install swagger-ui-express
npm install --save-dev @types/swagger-ui-express
```

Dans notre fichier `server.ts`, on ajoute des lignes pour servir ce contenu :

```ts
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
```


En lançant l'api, le fichier `swagger.json` est disponible à [http://localhost:5050/swagger.json](http://localhost:5050/swagger.json).

Sinon, grâce à `swagger-ui-express` on peut le consulter sous la route `docs` à [http://localhost:5050/docs/](http://localhost:5050/docs/)


{% hint style="success" %}

Notez surtout que les commentaires dans notre code apparaissent maintenant comme de la documentation de notre API. Génial !

{% endhint %}


## Documentation manuelle

tsoa extrait automatiquement les champs et les commentaires du script de type pour créer la documentation, mais il arrive que nous ayons besoin d'une documentation personnalisée lorsque nous faisons quelque chose qui sort de l'ordinaire. Prenons l'exemple de notre procédure d'obtention d'un fichier.

Avec `tsoa` on peut personnaliser la documentation générée directement dans `tsoa.json` :

```json
{
  ...
  "spec": {
    ...
    "specMerging": "recursive",
    "spec": {
      "paths": {
        "/user/{userId}/file": {
          "post": {
            "requestBody": {
              "content": {
                "multipart/form-data": {
                  "schema": {
                    "type": "object",
                    "properties": {
                      "file": {
                        "type": "string",
                        "format": "binary"
                      }
                    }
                  }
                }
              }
            }            
          }
        }
      }
    }
  },
  ...
}
```




