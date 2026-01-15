---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j5-notebooks-jupyter-gouvernance-industrialisation-and-restitution/lab-07-bonus-automatiser-et-regrouper-sa-configuration/","noteIcon":""}
---


## Objectifs

Ce lab a pour objectif d'améliorer l'organisation et la gestion de votre infrastructure de données géospatiales en utilisant :

- Un fichier `.env` pour centraliser les variables de configuration
- Un `Makefile` pour simplifier l'exécution des commandes courantes
- Un dossier `config` pour stocker la configuration de certains services
- L'ajout de dépendances et contrôles d'état de santé des services 

---

## 1. Centralisation des variables dans un fichier `.env`

Nous allons extraire les variables de configuration de `docker compose.yml` et les placer dans un fichier `.env`.

### Création du fichier `.env`

Créez un fichier `.env` à la racine de votre projet et ajoutez les lignes suivantes :

```env
#
# -> Project name
COMPOSE_PROJECT_NAME=ensg-sdi

# -> Proxy docker networks
#  - APPS_NETWORK will be the network name you use in deployed compose services
APPS_NETWORK=sdi_apps

# -> PostGIS
# kartoza/postgis env variables https://github.com/kartoza/docker-postgis
POSTGIS_VERSION=16-3.4
POSTGRES_DB=ensgdb
POSTGRES_USER=ensgadmin
POSTGRES_PASS=ensgpassword
ALLOW_IP_RANGE=0.0.0.0/0
POSTGRES_PORT=32767
# -> pgAdmin4
PGADMIN_DEFAULT_EMAIL=admin@ensg.eu
PGADMIN_DEFAULT_PASSWORD=ensgpassword

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
INITIAL_MEMORY=500M
MAXIMUM_MEMORY=1G
# Data and extensions
SAMPLE_DATA=false
# Full compatibility list : https://github.com/kartoza/docker-geoserver/blob/master/build_data/ Look for *_plugins.txt
STABLE_EXTENSIONS=css-plugin,importer-plugin,wmts-multi-dimensional-plugin
COMMUNITY_EXTENSIONS=backup-restore-plugin,ogcapi-plugin,smart-data-loader-plugin,wmts-styles-plugin
```

### Modification de `docker compose.yml`

Modifiez votre fichier `docker compose.yml` pour utiliser ces variables :

> [!NOTE]- Solution partielle

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Faites de même pour pgAdmin et GeoServer, pour l'ensemble des variables d'environnement

---

## 2. Création d'un `Makefile` pour automatiser les tâches

Nous allons maintenant créer un `Makefile` pour simplifier l'exécution des commandes Docker Compose.

### Création du fichier `Makefile`

Créez un fichier `Makefile` à la racine du projet et ajoutez les commandes suivantes :

```make
# Charge les variables d'environnement
include .env
export $(shell sed 's/=.*//' .env)

# Lancement des services
up:
    docker network inspect ${APPS_NETWORK} >/dev/null 2>&1 || docker network create --subnet=172.24.0.0/16 --driver bridge sdi_apps
	docker compose up -d

down:
	docker compose down

restart:
	docker compose restart

logs:
	docker compose logs -f

ps:
	docker compose ps

clean:
	docker system prune -f
```

### Utilisation du `Makefile`

Exécutez les commandes suivantes pour tester :

```bash
make up
make logs
make ps
make down
```

Puis redémarrez votre stack.

---

## 3. Vérification et tests

1. Vérifiez que `docker compose` charge bien les variables de `.env` :
    
    ```bash
    docker compose config
    ```
    
2. Assurez-vous que votre infrastructure se lance correctement avec `make up`
3. Testez la connexion à PostgreSQL et pgAdmin

---

## 4. Quizz

1. Où se trouvent maintenant les variables d'environnement de notre infrastructure ?
2. Quelle commande permet de lister les services en cours d'exécution ?
3. Quelle commande du `Makefile` permet de redémarrer les services ?
4. Comment éviter d'avoir des fichiers de configuration sensibles dans un dépôt Git ?
5. Quelle est la commande pour voir la configuration complète après interpolation des variables d’environnement dans `docker compose.yml` ?

---

## 5. Pérenniser la configuration de `postgis` dans pgAdmin4

Si vous avez effectué un nettoyage via `make clean` ou des commandes directes telles que `docker system prune -a`, vous avez remarqué que pgAdmin4 a "perdu" la configuration du serveur par défaut. 

**2 solutions** : 
- Créer un volume embarquant la configuration de pgAdmin4
- Automatiser la création de la création du connecteur. Pratique lors de la mise en oeuvre mais risque d'écraser votre travail par la suite. 

Création du volume : ajoutez le volume suivant dans le service *pgadmin*

```yaml
    volumes:
      - pgadmin_data:/var/lib/pgadmin
      - pgadmin_config:/pgadmin4
```


### Automatisation de la configuration. 


Via une astuce technique, il est possible de peupler automatiquement le fichier contenant la description des bases de données. RDV sur le site de l'éditeur pour la description détaillée des possibilités de configuration : 
- https://www.pgadmin.org/docs/pgadmin4/latest/container_deployment.html

Dans le fichier `docker compose.yml`, remplacez la ligne `image: dpage/pgadmin4:latest`, par la configuration suivante : 


```yaml
  pgadmin:
    build:
      context: ./config/pgadmin
```


Puis, créez le fichier Dockerfile correspondant dans le dossier pointé pour le build 


```Dockerfile
FROM dpage/pgadmin4:latest

# >> Configuration file creation
# Create 'servers.json' configuration file
USER root
RUN echo -e '{ \n\
    "Servers": { \n\
        "1": { \n\
            "Name": "ENSG SDI", \n\
            "Group": "Servers", \n\
            "Host": "postgis", \n\
            "Port": 5432, \n\
            "MaintenanceDB": "ensgdb", \n\
            "Username": "ensgadmin", \n\
            "SSLMode": "prefer", \n\
            "PassFile": "/pgpassfile" \n\
        } \n\
    } \n\
}' > /pgadmin4/servers.json
```


Recréez le conteneur `pgadmin` et admirez le résultat! 😇


---

## 6. Dépendances et gestion des états des services

Le service `postgis` est une dépendance pour le bon fonctionnement des services suivants. 
- `geoserver`
- `pgadmin`

Ajoutez au sein de ces 2 services cette dépendance, et redémarrez la stack complète. 

> [!NOTE]- Solution partielle
>  La directive `depends_on:` permet de concrétiser cet arbre de dépendances

### Ajouter des contrôles d'état de santé des services

Puisque PostGIS est une dépendance pour Geoserver et PGAdmin, vérifions son état de santé au démarrage avec le bloc de configuration suivant. 

```yaml
# PostGIS → pg_isready vérifie si la base de données est opérationnelle.
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ensgadmin -d ensgdb"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Ajoutons la conditions de démarrage de geoserver que le service `postgis`  soit opérationnel. 

> [!NOTE]- Solution partielle
>  La directive `depends_on:` permet de concrétiser cet arbre de dépendances
>   et la `condition : service_healthy` permet d'attendre le démarrage de cette dépendance.

Redémarrez la stack complète, et observez que les contrôles sont bien exécutés. 

**Geoserver**

Ajoutez un contrôle de l'état de santé de Geoserver

```yaml 
healthcheck:
  # GeoServer → curl vérifie que l’interface web répond.
  test: ["CMD", "curl", "-f", "http://172.24.0.11:8080/geoserver/web"]
  interval: 15s
  timeout: 10s
  retries: 5
```

Tous les services dépendant de geoserver ou postgis pourront désormais le déclarer en toute confiance, et imposer la condition de bonne santé avant démarrage. 

Observer les logs relatifs aux états de santé de vos services : 

```shell
docker inspect postgis | grep -A10 Health
```

## Conclusion

Vous avez maintenant une infrastructure mieux organisée avec un `.env` pour centraliser les configurations et un `Makefile` pour automatiser les tâches courantes ! 🎉