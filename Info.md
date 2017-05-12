# Useful informational queries

## Find stat group (aka: "terminal number", "statistical group")
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
