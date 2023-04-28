# Docker Dev Container

Avant la création de notre API, il serait intéressant de créer un environnement de développement qui :
* facilement duplicable d'un ordinateur à l'autre
* indépendant du système d'exploitation
* contient le set de dépendances du projet (les versions des paquets par exemple)

Nous utilisons Docker et VSCode afin de satisfaire ces demandes via leur [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

On crée un container qui tourne uns version du système d'exploitation et interprète de notre choix. Idéalement ce sera les mêmes qu'on utilisera en production (par exemple, Ubuntu Linux).

VSCode s'attache à ce container de plusieurs façons :
- on monte le dossier de notre projet (en local) dans le container via l'élément `volumes` de `docker compose`. 
- quand on ouvre un terminal dans VSCode, c'est l'équivalent de lancer un `docker exec -it [ID du container]`. On lance donc un interprète *dans le container*. On peut ensuite installer et lancer des processus provenant de notre développement (par exemple, lancer l'API) dans son environnement précis.

Pour accomplir tout cela, VSCode exige la présence d'un dossier `.devcontainer` à la racine du workspace.

Nous commençons donc par créer ce dossier dans notre dossier de travail, et en ajoutant un fichier `devcontainer.json` dedans :

```json
{
  "name": "NodeJS Boilerplate API",
  // Pointer vers notre docker-compose.dev.yml
  "dockerComposeFile": [
    "../docker-compose.dev.yml"
  ],
  // Le service dans docker-compose.dev.yml auquel on va attacher VSCode
  "service": "vscode_api",
  // Le dossier de travail précisé dans Dockerfile.dev
  "workspaceFolder": "/home/hetic",
  // Set *default* container specific settings.json values on container create.
  "settings": {},
  // Quelques extensions VSCode à inclure par défaut pour notre projet
  "extensions": [
    "pmneo.tsimporter",
    "stringham.move-ts",
    "rbbit.typescript-hero",
    "ms-vscode.vscode-typescript-tslint-plugin",
    "streetsidesoftware.code-spell-checker-french"
  ],
  "forwardPorts": [ ]
}
```
