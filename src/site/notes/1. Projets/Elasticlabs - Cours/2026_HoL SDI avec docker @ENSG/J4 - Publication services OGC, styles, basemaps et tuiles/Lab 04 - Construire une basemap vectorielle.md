---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j4-publication-services-ogc-styles-basemaps-et-tuiles/lab-04-construire-une-basemap-vectorielle/","noteIcon":""}
---


Cheminement : 
- construction d'un geopackage / conteneur de base
- import de données OSM
- Chemin MVT : 
	- Construction des tuiles vecteur
	- Assemblage des styles
	- Serveur de tuiles : Martin dans docker-compose

## Objectif

Produire une **basemap vectorielle (.mbtiles)** à partir de `hauts-de-seine.osm.pbf`, utilisable dans MapLibre / Mapbox GL / GeoServer (via WMTS vectoriel).

### Construction d'un subset de vecteurs tuilés 

- Préparation  : 
	- Téléchargement d'une extraction OSM sur bbbike (https://download.bbbike.org/osm/extract/)
	- Si vous avez déjà déroulé le lab de basemap sur base geoserver, reprenez le meme fichier! 

Récupérez l'image de `planetiler`

```bash
docker pull ghcr.io/onthegomap/planetiler:latest
```

## Générer la basemap OSM (profil par défaut)

Exécutez la commande suivante : 
Pour trouver l'aire concernée, RDV sur `https://download.geofabrik.de/europe.html`, et choisissez au plus près de notre zone d’intérêt. 

```bash
docker run --rm \
  -v $(pwd)/data:/data \
  ghcr.io/onthegomap/planetiler:latest \
  --input=./data/input/hauts-de-seine.osm.pbf \
  --output=./data/osm_basemap.mbtiles \
  --area=<notre-zone> \
  --download=false

```

A la fin du processus, vous devez retrouver un fichier `osm_basemap.mbtiles` dans le répertoire `./data`. Il contient l'ensemble des vecteurs tuilés générés par `planetiler`.

Planetiler produit automatiquement :

- routes (hiérarchie OSM)
- bâtiments
- eau
- landuse
- POI
- frontières administratives

Le tout en **tuiles vectorielles (MVT)**.

## Vérifications

### En ligne de commande

```bash
sqlite3 ./data/osm_basemap.mbtiles ".tables"
```

Doit contenir : 

```bash
metadata       tiles          tiles_data     tiles_shallow
```

Inspection rapide

```bash
sqlite3 ./data/osm_basemap.mbtiles "SELECT * FROM metadata;"
```

## Diffusion

Depuis le docker-compose.yml du cours, démarrez le profil `basemap` afin de lancer le service `tileserver-gl`. 

Le style déployé par défault étant **`basic-preview`**, l’URL raster XYZ est :

```bash
http://tileserver-gl:8080/styles/basic-preview/{z}/{x}/{y}.png
```

(⚠️ depuis MapStore **dans Docker**, on utilise le **nom du service**, pas `localhost`)

## Configurer Mapstore2 

### MapStore2 : `localConfig.json` (catalog + service GeoServer optionnel)

Crée `./config/mapstore/localConfig.json` :

```json
{
  "initialState": {
    "defaultState": {
      "catalog": {
        "selectedService": "local_geoserver",
        "services": {
          "local_geoserver": {
            "type": "wms",
            "title": "GeoServer local",
            "url": "http://geoserver:8080/geoserver/wms"
          }
        }
      }
    }
  }
}

```

### MapStore2 : `new.json` (fond XYZ depuis TileServer-GL)

Créez le fichier `new.json` de MapStore2, afin de rendre disponible tout de suite la basemap nouvellement créée. 

📄 Crée le fichier `./config/mapstore/new.json`

```json
{
  "version": 2,
  "map": {
    "projection": "EPSG:3857",
    "center": [2.25, 48.85],
    "zoom": 11,
    "layers": [
      {
        "id": "basemap_planetiler",
        "type": "tileprovider",
        "provider": "custom",
        "group": "background",
        "title": "Basemap OSM (Planetiler / TileServer-GL)",
        "visibility": true,
        "url": "http://tileserver-gl:8080/styles/basic-preview/{z}/{x}/{y}.png",
        "tileSize": 256
      }
    ]
  }
}
```

### Modification du service MapStore

Remplace ton service `mapstore` par ceci : 
```yaml
mapstore:
  image: geosolutionsit/mapstore2:latest
  container_name: ${COMPOSE_PROJECT_NAME}_mapstore
  restart: unless-stopped
  expose:
    - "8080"
  volumes:
    - ./mapstore/configs/localConfig.json:/usr/local/tomcat/webapps/mapstore/configs/localConfig.json:ro
    - ./mapstore/configs/new.json:/usr/local/tomcat/webapps/mapstore/configs/new.json:ro
  networks:
    ensg_sdi:
      ipv4_address: 172.24.10.5

```

Puis :

```bash
docker compose up -d --force-recreate mapstore
docker logs -f ${COMPOSE_PROJECT_NAME}_mapstore
```


## Tests (CLI ou navigateur)

### Tester que TileServer-GL répond

```bash
curl -I "http://172.24.10.7:8080/styles/basic-preview/#9/48.681/2.5028"
```

### Dans MapStore

- ouvrir : `http://172.24.10.5:8080/#/viewer/openlayers/new`
- ou `http://ensg-sdi.docker/mapstore/#/viewer/openlayers/new`

Tu dois voir :

- ✅ fond de carte OSM (Planetiler)
- ✅ zoom fluide
- ✅ pas de GeoServer requis pour la basemap

## Architecture finale (propre & moderne)

```text
Planetiler (.mbtiles)
        ↓
TileServer-GL (XYZ raster)
        ↓
MapStore2 (fond de plan)
        +
GeoServer / PostGIS (couches métier)
```

👉 **Best practice validée** :

- 🗺️ Basemap → Planetiler + TileServer-GL
- 📊 Données métier → PostGIS + GeoServer
- 🧭 Client → MapStore2

