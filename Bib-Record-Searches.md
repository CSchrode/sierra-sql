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