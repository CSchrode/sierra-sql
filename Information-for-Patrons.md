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