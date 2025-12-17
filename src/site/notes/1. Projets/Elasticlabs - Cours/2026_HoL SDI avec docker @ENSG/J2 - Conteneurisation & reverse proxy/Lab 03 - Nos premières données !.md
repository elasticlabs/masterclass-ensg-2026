---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j2-conteneurisation-and-reverse-proxy/lab-03-nos-premieres-donnees/","noteIcon":""}
---

# TUTO — Construire une basemap (OSM + fonds standards)
**Formation : SDI conteneurisée & pensée système**

---

## 🎯 Objectifs

À l’issue de ce tutoriel, vous saurez :

- construire une **basemap composite**
- combiner **fonds génériques** + **données locales OSM**
- styliser les routes selon :
  - un style **OSM Bright–like**
  - un style **praticabilité vélo**
- publier la basemap via **GeoServer (WMS/WMTS)**

---

## 🧠 Rappel conceptuel

Une **basemap n’est pas neutre** :
- elle guide la lecture
- elle prépare des usages (navigation, analyse, décision)
- elle peut déjà porter une **intention politique / sanitaire**

👉 Ici, la basemap prépare :
- le routage
- la lecture de l’exposome
- la mobilité douce

---

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

Télécharger depuis https://www.naturalearthdata.com :

- `ne_10m_land`
- `ne_10m_ocean`
- `ne_10m_lakes`
- `ne_10m_rivers_lake_centerlines`
- `ne_10m_urban_areas`

Importer dans PostGIS :

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

# 2️⃣ Données OSM : extraction ciblée

## 2.1 Source OSM

Options :
- Geofabrik (Île-de-France)
- BBBike
- Overpass (moins reproductible)

👉 Recommandé pour la formation : **Geofabrik**
Attention à resserrer l'emprise sur une zone permettant de futurs calculs. 

## 2.2. Importer dans OSM

```sql
CREATE SCHEMA IF NOT EXISTS osm;
```

Import : 

```bash
osm2pgsql \
  -d gis \
  --create \
  --slim \
  --hstore \
  --schema=osm \
  --proj=3857 \
  ile-de-france-latest.osm.pbf
```

👉 Tables créées :

- `osm.planet_osm_line`
- `osm.planet_osm_polygon`
- `osm.planet_osm_point`

# 3️⃣ Préparer les couches de basemap

## 3.1 Routes (vue dédiée)

```sql
CREATE OR REPLACE VIEW publish.osm_roads AS
SELECT
  way AS geom,
  highway,
  name,
  surface,
  bicycle,
  lanes,
  maxspeed
FROM osm.planet_osm_line
WHERE highway IS NOT NULL;
```

## 3.2 Eau & occupation du sol

```sql
CREATE OR REPLACE VIEW publish.osm_water AS
SELECT way AS geom
FROM osm.planet_osm_polygon
WHERE water IS NOT NULL
   OR waterway IS NOT NULL;
```

```sql
CREATE OR REPLACE VIEW publish.osm_landuse AS
SELECT way AS geom, landuse
FROM osm.planet_osm_polygon
WHERE landuse IS NOT NULL;
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

## 4.2 Exemple SLD (extrait)

```xml
<Rule>
  <Name>primary</Name>
  <Filter>
    <PropertyIsEqualTo>
      <PropertyName>highway</PropertyName>
      <Literal>primary</Literal>
    </PropertyIsEqualTo>
  </Filter>
  <LineSymbolizer>
    <Stroke>
      <CssParameter name="stroke">#fcd6a4</CssParameter>
      <CssParameter name="stroke-width">3</CssParameter>
    </Stroke>
  </LineSymbolizer>
</Rule>
```

👉 Créer un SLD avec règles par type `highway`.

# 5️⃣ Style 2 — Routes “praticabilité vélo”

🎯 **Objectif**  
Lire **où il est agréable / sûr de rouler**.

## 5.1 Indicateur vélo (heuristique simple)

Créer une **vue enrichie** :

```sql
CREATE OR REPLACE VIEW publish.osm_roads_bike AS
SELECT
  geom,
  highway,
  surface,
  bicycle,
  CASE
    WHEN highway IN ('cycleway') THEN 1
    WHEN highway IN ('residential','living_street')
         AND (surface IS NULL OR surface IN ('asphalt','paved')) THEN 2
    WHEN highway IN ('tertiary','secondary') THEN 3
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