# Find patrons with more than 1 name listed
```sql
SELECT
r.record_type_code || r.record_num || 'a' as record_number,
(
	SELECT
	string_agg(v.field_content, ',' order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.varfield_type_code = 'b'
) as barcodes,

(
	SELECT
	string_agg(n.last_name || ' ' || n.first_name || ' ' || n.middle_name, ', ' order by n.display_order)

	FROM
	sierra_view.patron_record_fullname as n

	WHERE
	n.patron_record_id = r.id
) as fullnames_listed

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.patron_record_fullname as f
ON
  f.patron_record_id = r.id

WHERE
r.record_type_code || r.campus_code = 'p'
and r.deletion_date_gmt is null

GROUP BY
r.id, r.record_type_code, r.record_num

HAVING
count(*) > 1
```

# Find duplicate patrons
***

## Find patron records that have duplicate barcodes
If barcode number is repeated on multiple records, this will display the patron record information

sample output:
```csv
created_date;barcode;patron_record_num;patron;ptype;last_circ_activity;expiration_date
2012-07-19 02:17:11-04;49143662;p1234567a;Patron, John Q 1947-02-26;0;2015-12-22 08:38:26-05;2020-12-22 04:00:00-05
2016-06-13 14:44:03-04;49143662;p1234568a;Patron, John Q 1947-02-26;0;2016-06-13 14:44:03-04;2021-06-13 04:00:00-04
2012-07-18 22:59:21-04;50948033;p1234569a;Patron, Patty L 1986-12-12;6;2015-05-26 12:30:29-04;2013-03-06 04:00:00-05
2013-06-15 16:07:41-04;50948033;p1234570a;Patty, Patty L 1986-12-12;0;2013-07-20 11:16:24.26-04;2018-06-15 04:00:00-04
```

```sql
SELECT
r.creation_date_gmt as created_date,
e.index_entry as barcode,
'p' || r.record_num || 'a' as patron_record_num,
n.last_name || ', ' ||n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') || ' ' || p.birth_date_gmt as patron,
p.ptype_code as ptype,
p.activity_gmt as last_circ_activity,
p.expiration_date_gmt as expiration_date

FROM
sierra_view.phrase_entry as e

JOIN
sierra_view.patron_record as p
ON
  p.record_id = e.record_id

JOIN
sierra_view.record_metadata as r
ON
  r.id = p.record_id

JOIN
sierra_view.patron_record_fullname AS n
ON
  n.patron_record_id = r.id

WHERE 
-- combining the index tag with the index entry returns more quickly
e.index_tag || e.index_entry IN (
-- 'b' || '63160170',
-- 'b' || '63274153'
	SELECT
	'b' || e.index_entry as barcode

	FROM
	sierra_view.phrase_entry AS e

	JOIN
	sierra_view.patron_record as p
	ON
	  p.record_id = e.record_id

	WHERE
	e.index_tag || e.varfield_type_code = 'bb'

	GROUP BY
	barcode

	HAVING 
	count(*) > 1
)

ORDER BY
barcode,
patron ASC
```

## Find patrons with *multiple* barcodes

```sql
SELECT
'p' || r.record_num || 'a' as record_num

FROM
sierra_view.record_metadata as r

WHERE
r.id IN
(
	SELECT
	-- count(*),
	e.record_id

	FROM
	sierra_view.patron_record as p

	JOIN
	sierra_view.phrase_entry as e
	ON
	  (e.record_id = p.record_id) AND (e.index_tag = 'b') AND (e.varfield_type_code = 'b')

	GROUP BY
	e.record_id

	HAVING
	count(*) > 1

	-- ORDER BY
	-- count(*)
)
```


## Find duplicate patrons by name + birthdate + patron code

sample output
```csv
2012-07-18 21:30:10-04;12345678;p1234567a;Patron, John Q 1977-01-01;0;2016-08-18 16:11:55-04;2019-05-10 04:00:00-04;c;$145.78
2015-01-07 13:41:06-05;12345679;p7654321a;Patron, John Q 1977-01-01;0;2016-08-21 15:04:40-04;2020-04-10 04:00:00-04;c;$214.53
```

```sql
SELECT
r.creation_date_gmt as created,
e.index_entry as barcode,
'p' || r.record_num || 'a' as patron_record_num,
pn.last_name || ', ' ||pn.first_name || COALESCE(' ' || NULLIF(pn.middle_name, ''), '') || ' ' || pr.birth_date_gmt as patron,
pr.ptype_code,
pr.activity_gmt,
pr.expiration_date_gmt,
pr.mblock_code as block_code,
pr.owed_amt::float8::numeric::money as owed_amt
-- pr.home_library_code,

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
	patron_name,
	p.ptype_code
	-- e.index_entry

	HAVING
	COUNT(*) > 1
)

ORDER BY
pn.last_name || pn.first_name || pr.birth_date_gmt || COALESCE(' ' || NULLIF(pn.middle_name, ''), ''),
pr.ptype_code ASC,
pr.activity_gmt DESC
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