---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j5-notebooks-jupyter-gouvernance-industrialisation-and-restitution/lab-05-exposome-complexe/","noteIcon":""}
---

## Objectifs

Ce lab a pour objectif de vous faire travailler sur un graphe de routage avec `pgrouting`, et étudier l'influence de l'exposome sur des calculs d'itinéraires.

“Un notebook n’est pas un script jetable :  
c’est un **maillon du système**, versionné, rejouable, explicable.”

## 1. Initialisation - Docker et Jupyterlab 

Jupyterlab doit être ajouté à votre stack. Elle sera hébergée dans un sous-dossier : "/jupyter". Les volumes suivants sont utilisés, et doivent être ajoutés à `filebrowser`, ainsi que dans l'inventaire des volumes en bas du fichier : 
- `jupyter-notebooks:/home/jovyan/work`
- `jupyter-data:/home/jovyan/data`

Ajoutez ensuite le sevice jupyterlab à votre pile docker compose : 

```yaml

services:

  [...]
  jupyterlab:
    image: quay.io/jupyter/datascience-notebook:python-3.11
    container_name: geodata-jupyterlab
    restart: unless-stopped
    environment:
      - JUPYTER_TOKEN=ensg
      - JUPYTER_ENABLE_LAB=yes
      - TZ=Europe/Paris
      # Connexion PostGIS (réutilisable dans les notebooks)
      - POSTGRES_HOST=postgis
      - POSTGRES_DB=ensgdb
      - POSTGRES_USER=ensgadmin
      - POSTGRES_PASSWORD=ensgpassword
    volumes:
      - jupyter-notebooks:/home/jovyan/work
      - jupyter-data:/home/jovyan/data
    command: >
      start-notebook.py
      --NotebookApp.base_url=/jupyter
      --NotebookApp.allow_origin='*'
      --NotebookApp.allow_remote_access=True
      --NotebookApp.trust_xheaders=True
      --NotebookApp.ip=0.0.0.0
      --NotebookApp.port=8888
      --NotebookApp.notebook_dir=/home/jovyan/work
    expose:
      - "8888"
    networks:
      ensg_sdi:
        ipv4_address: 172.24.10.8

```

Construisez ensuite votre conteneur et démarrez-le avec la commande suivante : 

`docker compose up -d --build jupyterlab`


### (Optionnel) Configurer le reverse proxy nginx

Si vous avez implémenté un reverse proxy Nginx dans un lab précédent, ajoutez le bloc de configuration suivant dans le fichier `ensg-sdi.docker.conf`. 

```nginx
location /jupyter/ {

	proxy_pass http://geodata-jupyterlab:8888/jupyter/;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
	  
	# Websocket (indispensable pour Jupyter)
	proxy_http_version 1.1;
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "upgrade";
}
```

Et redémarrez nginx-proxy :  `docker compose up -d --build nginx-proxy`
### Accès à jupyterlab

Accédez ensuite aux carnets jupyterlab depuis l'adresse suivante : 

`http://172.24.10.8:8888/jupyter/lab` ou `http://ensg-sdi.docker/jupyter/lab`, si vous avez implémenté un reverse proxy. 


### Dépendances Python utiles le sujet fil rouge exposome

L’image `datascience-notebook` est déjà bien fournie. Ajoutez juste au besoin :

- `psycopg2-binary` (connexion Postgres)
- `sqlalchemy` (plus propre)
- `geopandas`, `shapely` (souvent déjà)
- `rasterio` (si tu manipules GeoTIFF Sentinel)
- `osmnx` (optionnel)
- `folium` ou `leafmap` (viz rapide)

On peut soit :

- installer à la volée dans le notebook (`pip install ...`)
- **(notre choix)** ou monter un `requirements.txt` et faire un `pip install -r ...` au démarrage.

**Pourquoi cette méthode est idéale en formation**
- 🔁 **reproductible** (même environnement pour tous)
- 🧠 **lisible** (les libs sont explicites)
- 🧹 **jetable** (on détruit/recrée sans douleur)
- ❌ pas besoin de construire une image custom

Déposez via `filebrowser` dans le répertoire `jupyter-notebooks` le fichier `requirements.txt` suivant : 

```txt
# --- Base data science ---
numpy
pandas
matplotlib
seaborn

# --- Geo / spatial ---
geopandas
shapely
pyproj
rtree

# --- Raster / Sentinel ---
rasterio
xarray
rioxarray

# --- PostGIS / SQL ---
sqlalchemy
psycopg2-binary

# --- pgRouting / network ---
networkx

# --- Visualisation légère ---
folium
leafmap

# --- Utils ---
python-dotenv
tqdm

```

💡 Pourquoi ce choix :

- **aucune lib exotique**
- tout est **stable**, **pip-installable**, **compréhensible**
- compatible avec `datascience-notebook`

Une fois déposé, il est accessible dans le conteneur jupyter via le chemin : `/home/jovyan/work`. 

Installez les librairies requises via la commande : 
- `pip install --no-cache-dir -r /home/jovyan/work/requirements.txt`

### Vérification de l'installation

Si l'installation s'est bien déroulée, créez le **carnet jupyter suivant** afin de vérifier el bon fonctionnement et les versions des librairies déployées : 

#### `00_env_check.ipynb` 

À faire exécuter en premier :

```python
import geopandas as gpd
import rasterio
import sqlalchemy
import psycopg2
import folium

print("GeoPandas:", gpd.__version__)
print("Rasterio:", rasterio.__version__)
print("SQLAlchemy:", sqlalchemy.__version__)
print("Environment OK ✅")
```

👉 C'est une bonne pratique qui évite 80 % des problèmes dès le départ.

#### Un peu de confort !

Ajoute dans `requirements.txt` :

```txt
jupyterlab-lsp
python-lsp-server
```

Puis relancez l'installation des packages avec la commande vue plus haut. Seuls les packages nouvellement ajoutés seront déployés. 

Ces utilitaires ajoutent autocomplétion + confort dans l'IDE jupyterlab. 

## 2. Préparation des données 

Objectifs : 

- Remplacer le conteneur `postgis/postgis` par un conteneur **avec pgRouting préinstallé**
- Initialiser la base avec **extensions** + **schémas**
- Construire un **graphe RAW** (topologie pgRouting) sur un réseau routier de départ, avant tout calcul d’exposome

### 1. Remplacement minimal du service

Tu peux remplacer uniquement l’image (et garder volume + env + IP) :

```yaml
postgis:
  image: pgrouting/pgrouting:16-3.4-3.6.1
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
  healthcheck:
    test: "PGPASSWORD=${POSTGRES_PASSWORD} pg_isready -h 127.0.0.1 -U ${POSTGRES_USER} -d ${POSTGRES_DB}"
  networks:
    ensg_sdi:
      ipv4_address: 172.24.0.10

```

**Pourquoi ce tag ?** `pgrouting/pgrouting` fournit Postgres + PostGIS + pgRouting dans la même image, et les tags existent pour Postgres 16 / PostGIS 3.4 / pgRouting 3.6.x.

> [!warning] Attention variable healthcheck  
> Dans ton bloc d’origine, tu utilises `POSTGRES_PASSWORD` dans `environment`, mais le healthcheck appelle `${POSTGRES_PASS}`.  
> Harmonise (ex. `${POSTGRES_PASSWORD}`), sinon le healthcheck peut échouer.

Comme tu restes en **Postgres 16**, tu peux redémarrer à l'aide d'un simple :

```bash
docker compose up -d --force-recreate postgis
```

> [!warning] Si ça ne démarre pas  
>Si le volume `postgis_data` a été initialisé par une autre image/config incompatible (peu probable ici car même major 16), fais un backup/restore ou recrée le volume (en formation, c’est souvent acceptable).

Connectes-toi à ta base de données à l'aide de pgAdmin. 
Que constates-tu ? Pourquoi ? 

#### Activation de pgRouting

Exécutez les commandes SQL suivantes dans pgAdmin :

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pgrouting;
CREATE EXTENSION IF NOT EXISTS postgis_raster;

-- (optionnel, utile pour debug)
CREATE EXTENSION IF NOT EXISTS hstore;
```

Et observez le résultat dans les extensions installées. 

### 2. Création du graphe de routage pgRouting

L'itinéraire de test ne rentre pas dans Paris à priori, et va de Pizzabell, Meudon bellevue, vers Tour Sequoia La Défense.

La création de graphes de routage pouvant prendre énormément d'espace de stockage, nos allons resserrer l'étude autour de cet itinéraire. 

#### Enchaînement logique proposé

|                                                                                                                                                                                                                                                                                                                             |                                                                                                                                        |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| - départ : **Pizzabell (Meudon Bellevue)**<br>- arrivée : **Tour Sequoia**<br><br>- Import `idf.pbf` (osm2pgrouting ou équivalent)<br>- Création `study_area` (Paris 15/16 ou autre)<br>- Extraction du sous-graphe<br>- Jointure Air + Bruit<br>- Ajout des masques Sentinel<br>- Calcul des coûts<br>- Routage comparatif | ![https://upload.wikimedia.org/wikipedia/en/e/ee/TourSequoia.jpg\|200](https://upload.wikimedia.org/wikipedia/en/e/ee/TourSequoia.jpg) |

#### 2.1. Import des données de routage OSM

Dans le fichier `docker-compose`, la base est le service **`postgis`** sur le réseau externe **`ensg_sdi`**, avec :

- DB : `ensgdb`  
- user : `ensgadmin`
- password : `ensgpassword`
- host : `postgis`
- port : `5432`

---

Chez **BBBike**, certains fichiers nommés `*.osm.pbf` sont en réalité des **PBF compressés/packagés d’une manière qui pose problème** à certaines builds de `osm2pgrouting`. 

La conversion en `.osm` est la voie **la plus fiable** dans ce contexte Docker. Dans le cas présent, `osmium` est utilisé.

```bash
docker run --rm \
  -v "$(pwd)/data/input:/data" \
  debian:bookworm-slim \
  bash -lc 'apt-get update -qq \
    && apt-get install -y -qq osmium-tool \
    && osmium cat /data/hauts-de-seine.osm.pbf -f osm -o /data/hauts-de-seine.osm'
```

👉 Résultat :  
`./data/input/hauts-de-seine.osm`

---
**Import avec `osm2pgrouting` (format XML `.osm`)**

On reste sur l’image **`pgrouting/pgrouting-extra:16-3.5-3.8`**, qui est OK avec `.osm`.
Et le fichier OSM converti à l'étape précédente sur l’hôte : `./data/input/hauts-de-seine.osm.`
- La conversion simplifie également le fichier en supprimant une grande quantité d'informations ajoutés dans les fichiers PBF (métadonnées)
- Le fichier résultant est plus simple, mais contient tout ce dont `pgrouting` a besoin pour travailler!

```bash
docker run --rm \
  --network ensg_sdi \
  -v "$(pwd)/data/input:/data:ro" \
  -v "$(pwd)/mapconfig.xml:/mapconfig.xml:ro" \
  pgrouting/pgrouting-extra:16-3.5-3.8 \
  osm2pgrouting \
    --f /data/hauts-de-seine.osm \
    --conf /mapconfig.xml \
    --dbname ensgdb \
    --username ensgadmin \
    --host postgis \
    --port 5432 \
    -W ensgpassword \
    --clean

```

L'import doit prendre un moment, et le terminal afficher une sortie de ce type : 

```bash
[...]
[****************************************|          ] (80%) Total processed: 300000      Vertices inserted: 13863       Split ways inserted 20381
[******************************************|        ] (85%) Total processed: 320000      Vertices inserted: 13833       Split ways inserted 22499
[*********************************************|     ] (90%) Total processed: 340000      Vertices inserted: 17846       Split ways inserted 28573
[************************************************|  ] (96%) Total processed: 360000      Vertices inserted: 14642       Split ways inserted 26028
[**************************************************|] (100%) Total processed: 374628     Vertices inserted: 11257       Split ways inserted 19667
         out_edges modified: 176039      in_edges modified: 162098
Creating indexes ...

Processing Points of Interest ...
#########################
size of streets: 374628
Execution started at: Wed Jan 14 18:55:26 2026
Execution ended at:   Wed Jan 14 18:57:29 2026
Elapsed time: 122.206 Seconds.
User CPU time: -> 39.2789 seconds
#########################
```


### 3. Création du réseau multi-modal (pied / vélo / voiture) pgrouting

#### 🧭 Modèle de données (ce que l'on a maintenant)

**1️⃣ Table `ways`**

Colonnes importantes :
- `id`
- `source`, `target`
- `tag_id` ⟵ **clé vers `configuration`**
- `length_m` (ou équivalent)
- `cost`, `reverse_cost` (si créés par défaut)

**2️⃣ Table `configuration`**

Tu as (d’après la capture) :
- `tag_key` → ex: `highway`, `cycleway`
- `tag_value` → ex: `motorway`, `footway`, `cycleway`
- `maxspeed`, `maxspeed_forward`, `maxspeed_backward`
- `priority` (souvent inutilisé au début)
- `force`

➡️ **C’est ici que tu dois définir les règles de mobilité.**

Ce que nous allons faire : pour chaque tronçon (`ways`) :

- calculer un **temps de parcours**
- différent selon le **mode** :
    - 🚶 à pied
    - 🚲 à vélo
    - 🚗 en voiture

👉 On va créer **3 paires de coûts** :

- `cost_walk / reverse_cost_walk`
- `cost_bike / reverse_cost_bike`
- `cost_car / reverse_cost_car`

#### 1️⃣ Ajouter les colonnes de coût

Ouvrez `pgAdmin4`, allez dans le `Query tool`, et saisissez les commandes suivantes : 

```sql
ALTER TABLE ways
  ADD COLUMN IF NOT EXISTS cost_walk double precision,
  ADD COLUMN IF NOT EXISTS reverse_cost_walk double precision,
  ADD COLUMN IF NOT EXISTS cost_bike double precision,
  ADD COLUMN IF NOT EXISTS reverse_cost_bike double precision,
  ADD COLUMN IF NOT EXISTS cost_car double precision,
  ADD COLUMN IF NOT EXISTS reverse_cost_car double precision;
```


#### 2️⃣ Principe fondamental des pondérations

On travaille **en temps (secondes)** :
- Temps = Longueur (m) / Durée (s)

Conversion :

- 5 km/h → 1.39 m/s
- 15 km/h → 4.17 m/s
- 30 km/h → 8.33 m/s
- 50 km/h → 13.89 m/s

🚫 **Tronçon interdit** → coût très élevé (`1e9`)


#### Pondération 🚶 à pied

Règles simples (parfaites pour un cours) :

- ❌ interdit : `motorway`, `motorway_link`    
- ✅ autorisé ailleurs
- vitesse constante : **5 km/h**

```sql
UPDATE ways w
SET cost_walk =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    ELSE w.length_m / 1.3888889
  END,
  reverse_cost_walk =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    ELSE w.length_m / 1.3888889
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;
```

#### Pondération  🚲 à vélo

Règles :

- ❌ interdit : `motorway`, `motorway_link`    
- 🚲 rapide : `cycleway`
- 🟡 moyen : `path`, `track`
- par défaut : **15 km/h**, + vite sur `cycleway`


```sql
UPDATE ways w
SET cost_bike =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    WHEN c.tag_key='highway' AND c.tag_value='cycleway' THEN w.length_m / 5.5555556
    WHEN c.tag_key='highway' AND c.tag_value IN ('path','track') THEN w.length_m / 3.3333333
    ELSE w.length_m / 4.1666667
  END,
  reverse_cost_bike =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    WHEN c.tag_key='highway' AND c.tag_value='cycleway' THEN w.length_m / 5.5555556
    WHEN c.tag_key='highway' AND c.tag_value IN ('path','track') THEN w.length_m / 3.3333333
    ELSE w.length_m / 4.1666667
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;

```

#### Pondération 🚗 voiture

Règles :

- ❌ interdit : `footway`, `path`, `cycleway`, `pedestrian`
- vitesse basée sur `maxspeed` (ou défaut)
- maxspeed si dispo, sinon 50 km/h ; interdit sur certains highways

```sql
UPDATE ways w
SET cost_car =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('footway','path','cycleway','pedestrian') THEN 1e9
    WHEN c.maxspeed IS NOT NULL THEN w.length_m / (c.maxspeed * 1000.0/3600.0)
    ELSE w.length_m / 13.8888889
  END,
  reverse_cost_car =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('footway','path','cycleway','pedestrian') THEN 1e9
    WHEN c.maxspeed_backward IS NOT NULL THEN w.length_m / (c.maxspeed_backward * 1000.0/3600.0)
    WHEN c.maxspeed IS NOT NULL THEN w.length_m / (c.maxspeed * 1000.0/3600.0)
    ELSE w.length_m / 13.8888889
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;

```

#### Index (important pour les perfs)

```sql
CREATE INDEX IF NOT EXISTS ways_source_idx ON ways(source);
CREATE INDEX IF NOT EXISTS ways_target_idx ON ways(target);
ANALYZE ways;
```

#### Tests de routage (le moment fun 😄)

Regardons ce que donnent nos premiers tests de routage, de Pizzabell à la Tour Sequoia La Défense! 

🚗 **voiture**

Requête brute de fonderie, listant les arcs apr lesquels pgrouting calcule l'itinéraire : 

```sql
SELECT * FROM pgr_dijkstra(
  'SELECT id, source, target, cost_car AS cost, reverse_cost_car AS reverse_cost FROM ways',
  1000, 2000,
  directed := true
);
```

Afin de visualiser directement dans pgrouting l'itinéraire, nous allons créer une vue : 

```sql
UPDATE ways w
SET cost_car =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('footway','path','cycleway','pedestrian') THEN 1e9
    WHEN c.maxspeed IS NOT NULL THEN w.length_m / (c.maxspeed * 1000.0/3600.0)
    ELSE w.length_m / 13.8888889
  END,
  reverse_cost_car =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('footway','path','cycleway','pedestrian') THEN 1e9
    WHEN c.maxspeed_backward IS NOT NULL THEN w.length_m / (c.maxspeed_backward * 1000.0/3600.0)
    WHEN c.maxspeed IS NOT NULL THEN w.length_m / (c.maxspeed * 1000.0/3600.0)
    ELSE w.length_m / 13.8888889
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;
```


Vous pouvez visualiser le tracé directement dans pgAdmin !

---

🚲 **vélo**

Requête brute de fonderie, listant les arcs apr lesquels pgrouting calcule l'itinéraire : 

```sql
SELECT * FROM pgr_dijkstra(
  'SELECT id, source, target, cost_bike AS cost, reverse_cost_bike AS reverse_cost FROM ways',
  1000, 2000,
  directed := true
);
```

Afin de visualiser directement dans pgrouting l'itinéraire, nous allons créer une vue : 

```sql
UPDATE ways w
SET cost_bike =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    WHEN c.tag_key='highway' AND c.tag_value='cycleway' THEN w.length_m / 5.5555556
    WHEN c.tag_key='highway' AND c.tag_value IN ('path','track') THEN w.length_m / 3.3333333
    ELSE w.length_m / 4.1666667
  END,
  reverse_cost_bike =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    WHEN c.tag_key='highway' AND c.tag_value='cycleway' THEN w.length_m / 5.5555556
    WHEN c.tag_key='highway' AND c.tag_value IN ('path','track') THEN w.length_m / 3.3333333
    ELSE w.length_m / 4.1666667
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;

```

Vous pouvez visualiser le tracé directement dans pgAdmin !

---

🚶 **piéton**

Requête brute de fonderie, listant les arcs apr lesquels pgrouting calcule l'itinéraire : 

```sql
SELECT * FROM pgr_dijkstra(
  'SELECT id, source, target, cost_walk AS cost, reverse_cost_walk AS reverse_cost FROM ways',
  1000, 2000,
  directed := true
);
```

Afin de visualiser directement dans pgrouting l'itinéraire, nous allons créer une vue : 

```sql
UPDATE ways w
SET cost_walk =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    ELSE w.length_m / 1.3888889
  END,
  reverse_cost_walk =
  CASE
    WHEN c.tag_key='highway' AND c.tag_value IN ('motorway','motorway_link') THEN 1e9
    ELSE w.length_m / 1.3888889
  END
FROM configuration c
WHERE w.tag_id = c.tag_id;
```

Vous pouvez visualiser le tracé directement dans pgAdmin ! Passons maintenant à la construction de nos variables d'exposome. 

### 4. Calcul des paramètres d'exposome .

#### Architecture logique (propre & compréhensible)

**Chaîne conceptuelle**

```scss
Sentinel (1 journée)
   ↓
Indices (NDVI, NDWI, MI)
   ↓
Masques spatiaux (verdure / humidité)
   ↓
Modulation de l’exposition Air + Bruit
   ↓
Coût de routage exposome
```

“La pollution est mesurée,  
la verdure module l’exposition vécue.”


#### Pourquoi plusieurs couches ?

- **Air (Airparif / grilles / polygones)** : proxy d’une **pression atmosphérique** (NO₂, PM2.5…), souvent agrégée/modélisée. Sert à spatialiser des zones plus ou moins exposées.
- **Bruit (Bruitparif / Lden, Ln)** : proxy d’une **pression sonore** (cartes par zones). Complète l’air, car l’exposition urbaine n’est pas “mono-facteur”.
- **Sentinel (NDVI/NDWI/NDMI)** : **ne mesure pas la pollution**. Sert à produire des **masques environnementaux** :
    - verdure (NDVI) → atténuation/“confort” / environnement plus agréable,
    - humidité/fraîcheur (NDMI/NDWI) → proxy microclimat (ressenti, îlots de chaleur).  
        👉 On l’a cadré comme **modulateur** de l’exposition vécue, pas comme mesure sanitaire.

#### Notion de subjectivité

- L’exposome dans le lab est une **construction**, pas une vérité médicale :    
    - on choisit les variables,
    - on choisit la normalisation,
    - on choisit les poids (air vs bruit),
    - on choisit comment la verdure/humidité atténue (ou non) l’exposition.
- Ces choix sont **contextuels** (profil “sportif”, “fragile”, “enfant”, “cycliste”), donc **discutables**. Par exemple :

|Persona|Air|Bruit|Végétation|Usage|
|---|---|---|---|---|
|🧒 Enfant asthmatique|0.5|0.2|0.3|école|
|👩‍💼 Actif urbain|0.3|0.4|0.3|travail|
|👴 Senior fragile|0.2|0.5|0.3|santé|

👉 Les poids **sommant à 1**
👉 **Routage = décision (pas de neutralité)**

- Un algorithme optimise **le coût qu’on lui donne**.
- Comparer “distance” vs “exposome” illustre :
    - un compromis distance/temps ↔ santé/confort,
    - la possibilité d’effets secondaires (déplacement de trafic/exposition),
    - le fait que “meilleur itinéraire” dépend du critère choisi.


> “Un exposome n’est pas découvert, il est construit.  
> Un algorithme n’est pas neutre, il optimise ce qu’on lui demande.”

#### Ce que CE N’EST PAS

- ❌ pas une vérité sanitaire
- ❌ pas un modèle validé épidémiologiquement
- ❌ pas une donnée temps réel

#### Ce que C’EST

- ✅ un **système spatial cohérent**    
- ✅ une **chaîne de décision**
- ✅ un **outil de débat et de scénarios**


## 5. Lab QGIS (simple & efficace) pour préparer fichiers Sentinel `.SAFE` rapidement

### Rôle exact des indices Sentinel dans TON exposome

👉 Les indices **ne remplacent pas** Airparif / Bruitparif  
👉 Ils servent de **facteurs modulateurs spatiaux**

|Indice|Ce qu’il représente|Rôle dans l’exposome|
|---|---|---|
|**NDVI**|Densité de végétation|↓ exposition perçue (filtration, confort)|
|**NDWI**|Eau / humidité de surface|↓ stress thermique / poussières|
|**Moisture Index**|Humidité de la végétation|Proxy de fraîcheur micro-climatique|

#### Indices à calculer

### NDVI — Végétation


![https://images.squarespace-cdn.com/content/v1/58c95854c534a56689231265/1491931435643-J83RD8HP8T3MIYG00ALM/NDVI.png](https://images.squarespace-cdn.com/content/v1/58c95854c534a56689231265/1491931435643-J83RD8HP8T3MIYG00ALM/NDVI.png)

- `(B8 - B4) / (B8 + B4)`
- seuils simples :
    - `< 0.2` : minéral
    - `0.2–0.5` : végétation moyenne
    - `> 0.5` : végétation dense

### NDWI — Eau / humidité de surface

![https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel-2/ndwi/fig/fig1.jpg](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel-2/ndwi/fig/fig1.jpg)

- `(B3 - B8) / (B3 + B8)`
- utile pour :
    - parcs humides
    - berges
    - zones rafraîchissantes
### Moisture Index (optionnel mais intéressant)

![https://geovisualization.net/wp-content/uploads/2022/07/analysis-moisture-index-20220714-07.png](https://geovisualization.net/wp-content/uploads/2022/07/analysis-moisture-index-20220714-07.png)


- capte le **stress hydrique**    
- bon lien avec confort thermique



#### Création des masques dans QGIS

Objectif : sortir **3 GeoTIFF** directement utilisables dans le carnet Jupyter :

- `ndvi.tif`, `ndmi.tif` (+ option `ndwi.tif`)
- idéalement **déjà masqués nuages**
- rangés dans `data/sentinel/processed/`

### 1) Charger les bandes utiles

Dans QGIS :

- **Ajouter raster** depuis le `.SAFE`
- Charger :
    - **10 m** : B04 (Red), B08 (NIR), (B03 si tu veux NDWI)
    - **20 m** : B11 (SWIR) pour NDMI
    - **20 m** : **SCL** (Scene Classification Layer)

### 2) Construire un masque “valide” (nuages/ombres)

Raster Calculator sur **SCL** (20 m) :

- créer un raster binaire `valid_mask_20m` (1 = OK, 0 = à exclure)
- règle simple : exclure **nuages** + **ombres** + **cirrus** + **neige**
- puis **rééchantillonner à 10 m** (outil _Warp (reproject)_ ou _Align Rasters_)

_(En formation, le plus important est d’obtenir un masque cohérent, pas d’être exhaustif sur les codes.)_

### 3) Calculer NDVI & NDMI

Raster Calculator :

- **NDVI**  
    `(B08 - B04) / (B08 + B04)`
    
- **NDMI** (Moisture)  
    `(B08 - B11) / (B08 + B11)`  
    ⚠️ comme B11 est en 20 m, soit :
    - tu rééchantillonnes B11 en 10 m, soit
    - tu passes tout en 20 m (mais je recommande “tout en 10 m” pour simplifier)

### 4) Appliquer le masque nuages

Raster Calculator :

- `ndvi_masked = ndvi * valid_mask_10m`
- `ndmi_masked = ndmi * valid_mask_10m`

### 5) Découper à la zone d’étude

Outil : **Clip raster by mask layer** (ou _Découper un raster_)  
➡️ utiliser un polygone AOI (Paris 15/16, bbox, commune, etc.)

### 6) Exporter (GeoTIFF)

Exporter dans :

- `data/sentinel/processed/ndvi.tif`
- `data/sentinel/processed/ndmi.tif`
- (option) `data/sentinel/processed/ndwi.tif`


## 6. Carnets jupyter : calcul de l'exposome 

Dans le carnet `01_lab_exposome_end_to_end.ipynb`, la partie Air/Bruit est volontairement **générique** (table/colonnes à adapter), parce que leurs noms dépendent de ton import.  

Selon **les noms exacts de tes tables Airparif/Bruitparif** (et leurs champs + nom de la géométrie), la section 5 est très “plug-and-play” (sans TODO)

Un carnet spécifique vous permet de géocoder une adresse compatible en IDF, afin de pouvoir ensuite **copier-coller** `SRC_NODE` / `TGT_NODE` dans le carnet principal `01_lab_exposome_end_to_end.ipynb`. 

Ce carnet :

- géocode une **adresse postale en France** via l’API BAN (mode en ligne),
- propose un **mode hors-ligne** (tu saisis lon/lat),
- trouve l’arête `ways` la plus proche puis sort `SRC_NODE` / `TGT_NODE`,
- inclut un **test pgRouting** (walk/bike/car) pour valider.

---
