# Find specific item types in specific locations and group them by callnumber classes (work in progress)
```sql
﻿drop table if exists temp_search1;
drop table if exists temp_search2;
drop table if exists temp_search3;

create temp table temp_search1 AS
SELECT
-- i.location_code,
i.record_id as item_record_id,
l.bib_record_id,
(
	SELECT
-- 	string_agg(v.field_content, ',' order by v.occ_num)

-- 	v.field_content

	regexp_matches(
		regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'i'), -- get the call number strip the subfield indicators
		'(^[0-9]{3})'
	) as call_number

-- 	regexp_matches(v.field_content,'[0-9]{3,}') as call_number

-- 	regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'i') -- get the call number strip the subfield indicators

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = l.bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY v.occ_num

	LIMIT 1
)[1] as call_class
-- count(i.record_id)

-- count(i.record_id)
-- ,
-- NOW()::timestamp - r.creation_date_gmt::timestamp as age,
-- NOW()::timestamp - i.last_checkin_gmt::timestamp as time_since_last_checkin

FROM
sierra_view.item_record as i

JOIN
sierra_view.record_metadata as r
ON
   r.id = i.record_id

LEFT OUTER JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.item_record_id = r.id


LEFT OUTER JOIN sierra_view.checkout AS c ON
   i.id = c.item_record_id



WHERE
-- call_class is null
i.itype_code_num IN (0,157)
AND i.item_status_code IN 
-- ('o')
('-') 

-- DATE
-- commnet to "--/DATE" to get the entire group number
-- AND c.due_gmt is null 
-- AND NOW()::timestamp - r.creation_date_gmt::timestamp > INTERVAL '2 years'
-- AND (
--      NOW()::timestamp - i.last_checkin_gmt::timestamp > INTERVAL '2 years'
--      OR i.last_checkin_gmt is null
-- )
--/DATE


AND i.location_code IN (
     -- locations are anything at main
     SELECT
     code
--     , name

     FROM
     sierra_view.location_myuser as l

     WHERE
--     lower(l.code) ~ lower('((2ra.{0,})|3(ra|aa|ha|la).{0,})')
--      lower(l.code) ~ lower('((2ra.{0,})|(3ra)|(2ea)|(2ga)|(2sa)|(3aa)|(3ha)|(3la))')
	lower(l.code) ~ lower('((2ra.{0,})|(3ra))') -- matches all of 2ra* and 3ra
-- 	lower(l.code) ~ lower('((2ra.{0,}))') -- matches all of 2ra*
-- 	l.code = '2rabu'
)

-- group by
-- -- i.location_code,
-- call_class

-- i.location_code,

-- -- i.record_id,
-- i.location_code

-- ORDER BY
-- i.location_code,
-- call_class
-- i.location_code

;



------------------------------------------------------

create temp table temp_search2 AS
SELECT
-- i.location_code,
i.record_id as item_record_id,
l.bib_record_id,
(
	SELECT
-- 	string_agg(v.field_content, ',' order by v.occ_num)

-- 	v.field_content

	regexp_matches(
		regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'i'), -- get the call number strip the subfield indicators
		'(^[0-9]{3})'
	) as call_number

-- 	regexp_matches(v.field_content,'[0-9]{3,}') as call_number

-- 	regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'i') -- get the call number strip the subfield indicators

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = l.bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY v.occ_num

	LIMIT 1
)[1] as call_class
-- count(i.record_id)

-- count(i.record_id)
-- ,
-- NOW()::timestamp - r.creation_date_gmt::timestamp as age,
-- NOW()::timestamp - i.last_checkin_gmt::timestamp as time_since_last_checkin

FROM
sierra_view.item_record as i

JOIN
sierra_view.record_metadata as r
ON
   r.id = i.record_id

LEFT OUTER JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.item_record_id = r.id


LEFT OUTER JOIN sierra_view.checkout AS c ON
   i.id = c.item_record_id



WHERE
-- call_class is null
i.itype_code_num IN (0,157)
AND i.item_status_code IN 
-- ('o')
('-', 't', '!') 

-- DATE
-- commnet to "--/DATE" to get the entire group number
AND c.due_gmt is null 
AND NOW()::timestamp - r.creation_date_gmt::timestamp > INTERVAL '2 years'
AND (
     NOW()::timestamp - i.last_checkin_gmt::timestamp > INTERVAL '2 years'
     OR i.last_checkin_gmt is null
)
--/DATE


AND i.location_code IN (
     -- locations are anything at main
     SELECT
     code
--     , name

     FROM
     sierra_view.location_myuser as l

     WHERE
--     lower(l.code) ~ lower('((2ra.{0,})|3(ra|aa|ha|la).{0,})')
--      lower(l.code) ~ lower('((2ra.{0,})|(3ra)|(2ea)|(2ga)|(2sa)|(3aa)|(3ha)|(3la))')
	lower(l.code) ~ lower('((2ra.{0,})|(3ra))') -- matches all of 2ra* and 3ra
-- 	lower(l.code) ~ lower('((2ra.{0,}))') -- matches all of 2ra*
-- 	l.code = '2rabu'
)

-- group by
-- -- i.location_code,
-- call_class

-- i.location_code,

-- -- i.record_id,
-- i.location_code

-- ORDER BY
-- i.location_code,
-- call_class
-- i.location_code

;

------------------------------------------------------

-- SELECT
-- *
-- from
-- temp_search limit 100



-- SELECT
-- t.call_class,
-- count(t.item_record_id)
-- 
-- FROM
-- temp_search as t
-- 
-- GROUP BY
-- t.call_class
-- 
-- ORDER BY
-- t.call_class;

create temp table temp_search3 AS
SELECT
t1.call_class

from
temp_search1 as t1

UNION

SELECT
t2.call_class

FROM
temp_search2 as t2;



-- to produce the final data for the report, we want to run this one, and then run the next query, to produce our final results
-- NOTE: the items that don't have classifications will need to be queired seperately ... (count (*) )
--- query 1
-- SELECT
-- t3.call_class,
-- COUNT(t1.item_record_id) as count_items_search1
-- 
-- FROM
-- temp_search3 as t3
-- 
-- LEFT OUTER JOIN
-- temp_search1 as t1
-- ON
--   t1.call_class = t3.call_class
-- 
-- GROUP BY
-- t3.call_class
-- 
-- ORDER BY
-- t3.call_class
-- 
-- SELECT
-- count(*)
-- from temp_search1
-- 
-- where
-- call_class is null


--- query 2
-- SELECT
-- t3.call_class,
-- COUNT(t1.item_record_id) as count_items_search1
-- 
-- FROM
-- temp_search3 as t3
-- 
-- LEFT OUTER JOIN
-- temp_search1 as t1
-- ON
--   t1.call_class = t3.call_class
-- 
-- GROUP BY
-- t3.call_class
-- 
-- ORDER BY
-- t3.call_class
-- 
-- SELECT
-- count(*)
-- from temp_search1
-- 
-- where
-- call_class is null
```


# Find numbers of deleted records (by record type code) in a given range of time
```sql
SELECT
r.record_type_code,
count(*)

FROM
sierra_view.record_metadata as r

WHERE
r.campus_code = ''
AND r.deletion_date_gmt >= '2017-01-01'
AND r.deletion_date_gmt < '2018-01-01'

GROUP BY
r.record_type_code
```

# Find numbers of created records (by record type code) in a given range of time
```
SELECT
r.record_type_code,
count(*)

FROM
sierra_view.record_metadata as r

WHERE
r.campus_code = ''
AND r.creation_date_gmt >= '2017-01-01'
AND r.creation_date_gmt < '2018-01-01'

GROUP BY
r.record_type_code

ORDER BY
r.record_type_code ASC
```

# Extract the last X amount of transactions
```sql
SELECT
md5(r.record_type_code || r.record_num || 'a' || '1434') as hash_patron_record_num,
c.transaction_gmt,
p.ptype_code,
br.bcode2,
ir.itype_code_num,
ir.agency_code_num as item_agency_code_num,
c.id,

c.application_name,
-- c.op_code,
CASE 
  WHEN c.op_code = 'o' THEN 'checkout'
  WHEN c.op_code = 'i' THEN 'checkin'
  WHEN c.op_code = 'n' THEN 'hold'
  WHEN c.op_code = 'h' THEN 'hold w/ recall'
  WHEN c.op_code = 'nb' THEN 'bib lv hold'
  WHEN c.op_code = 'hb' THEN 'bib lv hold recall'
  WHEN c.op_code = 'ni' THEN 'item lv hold'
  WHEN c.op_code = 'hi' THEN 'item lv hold recall'
  WHEN c.op_code = 'nv' THEN 'vol lv hold'
  WHEN c.op_code = 'hv' THEN 'vol lv hold recall'
  WHEN c.op_code = 'f' THEN 'filled hold'
  WHEN c.op_code = 'r' THEN 'renewal'
  WHEN c.op_code = 'b' THEN 'booking'
  WHEN c.op_code = 'u' THEN 'use count'
END AS op_code_type,
c.stat_group_code_num,
n.name as stat_group_name,
ir.checkout_statistic_group_code_num,
ni.name as checkout_statistic_group_name,
-- c.stat_group_code_num,
c.bib_record_id,
c.item_record_id,
c.volume_record_id

FROM
sierra_view.circ_trans	as c

-- link stat_group_code_num to name
JOIN
sierra_view.statistic_group as g
ON
  g.code_num = c.stat_group_code_num
JOIN
sierra_view.statistic_group_name as n
ON
  n.statistic_group_id = g.id
-- /link stat_group_code_num to name

JOIN
sierra_view.patron_record as p
ON
  p.record_id = c.patron_record_id

JOIN
sierra_view.record_metadata as r
ON
  r.id = p.record_id

-- LEFT OUTER JOIN
-- sierra_view.bib_record_property as b
-- ON
--   b.bib_record_id = c.bib_record_id

LEFT OUTER JOIN
sierra_view.bib_record as br
ON
  br.record_id = c.bib_record_id

LEFT OUTER JOIN
sierra_view.item_record as ir
ON
  ir.record_id = c.item_record_id

-- link stat_group_code_num to name
LEFT OUTER JOIN
sierra_view.statistic_group as gi
ON
--   g.code_num = c.stat_group_code_num
  gi.code_num = ir.checkout_statistic_group_code_num
LEFT OUTER JOIN
sierra_view.statistic_group_name as ni
ON
  ni.statistic_group_id = gi.id
-- /link stat_group_code_num to name


ORDER BY
c.id DESC

limit 2000;
```