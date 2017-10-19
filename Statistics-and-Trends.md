# "Long Tail" - Items from main that have not circulated, or have only circulated more than 2 years ago, grouped by publication year -- combined with the inverse of that ... items that have circulated <= 2 years ago
```sql
drop table if exists temp_circs_at_main;
drop table if exists temp_old_circs_at_main;


CREATE TEMP TABLE temp_circs_at_main AS
SELECT
p.publish_year,
count(i.record_id) as count

FROM
sierra_view.item_record as i

JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.item_record_id = i.record_id

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = l.bib_record_id

LEFT OUTER JOIN
sierra_view.checkout as c
ON
  c.item_record_id = i.record_id
  
WHERE
-- item has not circulated in two year, or circulated at all
(
	age(NOW()::timestamp, i.last_checkin_gmt::timestamp) <= INTERVAL '2 years'
-- 	OR i.last_checkin_gmt IS NULL
)

-- item is not currently checked out
AND 
c.item_record_id IS NULL

-- item has a status of '-'
AND i.item_status_code = '-'

AND i.itype_code_num IN (0, 157)

-- and is in locations starting with '2ra' or are in locations: '3ra', '2ea', '2ga', '2sa', '3aa', '3ha', '3la'
AND i.location_code IN (
	SELECT
	l.code

	FROM
	sierra_view.location_myuser as l

	WHERE
	l.code ~* '^2ra.*'
	OR LOWER(l.code) IN (
		'3ra',
		'2ea',
		'2ga',
		'2sa',
		'3aa',
		'3ha',
		'3la'
	)
)

GROUP BY
p.publish_year;
---


CREATE TEMP TABLE temp_old_circs_at_main AS
SELECT
p.publish_year,
count(i.record_id) as count

FROM
sierra_view.item_record as i

JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.item_record_id = i.record_id

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = l.bib_record_id

LEFT OUTER JOIN
sierra_view.checkout as c
ON
  c.item_record_id = i.record_id
  
WHERE
-- item has not circulated in two year, or circulated at all
(
	age(NOW()::timestamp, i.last_checkin_gmt::timestamp) > INTERVAL '2 years'
	OR i.last_checkin_gmt IS NULL
)

-- item is not currently checked out
AND 
c.item_record_id IS NULL

-- item has a status of '-'
AND i.item_status_code = '-'

AND i.itype_code_num IN (0, 157)

-- and is in locations starting with '2ra' or are in locations: '3ra', '2ea', '2ga', '2sa', '3aa', '3ha', '3la'
AND i.location_code IN (
	SELECT
	l.code

	FROM
	sierra_view.location_myuser as l

	WHERE
	l.code ~* '^2ra.*'
	OR LOWER(l.code) IN (
		'3ra',
		'2ea',
		'2ga',
		'2sa',
		'3aa',
		'3ha',
		'3la'
	)
)

GROUP BY
p.publish_year;
---


-- SELECT * from temp_old_circs_at_main;
-- SELECT * from temp_circs_at_main;


DROP TABLE IF EXISTS temp_combined_pub_years;

CREATE TEMP TABLE temp_combined_pub_years (
id SERIAL NOT NULL,
publish_year INTEGER
);

INSERT INTO 
temp_combined_pub_years (publish_year)

SELECT
o.publish_year

FROM
temp_old_circs_at_main as o;
---

INSERT INTO 
temp_combined_pub_years (publish_year)
(
	SELECT
	o.publish_year

	FROM
	temp_circs_at_main as o
);
--

CREATE TEMP TABLE temp_combined_pub_years_unique AS
SELECT
t.publish_year

FROM
temp_combined_pub_years as t

GROUP BY
t.publish_year;
---

SELECT
t.publish_year,
c.count as recent_circs,
oc.count as old_and_non_circs

FROM
temp_combined_pub_years_unique as t

LEFT OUTER JOIN
temp_circs_at_main as c
ON
  c.publish_year = t.publish_year

LEFT OUTER JOIN
temp_old_circs_at_main as oc
ON
  oc.publish_year = t.publish_year;
```

# Last Week's Circulations from main branch, grouped by call number class (work in progress)
```sql
DROP TABLE IF EXISTS temp_call_groups;
CREATE TEMP TABLE temp_call_groups (
id SERIAL NOT NULL,
group_value VARCHAR(12),
group_name VARCHAR(512)
);

INSERT INTO temp_call_groups (group_value, group_name) VALUES 
('000', 'Computer science, information & general works'),
('100', 'Philosophy & psychology'),
('200', 'Religion'),
('300', 'Social sciences'),
('400', 'Language'),
('500', 'Science'),
('600', 'Technology'),
('700', 'Arts & recreation'),
('800', 'Literature'),
('900', 'History & geography'),
('fiction', 'PLCH fiction'),
('m','PLCH music score'),
('easy', 'PLCH easy'),
('other', 'PLCH other'),
('', 'PLCH none')
;

-- select * from temp_call_groups;

DROP TABLE IF EXISTS temp_circs_from_main;
CREATE TEMP TABLE temp_circs_from_main AS
SELECT
extract (year from t.transaction_gmt) || 'W' || extract(week from t.transaction_gmt) as transaction_week,
(
	SELECT
-- 	v.field_content as callnumber
	regexp_matches(
		regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'i'), -- get the call number strip the subfield indicators
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
	v.record_id = l.bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY 
	v.occ_num

	LIMIT 1
)[1] AS call_class,
count(t.item_record_id)

FROM
sierra_view.circ_trans AS t

LEFT OUTER JOIN
sierra_view.item_record AS i
ON
  i.record_id = t.item_record_id

LEFT OUTER JOIN
sierra_view.bib_record_item_record_link AS l
ON
  l.item_record_id = i.record_id

WHERE
-- transactions from last week
extract(week from t.transaction_gmt) = extract (week from (NOW() - interval '1 week'))

-- type of transaction is checkout 'o'
AND t.op_code = 'o'

-- item type is book (0) or music score (157)
AND i.itype_code_num IN (0,157)

-- item type is Juvenile Books (2, 22, 159)
-- item type is Teen Books (4, 24)
-- AND i.itype_code_num IN (2, 22, 159, 4, 24)

-- from all Main locations
-- to get an updated list check here ...
-- https://github.com/plch/sierra-sql/wiki/Info#find-stat-group-aka-terminal-number-statistical-group-code-number-and-name
AND t.stat_group_code_num IN (
0,
1,
9,
10,
11,
12,
13,
14,
21,
31,
41,
42,
43,
44,
51,
61,
62
)

GROUP BY
call_class,
transaction_week;

-- FULL OUTER JOIN two temp tables to get all values to our final counts

SELECT 
(
	SELECT
	extract(year from NOW()) || 'W' || extract(week from (NOW() - interval '1 week'))
) as transaction_week,
* 

FROM 
(
	SELECT 
	g.group_value,
	g.group_name

	FROM
	temp_call_groups as g
) as t1

FULL OUTER JOIN
(
	SELECT 
	CASE 	
		WHEN m.call_class ~ '^[0-9]{3}' THEN substring(m.call_class from 1 for 1) || '00'
		WHEN m.call_class ~* '^m[0-9]{1}.*' THEN lower(substring(m.call_class from 1 for 1))
		ELSE lower(m.call_class)
	END as class,
	SUM(m.count)

	FROM 
	temp_circs_from_main AS m

	GROUP BY
	class
) AS t2
ON t2.class = t1.group_value

ORDER BY
t1.group_value,
t2.class





```



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