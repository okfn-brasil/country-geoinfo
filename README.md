# country-geoinfo
Spatial relations between countries and  between another geographical standards

## Data

Data comes from multiple sources as follows.

* [country-codes](https://github.com/datasets/country-codes): the main dataset, where *ISO 3166-1* data is organized and other standard codes can be related to *Alpha-1* codes.

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
shp2pgsql -s 4326 ne_10m_admin_0_map_units  public.ne10m_units | psql -h localhost -U postgres sandbox
rm *.*
# get secondary mundi map, witn all ~240 countries
wget -c http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_countries.zip
unzip ne_10m_admin_0_countries.zip
shp2pgsql -s 4326 ne_10m_admin_0_countries  public.ne10m_countries | psql -h localhost -U postgres sandbox
rm *.*
# get UTM Zones
mkdir utm_zones;  cd utm_zones
wget -c http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/at_download/file -O global-utm-zones.zip
unzip global-utm-zones.zip
shp2pgsql  -s 4326 utm_zones_final.dbf public.utm_zones | psql -h localhost -U postgres sandbox
rm *.*

psql -h localhost -U postgres sandbox
```
them create the *mundi* view, as main map:
```sql
CREATE VIEW mundi AS
     SELECT iso_a2, st_Area(geom) as area, geom 
     FROM ne10m_units WHERE iso_a2!='-99' -- no ocean
     UNION
     SELECT iso_a2, st_Area(geom) as area, geom 
     FROM ne10m_countries WHERE iso_a2 NOT IN (SELECT iso_a2 FROM ne10m_units)
; -- area for simplify area-factor calculations in next steps
```
### Country neighbors
```sql
CREATE VIEW neighbors AS
  WITH pairs AS (
    SELECT m.iso_a2 as a, scan.iso_a2 AS b
    FROM mundi AS m INNER JOIN mundi AS scan
         ON ST_DWithin(m.geom, scan.geom, 0.0001)
    WHERE m.gid>scan.gid
    ORDER BY 1, 2
  ) 
  SELECT a as iso_a2, array_agg(b) AS neighbor_list
  FROM (
    SELECT a, b FROM pairs  UNION  SELECT b, a FROM pairs 
  ) t
  GROUP BY 1 ORDER BY 1;
```
`psql -h localhost -U postgres sandbox -c "SELECT *, array_length(neighbor_list,1) as list_len FROM neighbors"`

### Centroid
```sql
    SELECT iso_a2, st_astext(ST_PointOnSurface(geom)) as centroid -- long lat
    FROM mundi ORDER BY 1;
```

### UTM references
Antartica (AQ) have [this special definition](http://portal.uni-freiburg.de/AntSDI/standardsspecifications/refsystemandprojections/projections/utm.gif/image_view_fullscreen).

```sql
CREATE VIEW UTM_references AS
    SELECT  m.iso_a2, array_agg(u.code) AS utm_codes
    FROM utm_zones u INNER JOIN mundi m 
      ON (m.geom && u.geom AND st_intersects(m.geom,u.geom))
    WHERE u.code!='AQ' -- Antartica have separate def.
    GROUP BY 1 ORDER BY 1
 ;
```
`psql -h localhost -U postgres sandbox -c "SELECT *, array_length(utm_codes,1) AS list_len FROM UTM_references"`

The source preparation used [enviroprojects.org/geospatial-services/gis-resources global-utm-zones](http://www.enviroprojects.org/geospatial-services/gis-resources/global-utm-zones/view).
Each country is under one or more cells of the UTM-grid, that is the main standard to describe contry territory in a local-planar projection. See [Utm-zones.jpg](https://upload.wikimedia.org/wikipedia/commons/e/ed/Utm-zones.jpg).
