
# Find cases where an item record is linked to more than one bib record (improved)
```sql
--- find same item record linked to multiple bib records 
SELECT
r.record_type_code || r.record_num || 'a' as bib_record_num,
ir.record_type_code || ir.record_num || 'a' as item_record_num

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.bib_record_id = r.id

JOIN
sierra_view.record_metadata as ir
ON
  ir.id = l.item_record_id

WHERE
l.item_record_id IN (
	SELECT
	l.item_record_id
	-- ,count(item_record_id)

	FROM
	sierra_view.bib_record_item_record_link as l

	GROUP BY
	l.item_record_id

	HAVING
	count(item_record_id) > 1
);
```

# Find cases where an item record is linked to more than one bib record
```sql
-- find item records that are linked to more than one bib record
SELECT
l.item_record_id,
count(bib_record_id)

FROM
sierra_view.bib_record_item_record_link as l

GROUP BY
l.item_record_id

HAVING
count(bib_record_id) > 1
```


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