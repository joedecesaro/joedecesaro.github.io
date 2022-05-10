# Potential Wind Power in Iowa
#### Authors: Kirsten Hodgson and Joe DeCesaro

Date: 2021-12-02

## Introduction
In this assignment, we evaluate the maximum potential annual wind energy production available to the state of Iowa for two possible siting scenarios: where residential buildings require a buffer of 3 times the height of the turbine hub, and where they require a buffer of 10 times the hub height. Data is extracted from the PostGIS database and most feature data originates from OpenStreetMap. We first query the database, then merge the siting constraints and subtract them from the gridded annual wind velocity in Iowa. We use the results to calculate the number of wind turbines and maximum amount of wind power that is possible in Iowa according to our siting constraints.


```python
import psycopg2
import sqlalchemy
import geopandas as gpd
import pandas as pd
import math
```

## Establish Database Connection and Siting Constraints


```python
pg_uri_template = 'postgresql+psycopg2://{user}:{password}@{host}/{database}'
db_uri = pg_uri_template.format(
    host = '128.111.89.111', 
    database = 'osmiowa',
    user = 'eds223_students',
    password = 'eds223',
)
db_uri
db = sqlalchemy.create_engine(db_uri)
db
```




    Engine(postgresql+psycopg2://eds223_students:***@128.111.89.111/osmiowa)




```python
H3 = 150 * 3
H10 = 150 * 10 
airport = 7500 
H2 = 150 * 2
H1 = 150
d = 136
turbine_area = (math.pi *((5 * d)**2))
```

## Create Subqueries

Here we create the subquery for each siting constraint, in the order they are presented in the assignment documentation.


```python
building_query_3h = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
ST_BUFFER(way, {H3}) as way 
FROM 
planet_osm_polygon 
WHERE 
building in ('yes', 'residential', 'apartments', 'house', 'static_caravan', 'detached')
OR 
landuse = 'residential'
OR 
place = 'town'
"""
# buildings_1 = gpd.read_postgis(building_query_3h, con = db, geom_col = 'way')
```


```python
building_query_10h = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
ST_BUFFER(way, {H10}) as way 
FROM 
planet_osm_polygon 
WHERE 
building in ('yes', 'residential', 'apartments', 'house', 'static_caravan', 'detached')
OR 
landuse = 'residential'
OR 
place = 'town'
"""
# buildings_2 = gpd.read_postgis(building_query_10h, con = db, geom_col = 'way')
```


```python
building_query_non_res = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
ST_BUFFER(way, {H3}) as way 
FROM 
planet_osm_polygon 
WHERE 
building is not NULL 
OR 
building not in ('yes', 'residential', 'apartments', 'house', 'static_caravan', 'detached')
"""
# buildings_non_res = gpd.read_postgis(building_query_non_res, con = db, geom_col = 'way')
```


```python
airport_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
ST_BUFFER(way, {airport}) as way 
FROM 
planet_osm_polygon 
WHERE 
aeroway is not NULL 
"""
# airports = gpd.read_postgis(airport_query, con = db, geom_col = 'way')
```


```python
military_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
way 
FROM 
planet_osm_polygon 
WHERE 
military is not NULL 
OR
landuse = 'military'
"""
# military = gpd.read_postgis(military_query, con = db, geom_col = 'way')
```


```python
highway_railroad_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_line.natural, planet_osm_line.power, "generator:source", water, waterway,
ST_BUFFER(way, {H2}) as way 
FROM 
planet_osm_line 
WHERE 
railway is not NULL
AND
railway not in ('abandoned', 'disused')
OR
highway in ('motorway','motorway_link', 'trunk', 'trunk_link', 'primary', 'primary_link', 'secondary', 'secondary_link')
"""
# highway_rail = gpd.read_postgis(highway_railroad_query, con = db, geom_col = 'way')
```


```python
reserve_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
way 
FROM 
planet_osm_polygon  
WHERE 
leisure = 'nature_reserve'
OR 
landuse in ('salt_pond', 'conservation')
OR 
"natural" in ('wetland')
"""
# reserve = gpd.read_postgis(reserve_query, con = db, geom_col = 'way')
```


```python
river_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_line.natural, planet_osm_line.power, "generator:source", water, waterway,
ST_BUFFER(way, {H1}) as way 
FROM 
planet_osm_line 
WHERE 
waterway = 'river' 
"""
# river = gpd.read_postgis(river_query, con = db, geom_col = 'way')
```


```python
lake_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
way 
FROM 
planet_osm_polygon  
WHERE 
water = 'lake'
"""
# lake = gpd.read_postgis(lake_query, con = db, geom_col = 'way')
```


```python
power_line_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_line.natural, planet_osm_line.power, "generator:source", water, waterway,
ST_BUFFER(way, {H2}) as way 
FROM 
planet_osm_line 
WHERE 
power is not NULL 
"""
# power_line = gpd.read_postgis(power_line_query, con = db, geom_col = 'way')
```


```python
power_plant_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_polygon.natural, planet_osm_polygon.power, "generator:source", water, waterway,
ST_BUFFER(way, {H1}) as way 
FROM 
planet_osm_polygon 
WHERE 
power is not NULL 
"""
# power_plant = gpd.read_postgis(power_plant_query, con = db, geom_col = 'way')
```


```python
turbine_query = f"""
SELECT 
osm_id, building, landuse, aeroway, military, highway, railway, 
leisure, planet_osm_point.natural, planet_osm_point.power, "generator:source", water, waterway,
ST_BUFFER(way, {5 * d}) as way 
FROM 
planet_osm_point 
WHERE 
"generator:source" = 'wind' 
"""
# turbines = gpd.read_postgis(turbine_query, con = db, geom_col = 'way')
```

## Merge Subqueries Scenario 1 (3H)

Next we merge all the subqueries for scenario 1 using `union` and create a geodataframe of the result.


```python
scenario_1_query = f"""
{building_query_3h} UNION 
{building_query_non_res} UNION
{airport_query} UNION
{military_query} UNION
{highway_railroad_query} UNION
{reserve_query} UNION
{river_query} UNION
{lake_query} UNION
{power_line_query} UNION
{power_plant_query} UNION
{turbine_query}
"""
```


```python
scenario_1 = gpd.read_postgis(scenario_1_query, con = db, geom_col = 'way')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>osm_id</th>
      <th>building</th>
      <th>landuse</th>
      <th>aeroway</th>
      <th>military</th>
      <th>highway</th>
      <th>railway</th>
      <th>leisure</th>
      <th>natural</th>
      <th>power</th>
      <th>generator:source</th>
      <th>water</th>
      <th>waterway</th>
      <th>way</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-13429329</td>
      <td>roof</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>POLYGON ((1585976.821 1115123.834, 1585976.722...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>-13429328</td>
      <td>yes</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>POLYGON ((1585990.543 1115115.731, 1585990.453...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>-13429327</td>
      <td>roof</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>POLYGON ((1586670.363 1115704.927, 1586682.846...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>-13429326</td>
      <td>yes</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>POLYGON ((1586230.437 1115255.462, 1586230.465...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-13429325</td>
      <td>yes</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>POLYGON ((1586289.806 1115743.198, 1586290.258...</td>
    </tr>
  </tbody>
</table>
</div>



## Merge Subqueries Scenario 2 (10H)

Now we repeat the above steps to merge the subqueries for scenario 2.


```python
scenario_2_query = f"""
{building_query_10h} UNION 
{building_query_non_res} UNION
{airport_query} UNION
{military_query} UNION
{highway_railroad_query} UNION
{reserve_query} UNION
{river_query} UNION
{lake_query} UNION
{power_line_query} UNION
{power_plant_query} UNION
{turbine_query}
"""
```


```python
scenario_2 = gpd.read_postgis(scenario_2_query, con = db, geom_col = 'way')
```

## Wind Data

Here we query the wind data from the database and subtract the siting constraints for each scenario from it.


```python
wind_data_query = f"""
SELECT 
*
FROM 
wind_cells_10000 
"""
```


```python
wind_data = gpd.read_postgis(wind_data_query, con=db, geom_col='geom')
```

### Subtracting Scenario 1


```python
suitable_3h = wind_data.overlay(scenario_1, how='difference', keep_geom_type=False)
```

### Subtracting Scenario 2


```python
suitable_10h = wind_data.overlay(scenario_2, how='difference', keep_geom_type=False)
```

## Power Production

Now, we calculate the total number of turbines that could fit in the state of Iowa in each scenario, then use this to calculate the total wind production possible in Iowa.

### Scenario 1

The calculations for the possible number of wind turbines in scenario 1:


```python
suitable_3h['area_sq_m'] = suitable_3h['geom'].area
```


```python
suitable_3h['turbine_fit'] = suitable_3h['area_sq_m']/turbine_area
```


```python
total_turbines_3h = suitable_3h['turbine_fit'].sum()
total_turbines_3h
```




    57395.62981630259



The calculations for the total possible amount of wind power production in scenario 1:


```python
suitable_3h['wind_production_per_cell'] = (2.6*suitable_3h['wind_speed']-5)*suitable_3h['turbine_fit']
```


```python
total_wind_production_3h = suitable_3h['wind_production_per_cell'].sum()
total_wind_production_3h
```




    1066157.7002702449




```python
print("The total number of turbines possible for siting scenario 2 is", total_turbines_3h, ". The total annual wind production for this scenario is", total_wind_production_3h, "GWh.")
```

    The total number of turbines possible for siting scenario 2 is 57395.62981630259 . The total annual wind production for this scenario is 1066157.7002702449 GWh.


### Scenario 2

The calculations for the possible number of wind turbines in scenario 2:


```python
suitable_10h['area_sq_m'] = suitable_10h['geom'].area
```


```python
suitable_10h['turbine_fit'] = suitable_10h['area_sq_m']/turbine_area
```


```python
total_turbines_10h = suitable_10h['turbine_fit'].sum()
total_turbines_10h
```




    52121.75830678892



The calculations for the total possible amount of wind power production in scenario 2:


```python
suitable_10h['wind_production_per_cell'] = (2.6*suitable_10h['wind_speed']-5)*suitable_10h['turbine_fit']
```


```python
total_wind_production_10h = suitable_10h['wind_production_per_cell'].sum()
total_wind_production_10h
```




    968212.2828864241




```python
print("The total number of turbines possible for siting scenario 2 is", total_turbines_10h, ". The total annual wind production for this scenario is", total_wind_production_10h, "GWh.")
```

    The total number of turbines possible for siting scenario 2 is 52121.75830678892 . The total annual wind production for this scenario is 968212.2828864241 GWh.

