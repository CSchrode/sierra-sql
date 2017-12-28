# Checkout totals by top bib records having items with specific item types
```sql
-- 
-- This query will find the top entries for a title given attached item types.
-- To produce output, comment and uncomment the item types in the first where clause
-- * note, that changing the call number search in the second query on the temp table limits to fiction vs non-fiction

DROP TABLE IF EXISTS temp_checkouts_ytd;
CREATE TEMP TABLE temp_checkouts_ytd AS

SELECT 
SUM ( i.last_year_to_date_checkout_total ) as sum_checkouts_ytd,
l.bib_record_id,
(
	SELECT
	regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig') as call_number -- get the call number strip the subfield indicators

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = l.bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY
	v.occ_num

	LIMIT 1
) as call_number

-- bib_view.record_num,
-- bib_view.title,
--varfield.field_content as callnum

FROM
sierra_view.item_record as i

JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.item_record_id = i.record_id

WHERE
--Call Number:
--AND sierra_view.varfield.record_id = sierra_view.bib_view.id 
--AND sierra_view.varfield.record_type_code = 'b' 
--AND sierra_view.varfield.varfield_type_code = 'c' 
--AND field_content ~* '.*fiction.*'  --Fiction
--AND field_content ~ '^\|a(?:\d\d|M)\d'  --Non-fiction
 
--Item Type: 
-- un-comment for specific output
--AND i.itype_code_num NOT IN ( 30, 31, 32, 33, 34, 35, 37, 136 )  --Overall (no magazines)
i.itype_code_num IN ( 0, 1, 20, 21, 46 )  --Adult Books
--AND i.itype_code_num IN ( 4, 5 )  --Teen Books
--AND i.itype_code_num IN ( 2, 3 )  --Juvenile Books  
--AND i.itype_code_num IN ( 70, 60, 77, 90, 93, 100, 101, 82, 65, 130, 134, 163 )  --Adult AV
--AND i.itype_code_num IN ( 72, 92 ) --Teen AV
--AND i.itype_code_num IN ( 71, 61, 66, 78, 91 ) --Juvenile AV
  
--AND i.itype_code_num IN ( 30 ) --Adult Magazine
--AND i.itype_code_num IN ( 32 ) --Teen Magazine
--AND i.itype_code_num IN ( 31 ) --Juvenile Magazine

--AND i.itype_code_num IN ( 70, 71, 72, 73 ) --Audiobooks on CD
--AND i.itype_code_num IN ( 71 ) --Juvenile Audiobooks on CD
--AND i.itype_code_num IN ( 72 ) --Teen Audiobooks on CD

--AND i.itype_code_num IN ( 77, 78, 79 ) --Music on CD
--AND i.itype_code_num IN ( 78 ) --Juvenile Music on CD

--AND i.itype_code_num IN ( 20, 21, 22, 23, 24, 26, 27 ) --Large Print Books

--AND i.itype_code_num IN ( 90, 91, 92, 93, 94 ) --Playaways
--AND i.itype_code_num IN ( 91 ) --Juvenile Playaways
--AND i.itype_code_num IN ( 92 ) --Teen Playaways

--AND i.itype_code_num IN ( 100, 101 ) --Video
  
GROUP BY
l.bib_record_id,
call_number

HAVING
SUM ( i.last_year_to_date_checkout_total ) > 0
;
---


CREATE INDEX index_sum_checkouts_ytd ON temp_checkouts_ytd (sum_checkouts_ytd);
---


SELECT 
t.sum_checkouts_ytd,
p.best_title,
r.record_num,
null as null,
*

FROM
temp_checkouts_ytd as t

JOIN
sierra_view.record_metadata as r
ON
  r.id = t.bib_record_id

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = t.bib_record_id

WHERE
-- t.call_number ~* '.*fiction.*' -- fiction
t.call_number !~* '.*fiction.*' -- non fiction

ORDER BY
t.sum_checkouts_ytd DESC

limit 200
;
```