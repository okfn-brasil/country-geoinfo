# country-geoinfo
Spatial relations between countries and  between another geographical standards

## Data
This project is  a "[country-codes](https://github.com/datasets/country-codes) derivative",  where *ISO 3166-1* data is a base information, and all CSV files can be [joined](https://en.wikipedia.org/wiki/Join_(SQL)) by *Alpha-2* codes.
Data of this project comes from multiple sources as follows.

* [Standard mundi map](http://wiki.okfn.org/Datasets_preparation/Country-code_derivatives#Standard_mundi_map): the choice of geographical data, that is also organized by *ISO 3166-1 Alpha-1* country codes. 

* *UTM zones* from [enviroprojects.org/geospatial-services/gis-resources global-utm-zones](http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/view).
 
* *Territory containment* from [CLDR](http://www.unicode.org/cldr/charts/28/supplemental/territory_containment_un_m_49.html) and [primay source at un.org](http://unstats.un.org/unsd/methods/m49/m49regin.htm), describing standard geographical agregators as European Union (EU) or Soth America (005).  Updated at a [friendly spreadsheet](https://docs.google.com/spreadsheets/d/15zt6TS8jZps11tUvX14BqmnErMl4xFMzhpDMldDde2Q/edit?usp=sharing).


## Preparation
As [described here](http://wiki.okfn.org/Datasets_preparation/Country-code_derivatives#Standard_mundi_map), the first step is to load contry polygons into PostGIS, 

```shell
# get main mundi map, witn only ~190 "country units" (WGS-84 is the SRID-4326)
mkdir sandbox
cd sandbox
wget -c http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_map_units.zip
unzip ne_10m_admin_0_map_units.zip
shp2pgsql -G -s 4326 ne_10m_admin_0_map_units  public.ne10m_units | psql -h localhost -U postgres sandbox
rm *.*
# get secondary mundi map, witn all ~240 countries
wget -c http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_countries.zip
unzip ne_10m_admin_0_countries.zip
shp2pgsql -G -s 4326 ne_10m_admin_0_countries  public.ne10m_countries | psql -h localhost -U postgres sandbox
rm *.*
# get UTM Zones
mkdir utm_zones;  cd utm_zones
wget -c http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/at_download/file -O global-utm-zones.zip
unzip global-utm-zones.zip
shp2pgsql -G -s 4326 utm_zones_final.dbf public.utm_zones | psql -h localhost -U postgres sandbox
rm *.*

psql -h localhost -U postgres sandbox
```
them create the *mundi* view, as main map:
```sql
CREATE VIEW mundi AS
     SELECT iso_a2, st_Area(geog,true) as area, geog 
     FROM ne10m_units WHERE iso_a2!='-99' -- all iso-alpha2 defined
     UNION
     SELECT iso_a2, st_Area(geog,true) as area, geog  -- the few remained countries
     FROM ne10m_countries WHERE iso_a2 NOT IN (SELECT iso_a2 FROM ne10m_units)
; -- area for simplify area-factor calculations in next steps
```
For data analysis, each preparation CSV can be isolated in an intermediary output. Example, isolating UTM_zones:
```shell
psql -h localhost -U postgres sandbox -c "
  COPY (SELECT * FROM UTM_zone_lists_out) TO '/tmp/utm_zones.csv' WITH CSV HEADER
"
```
for final output, check final JOIN. 

### Country neighbors
```sql
CREATE VIEW neighbors AS
  WITH pairs AS (
    SELECT m.iso_a2 as a, scan.iso_a2 AS b
    FROM mundi AS m INNER JOIN mundi AS scan
         ON ST_DWithin(m.geog, scan.geog, 100)
    WHERE m.iso_a2>scan.iso_a2
    ORDER BY 1, 2
  )
  SELECT a as iso_a2, array_agg(b) AS neighbor_list
  FROM ( -- original and commutative pairs 
    SELECT a, b FROM pairs  UNION  SELECT b, a FROM pairs 
  ) t
  GROUP BY 1 ORDER BY 1;
  
CREATE TABLE neighbors_out AS 
    SELECT iso_a2, 
           array_to_string(neighbor_list, ' ') as neighbor_list, 
           array_length(neighbor_list,1) AS list_len 
    FROM neighbors;
```

### Area
In km<sup>2</sup>, using spheroid,
```sql
SELECT iso_a2, (area*1e-6)::int from mundi where iso_a2!='-99' order by 1;
```

### Centroid
Cast to geometry is not precise (in geometries as RU), but result is razoable
```sql
  WITH cs AS (
   SELECT iso_a2, ST_Centroid(geog::geometry) as c
   FROM mundi ORDER BY 1
  ) SELECT iso_a2, ST_XMax(c) as cnt_lat, ST_YMax(c) as cnt_lon;
```

### UTM references
Antartica (AQ) have [this special definition](http://portal.uni-freiburg.de/AntSDI/standardsspecifications/refsystemandprojections/projections/utm.gif/image_view_fullscreen).

```sql
CREATE VIEW UTM_zone_lists AS
    SELECT  m.iso_a2, array_agg(u.code) AS utm_zones
    FROM utm_zones u INNER JOIN mundi m 
      ON (m.geog && u.geog AND st_intersects(m.geog,u.geog))
    WHERE u.code!='AQ' -- Antartica have separate def.
    GROUP BY 1 ORDER BY 1
 ;
CREATE TABLE UTM_zone_lists_out1 AS 
    SELECT iso_a2, 
           array_to_string(utm_zones, ' ') as UTMgrid_cells, 
           array_length(utm_zones,1) AS list_len
    FROM UTM_zone_lists;
```

The source preparation used [enviroprojects.org/geospatial-services/gis-resources global-utm-zones](http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/view).
Each country is under one or more cells of the UTM-grid, that is the main standard to describe contry territory in a local-planar projection. See [Utm-zones.jpg](https://upload.wikimedia.org/wikipedia/commons/e/ed/Utm-zones.jpg).

To list only "relevant" UTM cells, as we [see in a coarse map](http://earth-info.nga.mil/GandG/coordsys/grids/utm_1km_polyline_dloads.html) (ex. and ignoring small islands), a "area factor" must be noted, 

```sql
CREATE VIEW UTM_zone_lists2 AS
  WITH izones AS (
    SELECT  m.iso_a2, u.code as zone, u.area, st_area(st_intersection(m.geog,u.geog)) as ia
    FROM utm_zones u INNER JOIN mundi m 
      ON (m.geog && u.geog AND st_intersects(m.geog,u.geog))
    WHERE u.code!='AQ' -- Antartica have separate def.
  ) SELECT iso_a2, array_agg(zone) AS utm_zones
    FROM izones
    WHERE (100*area/ia)::int>0 -- minimal of 1% in area factor
    GROUP BY 1 ORDER BY 1
 ;
 
CREATE TABLE UTM_zone_lists_out2 AS 
    SELECT iso_a2, 
           array_to_string(utm_zones, ' ') as UTMgrid_cells, 
           array_length(utm_zones,1) AS list_len
    FROM UTM_zone_lists2;
```

## Adicional information 

About other [topological information](https://en.wikipedia.org/wiki/DE-9IM). All the `list_len=0` are usual islands, some `list_len=1` like Vatican City (VA) are [enclaves](https://en.wikipedia.org/wiki/List_of_enclaves_and_exclaves). The [landlocked countries](https://en.wikipedia.org/wiki/Landlocked_country), like Bolivia (BO), are not indicated. Aboy

About preparation options, see [git of unicode-cldr](https://github.com/unicode-cldr/cldr-core).
