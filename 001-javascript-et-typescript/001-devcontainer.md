# Docker Dev Container

Avant la création de notre API, il serait intéressant de créer un environnement de développement qui :

* facilement duplicable d'un ordinateur à l'autre
* indépendant du système d'exploitation
* contient le set de dépendances du projet (les versions des paquets par exemple)

Notamment, nous allons créer un API avec NodeJS (v.18) qui tourne en Ubuntu Linux (bullseye).

Nous utilisons Docker et VSCode afin de satisfaire ces demandes via leur [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

Il faut d'abord installer Docker et VSCode avant de procéder aux étapes suivantes. Un guide complèt se trouve [ici](https://docs.glassworks.tech/unix-shell/introduction/010-introduction/installation-party).

On crée un container qui tourne une version du système d'exploitation et interprète de notre choix. Idéalement, ce seront les mêmes que l'on utilisera en production (par exemple, Ubuntu Linux).

VSCode s'attache à ce container de plusieurs façons :

* on monte le dossier de notre projet (en local) dans le container via l'élément `volumes` de `docker compose`.
* quand on ouvre un terminal dans VSCode, c'est l'équivalent de lancer un `docker exec -it [ID du container]`. On lance donc un interprète _dans le container_. On peut ensuite installer et lancer des processus provenant de notre développement (par exemple, lancer l'API) dans son environnement précis.

Pour accomplir tout cela, VSCode exige la présence d'un dossier `.devcontainer` à la racine du workspace.

Nous commençons donc par créer ce dossier dans notre dossier de travail, et en ajoutant un fichier `devcontainer.json` dedans :

{% code title=".devcontainer/devcontainer.json" lineNumbers="true" %}
```json
{
  "name": "NodeJS API",
  // Pointer vers notre docker-compose.dev.yml
  "dockerComposeFile": [
    "../docker-compose.dev.yml"
  ],
  // Le service dans docker-compose.dev.yml auquel on va attacher VSCode
  "service": "vscode_api",
  // Le dossier de travail précisé dans Dockerfile.dev
  "workspaceFolder": "/home/dev",
  // Set *default* container specific settings.json values on container create.
  "customizations": {
    "settings": {},
    "extensions": [
      "pmneo.tsimporter",
      "stringham.move-ts",
      "rbbit.typescript-hero",      
    ]
  },
  // Quelques extensions VSCode à inclure par défaut pour notre projet 
  "forwardPorts": [ 5050 ]
}
```
{% endcode %}

Si on analyse bien ce fichier, on voit que VSCode va consulter un fichier `docker-compose.dev.yml` qui existe dans le répertoire parent pour lancer les services nécessaires pour ce parent.

Ce docker-compose pourrait à la fois contenir un service pour notre VSCode, mais aussi une base de données (de l'image mariadb), et d'autres services comme **redis**, ou autre.

Pour le moment, nous créons uniquement le service pour VSCode, qui doit porter le même nom que le champ `service` dans `devcontainer.json`.

{% code title="docker-compose.dev.yml" lineNumbers="true" %}
```yaml
version: '3.9'

services:
  vscode_api:
    image: rg.fr-par.scw.cloud/api-code-samples-vscode/vscode_api:2.0.0
    command: /bin/bash -c "while sleep 1000; do :; done"
    working_dir: /home/dev
    networks:
      - api-network
    volumes:
      - ./:/home/dev:cached
      
networks:
  api-network:
    driver: bridge
    name: api-network
```
{% endcode %}

Remarquez le service `vscode_api`, qui finalement tourne une commande `/bin/bash` en boucle infinie. VSCode s'attache à ce service. Le container est créé à partir d'une image que je vous ai déjà crée.

Nous sommes prêts à lancer notre Dev Container. Vérifiez bien la structure de votre projet VSCode.

<figure><img src="./structure.png" alt=""><figcaption></figcaption></figure>

Vous pouvez ensuite lancer votre Dev Container en appuyant sur **F1**, et puis **Dev Containers : Rebuild and Reopen in Container**.

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Une fois lancé, ouvrez un nouveau terminal.

Vous allez voir que NodeJS est déjà installé :

```bash
node -v
# v18.13.0
```

Au passage, nous avons installé **mycli** aussi, car on va travailler avec un SGBDR :

```bash
 mycli --version
 # Version: 1.23.2
```

Pour utiliser **typescript**, nous avons aussi ajouté **ts-node** et **typescript** comme commandes globales :

```bash
ts-node -v
# v10.9.1

tsc -v
# Version 5.0.4
```

{% hint style="danger" %}
Si vous avez des difficultés à démarrer votre dev-container, essayez ce qui suit :

- Ouvrez le Docker Dashboard, et essayez de supprimer tous les conteneurs qui pourraient être en conflit avec votre conteneur.
- Essayez de lancer `docker system prune -a --volumes` pour vider toutes les caches locales
- Redémarrez Docker
- Redémarrez votre ordinateur
{% endhint %}
