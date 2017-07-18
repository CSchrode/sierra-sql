## Find bib records with more than one indexed call number on the record.
Also reports the fields so that they may more easily be found.
```sql
SELECT
r.record_type_code || r.record_num || 'a' as bib_record_num,
( 
	SELECT
	string_agg(v.marc_tag || ' ' || v.field_content, ', ') as call_numbers_list

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.varfield_type_code = 'c'

	GROUP BY
	v.record_id

) as index_call_numbers_list

FROM
sierra_view.record_metadata as r

WHERE
r.id IN (
	SELECT
	v.record_id

	FROM
	sierra_view.bib_record as b

	JOIN
	sierra_view.varfield as v
	ON
	  v.record_id = b.record_id

	WHERE
	v.varfield_type_code = 'c'

	GROUP BY
	v.record_id

	HAVING
	count(*) > 1
)
```


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