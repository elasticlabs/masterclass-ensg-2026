---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j2-conteneurisation-and-reverse-proxy/lab-02-construire-le-socle-de-l-idg/","noteIcon":""}
---


## 🎯 Objectifs

À l’issue de ce lab, vous serez capables de :

- comprendre le **fonctionnement de Docker**
- lancer et inspecter des conteneurs
- structurer une pile **docker-compose**
- exposer des services via des ports
- ajouter des **healthchecks**
- gérer les **dépendances entre services**
- raisonner “infra comme un système”


Aujourd’hui, nous construisons le **socle technique** :

- **conteneurs** = briques isolées
- **compose** = graphe de dépendances
- **healthchecks** = état de santé (stocks d’état)
- **logs** = signaux du système

👉 Sans ce socle, une SDI devient fragile et opaque.

---

## Vérifications locales

```bash
docker version
docker compose version
```

Si l’une des commandes échoue → corriger **avant** d’aller plus loin.

## 1. Premier docker-compose (Portainer)

**Pourquoi Portainer ?**
- visualiser conteneurs, volumes, réseaux
- comprendre **ce que fait Docker** graphiquement

Comme la plupart des aspects de docker, `compose` est très bien documenté! 
- RDV sur https://docs.docker.com/reference/compose-file/

### Créez le réseau `ensg_sdi` 

Ce réseau supportera l'ensemble de nos services par la suite. Pour des raisons de simplification du développement (mais pas de la configuration 😇), les adresses IP sont fixées pour chaque service. 

```shell
docker network create --subnet=172.24.0.0/16 --driver bridge ensg_sdi
```

### 2.2 docker-compose.yml (Portainer seul)

Ajoutez le service `portainer` dans un fichier `docker-compose.yml` (attention! en soi, ce n'est pas du tout suffisant pour faire un fichier fonctionnel!)
- Image : portainer/portainer-ce
- Toujours redémarrer
- exposer le port 9000
- Volumes par défaut (portainer_data en volume nommé)
- Réseau sdi_apps, fixer l'adresse IP à 172.24.0.22

> [!NOTE]- Solution partielle

```yaml
services:  
  portainer:
    image: portainer/portainer-ce
    container_name: portainer
    restart: always
    expose:
      - "9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      ensg_sdi:
        ipv4_address: 172.24.0.22

volumes:
  portainer_data:

networks:
  ensg_sdi:
    external: true
```

<u>Question</u> : que devez-vous ajouter pour compléter cette configuration ? 
- Indice : pensez aux volumes, au réseau, et à la structure générale d'un fichier docker compose!

### Déploiement

Lancez Portainer avec :

```bash
docker-compose up -d portainer
```

Accédez à l'interface de Portainer : `http://172.24.0.22:9000`

1. Créez un compte administrateur
2. Connectez Portainer à l'environnement Docker local

<u>Quizz</u> : qu'elle est la différence entre les directive `expose` et `ports` ?


## 2. Ajouter Dozzle (logs temps réel)

**Rôle de Dozzle**

- visualiser les **logs**
- comprendre le comportement d’un système vivant
### 3.1 Ajout du service

```bash
  dozzle:
    image: amir20/dozzle:latest
    container_name: ensg-dozzle
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DOZZLE_LEVEL=info
    networks:
      ensg_sdi:
        ipv4_address: 172.28.0.30
```

Relancer : 

```bash
docker compose up -d
```

Accès :

- http://localhost:8081

👉 Observer les logs de Portainer.


## 3. Ajouter Filebrowser (données & volumes)

Pourquoi Filebrowser ?

- visualiser les **volumes**
- préparer la notion de **stock de données**
- éviter le “docker = boîte noire”

### Ajout du service 

```bash
    image: filebrowser/filebrowser:latest
    container_name: ensg-filebrowser
    restart: unless-stopped
    volumes:
      - ./data:/srv
      - ./data/filebrowser/filebrowser.db:/database/filebrowser.db
    command: ["--noauth"]
    networks:
      ensg_sdi:
        ipv4_address: 172.28.0.40
```

Accès :

- http://localhost:8082

👉 Explorer le dossier `data/`.

## 4. Ajouter des healthchecks

### 5.1 Pourquoi ?

- un conteneur “lancé” ≠ un service “opérationnel”
- le **healthcheck** est un signal système 

### 5.2 Exemples de healthchecks

#### Portainer

```bash
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9000"]
      interval: 30s
      timeout: 5s
      retries: 3
```

#### Dozzle

```bash
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080"]
      interval: 30s
      timeout: 5s
      retries: 3
```

Vérification avec commande `docker ps`, observez la colonne `STATUS` `


## 5. Ajouter PostgreSQL avec PostGIS

Nous allons maintenant ajouter un service PostgreSQL avec l'extension PostGIS.

### Ajout du service PostgreSQL/PostGIS

Ajoutez le service `postgis` dans `docker-compose.yml` :

> [!NOTE]- Solution partielle

```yaml
  postgis:
    image: postgis/postgis:16-3.4
    container_name: postgis
    restart: always
    expose:
      - "5432"
    environment:
      POSTGRES_DB: ensgdb
      POSTGRES_USER: ensgadmin
      POSTGRES_PASSWORD: ensgpassword
      ALLOW_IP_RANGE: 0.0.0.0/0
      FORCE_SSL: FALSE
    volumes:
      - postgis_data:/var/lib/postgresql/data
    networks:
      ensg_sdi:
        ipv4_address: 172.24.0.10
```

Lancez le service :

```bash
docker-compose up -d postgis
```

Vérifiez qu'il fonctionne :

```bash
docker ps
```

Testez la connexion :

```bash
docker exec -it postgis psql -U ensgadmin -d ensgdb
```

### Test de connexion depuis VS Code

1. Installez l'extension **PostgreSQL** dans VS Code.
2. Ouvrez le panneau `PostgreSQL Explorer`.
3. Cliquez sur `Add Connection` et renseignez :
    - **Host** : `172.24.0.10`
    - **Port** : `5432`
    - **User** : `ensgadmin`
    - **Password** : `ensgpassword`
    - **Database** : `ensgdb`
4. Testez la connexion.

---

## 6. Ajouter et configurer pgAdmin 4

pgAdmin est une interface web pour gérer PostgreSQL.

### Ajout du service pgAdmin

Ajoutez le service `pgAdmin` dans `docker-compose.yml` :

> [!NOTE]- Solution partielle

```yaml
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@ensg.eu
      PGADMIN_DEFAULT_PASSWORD: ensgpassword
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    expose:
      - "5050:80"
    networks:
      ensg_sdi:
        ipv4_address: 172.24.0.20
```

### Déploiement

```bash
docker-compose up -d pgadmin
```

Accédez à `http://172.24.0.20:5050` et connectez-vous avec :

- **Email** : `admin@ensg.eu`
- **Mot de passe** : `ensgpassword`

### Installation des extensions PostgreSQL

Dans l'interface pgAdmin, exécutez les commandes suivantes. Comment expliquez-vous les résultats remontés ?

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION hstore;
CREATE EXTENSION pg_stat_statements;
```

## 7. Ajouter et déployer GeoServer

GeoServer permet de diffuser des données géospatiales via des services OGC.

### Ajout du service GeoServer

Ajoutez le bloc suivant dans `docker-compose.yml` :

```yaml
  geoserver:
    image: kartoza/geoserver
    container_name: geoserver
    restart: always
    environment:
      # - RDV sur https://github.com/kartoza/docker-geoserver?tab=readme-ov-file#environment-variables pour une liste exhaustive, à vous de jouer!
      - TODO
    expose:
      - "8080"
    volumes:
      - geoserver_data:/opt/geoserver/data_dir
      - geoserver_bal:/opt/geoserver/data_dir/_BAL_
      - geoserver_settings:/settings
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.11
```

> [!NOTE]- Variables d'environnement geoserver - Solution partielle
>    - "CORS_ENABLED=true"
      - GEOSERVER_ADMIN_PASSWORD=geoserver
      - GEOSERVER_ADMIN_USER=admin
      - INITIAL_MEMORY=500M
      - MAXIMUM_MEMORY=1G
      - GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
      - GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc
      - ROOT_WEBAPP_REDIRECT=true
      - TOMCAT_EXTRAS=false
      - SAMPLE_DATA=false
      # Extensions set to be installed
      - "INSTALL_EXTENSIONS=true"
      - STABLE_EXTENSIONS=css-plugin,importer-plugin,wmts-multi-dimensional-plugin
      - COMMUNITY_EXTENSIONS=backup-restore-plugin,ogcapi-plugin,smart-data-loader-plugin,wmts-styles-plugin

### Déploiement

```bash
docker-compose up -d geoserver
```

Accédez à `http://172.24.0.11:8080/geoserver` et connectez-vous avec :

- **Utilisateur** : `admin`
- **Mot de passe** : `geoserver`

### Connexion à PostgreSQL

1. Dans GeoServer, allez dans `Stores > Add new Store`
2. Choisissez `PostGIS`
3. Remplissez les champs :
    - **Database** : `ensgdb`
    - **User** : `ensgadmin`
    - **Password** : `ensgpassword`
    - **Host** : `postgis`
    - **Port** : `5432`
4. Cliquez sur `Save`

## 8. Ajout et configuration du reverse proxy NGinx

### URL (via `/etc/hosts`)
Dans ce lab, on configure :

```bash
127.0.0.1  ensg-sdi.docker
```

Puis on chercher à mettre en oeuvre les accès suivants :

- `http://ensg-sdi.docker/geoserver` ✅ (GeoServer est nativement sous `/geoserver`) [Documentation GeoServer](https://docs.geoserver.org/main/en/user/installation/docker.html?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/files` ✅ (Filebrowser avec baseurl) [GitHub+1](https://github.com/filebrowser/filebrowser/issues/1557?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/logs` ✅ (Dozzle avec `DOZZLE_BASE`) [Dozzle+1](https://dozzle.dev/guide/changing-base?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/pgadmin` ✅ (pgAdmin sous sous-répertoire via `SCRIPT_NAME`)



## Conclusion

Félicitations ! Vous avez mis en place une infrastructure de données géospatiales avec Docker Compose. Vous pouvez maintenant exploiter ces données dans vos applications SIG.