# Userful informational queries


## Find stat group (aka: "terminal number", "statistical group" 
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
