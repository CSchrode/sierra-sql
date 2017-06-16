# Count number of items by criteria
```sql
SELECT
-- i.agency_code_num,
p.name as item_agency,
count(*)

FROM
sierra_view.item_record as i

JOIN
sierra_view.agency_property_myuser as p
ON
  p.code = i.agency_code_num

WHERE
i.item_status_code IN (
'-',
'o',
'r',
't',
'!'
)
AND substring(i.location_code,1,2) NOT IN (
'vi',
'yt',
'ym',
'yc',
'yh'
)

GROUP BY
p.name,
p.display_order

ORDER BY
p.display_order
```