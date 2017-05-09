# Find duplicate patrons
***

## Find duplicate patrons by name + birthdate

sample output
```csv
"p1234567a";"Patron, John Q 1977-01-01";15;"sm";"2015-08-26 17:44:31-04"
"p7654321a";"Patron, John Q 1977-01-01";0;"sm";"2017-05-06 08:35:19-04"
```

```sql
SELECT
'p' || r.record_num || 'a' as patron_record_num,
pn.last_name || ', ' ||pn.first_name || COALESCE(' ' || NULLIF(pn.middle_name, ''), '') || ' ' || pr.birth_date_gmt,
pr.ptype_code,
pr.home_library_code,
pr.activity_gmt

FROM
sierra_view.patron_record_fullname as pn

JOIN
sierra_view.patron_record as pr
ON
  pr.record_id = pn.patron_record_id

JOIN
sierra_view.record_metadata as r
ON
  r.id = pr.record_id

WHERE
pr.birth_date_gmt || pn.first_name || COALESCE(' ' || NULLIF(pn.middle_name, ''), '') || ' ' || pn.last_name 
IN
(
	SELECT

	p.birth_date_gmt ||
	n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') || ' ' || n.last_name as patron_name
	-- e.index_entry,
	-- count(*) as matches

	FROM
	sierra_view.record_metadata AS r

	JOIN
	sierra_view.patron_record AS p
	ON
	  p.record_id = r.id

	JOIN
	sierra_view.patron_record_fullname AS n
	ON
	  n.patron_record_id = r.id

	-- JOIN
	-- sierra_view.phrase_entry AS e
	-- ON
	--   (e.record_id = r.id) AND (e.index_tag = 'b') AND (e.varfield_type_code = 'b')

	WHERE 
	r.record_type_code = 'p'
	-- and r.creation_date_gmt >= '2017-05-01'

	GROUP BY
	p.birth_date_gmt,
	patron_name
	-- e.index_entry

	HAVING
	COUNT(*) > 1
)

ORDER BY
pn.last_name || pn.first_name || pr.birth_date_gmt || COALESCE(' ' || NULLIF(pn.middle_name, ''), '')
```
...including barcode
```sql
SELECT
'p' || r.record_num || 'a' as patron_record_num,
pn.last_name || ', ' ||pn.first_name || COALESCE(' ' || NULLIF(pn.middle_name, ''), '') || ' ' || pr.birth_date_gmt as patron,
pr.ptype_code,
pr.home_library_code,
pr.activity_gmt,
e.index_entry as barcode

FROM
sierra_view.patron_record_fullname as pn

JOIN
sierra_view.patron_record as pr
ON
  pr.record_id = pn.patron_record_id

JOIN
sierra_view.record_metadata as r
ON
  r.id = pr.record_id

JOIN
sierra_view.phrase_entry AS e
ON
  (e.record_id = r.id) AND (e.index_tag = 'b') AND (e.varfield_type_code = 'b')
  
WHERE
pr.birth_date_gmt || pn.first_name || COALESCE(' ' || NULLIF(pn.middle_name, ''), '') || ' ' || pn.last_name 
IN
(
	SELECT

	p.birth_date_gmt ||
	n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') || ' ' || n.last_name as patron_name
	-- e.index_entry,
	-- count(*) as matches

	FROM
	sierra_view.record_metadata AS r

	JOIN
	sierra_view.patron_record AS p
	ON
	  p.record_id = r.id

	JOIN
	sierra_view.patron_record_fullname AS n
	ON
	  n.patron_record_id = r.id

	-- JOIN
	-- sierra_view.phrase_entry AS e
	-- ON
	--   (e.record_id = r.id) AND (e.index_tag = 'b') AND (e.varfield_type_code = 'b')

	WHERE 
	r.record_type_code = 'p'
	-- and r.creation_date_gmt >= '2017-05-01'

	GROUP BY
	p.birth_date_gmt,
	patron_name
	-- e.index_entry

	HAVING
	COUNT(*) > 1
)

ORDER BY
pn.last_name || pn.first_name || pr.birth_date_gmt || COALESCE(' ' || NULLIF(pn.middle_name, ''), '')
```



## Find duplicate patrons by birthdate + name + barcode

sample output:
```csv
"1990-01-01";"John Q Patron";"12345678";2
"1977-02-02";"Betty J Patron"; "12345679";2
```

```sql
SELECT

p.birth_date_gmt,
n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') || ' ' || n.last_name as patron_name,
e.index_entry,
count(*) as matches

FROM
sierra_view.record_metadata AS r

JOIN
sierra_view.patron_record AS p
ON
  p.record_id = r.id

JOIN
sierra_view.patron_record_fullname AS n
ON
  n.patron_record_id = r.id

JOIN
sierra_view.phrase_entry AS e
ON
  (e.record_id = r.id) AND (e.index_tag = 'b') AND (e.varfield_type_code = 'b')

WHERE 
r.record_type_code = 'p'
-- and r.creation_date_gmt >= '2017-05-01'

GROUP BY
p.birth_date_gmt,
patron_name,
e.index_entry

HAVING
COUNT(*) > 1
```

***

## Find duplicate patron barcodes that start with “SR”
temporary patron cards are issued barcodes starting with SR, but are sometimes accidentally duplicated. This query will find those patron record nums.

sample output:
```csv
"p1234567a";"sr123456";"2017-04-23 11:20:59-04"
"p1234568a";"sr123456";"2017-04-04 15:11:56-04"
```

```sql 
SELECT
-- patron record number, barcode, and create date. 
'p' || r.record_num || 'a',
p.index_entry,
r.creation_date_gmt

FROM
sierra_view.phrase_entry as p

JOIN
sierra_view.record_metadata as r
ON
  r.id = p.record_id

JOIN
sierra_view.patron_record as pr
ON
  pr.record_id = r.id

where 
p.index_tag || p.index_entry IN
(
SELECT
'b' || e.index_entry

FROM
sierra_view.phrase_entry as e

JOIN
sierra_view.record_metadata as r
ON
  r.id = e.record_id

WHERE
e.index_tag || e.index_entry SIMILAR TO 'b' || 'sr[0-9]{1,}'

GROUP BY
e.index_entry

HAVING COUNT(e.index_entry) > 1
)

ORDER BY 
r.creation_date_gmt DESC
```

## Find patrons with note of "ConnectED"
```sql
SELECT
*

FROM

sierra_view.record_metadata as r

JOIN
sierra_view.patron_record  as p
ON
  p.record_id = r.id

JOIN
sierra_view.varfield as v 
ON
  (v.record_id = r.id) AND (v.varfield_type_code = 'x') AND (v.field_content = 'ConnectED')

where r.record_type_code = 'p'
```