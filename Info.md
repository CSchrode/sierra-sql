# Useful informational queries

## Find stat group (aka: "terminal number", "statistical group") code number and name 
sample output
```csv
0;Main Unknown
1;Main
9;Main Pickup Window
10;Main Sorter-xmnsort1
11;Main cir schk01
```

```sql
SELECT
g.code_num, n.name

FROM 
sierra_view.statistic_group as g 

JOIN
sierra_view.statistic_group_name as n
on
  n.statistic_group_id = g.id

ORDER BY 
g.code_num
```

## Find "Agency" code number and name

```sql
SELECT

p.code_num as code,
n.name

FROM 
sierra_view.agency_property as p

JOIN
sierra_view.agency_property_name as n
ON
  n.agency_property_id = p.id

ORDER BY code_num
```

## Find location code numbers and names
SELECT
l.code,
n.name

FROM 
sierra_view.location as l

JOIN
sierra_view.location_name as n
ON
  n.location_id = l.id

ORDER BY 
l.code