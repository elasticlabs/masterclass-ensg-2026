---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j4-publication-services-ogc-styles-basemaps-et-tuiles/lab-03-nos-premieres-donnees/","noteIcon":""}
---

# Construire une basemap (OSM + fonds standards) simple avec Geoserver + Styles SLD/CSS

## 🎯 Objectifs

À l’issue de ce tutoriel, vous saurez :

- construire une **basemap composite**
- combiner **fonds génériques** + **données locales OSM**
- styliser les routes selon :
  - un style **OSM Bright–like**
  - un style **praticabilité vélo**
- publier la basemap via **GeoServer (WMS/WMTS)**

## 🧠 Rappel conceptuel

Une **basemap n’est pas neutre** :
- elle guide la lecture
- elle prépare des usages (navigation, analyse, décision)
- elle peut déjà porter une **intention politique / sanitaire**

👉 Ici, la basemap prépare :
- le routage
- la lecture de l’exposome
- la mobilité douce

# 🧱 Étape 1 — Fonds standards (Natural Earth)

## 1.1 Pourquoi Natural Earth ?
- données libres
- multi-échelles
- parfaites pour :
  - eau
  - pays
  - zones urbaines
  - contexte global

👉 Pas adaptées au routage, mais idéales en **arrière-plan**

## 1.2 Couches utiles (échelle urbaine)

Que vous choisissiez de créer une basemap depuis un outil automatisé de vecteurs tuilés, ou depuis geoserver en direct, la démarche débute souvent par le téléchargement de données  depuis https://www.naturalearthdata.com :

- `ne_10m_land`
- `ne_10m_ocean`
- `ne_10m_lakes`
- `ne_10m_rivers_lake_centerlines`
- `ne_10m_urban_areas`

### La méthode artisanale

Importer les données téléchargées directement dans PostGIS, avant publication dans geoserver :

Nota : $PG_CONN contient les données de connection à votre conteneur postgis. Faites une recherche via par exemple chatgpt, en lui donnant vos éléments d'entrée, afin d'injecter avec succès ces données via **ogr2ogr**

```bash
ogr2ogr -f "PostgreSQL" PG:"$PG_CONN" \
  ne_10m_land.shp \
  -nln basemap.ne_land \
  -lco GEOMETRY_NAME=geom \
  -lco FID=id \
  -t_srs EPSG:3857 \
  -overwrite
```

👉 Répéter pour les autres couches (`ocean`, `lakes`, `rivers`).

### (conseillée) La méthode *geopackage*

L'éditeur de *geoserver* a décidé de publier un tutoriel de création d'un style très apprécié des utilisateurs : OSM Bright.

Téléchargez le fichier geopackage fourni par l'éditeur : https://github.com/geosolutions-it/osm-styles?tab=readme-ov-file#the-low-resolution-geopackage

Observez son contenu dans QGis. Vous devez retrouver les calques suivants : 

```bash
osm-lowres.gpkg
- builtup_area
- icesheet_outlines
- icesheet_polygons
- land_polygons
- ne_10m_admin_0_boundary_lines_land
- ne_10m_admin_0_countries_points
- ne_10m_admin_1_states_provinces_lines
- ne_10m_bathymetry
- ne_10m_bathymetry_gen0
- ne_10m_bathymetry_gen1
- ne_10m_bathymetry_gen2
- ne_10m_geography_marine_polys
- simplified_land_polygons
- simplified_water_polygons
- water_polygons
```

Utilisez Filebrowser afin de déposer l'archive geopackage directement dans un volume *data* de geoserver. 

Dans geoserver : 
- Créez un nouveau workspace
	- Activez les services WMS, WMTS, WFS
- Créez un entrepôt de type *geopackage*
- Publiez et prévisualisez les couches dans geoserver
- Générez un nouveau style dans l'espace de travail courant, sur la base d'un style par défaut
- Dans filebrowser, explorez votre espace de travail, et téléversez les styles fournis par l'éditeur à l'adresse : https://github.com/geosolutions-it/osm-styles/tree/master/workspaces/osm/styles, directement par glisser-déposer dans un dossier `geoserver-data/uploads/styles/`
- Dans geoserver, vous devez les ajouter ensuite manuellement, afin de permettre leur enregistrement ad-hoc. Oui, oui... 
	- Une autre solution est d'automatiser leur création par script via l'API REST geoserver, proposée dans le sous-chapitre suivant (automatisation).
- Créez ensuite un layergroup, avec les calques suivants et styles associés : 
	- ne_10m_bathymetry_gen2, ne_10m_bathymetry_gen1, ne_10m_bathymetry_gen0, ne_10m_bathymetry
	- ne_10m_geography_marine_polys
	- simplified_water_polygons, water_polygons
	- simplified_land_polygons, land_polygons
	- builtup_area
	- icesheet_polygons, icesheet_outlines
	- ne_10m_admin_0_boundary_lines_land, ne_10m_admin_1_states_provinces_lines
	- ne_10m_admin_0_countries_points
- Testez l'affichage de votre basemap générique dans un client tel que mapstore, en WMS.
- Créez un serveur de tuiles WMTS
	- Vérifiez que votre `layergroup` est visible dans le serveur de tuiles; `geowebcache`, bien que présenté dans une interface unifiée avec Geoserver, est un service bien distinct. 
	- supprimez les grilles de tuilage superflues
- Configurez la connexion avec votre basemap initiale dans mapstore2

# 2️⃣ Données OSM : extraction ciblée

## 2.1 Source OSM

Options :
- Geofabrik (Île-de-France)
- BBBike
- Overpass (moins reproductible)

👉 Recommandé pour la formation : **bbbike**
Attention à resserrer l'emprise sur une zone permettant de futurs calculs. 

## 2.2. Importer dans OSM

Saisissez la commande SQL suivante dans pgadmin :

```sql
CREATE SCHEMA IF NOT EXISTS osm;
```

En supposant que votre fichier PBF se situe dans `./data/input`, et se nomme `hauts-de-seine.osm.pbf`, exécutez les commandes suivantes dans un terminal afin d'importer son contenu dans postgis :

```bash

osm2pgsql \
  -H 172.24.0.10 \
  -P 5432 \
  -d ensgdb \
  -U ensgadmin \
  -W \
  --create \
  --slim \
  --hstore \
  ./data/input/hauts-de-seine.osm.pbf

```

👉 Entrer le mot de passe PostgreSQL quand demandé.

**Vérification (ligne de commande)**

Exécutez l)es commandes suivantes : 

```bash
psql -h 172.24.0.10 -p 5432 -U ensgadmin -d ensgdb

*puis*

\dt
```

👉 Tables attendues (exemple) :

- `planet_osm_point`
- `planet_osm_line`
- `planet_osm_polygon`
- `planet_osm_roads`

Test simple :

```sql 
SELECT COUNT(*) FROM planet_osm_point;
```

**Vérifications PgAdmin4**

- Se connecter au serveur PostGIS
- Base : `ensgdb` → Schéma : `public`
- Vérifier la présence des tables `planet_osm_*`
- Lancer une requête :

```sql 
SELECT ST_Extent(way) FROM planet_osm_polygon;
```

Les données OSM des **Hauts-de-Seine** sont accessibles dans PostGIS et prêtes pour l’analyse ou la visualisation SIG.

# 3️⃣ Préparer les couches de basemap

## 3.1 Routes (vue dédiée)

`osm2pgsql` **ne crée pas une colonne par tag OSM**.  
Les tags non “promus” sont stockés dans la colonne **`tags` (hstore)**.

👉 Dans `planet_osm_line`, les colonnes comme `lanes`, `surface`, `bicycle`, `maxspeed` **n’existent pas en colonnes natives**. Correction : extraire les tags depuis `tags`

Version **fonctionnelle** de la vue :

```sql
CREATE OR REPLACE VIEW public.osm_roads AS
SELECT
  way AS geom,
  highway,
  name,
  tags->'surface'   AS surface,
  tags->'bicycle'   AS bicycle,
  tags->'lanes'     AS lanes,
  tags->'maxspeed'  AS maxspeed
FROM public.planet_osm_line
WHERE highway IS NOT NULL;
```

Vérification rapide : 

```sql
SELECT highway, lanes, maxspeed
FROM public.osm_roads
LIMIT 10;
```

# 4️⃣ Style 1 — Routes “OSM Bright–like”

🎯 **Objectif**

- lisibilité
- hiérarchie claire
- neutralité

## 4.1 Logique de style

| highway     | couleur  | largeur  |
| ----------- | -------- | -------- |
| motorway    | \#e892a2 | large    |
| primary     | \#fcd6a4 | moyen    |
| secondary   | \#f7fabf | moyen    |
| residential | \#ffffff | fin      |
| service     | \#eeeeee | très fin |
etc. 

## 4.2. Style type OSM-bright complet

Copier le fichier d'entrée nommé `osm-bright-roads.sld` depuis le dépôt du cours vers un dossier interne à `geoserver`. 

Quel outil allez-vous utiliser ? 

### ✅ Côté GeoServer

- DataStore → PostGIS → schéma `public` 
- Publier la vue `osm_roads`
- CRS : **EPSG:3857** (ou 4326 selon ton import)
- Style simple basé sur `highway` (couleur/épaisseur), ou style complet, selon vos envies!

👉 On a  maintenant un **fond de plan routier OSM fonctionnel** pour un MVP.

# 5️⃣ Style 2 — Routes “praticabilité vélo”

🎯 **Objectif**  
Lire **où il est agréable / sûr de rouler**.

## 5.1 Indicateur vélo (heuristique simple)

Créer une **vue enrichie** :

```sql
CREATE OR REPLACE VIEW public.osm_roads_bike AS
SELECT
  geom,
  highway,
  surface,
  bicycle,
  CASE
    -- 1) Infrastructures vélo évidentes
    WHEN highway IN ('cycleway') THEN 1

    -- 2) Chemins souvent praticables (si pas explicitement interdit)
    WHEN highway IN ('path','footway','track')
         AND COALESCE(LOWER(bicycle), 'yes') NOT IN ('no','dismount','use_sidepath') THEN 2

    -- 3) Rues calmes / résidentielles avec surface correcte
    WHEN highway IN ('residential','living_street')
         AND COALESCE(LOWER(bicycle), 'yes') NOT IN ('no','dismount','use_sidepath')
         AND (surface IS NULL OR LOWER(surface) IN ('asphalt','paved','concrete','concrete:lanes','concrete:plates')) THEN 2

    -- 4) Routes plus circulées : on distingue si piste/bande existe
    WHEN highway IN ('tertiary','secondary')
         AND COALESCE(LOWER(bicycle), 'yes') NOT IN ('no','dismount','use_sidepath') THEN 3

    -- 5) Cas défavorables
    WHEN highway IN ('primary','trunk','motorway','motorway_link','trunk_link')
         OR COALESCE(LOWER(bicycle), '') IN ('no','dismount') THEN 4

    -- 6) Par défaut
    ELSE 4
  END AS bike_level
FROM publish.osm_roads;

```

### Interprétation

|bike_level|sens|
|---|---|
|1|très favorable|
|2|favorable|
|3|moyen|
|4|défavorable|
## 5.2 Style SLD “vélo”

Couleurs :

- 🟢 vert = favorable
- 🟡 jaune = moyen
- 🔴 rouge = défavorable


```xml
<Rule>
  <Name>Très favorable</Name>
  <Filter>
    <PropertyIsEqualTo>
      <PropertyName>bike_level</PropertyName>
      <Literal>1</Literal>
    </PropertyIsEqualTo>
  </Filter>
  <LineSymbolizer>
    <Stroke>
      <CssParameter name="stroke">#2ecc71</CssParameter>
      <CssParameter name="stroke-width">3</CssParameter>
    </Stroke>
  </LineSymbolizer>
</Rule>

```

Copier le fichier d'entrée `osm-bike-practicability.sld` depuis les fichiers du cours, vers un dossier geoserver, via filebrowser, puis importer le style dans geoserver. 

### ✅ Côté GeoServer

- DataStore → PostGIS → schéma `public` 
- Publier la vue `osm_roads_bike`
- CRS : **EPSG:3857** (ou 4326 selon ton import)
- Style complet, placé dans un répertoire geoserver précédemment !

👉 On a  maintenant un **fond de plan routier OSM fonctionnel** pour un MVP.

**Conseils GeoServer (rapide)**

- Ordre des règles : **1 → 4** (déjà OK)
- CRS recommandé : **EPSG:3857**
- Si surcharge visuelle : réduire `stroke-width` de 0.5

**Résultat attendu**

- 🟢 pistes et rues calmes immédiatement visibles
- 🟡 axes tolérables mais moins confortables
- 🔴 axes à éviter pour un itinéraire vélo

# 6️⃣ Publication GeoServer

### 6.1 Couches à publier

- `basemap.ne_land`
- `basemap.ne_water`
- `publish.osm_landuse`
- `publish.osm_roads`
- `publish.osm_roads_bike`

### 6.2 Ordre d’affichage

1. ocean / land
2. landuse
3. water
4. roads
### 6.3 Activer GeoWebCache

- WMTS
- XYZ
- EPSG:3857
- zooms : 6 → 18

# 7️⃣ Test de la basemap

### QGIS

- connexion WMTS
- vérifier lisibilité multi-échelles

### Web

- OpenLayers / Mapstore
- comparer styles _Bright_ vs _Vélo_