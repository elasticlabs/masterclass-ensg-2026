---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j5-notebooks-jupyter-gouvernance-industrialisation-and-restitution/lab-06-implementation-du-routage-exposome-aware/","noteIcon":""}
---

## API de routage (FastAPI) — service “bout-en-bout”

> Objectif : exposer un endpoint HTTP simple qui transforme une intention utilisateur (A→B, mode, profil) en un itinéraire calculé par PostGIS/pgRouting, renvoyé en GeoJSON.

### Pourquoi une API ?

- **Découplage** : le client (MapStore) ne parle pas directement à PostGIS.
- **Reproductibilité** : même requête = même résultat (versionné).
- **Évolutivité** : demain on ajoute des profils, de l’auth, du cache, des métriques, etc.

### Contrat d’API (minimal)

- `GET /api/health` → test de vie
- `GET /api/route?mode=walk|bike|car&profile=distance|exposome&from=lon,lat&to=lon,lat`
- renvoie un objet JSON contenant au minimum :
	- `mode`, `profile`
	- `from`, `to`
	- `geometry` (GeoJSON geometry ou Feature)
	- `metrics` (distance, coût…)
  

> Remarque : si vous avez mis en place le carnet `05_address_to_node.ipynb`, vous pouvez aussi tester l’API avec des coordonnées récupérées via géocodage BAN.

### Mise en oeuvre de FastAPI
#### Fichiers de composition docker

Créez le répertoire `./config/api_exposome`, et créez les fichiers suivants : 

**Fichier Dockerfile**

```Dockerfile
FROM python:3.11-slim
  
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
  
RUN apt-get update -qq && apt-get install -y -qq --no-install-recommends gcc && rm -rf /var/lib/apt/lists/*
  
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt
  
COPY main.py /app/main.py
  
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Fichier de configuration de FastAPI** : `main.py`

Copiez-collez directement le fichier fourni dans le répertoire `./config/api_exposome`, et discutez avec les autres élèves et l'enseignant de son contenu. 

**Fichier `requirements.txt`** :

Copiez-collez le contenu suivant : 

```txt
fastapi==0.115.6
uvicorn[standard]==0.30.6
SQLAlchemy==2.0.36
psycopg2-binary==2.9.10
```

**Fichier docker-compose.yml** :

Ajoutez le bloc suivant au fichier `docker-compose.yml` : 

```yaml

api:
  build:
    context: ./config/api_exposome
  container_name: ${COMPOSE_PROJECT_NAME}_ensg-api
  restart: unless-stopped
  environment:
    - TZ=Europe/Paris
    - POSTGRES_HOST=postgis
    - POSTGRES_DB=ensgdb
    - POSTGRES_USER=ensgadmin
    - POSTGRES_PASSWORD=ensgpassword
  expose:
    - "8000"
  profiles:
    - data
  networks:
    ensg_sdi:
      ipv4_address: 172.24.10.9
```

Redémarrez complètement le profil `data` docker compose, afin de construire FastAPI

```bash
docker compose --profiles data up -d --build
```

#### (Optionel) Si proxy configuré, ajouté l'endpoint `/api`

Dans ton NGINX (vhost `ensg-sdi.docker.conf`), ajoute le contenu suivant : 

```nginx
# --- FastAPI behind /api ---

location /api/ {
	proxy_pass http://geodata-api:8000/;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
	  
	# (optionnel) timeouts
	proxy_connect_timeout 60s;
	proxy_read_timeout 60s;
}
```

Et redémarrez le profil `proxyfied`


#### Tests rapides (curl ou navigateur)


```bash

curl -s "http://ensg-sdi.docker/api/health"

curl -s "http://ensg-sdi.docker/api/route?mode=walk&profile=exposome&from=2.2945,48.8584&to=2.2369,48.8925" | jq .

```

  ### À observer / discuter

- Quelle différence entre `profile=distance` et `profile=exposome` ?
- Que se passe-t-il si on change `ALPHA_*` dans le carnet Jupyter puis on relance la route ?
- Quels sont les paramètres **subjectifs** (choix de pondérations) ?


---

  

## Publication dans GeoServer

> Objectif : rendre accessibles les couches de “preuves” (air/bruit/masques/score) et la couche “résultat” (route) via des services standards (WMS/WFS).

  ### Couches minimales à publier

1) **Air** (Airparif) — WMS
2) **Bruit** (Bruitparif) — WMS
3) **Réseau** `ways` stylé sur `expo_final` — WMS (et WFS si besoin)
4) **Routes** (option 1) : table `routes_demo` (si vous stockez les routes en DB) — WMS/WFS

**ou**

**Routes** (option 2) : directement en GeoJSON via l’API FastAPI (si vous ne stockez pas)

  ### Styles (SLD) : l’essentiel
- `ways` : 5 classes “faible → fort” sur `expo_final`
- `air` : classes (ou dégradé) sur NO₂ / PM2.5
- `noise` : classes sur Lden


> Bon réflexe : un style = une intention (expliquer une variable), pas “faire joli”.

---

## Visualisation & comparaison (MapStore)

> Objectif : permettre à un utilisateur d’explorer les couches “preuves” et de comparer deux itinéraires (distance vs exposome).

  ### Pourquoi MapStore (dans ce lab)

- Très bon support WMS/WFS GeoServer
- Permet une carte “application” (catalogue, légende, gestion des couches)
- S’intègre bien avec la logique SDI (services standards + reverse proxy)


### Contenu minimal de la carte “Exposome Routing Demo”

**Fond :**
- Tileserver-GL (style OSM)

**Couches “preuves” :**
- Airparif (WMS)
- Bruitparif (WMS)
- `ways` (WMS) stylé sur `expo_final`
- (option) `ways` stylé sur `green_score`

**Couches “résultats” :**
- Itinéraire distance (GeoJSON API)
- Itinéraire exposome (GeoJSON API)

### Démo (scénario de restitution)

1) activer Air + Bruit + Réseau exposome
2) appeler la route `distance`
3) appeler la route `exposome`
4) comparer : distance, coût, traversée des zones


Il y a 2 façons propres (je recommande la 1) :

#### 1) Recommandé : config MapStore montée via volume (versionnée)

- Dépose `exposome-routing.json` dans ton dossier de config MapStore (ex : `./mapstore/configs/projects/`)
- Monte ce dossier dans le conteneur MapStore via Docker compose (volume)
- Le projet est “artefact d’infra”, rejouable.

#### 2) Alternative : importer dans l’UI MapStore

OK pour démo rapide, **mais moins reproductible**.

> Dans le JSON, tu dois remplacer :

- `workspace:air_layer`
- `workspace:noise_layer`
- `workspace:ways`  
    par tes noms réels GeoServer (workspace + layername).  
    Et ajuster l’URL Tileserver-GL si besoin : `/tiles/styles/osm-bright/style.json`.

---

## 9. Discussion & limites (conclusion)

### Ce qui est “mesuré” vs “construit”

- Air/Bruit : proxies issus de modèles / agrégations
- Sentinel : contexte (verdure/humidité) **modulateur**
- Exposome final : **construction subjective** (normalisation, poids, atténuation)

### Questions finales (à faire formuler par les groupes)

- Quel choix de pondération justifiez-vous pour un “profil fragile” ?
- Quel risque de “déplacement” de l’exposition (report sur axes secondaires) ?
- Quelles données manquent pour aller vers un usage opérationnel ?