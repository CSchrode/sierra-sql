## Find titles with a cat date within the last 60 days
```sql
DROP TABLE IF EXISTS temp_new_titles;

CREATE TEMP TABLE temp_new_titles AS
SELECT
r.id as bib_record_id,
p.bib_level_code,
p.material_code,
p.best_title as title,
p.best_author as author,
r.creation_date_gmt::date as creation_date,
r.record_type_code || r.record_num || 'a' as record_num,
'http://catalog.cincinnatilibrary.org/iii/encore/record/C__R' 
	|| r.record_type_code
	|| r.record_num AS encore_link,
b.cataloging_date_gmt::date as cat_date,
p.best_title_norm as title_norm,
p.publish_year


FROM
sierra_view.record_metadata as r

JOIN
sierra_view.bib_record as b
ON
  b.record_id = r.id

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = r.id

WHERE
r.record_type_code || r.campus_code = 'b'
AND r.deletion_date_gmt IS NULL
AND b.is_suppressed IS FALSE
AND age(b.cataloging_date_gmt) <= '60 days'::INTERVAL
;

CREATE INDEX sort_temp_new_titles ON temp_new_titles (bib_level_code, cat_date, title_norm)
;

ANALYZE temp_new_titles
;

SELECT
* 

FROM
temp_new_titles as t

ORDER BY
t.bib_level_code, 
t.cat_date,
t.title_norm
;
```


## Find Bib and Volume records with a cat date in the last week
```sql
-- example of grabbing bib and volume records with a cat date in the last week,
-- and creating a temp table from the results for use with other query output
-- (volume records don't have a cat date, so checking the linked bib)

-- the example interval
-- SELECT
-- (NOW()::date - INTERVAL '1 week')::date AS week_ago,
-- NOW()::date AS today

DROP TABLE IF EXISTS temp_bib_vol_with_cat_date_today;

CREATE TEMP TABLE temp_bib_vol_with_cat_date_today AS
SELECT
l.bib_record_id,
l.volume_record_id,
b.cataloging_date_gmt

FROM
sierra_view.bib_record_volume_record_link AS l

JOIN
sierra_view.bib_record AS b
ON
  b.record_id = l.bib_record_id

WHERE
b.cataloging_date_gmt::date BETWEEN 
	(NOW()::date - INTERVAL '1 week')::date 
	AND NOW()::date

UNION 

SELECT
b.record_id,
l.volume_record_id,
b.cataloging_date_gmt

FROM
sierra_view.bib_record AS b

LEFT OUTER JOIN
sierra_view.bib_record_volume_record_link AS l
ON
  l.bib_record_id = b.record_id

WHERE
l.volume_record_id IS NULL
AND b.cataloging_date_gmt::date BETWEEN 
	(NOW()::date - INTERVAL '1 week')::date 
	AND NOW()::date
;
---


CREATE INDEX temp_bib_vol_with_cat_date_today_bib ON temp_bib_vol_with_cat_date_today (bib_record_id);
CREATE INDEX temp_bib_vol_with_cat_date_today_vol ON temp_bib_vol_with_cat_date_today (volume_record_id);
CREATE INDEX temp_bib_vol_with_cat_date_today_cat_date ON temp_bib_vol_with_cat_date_today (cataloging_date_gmt);

-- sample query to test what we get back
-- SELECT id2reckey(bib_record_id), * FROM temp_bib_vol_with_cat_date_today order by bib_record_id, volume_record_id, cataloging_date_gmt;
```



## Bib Records With:
* **a cataloging date**
* **all attached items are suppressed**
* **no holds on bib**
* **no holds on attached items**
* **no active orders** :
   * (`order_record_status` not equal to `o`)

```sql

---
-- Find bib records where all attached items are suppressed
---
DROP TABLE IF EXISTS temp_bibs_all_attached_suppressed;
---
CREATE TEMP TABLE temp_bibs_all_attached_suppressed AS
SELECT
l.bib_record_id
-- r.record_type_code || r.record_num || 'a' as bib_record_num

FROM
sierra_view.bib_record_item_record_link as l

JOIN
sierra_view.record_metadata as r
ON
  r.id = l.bib_record_id

JOIN
sierra_view.bib_record as b
ON
  b.record_id = l.bib_record_id

LEFT OUTER JOIN
sierra_view.bib_record_order_record_link as ol
ON
  ol.bib_record_id = l.bib_record_id

LEFT OUTER JOIN
sierra_view.order_record as o
ON
  o.record_id = ol.order_record_id

WHERE
b.cataloging_date_gmt IS NOT NULL
AND b.is_suppressed IS FALSE
AND o.order_status_code != 'o'
AND l.bib_record_id NOT IN
(
	SELECT
	l_one.bib_record_id

	FROM
	sierra_view.bib_record_item_record_link as l_one

	JOIN
	sierra_view.item_record as i_one
	ON
	  i_one.record_id = l_one.item_record_id
	  AND i_one.is_suppressed IS FALSE

	GROUP BY
	l_one.bib_record_id
)

GROUP BY
l.bib_record_id
;
---

CREATE INDEX index_bib_record_id ON temp_bibs_all_attached_suppressed (bib_record_id);
---


-- Look for bib records that have holds, and remove them from the list
DELETE FROM 
temp_bibs_all_attached_suppressed

WHERE
bib_record_id IN (

	SELECT
	t.bib_record_id

	FROM
	temp_bibs_all_attached_suppressed as t

	LEFT OUTER JOIN
	sierra_view.hold as h
	ON
	  t.bib_record_id = h.record_id

	WHERE
	h.record_id IS NOT NULL
);
---


-- Look for bib records that have attached items that have holds, and remove them from the list
DELETE FROM 
temp_bibs_all_attached_suppressed

WHERE
bib_record_id IN (

	SELECT
	t.bib_record_id

	FROM
	temp_bibs_all_attached_suppressed as t

	JOIN
	sierra_view.bib_record_item_record_link as l
	ON
	  l.bib_record_id = t.bib_record_id

	LEFT OUTER JOIN
	sierra_view.hold as h
	ON
	  l.item_record_id = h.record_id

	WHERE
	h.record_id IS NOT NULL

	GROUP BY
	t.bib_record_id
);
---


SELECT
r.record_type_code || r.record_num || 'a' as record_num,
r.record_last_updated_gmt,
p.bib_level_code,
p.material_code,
p.best_title,
p.best_author,
p.publish_year

FROM
temp_bibs_all_attached_suppressed as t

JOIN
sierra_view.record_metadata as r
ON
  r.id = t.bib_record_id

LEFT OUTER JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = t.bib_record_id

ORDER BY
r.record_last_updated_gmt
```



## Find bib records with more than 1 attached item record
```sql
﻿DROP TABLE IF EXISTS temp_bib_records;
CREATE TEMP TABLE temp_bib_records AS
SELECT
l.bib_record_id,
count(l.bib_record_id) as count_linked_items

FROM
sierra_view.bib_record_item_record_link as l

GROUP BY
l.bib_record_id

HAVING
count(l.bib_record_id) > 1;
---
3

CREATE INDEX index_bib_record_id ON temp_bib_records (bib_record_id);
---


-- now we can do joins on our temp table to get what we need out of it ...
SELECT
*

FROM
temp_bib_records as t

JOIN
sierra_view.bib_record as b
ON
  b.record_id = t.bib_record_id

LIMIT 100;
---
```



### Find bib records with more than 1 attached item record, including the location code count of items
```sql
DROP TABLE IF EXISTS temp_item_record_location_count;
CREATE TEMP TABLE temp_item_record_location_count AS
SELECT
l.bib_record_id,
i.location_code,
count(i.location_code) as count_by_location_code

FROM
sierra_view.bib_record_item_record_link as l

JOIN
sierra_view.item_record as i
ON
  i.record_id = l.item_record_id

GROUP BY
l.bib_record_id,
i.location_code

HAVING
count(i.location_code) > 1;
---


CREATE INDEX index_location_code on temp_item_record_location_count (location_code);
CREATE INDEX index_bib_record_id on temp_item_record_location_count (bib_record_id);
---


-- now we can do our searches on the temp table by location code
SELECT
*
FROM
temp_item_record_location_count AS t

WHERE
t.location_code = '2ea'

LIMIT 100
```


### Find ISBNs existing in the ILS from a list provided.
```sql
DROP TABLE IF EXISTS temp_search_isbns;
CREATE TEMP TABLE temp_search_isbns (
	id SERIAL NOT NULL,
	isbn VARCHAR(30)
);
---


-- here we insert the ISBNs we want to search on. This list can be produced my importing a .csv file, and 
-- replacing \n (newline characters with '\')\n(\'' values to get the input into the format like the example below
INSERT INTO temp_search_isbns (isbn) VALUES
('076141102X'),
('9781683900962'),
('1683900960'),
('9780531268360'),
('9780553213119'),
('085478473X');

CREATE INDEX index_isbn ON temp_search_isbns (isbn);
---


DROP TABLE IF EXISTS temp_isbn;
CREATE TEMP TABLE temp_isbn AS
SELECT
v.id,
(
	-- performing subquery so that we can return one result for our extracted isbn
	SELECT
	regexp_matches(
		--regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig'), -- get the call number strip the subfield indicators
		v.field_content,
		'[0-9]{9,10}[x]{0,1}|[0-9]{12,13}[x]{0,1}', -- the regex to match on (10 or 13 digits, with the possibility of the 'X' character in the check-digit spot)
		'i' -- regex flags; ignore case
	)
	FROM
	sierra_view.varfield as v1

	WHERE
	v1.record_id = v.record_id

	LIMIT 1
)[1]::varchar(30) as isbn_extracted,
v.field_content, -- selecting this so we can spot check our extracted number against the field contents 
v.record_id 

FROM
sierra_view.varfield as v

WHERE
v.marc_tag || v.varfield_type_code = '020i' -- indexed as isbn and comes from the marc 020 field
;
---


-- add an index on the isbn number so we can more quicky match on it.
CREATE INDEX index_isbn_extracted ON temp_isbn (isbn_extracted);
---


-- run our search!
SELECT
*
FROM
temp_search_isbns as s

LEFT OUTER JOIN
temp_isbn as t
ON
  LOWER(s.isbn) = LOWER(t.isbn_extracted) -- make sure that the check digit 'x' is matched despite case

LEFT OUTER JOIN
sierra_view.record_metadata as r
ON
  r.id = t.record_id

LEFT OUTER JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = r.id
;
---

```



### Find Bib records where only one item is attached, the item is available, not checked out, and is not part of specific locations.
```sql
-- create a temp table with the bibs having one item record
DROP TABLE IF EXISTS temp_bibs_one_item;
CREATE TEMP TABLE temp_bibs_one_item AS
SELECT
l.bib_record_id

FROM
sierra_view.bib_record_item_record_link as l

GROUP BY
l.bib_record_id

HAVING
count(l.bib_record_id) = 1
;
---


---
-- remove the bib ids from the temp table that are not our bib records
DELETE 

FROM
temp_bibs_one_item as b

WHERE
b.bib_record_id IN
(
	SELECT
	b.bib_record_id

	FROM
	temp_bibs_one_item as b
	
	JOIN
	sierra_view.record_metadata as r
	ON
	  b.bib_record_id = r.id

	WHERE
	r.campus_code != ''
)
;
---


---
-- search the bibs attached item and see if it has a status of '-' and is not checked out
SELECT
r.record_type_code || r.record_num || 'a' as record_num

FROM
sierra_view.bib_record_item_record_link as l1

JOIN
sierra_view.record_metadata as r
ON
  r.id = l1.bib_record_id

JOIN
sierra_view.item_record as i
ON
  i.record_id = l1.item_record_id

LEFT OUTER JOIN
sierra_view.checkout as c
ON
  c.item_record_id = l1.item_record_id

WHERE
i.item_status_code = '-' -- item status code available
AND i.location_code NOT IN ('vidow') -- put a comma seperated list of location codes to exclude here ...
AND c.id IS NULL -- item doesn't have a checkout associated with it, and therefore not checked out
AND l1.bib_record_id IN (
	SELECT
	b.bib_record_id
	
	FROM
	temp_bibs_one_item as b
);
```



## Find bib records where all attached items are suppressed
```sql
SELECT
-- l.bib_record_id
r.record_type_code || r.record_num || 'a' as bib_record_num

FROM
sierra_view.bib_record_item_record_link as l

JOIN
sierra_view.record_metadata as r
ON
  r.id = l.bib_record_id

JOIN
sierra_view.bib_record as b
ON
  b.record_id = l.bib_record_id

WHERE
b.is_suppressed IS FALSE
AND l.bib_record_id NOT IN
(
	SELECT
	l_one.bib_record_id

	FROM
	sierra_view.bib_record_item_record_link as l_one

	JOIN
	sierra_view.item_record as i_one
	ON
	  i_one.record_id = l_one.item_record_id 
	  AND i_one.is_suppressed IS FALSE

	GROUP BY
	l_one.bib_record_id
)

GROUP BY
l.bib_record_id,
r.record_type_code,
r.record_num
```


## Find and return bib record information (including multiple varfields) based on local marc 9xx varfield data in the bib record
```sql
SELECT
r.record_type_code || r.record_num || 'a' as bib_record_num,
bp.best_title,
bp.best_author,
-- MARC 250 Edition info
(
	SELECT
	v.field_content

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.marc_tag = '250'

	ORDER BY
	v.occ_num

	LIMIT 1
		
) as marc_250_edition, 

-- MARC 260 publication info
(
	SELECT
	v.field_content

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.marc_tag = '260'

	ORDER BY
	v.occ_num

	LIMIT 1
		
) as marc_260_publication_info,

-- MARC 856 URL data,
(
	SELECT
	array_agg(v.field_content order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.marc_tag = '856'

	GROUP BY
	v.record_id
		
) as marc_856_fields,

-- MARC 956 Local label
(
	SELECT
	array_agg(v.field_content order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.marc_tag = '956'

	GROUP BY
	v.record_id
		
) as marc_956_fields,

-- MARC 996 Local label
(
	SELECT
	-- '[''' || string_agg(v.field_content, ''', ''') || ''']' as marc_996_fields
	array_agg(v.field_content order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.marc_tag = '996'

	GROUP BY
	v.record_id
		
) as marc_996_fields

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.bib_record_property as bp
ON
  bp.bib_record_id = r.id

WHERE
r.id IN
(
	SELECT
	v.record_id

	FROM
	sierra_view.varfield as v

	JOIN
	sierra_view.record_metadata as r
	ON
	  r.id = v.record_id AND r.record_type_code = 'b'

	WHERE
	v.marc_tag = '996'
	AND v.field_content is not null

	GROUP BY
	v.record_id
)
```