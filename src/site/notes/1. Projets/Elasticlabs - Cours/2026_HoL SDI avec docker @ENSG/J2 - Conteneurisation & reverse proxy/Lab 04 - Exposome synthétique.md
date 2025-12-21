---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j2-conteneurisation-and-reverse-proxy/lab-04-exposome-synthetique/","noteIcon":""}
---

## 1) Pack “jeu de données fil rouge” (synthétique, léger)

📦 **Téléchargement :** exposome_paris_ouest_pack.zip

Contenu :
- **Vecteurs (CSV + WKT, EPSG:3857)**
    - `admin_area.csv` (emprise approx)
    - `network_nodes.csv` (nœuds)
    - `network_edges.csv` (arêtes + `length_m`, `air_score`, `noise_score`, `heat_score`, `exposure_cost`)
    - `exposome_grid_250m.csv` (grille 250 m + score exposome)
- **Rasters GeoTIFF (EPSG:3857, indices 0–1)**
    - `air_no2_index_epsg3857.tif`
    - `noise_index_epsg3857.tif`
    - `heat_uhii_index_epsg3857.tif`
- `README.md` avec les commandes d’import (ogr2ogr + raster2pgsql optionnel)

➡️ Le README donne une importation directe dans PostGIS via `ogr2ogr` en recréant la géométrie depuis `geom_wkt` (colonne WKT).

Voici un **lab complet** (format atelier) pour : **importer les données**, **construire le réseau “exposome”** (coûts + vues de publication), et **créer/charger des styles SLD** dans GeoServer.

---

## Lab 1 — Import PostGIS + préparation des schémas

### 0) Pré-requis (dans le conteneur / VM)

- PostGIS dispo + accès `psql`
- GDAL `ogr2ogr`
- (option) `pgrouting` si tu veux faire du routage côté DB

### 1) Créer les schémas + extensions

```sql 
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_raster;

CREATE SCHEMA IF NOT EXISTS network;
CREATE SCHEMA IF NOT EXISTS exposome;

-- Option routage
CREATE EXTENSION IF NOT EXISTS pgrouting;
```

### 2) Importer les CSV (WKT → geom)

Depuis le dossier du pack (où sont les CSV) :

```bash
# admin
ogr2ogr -f "PostgreSQL" PG:"host=$PGHOST dbname=$PGDATABASE user=$PGUSER password=$PGPASSWORD" \
  admin_area.csv \
  -oo GEOM_POSSIBLE_NAMES=geom_wkt -oo KEEP_GEOM_COLUMNS=NO \
  -a_srs EPSG:3857 -nln public.admin_area -overwrite

# nodes
ogr2ogr -f "PostgreSQL" PG:"host=$PGHOST dbname=$PGDATABASE user=$PGUSER password=$PGPASSWORD" \
  network_nodes.csv \
  -oo GEOM_POSSIBLE_NAMES=geom_wkt -oo KEEP_GEOM_COLUMNS=NO \
  -a_srs EPSG:3857 -nln network.nodes -overwrite

# edges
ogr2ogr -f "PostgreSQL" PG:"host=$PGHOST dbname=$PGDATABASE user=$PGUSER password=$PGPASSWORD" \
  network_edges.csv \
  -oo GEOM_POSSIBLE_NAMES=geom_wkt -oo KEEP_GEOM_COLUMNS=NO \
  -a_srs EPSG:3857 -nln network.edges -overwrite

# grille 250 m
ogr2ogr -f "PostgreSQL" PG:"host=$PGHOST dbname=$PGDATABASE user=$PGUSER password=$PGPASSWORD" \
  exposome_grid_250m.csv \
  -oo GEOM_POSSIBLE_NAMES=geom_wkt -oo KEEP_GEOM_COLUMNS=NO \
  -a_srs EPSG:3857 -nln exposome.grid_250m -overwrite

```

### 3) Index spatiaux + contraintes minimales

```sql 
ALTER TABLE network.nodes  ALTER COLUMN geom SET NOT NULL;
ALTER TABLE network.edges  ALTER COLUMN geom SET NOT NULL;
ALTER TABLE exposome.grid_250m ALTER COLUMN geom SET NOT NULL;

CREATE INDEX IF NOT EXISTS idx_nodes_geom ON network.nodes USING GIST (geom);
CREATE INDEX IF NOT EXISTS idx_edges_geom ON network.edges USING GIST (geom);
CREATE INDEX IF NOT EXISTS idx_grid_geom  ON exposome.grid_250m USING GIST (geom);

ANALYZE network.nodes;
ANALYZE network.edges;
ANALYZE exposome.grid_250m;

```

---

## Lab 2 — Import rasters + “basemaps” (option A : GeoServer lit les GeoTIFF)

### Option A (recommandée pour formation)

- Tu **ne charges pas** les rasters dans PostGIS.
- GeoServer lit directement les `*.tif` (File-based store).

**Dans GeoServer**

1. _Stores → Add new Store → GeoTIFF_
2. Choisis le fichier `air_no2_index_epsg3857.tif`
3. Publish → vérifie SRS = EPSG:3857
4. Répéter pour `noise_index…` et `heat_uhii_index…`

### Option B (si tu veux tout dans PostGIS)

```bash
raster2pgsql -s 3857 -I -C -M air_no2_index_epsg3857.tif public.r_air_no2 | psql "$PGURL"
raster2pgsql -s 3857 -I -C -M noise_index_epsg3857.tif public.r_noise | psql "$PGURL"
raster2pgsql -s 3857 -I -C -M heat_uhii_index_epsg3857.tif public.r_heat | psql "$PGURL"
```

Puis dans GeoServer : store “PostGIS Raster”.

---

## Lab 3 — Construire le “réseau d’exposome” (coûts & vues)

### 1) Normaliser un score “exposome” par arête

On explicite un score (0–1) et un coût composite paramétrable.

```sql
ALTER TABLE network.edges
  ADD COLUMN IF NOT EXISTS exposome_score double precision,
  ADD COLUMN IF NOT EXISTS cost_short double precision,
  ADD COLUMN IF NOT EXISTS cost_expo double precision,
  ADD COLUMN IF NOT EXISTS cost_balance double precision;

-- exposome_score : moyenne pondérée des 3 indices (déjà 0–1 dans le pack)
UPDATE network.edges
SET exposome_score = (0.5*air_score + 0.3*noise_score + 0.2*heat_score);

-- coûts (exemples)
UPDATE network.edges
SET
  cost_short   = length_m,
  cost_expo    = length_m * exposome_score,              -- exposition cumulée
  cost_balance = 0.6*length_m + 0.4*(length_m*exposome_score);

```

### 2) Créer des vues “publication” (clean & stable)

```sql
CREATE OR REPLACE VIEW exposome.v_edges_publish AS
SELECT
  edge_id,
  source, target,
  length_m,
  exposome_score,
  exposure_cost,
  cost_short, cost_expo, cost_balance,
  geom
FROM network.edges;

CREATE OR REPLACE VIEW exposome.v_grid_publish AS
SELECT
  grid_id,
  air_mean, noise_mean, heat_mean,
  exposome_score,
  geom
FROM exposome.grid_250m;

```

(Option pgRouting : tu peux ajouter une topologie `pgr_createTopology` si tu veux routage côté DB sur `cost_*`.)

---

## Lab 4 — Publier dans GeoServer (PostGIS) + WMS/WFS

### 1) Store PostGIS

- _Stores → Add new store → PostGIS NG_
- Connexion DB
- Publish :
    - `exposome:v_edges_publish`
    - `exposome:v_grid_publish`
    - (option) `admin_area`
### 2) Activer GeoWebCache

- Dans chaque layer : coche _Enabled_ GeoWebCache
- Expose WMTS / (selon config) tuiles raster

---

# Lab 5 — Styles SLD (prêts à copier-coller)

## A) SLD lignes — réseau coloré par `exposome_score`

> 5 classes (0–1). Simple, lisible en formation.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sld:StyledLayerDescriptor version="1.0.0"
 xmlns:sld="http://www.opengis.net/sld"
 xmlns:ogc="http://www.opengis.net/ogc"
 xmlns="http://www.opengis.net/sld"
 xmlns:xlink="http://www.w3.org/1999/xlink"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://www.opengis.net/sld http://schemas.opengis.net/sld/1.0.0/StyledLayerDescriptor.xsd">
  <sld:NamedLayer>
    <sld:Name>edges_exposome</sld:Name>
    <sld:UserStyle>
      <sld:Title>Réseau - Exposome score</sld:Title>
      <sld:FeatureTypeStyle>

        <sld:Rule><sld:Title>0.0–0.2</sld:Title>
          <ogc:Filter><ogc:PropertyIsBetween>
            <ogc:PropertyName>exposome_score</ogc:PropertyName>
            <ogc:LowerBoundary><ogc:Literal>0.0</ogc:Literal></ogc:LowerBoundary>
            <ogc:UpperBoundary><ogc:Literal>0.2</ogc:Literal></ogc:UpperBoundary>
          </ogc:PropertyIsBetween></ogc:Filter>
          <sld:LineSymbolizer><sld:Stroke><sld:CssParameter name="stroke">#2DC937</sld:CssParameter><sld:CssParameter name="stroke-width">2</sld:CssParameter></sld:Stroke></sld:LineSymbolizer>
        </sld:Rule>

        <sld:Rule><sld:Title>0.2–0.4</sld:Title>
          <ogc:Filter><ogc:PropertyIsBetween>
            <ogc:PropertyName>exposome_score</ogc:PropertyName>
            <ogc:LowerBoundary><ogc:Literal>0.2</ogc:Literal></ogc:LowerBoundary>
            <ogc:UpperBoundary><ogc:Literal>0.4</ogc:Literal></ogc:UpperBoundary>
          </ogc:PropertyIsBetween></ogc:Filter>
          <sld:LineSymbolizer><sld:Stroke><sld:CssParameter name="stroke">#99C140</sld:CssParameter><sld:CssParameter name="stroke-width">2</sld:CssParameter></sld:Stroke></sld:LineSymbolizer>
        </sld:Rule>

        <sld:Rule><sld:Title>0.4–0.6</sld:Title>
          <ogc:Filter><ogc:PropertyIsBetween>
            <ogc:PropertyName>exposome_score</ogc:PropertyName>
            <ogc:LowerBoundary><ogc:Literal>0.4</ogc:Literal></ogc:LowerBoundary>
            <ogc:UpperBoundary><ogc:Literal>0.6</ogc:Literal></ogc:UpperBoundary>
          </ogc:PropertyIsBetween></ogc:Filter>
          <sld:LineSymbolizer><sld:Stroke><sld:CssParameter name="stroke">#E7B416</sld:CssParameter><sld:CssParameter name="stroke-width">2</sld:CssParameter></sld:Stroke></sld:LineSymbolizer>
        </sld:Rule>

        <sld:Rule><sld:Title>0.6–0.8</sld:Title>
          <ogc:Filter><ogc:PropertyIsBetween>
            <ogc:PropertyName>exposome_score</ogc:PropertyName>
            <ogc:LowerBoundary><ogc:Literal>0.6</ogc:Literal></ogc:LowerBoundary>
            <ogc:UpperBoundary><ogc:Literal>0.8</ogc:Literal></ogc:UpperBoundary>
          </ogc:PropertyIsBetween></ogc:Filter>
          <sld:LineSymbolizer><sld:Stroke><sld:CssParameter name="stroke">#DB7B2B</sld:CssParameter><sld:CssParameter name="stroke-width">2</sld:CssParameter></sld:Stroke></sld:LineSymbolizer>
        </sld:Rule>

        <sld:Rule><sld:Title>0.8–1.0</sld:Title>
          <ogc:Filter><ogc:PropertyIsBetween>
            <ogc:PropertyName>exposome_score</ogc:PropertyName>
            <ogc:LowerBoundary><ogc:Literal>0.8</ogc:Literal></ogc:LowerBoundary>
            <ogc:UpperBoundary><ogc:Literal>1.0</ogc:Literal></ogc:UpperBoundary>
          </ogc:PropertyIsBetween></ogc:Filter>
          <sld:LineSymbolizer><sld:Stroke><sld:CssParameter name="stroke">#CC3232</sld:CssParameter><sld:CssParameter name="stroke-width">2.5</sld:CssParameter></sld:Stroke></sld:LineSymbolizer>
        </sld:Rule>

      </sld:FeatureTypeStyle>
    </sld:UserStyle>
  </sld:NamedLayer>
</sld:StyledLayerDescriptor>
```

**Import GeoServer** : _Styles → Add a new style → Upload → Save → Apply au layer `v_edges_publish`_.

---

## B) SLD polygones — grille exposome (remplissage + contour léger)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sld:StyledLayerDescriptor version="1.0.0"
 xmlns:sld="http://www.opengis.net/sld"
 xmlns:ogc="http://www.opengis.net/ogc"
 xmlns="http://www.opengis.net/sld"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://www.opengis.net/sld http://schemas.opengis.net/sld/1.0.0/StyledLayerDescriptor.xsd">
  <sld:NamedLayer>
    <sld:Name>grid_exposome</sld:Name>
    <sld:UserStyle>
      <sld:FeatureTypeStyle>

        <sld:Rule>
          <ogc:Filter><ogc:PropertyIsLessThan><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.2</ogc:Literal></ogc:PropertyIsLessThan></ogc:Filter>
          <sld:PolygonSymbolizer>
            <sld:Fill><sld:CssParameter name="fill">#2DC937</sld:CssParameter><sld:CssParameter name="fill-opacity">0.55</sld:CssParameter></sld:Fill>
            <sld:Stroke><sld:CssParameter name="stroke">#333333</sld:CssParameter><sld:CssParameter name="stroke-opacity">0.15</sld:CssParameter></sld:Stroke>
          </sld:PolygonSymbolizer>
        </sld:Rule>

        <sld:Rule>
          <ogc:Filter><ogc:And>
            <ogc:PropertyIsGreaterThanOrEqualTo><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.2</ogc:Literal></ogc:PropertyIsGreaterThanOrEqualTo>
            <ogc:PropertyIsLessThan><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.4</ogc:Literal></ogc:PropertyIsLessThan>
          </ogc:And></ogc:Filter>
          <sld:PolygonSymbolizer>
            <sld:Fill><sld:CssParameter name="fill">#99C140</sld:CssParameter><sld:CssParameter name="fill-opacity">0.55</sld:CssParameter></sld:Fill>
            <sld:Stroke><sld:CssParameter name="stroke">#333333</sld:CssParameter><sld:CssParameter name="stroke-opacity">0.15</sld:CssParameter></sld:Stroke>
          </sld:PolygonSymbolizer>
        </sld:Rule>

        <sld:Rule>
          <ogc:Filter><ogc:And>
            <ogc:PropertyIsGreaterThanOrEqualTo><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.4</ogc:Literal></ogc:PropertyIsGreaterThanOrEqualTo>
            <ogc:PropertyIsLessThan><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.6</ogc:Literal></ogc:PropertyIsLessThan>
          </ogc:And></ogc:Filter>
          <sld:PolygonSymbolizer>
            <sld:Fill><sld:CssParameter name="fill">#E7B416</sld:CssParameter><sld:CssParameter name="fill-opacity">0.55</sld:CssParameter></sld:Fill>
            <sld:Stroke><sld:CssParameter name="stroke">#333333</sld:CssParameter><sld:CssParameter name="stroke-opacity">0.15</sld:CssParameter></sld:Stroke>
          </sld:PolygonSymbolizer>
        </sld:Rule>

        <sld:Rule>
          <ogc:Filter><ogc:And>
            <ogc:PropertyIsGreaterThanOrEqualTo><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.6</ogc:Literal></ogc:PropertyIsGreaterThanOrEqualTo>
            <ogc:PropertyIsLessThan><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.8</ogc:Literal></ogc:PropertyIsLessThan>
          </ogc:And></ogc:Filter>
          <sld:PolygonSymbolizer>
            <sld:Fill><sld:CssParameter name="fill">#DB7B2B</sld:CssParameter><sld:CssParameter name="fill-opacity">0.55</sld:CssParameter></sld:Fill>
            <sld:Stroke><sld:CssParameter name="stroke">#333333</sld:CssParameter><sld:CssParameter name="stroke-opacity">0.15</sld:CssParameter></sld:Stroke>
          </sld:PolygonSymbolizer>
        </sld:Rule>

        <sld:Rule>
          <ogc:Filter><ogc:PropertyIsGreaterThanOrEqualTo><ogc:PropertyName>exposome_score</ogc:PropertyName><ogc:Literal>0.8</ogc:Literal></ogc:PropertyIsGreaterThanOrEqualTo></ogc:Filter>
          <sld:PolygonSymbolizer>
            <sld:Fill><sld:CssParameter name="fill">#CC3232</sld:CssParameter><sld:CssParameter name="fill-opacity">0.55</sld:CssParameter></sld:Fill>
            <sld:Stroke><sld:CssParameter name="stroke">#333333</sld:CssParameter><sld:CssParameter name="stroke-opacity">0.15</sld:CssParameter></sld:Stroke>
          </sld:PolygonSymbolizer>
        </sld:Rule>

      </sld:FeatureTypeStyle>
    </sld:UserStyle>
  </sld:NamedLayer>
</sld:StyledLayerDescriptor>

```

---

## C) SLD raster — “pseudo-color” (pour air/noise/heat)

À appliquer à un GeoTIFF mono-bande normalisé 0–1.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sld:StyledLayerDescriptor version="1.0.0"
 xmlns:sld="http://www.opengis.net/sld"
 xmlns="http://www.opengis.net/sld"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://www.opengis.net/sld http://schemas.opengis.net/sld/1.0.0/StyledLayerDescriptor.xsd">
  <sld:NamedLayer>
    <sld:Name>raster_index</sld:Name>
    <sld:UserStyle>
      <sld:FeatureTypeStyle>
        <sld:Rule>
          <sld:RasterSymbolizer>
            <sld:Opacity>0.75</sld:Opacity>
            <sld:ColorMap type="ramp">
              <sld:ColorMapEntry color="#2DC937" quantity="0.0" label="faible"/>
              <sld:ColorMapEntry color="#99C140" quantity="0.25" label="moyen bas"/>
              <sld:ColorMapEntry color="#E7B416" quantity="0.50" label="moyen"/>
              <sld:ColorMapEntry color="#DB7B2B" quantity="0.75" label="moyen haut"/>
              <sld:ColorMapEntry color="#CC3232" quantity="1.0" label="élevé"/>
            </sld:ColorMap>
          </sld:RasterSymbolizer>
        </sld:Rule>
      </sld:FeatureTypeStyle>
    </sld:UserStyle>
  </sld:NamedLayer>
</sld:StyledLayerDescriptor>

```

---

## “Definition of Done” du lab (checklist)

-  Tables importées : `network.nodes`, `network.edges`, `exposome.grid_250m`
-  Colonnes calculées : `exposome_score`, `cost_short`, `cost_expo`, `cost_balance`
-  Vues publiées : `exposome.v_edges_publish`, `exposome.v_grid_publish`
-  GeoServer : layers publiés (WMS/WFS) + rasters (GeoTIFF)
-  Styles SLD appliqués (edges + grid + raster)
-  Test QGIS : connexion WMS/WFS + rendu correct

