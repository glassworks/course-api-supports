# Variables d'environnement

> Vous trouverez le projet fonctionnel de ce chapitre [ici](https://dev.glassworks.tech/courses/api/api-code-samples/-/tree/007-deploy-monitor)

Nous avons déjà testé une mise en prod de notre API [ici](../003-security/050-https.md).

Pour un déploiement sécurisé, il serait prudent de changer des variables d'environnement :

- Pour l'accès au SGBDR
- Accès à des buckets de stockage pour la production
- Compte de mailing production
- ...


Ceci peut être fait en modifiant notre `docker-compose.stage.yml`  (sur le serveur uniquement) :

```yml
version: '3.9'

services:
  api:
    build: 
      context: ./
      dockerfile: ./docker/Dockerfile.stage
    container_name: api
    ports:
      - "5050:5050"
    environment:
      - NODE_ENV=prod
      - PORT=5050
      - FRONT_URL=https://test.api.hetic.glassworks.tech
      - DB_HOST=stage-dbms
      - DB_USER=api-stage
      - DB_PASSWORD=api-stage-password
      - DB_DATABASE=school     
      - AWS_ACCESS_KEY_ID=
      - AWS_SECRET_ACCESS_KEY=
      - STORAGE_REGION=fr-par
      - STORAGE_ENDPOINT=https://s3.fr-par.scw.cloud
      - STORAGE_BUCKET=object-storage-playground
      - MJ_APIKEY=
      - MJ_APISECRET=
      - MJ_EMAIL_FROM=kevin@nguni.fr
      - MJ_EMAIL_NAME=Kevin
    networks:
      - api-network
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
      - MYSQL_ROOT_PASSWORD=Hv4ffyMpbW8tjJby
    command: [
      "--character-set-server=utf8mb4",
      "--collation-server=utf8mb4_unicode_ci",
    ]
    volumes:
      - ./dbms/dbms-data:/var/lib/mysql
      - ./dbms/mariadb.cnf:/etc/mysql/mariadb.cnf
    networks:
      - api-network

  proxy:
    image: nginx
    ports:
      - "80:80"
      - "443:443"
    restart: always
    volumes:
      - ./nginx/api-https-final.nginx.conf:/etc/nginx/conf.d/api.nginx.conf
      - ./nginx/certbot/www:/var/www/certbot/:ro
      - ./nginx/certbot/conf/:/etc/nginx/ssl/:ro
    networks:
      - api-network

  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./nginx/certbot/www/:/var/www/certbot/:rw
      - ./nginx/certbot/conf/:/etc/letsencrypt/:rw
    networks:
      - api-network

networks:
  api-network:
    driver: bridge
    name: api-network
  
```

Notez le suivant :

- les entrées `environment` sur les services `api` et `dbms`
- l'entrée `labels` sur le service `api`
  - Un `label` est une étiquette qu'on attache au container Docker afin d'être plus facilement identifiable par la suite. Nous allons l'utiliser plus tard pour extraire uniquement les journaux provenant de ce container.


Lancez le stack sur votre instance, et n'oubliez pas de se connecter à votre SGBDR une première fois.

```bash
docker exec -it [ID du container] mariadb -u root -p
```

Créer la base de données :

```sql
create database mtdb_stage;
```

Créer l'utilisateur pour notre API:

```sql
create user IF NOT EXISTS 'api-stage'@'%.%.%.%' identified by 'api-stage-password';
grant select, update, insert, delete on mtdb_stage.* to 'api-stage'@'%.%.%.%';
flush privileges;
```

Créer le schéma :

```sql
use mtdb_stage;

/* user */
create table if not exists user (
  userId int auto_increment not null,
  email varchar(256) unique not null, 
  familyName varchar(256), 
  givenName varchar(256), 
  primary key(userId)
);

drop trigger if exists before_insert_user;

create trigger before_insert_user
before insert
on user for each row set new.email = lower(trim(new.email));

/* ... */


/* Fichier d'un utilisateur */
create table if not exists user_file (
  fileId int auto_increment not null,
  userId int not null,
  storageKey varchar(512) not null,
  filename varchar(256),
  mimeType varchar(256),
  primary key(fileId),
  foreign key(userId) references user(userId) on delete cascade
);

```

