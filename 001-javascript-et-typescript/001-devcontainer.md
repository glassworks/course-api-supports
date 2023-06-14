# Docker Dev Container

Avant la création de notre API, il serait intéressant de créer un environnement de développement qui :

* facilement duplicable d'un ordinateur à l'autre
* indépendant du système d'exploitation
* contient le set de dépendances du projet (les versions des paquets par exemple)

Notamment, nous allons créer un API avec NodeJS (v.18) qui tourne dans Ubuntu Linux (bullseye).

Nous utilisons Docker et VSCode afin de satisfaire ces demandes via leur [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

On crée un container qui tourne uns version du système d'exploitation et interprète de notre choix. Idéalement ce sera les mêmes qu'on utilisera en production (par exemple, Ubuntu Linux).

VSCode s'attache à ce container de plusieurs façons :

* on monte le dossier de notre projet (en local) dans le container via l'élément `volumes` de `docker compose`.
* quand on ouvre un terminal dans VSCode, c'est l'équivalent de lancer un `docker exec -it [ID du container]`. On lance donc un interprète _dans le container_. On peut ensuite installer et lancer des processus provenant de notre développement (par exemple, lancer l'API) dans son environnement précis.

Pour accomplir tout cela, VSCode exige la présence d'un dossier `.devcontainer` à la racine du workspace.

Nous commençons donc par créer ce dossier dans notre dossier de travail, et en ajoutant un fichier `devcontainer.json` dedans :

{% code title=".devcontainer/devcontainer.json" lineNumbers="true" %}
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
  "workspaceFolder": "/home/dev",
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
{% endcode %}

Si on analyse bien ce fichier, on voit que VSCode va consulter un fichier `docker-compose.dev.yml` wui existe dans le répertoire parent pour lancer les services nécéssaires pour ce parent.

Ce docker-compose pourrait à la fois contenir un service pour notre VSCode, mais aussi une base de données (de l'image mariadb), et d'autres services comme **redis**, ou autre.

Pour le moment, nous créons uniquement le service pour VSCode, qui doit porter le même nom que le champ `service` dans `devcontainer.json`.

{% code title="docker-compose.dev.yml" lineNumbers="true" %}
```yaml
version: '3.9'

services:
  vscode_api:
    build: 
      context: ./
      dockerfile: ./docker/Dockerfile.dev
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

Remarquez le service `vscode_api`, qui finalement tourne une commande `/bin/bash` en boucle infinei. VSCode s'attache à ce service. Le container est crée à partir d'une image personnalisé. Les instructions des création de cette image doivent se trouver dans `docker/Dockerfile.dev` :

{% code title="docker/Dockerfile.dev" lineNumbers="true" %}
```docker
FROM node:18
# Forcer le faite d'utiliser les miroirs français quand on utilise apt ...
# Attention, le chemin source est rélative à l'emplacement du fichier docker-compose
COPY ./docker/sources.list /etc/apt/sources.list

ARG USERNAME=dev
ARG USER_UID=1001
ARG USER_GID=$USER_UID

# Créer l'utilisateur et son groupe, installer des paquets
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \    
    && apt-get update \        
    && apt-get install -y sudo \
    && apt-get install -y mycli \
    && apt-get install -y tzdata \    
    && npm install -g typescript \
    && npm install -g ts-node \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME


# Changer le user à dev
USER $USERNAME

# Fixer le fuseau horaire
ENV TZ Europe/Paris

# L'interprète par défaut
ENV SHELL /bin/bash

# Le repertoire maison par défaut
WORKDIR /home/dev
RUN /bin/bash
```
{% endcode %}

Enfin, remarquez que le Dockerfile remplace un fichier dans /etc/apt/sources.list par une version dans docker/sources.list. C'est la liste de dépôts Ubuntu duquel on charge les paquets connus par apt. Pour fonctionner dans l'école (sans être bloque) on est obligé de passer par des miroirs français :&#x20;

{% code title="docker/sources.list" lineNumbers="true" %}
```
# Privilégier d'abord les miroirs français

deb http://ftp.fr.debian.org/debian bullseye main
deb http://ftp.fr.debian.org/debian-security bullseye-security main
deb http://ftp.fr.debian.org/debian bullseye-updates main

# En derniers recours, essayez les dépôts principaux

# deb http://deb.debian.org/debian bullseye main
# deb http://security.debian.org/debian-security bullseye-security main
# deb http://deb.debian.org/debian bullseye-updates main
```
{% endcode %}

Si jamais vous avez un souci de version (et vous n'êtes pas à l'école) vous pouvez décommenter les dernières lignes pour utiliser les dépots officiaux.&#x20;

Nous sommes prêts à lancer notre Dev Container. Vérifiez bien la structure de votre projet VSCode.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Vous pouvez ensuite lancer votre Dev Container en appuyant sur **F1**, et puis **Dev Containers : Rebuild and Reopen in Container**.

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Une fois lancé, ouvrez un nouveau terminal.

Vous allez voir que NodeJS est déjà installé :&#x20;

```bash
node -v
# v18.13.0
```

Au passage, nous avons installés **mycli** aussi, car on va travailler avec un SGBDR :&#x20;

```bash
 mycli --version
 # Version: 1.23.2
```

Pour utiliser **typescript**, nous avons aussi ajouté **ts-node** et **typescript** comme commandes globales :&#x20;

```bash
ts-node -v
# v10.9.1

tsc -v
# Version 5.0.4
```
