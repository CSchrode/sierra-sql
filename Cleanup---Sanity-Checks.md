## Find location codes for all items, and the count of items in each of those locations
```sql
SELECT
i.location_code,
count(*)

FROM
sierra_view.item_record as i

GROUP BY
i.location_code

ORDER BY
count(*) DESC
```

## Find duplicated control tags
```sql
SELECT

'http://classic.cincinnatilibrary.org=' || id2reckey(record_id) || 'a' AS link,
id2reckey(record_id) || 'a' as bib_record_num,
index_entry as control_number

FROM
sierra_view.phrase_entry

WHERE
index_tag = 'o' AND
index_entry IN (
	SELECT
	index_entry
	
	FROM
	sierra_view.phrase_entry
	
	WHERE
	index_tag = 'o'
	
	GROUP BY
	index_entry
	
	HAVING
	    count(id) > 1
)
ORDER BY
    control_number, bib_record_num
```