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
docker compose up -d portainer
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
      - DOZZLE_BASE=/logs
    networks:
      ensg_sdi:
        ipv4_address: 172.24.0.30
```

Relancer : 

```bash
docker compose up -d
```

Accès :

- http://172.24.0.30:8080/logs

👉 Observer les logs de Portainer, directement dans l'application Dozzle.


## 3. Ajouter Filebrowser (données & volumes)

Pourquoi Filebrowser ?

- visualiser les **volumes**
- préparer la notion de **stock de données**
- éviter le “docker = boîte noire”

### Ajout du service 

```bash
filebrowser:
  image: filebrowser/filebrowser
  container_name: ${COMPOSE_PROJECT_NAME}_filebrowser
  restart: unless-stopped
  environment:
    - PUID=$(id -u)
    - PGID=$(id -g)
    - FB_BASEURL=/data
    # The list of avalable options can be found here : https://filebrowser.org/cli/filebrowser#options.
  expose:
    - "8080"
  volumes:
	- filebrowser_data:/srv
	- filebrowser_database:/database
	- filebrowser_config:/config
  networks:
    ensg_sdi:
      ipv4_address: 172.24.0.40
```

Accès :

- http://72.24.0.40/data

Lors du 1er lancement, observez les logs du conteneur afin de récupérer l'utilisateur par défaut (`admin`), et son mot de passe généré automatiquement une seule fois à la 1ère connexion. 

```docker
docker compose logs filebrowser
```

👉 Explorer le dossiers disponibles. 

Par la suite, afin de permettre la gestion des données de vos COTS (e.g. geoserver, etc.) via une interface graphique, il sera nécessaire d'associer les volumes dédiés tels que déclarés dans par exemple `geoserver`, dans la liste des volumes de `filebrowser` également. 

## 4. Ajouter PostgreSQL avec PostGIS

Nous allons maintenant ajouter un service PostgreSQL avec l'extension PostGIS.

### Ajout du service PostgreSQL/PostGIS

Ajoutez le service `postgis` dans `docker-compose.yml` :
- La configuration suivante crée la base `ensgdb`, l'utilisateur d'administration `ensgadmin`, ayant pour mot de passe `ensgpassword`

> [!NOTE]- Solution partielle

```yaml
  postgis:
    image: postgis/postgis:16-3.4
    container_name: ${COMPOSE_PROJECT_NAME}_postgis
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

Vérifiez qu'il fonctionne depuis `portainer` ou `dozzle`, ou encore `docker ps`

Testez la connexion :

```bash
docker compose exec -it postgis psql -U ensgadmin -d ensgdb
```

## 6. Ajouter et configurer pgAdmin 4

pgAdmin est une interface web pour gérer PostgreSQL.
### Ajout du service pgAdmin

Ajoutez le service `pgAdmin` dans `docker-compose.yml` :

> [!NOTE]- Solution partielle

```yaml
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: ${COMPOSE_PROJECT_NAME}_pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@ensg.eu
      PGADMIN_DEFAULT_PASSWORD: ensgpassword
      SCRIPT_NAME=/pgadmin
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      ensg_sdi:
        ipv4_address: 172.24.0.50
```

### Déploiement

```bash
docker-compose up -d pgadmin
```

Accédez à `http://172.24.0.50`/pgadmin et connectez-vous avec :

- **Email** : `admin@ensg.eu`
- **Mot de passe** : `ensgpassword`

### Installation des extensions PostGIS

Dans l'interface pgAdmin, commencez par enregistrer le serveur `postgis`. Exécutez ensuite  les commandes suivantes. Comment expliquez-vous les résultats remontés ?

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION hstore;
CREATE EXTENSION pg_stat_statements;
```

## 7. Ajouter et déployer GeoServer

GeoServer permet de diffuser des données géospatiales via des services OGC.

### Ajout d'un `healthcheck` à postgis

Geoserver dépend très fortement de ses founisseurs de données ; on va donc s'assurer qu'il ne cherche pas à démarrer avant que ces derniers soient prets. 

Ajouter un `healthcheck` au service `postgis` 

```bash
    healthcheck:
      test: "PGPASSWORD=${POSTGRES_PASS} pg_isready -h 127.0.0.1 -U ${POSTGRES_USER} -d ${POSTGRES_DB}"
```

### Ajout du service GeoServer

Ajoutez le bloc suivant dans `docker-compose.yml` . Les informations entre `${...}` représentent des **variables d'environnement**, attendues par `docker compose` dans un fichier `.env` co-localisé (voir juste après le bloc de service)

```yaml
  geoserver:
    container_name: ${COMPOSE_PROJECT_NAME}_geoserver
    image: kartoza/geoserver:${GS_VERSION}
    expose:
      - "8080"
    depends_on:
      postgis:
        condition: service_healthy
    environment:
      - GEOSERVER_ADMIN_PASSWORD=${GEOSERVER_ADMIN_PASSWORD}
      - GEOSERVER_ADMIN_USER=${GEOSERVER_ADMIN_USER}
      - INITIAL_MEMORY=${INITIAL_MEMORY}
      - MAXIMUM_MEMORY=${MAXIMUM_MEMORY}
      - GEOSERVER_DATA_DIR=${GEOSERVER_DATA_DIR}
      - GEOWEBCACHE_CACHE_DIR=${GEOWEBCACHE_CACHE_DIR}
      - ROOT_WEBAPP_REDIRECT=${ROOT_WEBAPP_REDIRECT}
      - TOMCAT_EXTRAS=${TOMCAT_EXTRAS}
      - SAMPLE_DATA=${SAMPLE_DATA}
      # Extensions set to be installed
      - STABLE_EXTENSIONS=${STABLE_EXTENSIONS}
      - COMMUNITY_EXTENSIONS=${COMMUNITY_EXTENSIONS}
    volumes:
      - geoserver-data:/opt/geoserver/data_dir
      - geoserver-injected-data:/opt/geoserver/data_dir/injected
      - geoserver-settings:/settings
    healthcheck:
      test: "curl --fail --silent --write-out 'HTTP CODE : %{http_code}\n' --output /dev/null -u ${GEOSERVER_ADMIN_USER}:'${GEOSERVER_ADMIN_PASSWORD}' http://localhost:8080/geoserver/rest/about/version.xml"
      interval: 1m30s
      timeout: 10s
      retries: 3
    networks:
      ensg_sdi:
        ipv4_address: 172.24.10.3
```

Ajoutez le contenu suivant dans un fichier `.env` situé au meme niveau que votre fichier de composition : 

```bash
#
# -> Project name
COMPOSE_PROJECT_NAME=ensg-sdi-hub

#
# -> PostGIS
# kartoza/postgis env variables https://github.com/kartoza/docker-postgis
POSTGIS_VERSION=16-3.4
POSTGRES_DB=geoserver
POSTGRES_USER=postgis
POSTGRES_PASS=postgis
ALLOW_IP_RANGE=0.0.0.0/0
POSTGRES_PORT=32767
# -> pgAdmin4
PGADMIN_MAIL=admin@ensg.eu
PGADMIN_PASSWORD=postgis

#
# -> Geoserver
GS_VERSION=2.24.2
GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=geoserver
# https://docs.geoserver.org/latest/en/user/datadirectory/setting.html
GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
# https://docs.geoserver.org/latest/en/user/geowebcache/config.html#changing-the-cache-directory
GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc
# Show the tomcat manager in the browser
TOMCAT_EXTRAS=false
ROOT_WEBAPP_REDIRECT=true
# https://docs.geoserver.org/stable/en/user/production/container.html#optimize-your-jvm
INITIAL_MEMORY=2G
MAXIMUM_MEMORY=4G
#
# Data and extensions
SAMPLE_DATA=false
# Full compatibility list : https://github.com/kartoza/docker-geoserver/blob/master/build_data/ Look for *_plugins.txt
STABLE_EXTENSIONS=css-plugin,imagemap-plugin,importer-plugin,wmts-multi-dimensional-plugin,ysld-plugin,h2-plugin
COMMUNITY_EXTENSIONS=backup-restore-plugin,geopkg-plugin,notification-plugin,ogcapi-plugin,smart-data-loader-plugin,wmts-styles-plugin
```

Sauvegardez le fichier.

Ajoutez les volumes `data` de `geoserver` parmi les volumes connus de `filebrowser`, afin d'en permettre l'exploration via cet outil. 

```yaml
filebrowser:
  image: filebrowser/filebrowser
  container_name: ${COMPOSE_PROJECT_NAME}_filebrowser
  restart: unless-stopped
  environment:
    - PUID=$(id -u)
    - PGID=$(id -g)
    # - FB_BASE_URL=/data
    # The list of avalable options can be found here : https://filebrowser.org/cli/filebrowser#options.
  expose:
    - "8080"
  volumes:
    - filebrowser_data:/srv
    - filebrowser_database:/database
    - filebrowser_config:/config
    - geoserver-data:/srv/geoserver-data
    - geoserver-injected-data:/srv/geoserver-injected
    - geoserver-settings:/srv/geoserver-settings
  networks:
    ensg_sdi:
      ipv4_address: 172.24.0.40
```

### Déploiement

```bash
docker compose up -d
```

Seuls les services `geoserver` et `filebrowser` sont redémarrés; pourquoi ? 

> [!tip]- Réponse
> docker compose fait l'inventaire des modifications de configuration, et de recrée ou redémarre que les seules ressources nécessaires !


Accédez à `http://172.24.0.11:8080/geoserver` et connectez-vous avec :

- **Utilisateur** : `admin`
- **Mot de passe** : `geoserver`

### Connexion à PostgreSQL

1. Dans GeoServer, créez dans cet ordre : 
	1. un nouvel `espace de travail (workspace)` 
	2. un nouvel `entrepôt (Store) `
2. Choisissez `PostGIS`
3. Remplissez les champs :
    - **Database** : `ensgdb`
    - **User** : `ensgadmin`
    - **Password** : `ensgpassword`
    - **Host** : `postgis`
    - **Port** : `5432`
4. Cliquez sur `Save`

Votre entrepôt configuré, vous ne pouvez pas encore ajouter de couches. 
Pourquoi ? 

> [!tip]- Réponse
> La base PostGIS ne contient pas encore de données; il est donc cohérent de ne pas en retrouver ici, non plus!

### Ajouter un client carto efficace : `mapstore2`

Ajoutez simplement le service suivant dans votre `docker-compose.yml` 

```bash
  #  -> MapStore2
  mapstore2:
    image: geosolutionsit/mapstore2:latest
    container_name: ${COMPOSE_PROJECT_NAME}_mapstore2
    restart: unless-stopped
    expose: 
      - "8080"
    networks:
      eNSG_SDI:
        ipv4_address: 172.24.10.5
```

## 8. Synthèse des outils disponibles à cette étape 

Bravo! Vous avez déployé un grand nombre de services, inventoriés très simplement dans un fichier `docker-compose.yml`, et pouvez désormais les lier entre eux pour animer une **Infrastructure de Données Géospatiales (IDG)** simple. 

**Outils d'administration** : 
- **Portainer**: http://172.24.0.22:9000 , votre outil d'administration des conteneurs.
- **Dozzle**: http://172.24.0.30:8080/ , votre outil centralisé de gestion des journaux des conteneurs
- **PostGIS via PGAdmin** : http://http://172.24.0.50/pgadmin/ (admin@ensg.eu/ensgpassword)
- **Filebrowser**: http://172.24.0.40/data (admin/mdp généré au lancement), votre outil Web de gestion des fichiers contenus par les volumes de votre IDG (e.g. geoserver)

**Serveurs et clients cartographiques** : 
- **Geoserver**: http://172.24.10.3:8080/geoserver/ (admin/geoserver), serveur cartographique de référence, simple et très intuitif
- **MapStore2**: http://172.24.10.5:8080/mapstore/#/ (admin/admin) , son alter-ego client naturel.

## (Bonus). Ajout et configuration du reverse proxy NGinx

Bien que vous n'en ayez pas strictement besoin pour un fonctionnement de bout en bout de votre infrastructure, la notion de reverse proxy est absolument essentielle dans un infrastructure informatique. 

Interlocuteur unique entre vos clients et votre IDG, le reverse proxy a notamment pour avantage : 
- de masquer les adresses IP et numéros de ports de vos services
- de masquer une grande partie desdits services (notamment ceux d'administration)
- (non implémenté dans ce lab) de centraliser la gestion des authentification
- (non implémenté dans ce lab) de centraliser la gestion des certificats pour le https
- (non implémenté dans ce lab) de s'occuper de la gestion de charge serveurs

### URL (via `/etc/hosts`)
Dans ce lab, on configure la ligne suivante dans le fichier `/etc/hosts` :

```bash
127.0.0.1  ensg-sdi.docker
```


**reverse proxy - le principe** : 
- Inventaire des URL désirées ✅
- Vérification des possibilités des COTS ✅
- Création du services de proxy
- Configuration des règles d'aiguillage
- Configuration des COTS concernés

On chercher à mettre en oeuvre les accès suivants :

-  http://ensg-sdi.docker/ : Mapstore2
- `http://ensg-sdi.docker/geoserver` : (GeoServer est nativement sous `/geoserver`) [Documentation GeoServer](https://docs.geoserver.org/main/en/user/installation/docker.html?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/files` : (Filebrowser avec baseurl) [GitHub+1](https://github.com/filebrowser/filebrowser/issues/1557?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/logs` : (Dozzle avec `DOZZLE_BASE`) [Dozzle+1](https://dozzle.dev/guide/changing-base?utm_source=chatgpt.com)
- `http://ensg-sdi.docker/pgadmin` : (pgAdmin sous sous-répertoire via `SCRIPT_NAME`)

La démarche peut se révéler très laborieuse pour certains logiciels qui n'ont pas été spécifiquement conçus pour se placer derrière un proxy. Il est parfois illusoire d'espérer configurer un service sous forme de `sous-dossier`. 

### Ajout du service nginx-proxy

Ajoutez le bloc suivant dans votre `docker-compose.yml` 

```bash
  #
  # Project reverse proxy
  nginx-proxy:
    image: ${COMPOSE_PROJECT_NAME}_proxy:latest
    container_name: ${COMPOSE_PROJECT_NAME}_proxy
    restart: unless-stopped
    expose:
      - "80"
    depends_on:
      - pgadmin
      - portainer
      - filebrowser
      - geoserver
    build:
      context: ./config/nginx-proxy
    environment:
      - DHPARAM_GENERATION=false
      - VIRTUAL_PORT=80
      - VIRTUAL_HOST=ensg-sdi.docker
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
    profiles:
      - proxyfied
    networks:
      ensg-sdi:
        ipv4_address: 172.24.0.2
```

Remarquez la directive `build` : elle pointe vers le répertoire contenant à minima un fichier Dockerfile de construction de service. 

### Configuration du proxy

Créez ce répertoire, et éditez le fichier `./config/Dockerfile` 

```Dockerfile
FROM nginx:alpine

COPY ensg-sdi.docker.conf /etc/nginx/conf.d/ensg-sdi.docker.conf
```

Discuter avec l'enseignant sur la signification de ces directives. 

#### Aiguillage de notre IDG

Le fichier `ensg-sdi.docker.conf` nous intéresse beaucoup ici, et va contenir les règles d'aiguillage des outils implémentés so far. 

```nginx

log_format sdi-vhost '$host $remote_addr - $remote_user [$time_local] '
'"$request" $status $body_bytes_sent '
'"$http_referer" "$http_user_agent"';

##
#client_max_body_size 4G;
#large_client_header_buffers 4 32k;

#
# Set resolver to docker default DNS
resolver 127.0.0.11 valid=30s;

# server blocks definition
server {
	#
	# Doit refl..ter le nom de domaine couvert par ce block server
	server_name ensg-sdi.docker;
	listen 80 ;
	access_log /var/log/nginx/access.log sdi-vhost;
	
	# (optionnel mais pratique) limite upload
	client_max_body_size 4G;
	large_client_header_buffers 4 32k;
	
	# Headers communs
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
	  
	#
	# ->
	location / {
		set $upstream homepage;
		proxy_pass http://$upstream:3000;
		proxy_http_version 1.1;
	}
	# -> Geoserver
	# See (https://github.com/kartoza/docker-geoserver)
	location /geoserver/ {
		# On d..finit dans une variable pour ..viter un crash du proxy en cas d'indispo du service
		set $upstream geoserver;
		#
		proxy_pass http://$upstream:8080;
		proxy_redirect off;
		proxy_set_header X-Forwarded-Prefix /geoserver;
		proxy_http_version 1.1;
	}
	
	## -> Mapstore2
	# See Tomcat behind reverse proxy -> https://clouding.io/hc/en-us/articles/360010691359-How-to-Install-Tomcat-with-Nginx-as-a
	location /mapstore/ {
		# On d..finit dans une variable pour ..viter un crash du proxy en cas d'indispo du service
		set $upstream mapstore2;
		#
		proxy_pass http://$upstream:8080;
		proxy_set_header X-Forwarded-Prefix /mapstore;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Server $host;
	}
	
	#
	# -> Filebrowser : files admin web GUI for our stack
	location /data {
		# On d..finit dans une variable pour ..viter un crash du proxy en cas d'indispo du service
		set $upstream filebrowser;
		# prevents 502 bad gateway error
		proxy_buffers 8 32k;
		proxy_buffer_size 64k;
		client_max_body_size 75M;
		# redirect all HTTP traffic to localhost:8088;
		proxy_pass http://$upstream;
		proxy_set_header X-Forwarded-Prefix /data;
		# enables WS support
		proxy_http_version 1.1;
	}
	
	location /logs/ {
		set $upstream dozzle;
		proxy_set_header X-Forwarded-Prefix /logs;
		proxy_pass http://$upstream:8080;
		proxy_redirect off;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
	}
	
	# -> pgadmin
	# See (https://www.pgadmin.org/docs/pgadmin4/6.21/container_deployment.html#http-via-nginx)
	location /pgadmin/ {
		# On d..finit dans une variable pour ..viter un crash du proxy en cas d'indispo du service
		set $upstream pgadmin;
		
		# Configuration du routage
		proxy_set_header X-Script-Name /pgadmin;
		proxy_set_header Host $host;
		proxy_pass http://$upstream;
		proxy_redirect off;
	}
	
	# -> Portainer
	location /portainer {
		return 301 $scheme://$host/portainer/;
	}
	
	location ^~ /portainer/ {
		# include /config/nginx/resolver.conf;
		set $upstream_app portainer;
		set $upstream_port 9000;
		set $upstream_proto http;
		proxy_pass $upstream_proto://$upstream_app:$upstream_port;
		rewrite /portainer(.*) $1 break;
		proxy_hide_header X-Frame-Options; # Possibly not needed after Portainer 1.20.0
	}
	
	location ^~ /portainer/api {
		# include /config/nginx/resolver.conf;
		set $upstream_app portainer;
		set $upstream_port 9000;
		set $upstream_proto http;
		proxy_pass $upstream_proto://$upstream_app:$upstream_port;
		rewrite /portainer(.*) $1 break;
		proxy_hide_header X-Frame-Options; # Possibly not needed after Portainer 1.20.0
	}
	##
}

```

Discussion avec l'enseignant pour l'explication de ces directives. 

#### Lancement et vérifications

Lorsque vous etes prets à tester votre configuration, exécutez la commande suivante afin de lancer le proxy : 

```bash
docker compose up -d
```

Que fais cette commande ? 
Que se passe-t-il ? 
Où est le service de proxy ? 

Lancez maintenant de manière explicite le "mode proxy", grace au profil associé : 

```bash
docker compose --profile proxyfied up -d
```

Debug en groupe, service par service. 
Les stagiaires les moins moteurs pourront passer directement à la création de basemap. 

## (Bonus) Homepage – Portail de la GeoStack

Mettre en place une **page d’accueil unifiée** pour la stack SDI, servant de :
- point d’entrée unique pour les outils,
- support pédagogique pour comprendre l’architecture, 
- interface de navigation simple pour les utilisateurs non techniques.

La solution retenue est **Homepage (gethomepage)**, servie à la racine via le reverse proxy NGINX.

URL cible :  
👉 `http://ensg-sdi.docker/`

### 🧩 Rôle de Homepage dans l’architecture

Homepage **ne remplace aucun service** :
- ❌ pas un proxy
- ❌ pas un orchestrateur
- ❌ pas un catalogue

👉 C’est un **tableau de bord statique**, qui :
- documente l’existant,
- expose les URLs propres,
- rend visible la structure du système.

### 🐳 Ajout du service docker compose

Le service Homepage est ajouté au `docker-compose.yml` :

```yaml
# -> Homepage
homepage:
  image: gethomepage/homepage:latest
  container_name: ${COMPOSE_PROJECT_NAME}_homepage
  restart: unless-stopped
  expose:
    - "3000"
  volumes:
    - ./config/homepage:/app/config
    - /var/run/docker.sock:/var/run/docker.sock:ro
  environment:
    - HOMEPAGE_ALLOWED_HOSTS=ensg-sdi.docker
    - PUID=1000
    - PGID=1000
  networks:
    ensg_sdi:
      ipv4_address: 172.24.10.6
```

Arborescence associée :

```arduino
homepage/
├── config/
│   ├── settings.yaml
│   ├── services.yaml
│   └── widgets.yaml   (optionnel)
└── icons/             (optionnel)
```

👉 **Aucun code**, uniquement de la configuration déclarative.

### Fichiers de configuration

`settings.yaml` (minimal et propre)

```yaml
title: ENSG SDI – GeoStack
theme: dark
color: slate

background:
  image: /icons/bg.jpg   # optionnel
  blur: sm
  saturate: 120

layout:
  Infrastructure:
    style: row
    columns: 3

headerStyle: boxed
hideErrors: true

```


`services.yaml` (aligné avec tes URLs proxy)


```yaml
- Infrastructure:
    - GeoServer:
        icon: geoserver
        href: http://ensg-sdi.docker/geoserver
        description: Services OGC (WMS/WFS/WCS)

    - MapStore:
        icon: mapstore
        href: http://ensg-sdi.docker/mapstore
        description: Applications cartographiques

    - GeoNetwork:
        icon: geonetwork
        href: http://ensg-sdi.docker/geonetwork
        description: Catalogue de métadonnées

- Administration:
    - pgAdmin:
        icon: postgres
        href: http://ensg-sdi.docker/pgadmin
        description: Administration PostgreSQL

    - Portainer:
        icon: docker
        href: http://ensg-sdi.docker/portainer
        description: Containers & stacks

    - Dozzle:
        icon: logs
        href: http://ensg-sdi.docker/logs
        description: Logs temps réel

    - Filebrowser:
        icon: folder
        href: http://ensg-sdi.docker/data
        description: Données & produits

```

💡 Même si GeoNetwork n’est pas encore branché côté NGINX, tu peux déjà le préparer ici.


### `.gitignore` – Homepage (formation)

Objectif :  
👉 **Versionner uniquement les fichiers YAML créés/modifiés en séance**,  
et ignorer tout le reste (cache, exemples, assets auto-générés).

```gitignore
# Ignore tout par défaut dans homepage
homepage/**

# Autorise uniquement les dossiers utiles
!homepage/config/
!homepage/icons/

# Autorise uniquement les fichiers YAML initiés
!homepage/config/settings.yaml
!homepage/config/services.yaml
!homepage/config/widgets.yaml

# Autorise les icônes ajoutées explicitement
!homepage/icons/**

# Sécurité
#.env
*.log
.DS_Store

```

## Conclusion

Félicitations ! Vous avez mis en place une infrastructure de données géospatiales avec Docker Compose. Vous pouvez maintenant exploiter ces données dans vos applications SIG.