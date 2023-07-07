# Stack de logging

> Vous trouverez le projet fonctionnel de ce chapitre [ici](https://dev.glassworks.tech:18081/courses/api/api-code-samples/-/tree/007-deploy-monitor)

Il est essentiel que l'on puisse suivre les logs générés par notre API. Bien sûr, il est possible juste de se connecter à l'instance qui tourne notre API, puis :

```sh
# Afficher les logs d'un container docker
docker logs [ID OU NOM DU CONTAINER] --follow
```

Si on a l'intention d'avoir plusieurs instances de notre API, sur plusieurs instances dans le cloud, on risque d'avoir plusieurs logs à regarder si jamais il y a un incident. Il serait pratique d'avoir un point central où on peut récupérer et regarder les logs.

Ici, on va faire rediriger tous nos logs vers une base NoSQL (mongodb) centralisé via un stack de plusieurs outils :

- `filebeat` : un service qui devrait être déployé sur toutes les instances de notre cloud qui va surveiller les journaux Docker sur chaque instance, et les envoyer vers un service centralisé pour traitement (`logstash`)
- `logstash` : qui tourne sur une instance dans notre cloud, et qui accepte tous les logs envoyés de partout, transforme/filtre les logs, et ensuite les insère dans notre MongoDB
- `mongodb` : notre base de données pour stocker et interroger des journaux

Nous allons utiliser un `docker-compose.logging.yml` pour créer ce stack. Il n'est pas obligé d'être dans le même stack que nos services de production.

## Filebeats

Filebeats tourne sur chaque instance de notre cloud, et lit les logs générés par Docker. On lui crée un service :

```yml
version: '3.8'

services:
  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.3
    user: root
    volumes:
      - ./logging/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - api-logging
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m" 
```

On a besoin de configurer Filebeats, via le fichier dans `./logging/filebeat.yml` :

```yml
# File to be duplicated on all worker nodes

filebeat.autodiscover:
  providers:
    - type: docker
      templates:
        - condition:
            equals:
              docker.container.labels.api_logging: "true"
          config:
            - type: container
              paths:
                - /var/lib/docker/containers/${data.docker.container.id}/*.log

processors:
- add_docker_metadata:
    host: "unix:///var/run/docker.sock"


- decode_json_fields:
    fields: ["message"]
    target: "logdata"
    overwrite_keys: true

output.logstash:
  hosts: ["logstash:5044"]

# logging.json: true
logging.metrics.enabled: false
# logging.level: debug

```

Ce fichier précise plusieurs choses :

- D'extraire uniquement les logs des containers avec le `label` de `api_logging: true`. Vous souvenez qu'on a [ajouté cet étiquette juste avant](./001-variables.md) ?
- Décoder le message (qui est en format JSON)
- Envoyer les messages vers `logstash`


## Logstash

On ajoute à notre `docker-compose.logging.yml` une entrée pour Logstash, qui récupère tous les journaux de Filebeats et les insère dans MongoDB.

```yml
  logstash:
    build: 
      context: ./
      dockerfile: ./logging/Dockerfile.logstash
    networks:
      - api-logging
    volumes:
      - ./logging/logstash.conf:/usr/share/logstash/pipeline/logstash.conf    
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"   
```

D'abord, on doit ajouter un `logging/Dockerfile.logstash` qui va construire une image pour logstash avec les modules pour MongoDB :

```Dockerfile
FROM docker.elastic.co/logstash/logstash:7.17.3

RUN bin/logstash-plugin install --version=3.1.5 logstash-output-mongodb && bin/logstash-plugin install logstash-filter-age
```

Logstash a besoin d'une configuration aussi (`logging/logstash.conf`) :


```conf
input {
  beats {
    port => 5044
  }
}

# Drop all older than 30 days
filter {
  age {}
  if [@metadata][age] > 2592000 {
    drop {}
  }
}

filter {
  # for logstash 5.x
  ruby { code => "event.set('epoc', event.get('@timestamp').to_i)" }
  # for logstash 2.x
  # ruby { code => "event['epoc'] = event['@timestamp'].to_i" }
}

filter {
  json {
    source => "message"
    target => "apilog"
  }
}

output {
  mongodb {
    id => "api-logcache"
    database => "api-logs"
    collection => "logs"
    uri => "mongodb://api_mongo_logger"
    # uri => "mongodb://[USER]:[PASS]@rms_mongo_logger"
    codec => "json"
  }
}
```

Ici, on précise le suivant :

- On écoute sur le port 5044
- On ignore tous les journaux plus vieux que 30 jours
- On ajoute un timestamp à chaque entrée
- On envoie les messages vers le service mongo, dans la base `api-logs`, et la collection `logs`


## Mongo

Enfin, on ajoute un service MongoDB pour accueillir nos journaux :

```yml
  api_mongo_logger:
    image: mongo:4.4.3
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all" ]
    ports: 
      - "5055:27017"
    volumes:
      - api_mongo_logs:/data/db
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"
    networks:
      - api-logging
```

Ici, on précise notamment qu'on va pouvoir accéder à cette base du port 5055.

## `docker-compose.logging.yml` entier

Voici le fichier complet :

```yml
version: '3.8'

services:
  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.3
    user: root
    volumes:
      - ./logging/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - api-logging
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m" 

  logstash:
    build: 
      context: ./
      dockerfile: ./logging/Dockerfile.logstash
    networks:
      - api-logging
    volumes:
      - ./logging/logstash.conf:/usr/share/logstash/pipeline/logstash.conf    
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"   

  api_mongo_logger:
    image: mongo:4.4.3
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all" ]
    ports: 
      - "5055:27017"
    volumes:
      - api_mongo_logs:/data/db
    logging:
      driver: "json-file"
      options:
        max-file: "5"
        max-size: "500m"
    networks:
      - api-logging

networks:
  api-logging:

volumes:
  api_mongo_logs:
```

Vous pouvez ensuite lancer votre stack de logging :


```bash
docker compose -f docker-compose.logging.yml up -d
```

A tout moment, on peut arrêter et déboguer le stack de logging sans interrompre notre stack de production. Pratique !

Télécharger un client mongodb ([MongoDB Compass](https://www.mongodb.com/try/download/shell) ou [Table Plus](https://tableplus.com)).

La connexion se fait via :

```
mongodb://[votre adresse IP]:5055/api-logs
```

<figure><img src="../.gitbook/assets/mongo.png" alt=""><figcaption></figcaption></figure>

{% hint style="warning" %}

Nous n'avons pas ajouté de la sécurisation sur notre instance MongoDB. Il faut faire attention à la sécuriser sur une vraie installation de production !

{% endhint %}

On peut par la suite, faire des interrogations, par exemple, afficher les requêtes sur une certaine route.

```js
db = db.getSiblingDB("api-logs");
db.getCollection("logs").find(
    {
        "apilog.url" : "/user"
    }
);
```

<figure><img src="../.gitbook/assets/mongo-logs.png" alt=""><figcaption></figcaption></figure>