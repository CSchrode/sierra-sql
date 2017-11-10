## Possible new books list (work in progress)
```sql
DROP TABLE IF EXISTS temp_new_bibs;

CREATE TEMP TABLE temp_new_bibs AS
SELECT
r.id as bib_record_id,
r.record_type_code || r.record_num || 'a' as bib_record_num,
r.creation_date_gmt::date,
b.language_code,
b.bcode1,
b.bcode2,
b.country_code,
b.cataloging_date_gmt::date,
p.best_title,
p.best_title_norm,
p.bib_level_code,
p.material_code,
m.name as material_code_name,
p.publish_year,
p.best_author,
p.best_author_norm,
(
	SELECT
-- 	v.field_content as callnumber
	regexp_matches(
		regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig'), -- get the call number strip the subfield indicators
		-- looking for :
		-- groups where 3 or more numbers in a row (dewey)
		-- words "fiction", "easy", or "other" appear anywhere in the string
		-- two or more letters appear in succession
		'((^[0-9]{3})|(^m.*)|(fiction.*)|(easy.*)|(other.*)|([a-z]{2,}))', 
		'gi'
	) AS call_number
	
	FROM
	sierra_view.varfield as v
	WHERE
	-- get the call number from the bib record
	v.record_id = bib_record_id
	AND v.varfield_type_code = 'c'
	ORDER BY 
	v.occ_num
	LIMIT 1
)[1] AS call_class


FROM
sierra_view.record_metadata AS r

JOIN
sierra_view.bib_record AS b
ON
  b.record_id = r.id

LEFT OUTER JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = r.id

LEFT OUTER JOIN
sierra_view.material_property_myuser as m
ON
  m.code = b.bcode2

WHERE
r.record_type_code || r.campus_code = 'b'
AND b.cataloging_date_gmt IS NOT NULL
AND b.is_suppressed IS FALSE
AND b.cataloging_date_gmt::timestamp >= '2017-01-01'::timestamp
AND b.cataloging_date_gmt::timestamp < '2017-02-01'::timestamp
AND b.bcode1 = 'm'
;
---

ANALYSE temp_new_bibs;
---


SELECT
*
FROM
temp_new_bibs
```

## Possible enhanced Courtesy Notice
This will show the list of titles due for patrons in two days. It will also compile a list of all other titles that are still due (including ones that are past due)
```sql
SELECT
c.patron_record_id,
(
	SELECT
	array_agg(p.best_title || ' [DUE:] ' || c_now.due_gmt ORDER BY c_now.due_gmt ASC, p.best_title ASC) AS titles_due_soon

	FROM
	sierra_view.checkout as c_now
	
	LEFT OUTER JOIN
	sierra_view.bib_record_item_record_link as l
	ON
	l.item_record_id = c_now.item_record_id

	LEFT OUTER JOIN
	sierra_view.bib_record_property as p
	ON
	  p.bib_record_id = l.bib_record_id

	WHERE
	c_now.patron_record_id = c.patron_record_id
	AND date_trunc( 'day', NOW() ) + interval '2 day' = date_trunc('day', c_now.due_gmt)
) as titles_due_soon,

(
	SELECT
	array_agg(p.best_title || ' [DUE:] ' || c_now.due_gmt ORDER BY c_now.due_gmt ASC, p.best_title ASC) AS titles_due_soon

	FROM
	sierra_view.checkout as c_now
	
	LEFT OUTER JOIN
	sierra_view.bib_record_item_record_link as l
	ON
	l.item_record_id = c_now.item_record_id

	LEFT OUTER JOIN
	sierra_view.bib_record_property as p
	ON
	  p.bib_record_id = l.bib_record_id

	WHERE
	c_now.patron_record_id = c.patron_record_id
	AND date_trunc( 'day', NOW() ) + interval '2 day' != date_trunc('day', c_now.due_gmt)
) as other_titles_due

FROM
sierra_view.checkout as c

WHERE
c.patron_record_id IN
(
	-- list of patrons where item(s) are due in two days from today's date
	SELECT
	r.id
	-- r.record_type_code || r.record_num || 'a' as patron_record_num

	FROM
	sierra_view.checkout as c

	JOIN
	sierra_view.record_metadata as r
	ON
	  (r.id = c.patron_record_id) 
	  AND (r.record_type_code = 'p') 
	  AND (r.campus_code = '')

	WHERE
	date_trunc( 'day', NOW() ) + interval '2 day' = date_trunc('day', c.due_gmt)

	GROUP BY
	r.id
	-- r.record_type_code,
	-- r.record_num
)

GROUP BY
c.patron_record_id
```