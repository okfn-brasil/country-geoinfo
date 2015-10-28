# country-geoinfo
Spatial relations between countries and  between another geographical standards

## Data
This project is  a "[country-codes](https://github.com/datasets/country-codes) derivative",  where *ISO 3166-1* data is a base information, and all CSV files can be [joined](https://en.wikipedia.org/wiki/Join_(SQL)) by *Alpha-2* codes.
Data of this project comes from multiple sources as follows.

* [Standard mundi map](http://wiki.okfn.org/Datasets_preparation/Country-code_derivatives#Standard_mundi_map): the choice of geographical data, that is also organized by *ISO 3166-1 Alpha-1* country codes. 

* *UTM zones* from [enviroprojects.org/geospatial-services/gis-resources global-utm-zones](http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/view)
 
* *Territory containment* from CLDR-tr35, describing standard geographical agregators as European Union (EU) or Soth America (005).


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
```
`psql -h localhost -U postgres sandbox -c "SELECT *, array_length(neighbor_list,1) as list_len FROM neighbors"`

### Area
In km<sup>2</sup>, using spheroid,
```sql
SELECT iso_a2, (ST_Area(geog,true)*1e-6)::int from ne10m_countries where iso_a2!='-99' order by 1;
```

### Centroid
cast to geometry and guaranteed to lie on the surface
```sql
    SELECT iso_a2, st_astext(ST_PointOnSurface(geog::geometry)) as centroid -- long lat
    FROM mundi ORDER BY 1;
```

### UTM references
Antartica (AQ) have [this special definition](http://portal.uni-freiburg.de/AntSDI/standardsspecifications/refsystemandprojections/projections/utm.gif/image_view_fullscreen).

```sql
CREATE VIEW UTM_references AS
    SELECT  m.iso_a2, array_agg(u.code) AS utm_codes
    FROM utm_zones u INNER JOIN mundi m 
      ON (m.geog && u.geog AND st_intersects(m.geog,u.geog))
    WHERE u.code!='AQ' -- Antartica have separate def.
    GROUP BY 1 ORDER BY 1
 ;
```
`psql -h localhost -U postgres sandbox -c "SELECT *, array_length(utm_codes,1) AS list_len FROM UTM_references"`

The source preparation used [enviroprojects.org/geospatial-services/gis-resources global-utm-zones](http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/view).
Each country is under one or more cells of the UTM-grid, that is the main standard to describe contry territory in a local-planar projection. See [Utm-zones.jpg](https://upload.wikimedia.org/wikipedia/commons/e/ed/Utm-zones.jpg).
