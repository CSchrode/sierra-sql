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