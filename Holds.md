## Freaky-holds
“Freaky” Holds
Active holds criteria:
* hold IS either bib 'b' OR volume= 'j' level
* hold.is_ir IS false (not INN-Reach)
* hold.is_ill IS false (not ILL)
* hold.is_frozen IS false -- not frozen hold
* patrons placing the hold has a ptype code IN (0, 1, 2, 3, 5, 6, 10, 11, 12, 15, 22, 30, 31, 32, 40, 41, 196)
* hold age (adjusted for delay_days) IS greater than 30

Title has attached items that have status `-` or (`g` or `m`)
```sql
DROP TABLE IF EXISTS temp_freaky_holds;
CREATE TEMP TABLE temp_freaky_holds AS
SELECT
CASE
-- 	we are not going to look at item level holds as part of this report, but could be useful later on...
-- 	WHEN r.record_type_code = 'i' THEN (
-- 		SELECT
-- 		l.bib_record_id
-- 		FROM
-- 		sierra_view.bib_record_item_record_link as l
-- 		WHERE
-- 		l.item_record_id = h.record_id
-- 		LIMIT 1
-- 	)
	WHEN r.record_type_code = 'j' THEN (
		SELECT
		l.bib_record_id

		FROM
		sierra_view.bib_record_volume_record_link as l

		WHERE
		l.volume_record_id = h.record_id

		LIMIT 1
	)

	WHEN r.record_type_code = 'b' THEN (
		h.record_id
	)
	ELSE NULL
END AS bib_record_id,
h.record_id,
r.record_type_code AS record_type_code,
h.id as hold_id,
h.patron_record_id,
p.ptype_code,
h.placed_gmt,
h.delay_days,
-- age of hold in days
(( extract(epoch FROM age(
	(h.placed_gmt + concat(h.delay_days, ' days')::INTERVAL)
)) / 3600 ) / 24 )::int AS age_days

FROM
sierra_view.hold as h

JOIN
sierra_view.record_metadata as r
ON
  r.id = h.record_id

JOIN
sierra_view.patron_record as p
ON
  p.record_id = h.patron_record_id

WHERE
(r.record_type_code = 'b' OR r.record_type_code = 'j')
AND h.is_ir is false -- not INN-Reach
AND h.is_ill is false -- not ILL
AND h.is_frozen is false -- not frozen hold
AND p.ptype_code IN (0, 1, 2, 3, 5, 6, 10, 11, 12, 15, 22, 30, 31, 32, 40, 41, 196)
AND (( extract(epoch FROM age(
	(h.placed_gmt + concat(h.delay_days, ' days')::INTERVAL)
)) / 3600 ) / 24 )::INTEGER > 30 
;

-- TESTING
-- SELECT * FROM temp_freaky_holds LIMIT 100;


-- aggregate the holds into the title, and count them
DROP TABLE IF EXISTS temp_freaky_holds_titles;
CREATE TEMP TABLE temp_freaky_holds_titles AS
SELECT
t.bib_record_id,
t.record_id,
t.record_type_code,
count(t.bib_record_id) AS count_holds

FROM
temp_freaky_holds AS t

GROUP BY
t.bib_record_id,
t.record_id,
t.record_type_code
;

-- testing
-- SELECT * FROM temp_freaky_holds_titles LIMIT 100;


-- get attached items
DROP TABLE IF EXISTS temp_freaky_holds_item_status;
CREATE TEMP TABLE temp_freaky_holds_item_status AS
SELECT
*,
(
	SELECT
	string_agg(DISTINCT i.location_code, ', ')

	FROM
	sierra_view.bib_record_item_record_link AS l

	JOIN
	sierra_view.item_record as i
	ON
	  i.record_id = l.item_record_id

	WHERE
	l.bib_record_id = t.bib_record_id
	AND i.item_status_code = '-'
) AS items_status_dash,
(
	SELECT
	string_agg(DISTINCT i.location_code, ', ')

	FROM
	sierra_view.bib_record_item_record_link AS l

	JOIN
	sierra_view.item_record as i
	ON
	  i.record_id = l.item_record_id

	WHERE
	l.bib_record_id = t.bib_record_id
	AND (i.item_status_code = 'g' OR i.item_status_code = 'm')
) AS items_status_g_or_m

FROM
temp_freaky_holds_titles as t
;

DROP TABLE temp_freaky_holds;
DROP TABLE temp_freaky_holds_titles;

-- output
SELECT
r.record_type_code || r.record_num || 'a' as bib_record_num,
t.count_holds,
(
-- 	SELECT 
-- 	v.field_content as callnumber

	SELECT
	-- get the call number strip the subfield indicators
	regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig')

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = t.bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY
	v.occ_num

	LIMIT 1
) as callnumber,
CASE
	WHEN t.record_type_code = 'j' THEN (
-- 		t.record_id
		SELECT
		v.field_content
		FROM
		sierra_view.varfield as v
		WHERE
		v.record_id = t.record_id
		AND v.varfield_type_code = 'v'
		ORDER BY
		v.occ_num
		LIMIT 1
	)
	ELSE NULL
END AS volume,
b.best_title, 
b.best_author,
t.items_status_dash,
t.items_status_g_or_m

FROM
temp_freaky_holds_item_status as t

JOIN
sierra_view.record_metadata as r
ON
  r.id = t.bib_record_id

JOIN
sierra_view.bib_record_property as b
ON
  b.bib_record_id = t.bib_record_id


WHERE
t.items_status_dash IS NOT NULL
OR t.items_status_g_or_m IS NOT NULL
;
```



## 90 day holds reports

### Bib-level 90 day holds report (testing)
```sql
SELECT
hold.record_id,
( 
	SELECT
	-- h.record_id,
	string_agg(h_list.pickup_location_code, ',') as pickup_location_list

	FROM
	sierra_view.hold as h_list

	WHERE
	h_list.record_id = hold.record_id

	GROUP BY
	h_list.record_id

) as pickup_location_list,

(
	-- get the count of OS over 90 days on hold
	SELECT
	COUNT(*)

	FROM
	sierra_view.hold as h_count

	WHERE
	h_count.record_id = hold.record_id
	AND h_count.pickup_location_code = 'os'
	AND h_count.is_frozen = 'f'
	AND (EXTRACT(epoch FROM (SELECT (NOW() - h_count.placed_gmt)))/86400)::int > 90
) as count_os_over_90,


bib_view.record_num,
bib_view.title,
date(bib_view.cataloging_date_gmt) as cat_date,
bib_view.bcode2,
COUNT (bib_view.id) AS nbr_holds,
(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view

	WHERE 
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'c'

	LIMIT 1 
) as callnum,
(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view
	
	WHERE
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'a'

	LIMIT 1
) as author,
(
	SELECT
	sierra_view.varfield_view.field_content

	FROM
	sierra_view.varfield_view
	
	WHERE
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b' 
	AND   sierra_view.varfield_view.varfield_type_code = 'p'
	LIMIT 1 
) as pubinfo,

(
	SELECT
	COUNT (item_record.id) as bib_items
	
	FROM
	sierra_view.bib_record_item_record_link , sierra_view.item_record

	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
) as nbr_items,

(
	SELECT
	COUNT(item_record.id ) as bib_items_active

	FROM
	sierra_view.bib_record_item_record_link,
	sierra_view.item_record
	
	LEFT OUTER JOIN sierra_view.checkout
	ON
	  sierra_view.item_record.id = sierra_view.checkout.item_record_id
	
	JOIN sierra_view.record_metadata
	ON
	  sierra_view.record_metadata.id = sierra_view.item_record.id
	  
	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
	-- AND item_record.item_status_code IN ( '-', 't', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' )
	AND (item_record.item_status_code IN ( '-', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' ) -- #20161116 only items in-transit for fewer than 60 days are active, per VM
	OR (item_record.item_status_code = 't' AND record_metadata.record_type_code = 'i'
	AND record_metadata.record_last_updated_gmt > now() - interval '60 days' ))
	AND ( -- #consider longoverdue items to be not active
		sierra_view.checkout.due_gmt is null  -- #item isnt checked out
		OR sierra_view.checkout.due_gmt > current_date - interval '60 days' -- #item is checked out, but due date after 60days ago
	)
) as nbr_active_items,

(
	SELECT 
	COUNT (item_record.id ) as bib_items_inporcessing 
	
	FROM
	sierra_view.bib_record_item_record_link , sierra_view.item_record
	
	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
	AND item_record.item_status_code IN ( 'p' )
) as nbr_inprocessing_items,

(
	SELECT 
	SUM(order_record_cmf.copies) as order_copies
	
	FROM
	sierra_view.bib_record_order_record_link,
	sierra_view.order_view,
	sierra_view.order_record_cmf

	WHERE
	hold.record_id = bib_record_order_record_link.bib_record_id 
	AND bib_record_order_record_link.order_record_id = order_view.id 
	AND order_view.record_id = order_record_cmf.order_record_id 
	AND order_view.received_date_gmt is null 
	AND order_record_cmf.location_code != 'multi'

	GROUP BY hold.record_id
) as nbr_ordered_copies

FROM 
sierra_view.hold

JOIN sierra_view.bib_view 
ON 
  hold.record_id = bib_view.id 
  AND bib_view.cataloging_date_gmt IS NOT NULL 

JOIN sierra_view.patron_record 
ON
hold.patron_record_id = patron_record.id 
AND patron_record.ptype_code IN (  0 , 1 , 2 , 5 , 6 , 10 , 11 , 12 , 15 , 22 , 30 , 31 , 32 , 40 , 41, 196 )

WHERE (
	(
		hold.is_frozen = 'f' 
		AND (
			(hold.delay_days = 0) 
			OR (EXTRACT(epoch FROM (SELECT (NOW() - hold.placed_gmt)))/86400::int > delay_days)
		)
	) 
	OR ( patron_record.ptype_code = 196 ) 
)  -- # Changed criteria to allow frozen/delay_days holds for Admin cards 20160504 LMK
-- AND (hold.placed_gmt < current_date - interval '90 days' ) removed RV
AND (EXTRACT(epoch FROM (SELECT (NOW() - hold.placed_gmt)))/86400) > 90.0

GROUP BY
hold.record_id, 
-- 	hold.pickup_location_code,
bib_view.id, 
bib_view.record_num,
bib_view.title,
bib_view.cataloging_date_gmt,
bib_view.bcode2

ORDER BY
title,
bcode2,
callnum
```

### Bib-level 90 day holds report (currently in production as of 2017-07-14)
```sql
SELECT
hold.record_id,
bib_view.record_num,
bib_view.title,
date(bib_view.cataloging_date_gmt) as cat_date,
bib_view.bcode2,
COUNT (bib_view.id) AS nbr_holds,
(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view

	WHERE 
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'c'

	LIMIT 1 
) as callnum,
(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view
	
	WHERE
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'a'

	LIMIT 1
) as author,
(
	SELECT
	sierra_view.varfield_view.field_content

	FROM
	sierra_view.varfield_view
	
	WHERE
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b' 
	AND   sierra_view.varfield_view.varfield_type_code = 'p'
	LIMIT 1 
) as pubinfo,

(
	SELECT
	COUNT (item_record.id) as bib_items
	
	FROM
	sierra_view.bib_record_item_record_link , sierra_view.item_record

	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
) as nbr_items,

(
	SELECT
	COUNT(item_record.id ) as bib_items_active

	FROM
	sierra_view.bib_record_item_record_link,
	sierra_view.item_record
	
	LEFT OUTER JOIN sierra_view.checkout
	ON
	  sierra_view.item_record.id = sierra_view.checkout.item_record_id
	
	JOIN sierra_view.record_metadata
	ON
	  sierra_view.record_metadata.id = sierra_view.item_record.id
	  
	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
	-- AND item_record.item_status_code IN ( '-', 't', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' )
	AND (item_record.item_status_code IN ( '-', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' ) -- #20161116 only items in-transit for fewer than 60 days are active, per VM
	OR (item_record.item_status_code = 't' AND record_metadata.record_type_code = 'i'
	AND record_metadata.record_last_updated_gmt > now() - interval '60 days' ))
	AND ( -- #consider longoverdue items to be not active
		sierra_view.checkout.due_gmt is null  -- #item isnt checked out
		OR sierra_view.checkout.due_gmt > current_date - interval '60 days' -- #item is checked out, but due date after 60days ago
	)
) as nbr_active_items,

(
	SELECT 
	COUNT (item_record.id ) as bib_items_inporcessing 
	
	FROM
	sierra_view.bib_record_item_record_link , sierra_view.item_record
	
	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
	AND item_record.item_status_code IN ( 'p' )
) as nbr_inprocessing_items,

(
	SELECT 
	SUM(order_record_cmf.copies) as order_copies
	
	FROM
	sierra_view.bib_record_order_record_link,
	sierra_view.order_view,
	sierra_view.order_record_cmf

	WHERE
	hold.record_id = bib_record_order_record_link.bib_record_id 
	AND bib_record_order_record_link.order_record_id = order_view.id 
	AND order_view.record_id = order_record_cmf.order_record_id 
	AND order_view.received_date_gmt is null 
	AND order_record_cmf.location_code != 'multi'

	GROUP BY hold.record_id
) as nbr_ordered_copies

FROM 
sierra_view.hold

JOIN sierra_view.bib_view 
ON 
  hold.record_id = bib_view.id 
  AND bib_view.cataloging_date_gmt IS NOT NULL 

JOIN sierra_view.patron_record 
ON
hold.patron_record_id = patron_record.id 
AND patron_record.ptype_code IN (  0 , 1 , 2 , 5 , 6 , 10 , 11 , 12 , 15 , 22 , 30 , 31 , 32 , 40 , 41, 196 )

WHERE (
	(
		hold.is_frozen = 'f' 
		AND (
			(hold.delay_days = 0) 
			OR (EXTRACT(epoch FROM (SELECT (NOW() - placed_gmt)))/86400::int > delay_days)
		)
	) 
	OR ( patron_record.ptype_code = 196 ) 
)  -- # Changed criteria to allow frozen/delay_days holds for Admin cards 20160504 LMK
AND (hold.placed_gmt < current_date - interval '90 days' )

GROUP BY
hold.record_id, 
bib_view.id, 
bib_view.record_num,
bib_view.title,
bib_view.cataloging_date_gmt,
bib_view.bcode2

ORDER BY
bcode2,
callnum,
title
```

## Volume-level 90 day holds report (currently in production as of 2017-07-14)
```sql
SELECT
hold.record_id,
bib_view.id,
bib_view.record_num,
bib_view.title,
date(bib_view.cataloging_date_gmt) as cat_date,
bib_view.bcode2,
COUNT (bib_view.id) as nbr_holds,

(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view
	
	WHERE 
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'c'
	
	LIMIT 
	1   
) as callnum,
	
(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view
	
	WHERE
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'a'

	LIMIT 
	1   
) as author,

( 
	SELECT
	sierra_view.varfield_view.field_content
	FROM
	sierra_view.varfield_view

	WHERE 
	sierra_view.varfield_view.record_id = sierra_view.bib_view.id
	AND sierra_view.varfield_view.record_type_code = 'b'
	AND sierra_view.varfield_view.varfield_type_code = 'p'
	
	LIMIT
	1   
) as pubinfo,

(
	SELECT
	COUNT (item_record.id) as vol_items
	
	FROM
	sierra_view.volume_record_item_record_link,
	sierra_view.item_record
	
	WHERE
	sierra_view.volume_record_item_record_link.volume_record_id = hold.record_id
	AND sierra_view.volume_record_item_record_link.item_record_id = item_record.id
) as nbr_items,

(
	SELECT
	COUNT (item_record.id ) AS vol_items_active

	FROM
	sierra_view.volume_record_item_record_link,
	sierra_view.item_record
	
	LEFT OUTER JOIN 
	sierra_view.checkout
	ON
	  sierra_view.item_record.id = sierra_view.checkout.item_record_id
	  
	JOIN sierra_view.record_metadata
	ON 
	  sierra_view.record_metadata.id = sierra_view.item_record.id

	WHERE
	sierra_view.volume_record_item_record_link.volume_record_id = hold.record_id
	AND sierra_view.volume_record_item_record_link.item_record_id = item_record.id
	-- AND item_record.item_status_code IN ( '-', 't', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' )
	AND (
		item_record.item_status_code IN ( '-', '!', 'b', 'p', '(', '\@', ')', '_', '=', '+' ) -- #20161116 only items in-transit for fewer than 60 days are active, per VM
		OR (
			item_record.item_status_code = 't' 
			AND record_metadata.record_type_code = 'i'
			AND record_metadata.record_last_updated_gmt > now() - interval '60 days' 
		)
	)
	-- consider longoverdue items to be not active
	AND ( 
		sierra_view.checkout.due_gmt is null --item isnt checked out
		OR sierra_view.checkout.due_gmt > current_date - interval '60 days' -- #item is checked out, but due date after 60days ago
	)
) as nbr_active_items,

(
	SELECT
	COUNT
	(item_record.id ) AS bib_items_inporcessing
	
	FROM
	sierra_view.bib_record_item_record_link,
	sierra_view.item_record

	WHERE
	sierra_view.bib_record_item_record_link.bib_record_id = hold.record_id
	AND sierra_view.bib_record_item_record_link.item_record_id = item_record.id
	AND item_record.item_status_code IN ( 'p' )
) as nbr_inprocessing_items,

(
	SELECT
	SUM(order_record_cmf.copies) AS order_copies

	FROM
	sierra_view.bib_record_order_record_link,
	sierra_view.order_view,
	sierra_view.order_record_cmf

	WHERE
	hold.record_id = bib_record_order_record_link.bib_record_id
	AND bib_record_order_record_link.order_record_id = order_view.id
	AND order_view.record_id = order_record_cmf.order_record_id
	AND order_view.received_date_gmt is null
	AND order_record_cmf.location_code != 'multi'

	GROUP BY 
	hold.record_id
) as nbr_ordered_copies,

(
	SELECT
	sierra_view.varfield_view.field_content
	
	FROM
	sierra_view.varfield_view

	WHERE
	sierra_view.varfield_view.record_id = sierra_view.volume_record.id
	AND sierra_view.varfield_view.record_type_code = 'j'
	AND sierra_view.varfield_view.varfield_type_code = 'v'
	
	ORDER BY
	field_content DESC 
	
	LIMIT 1 
) as volume_statement

FROM sierra_view.hold

JOIN sierra_view.volume_record
ON
  hold.record_id = volume_record.id

JOIN
sierra_view.bib_record_volume_record_link
ON
  volume_record.record_id = bib_record_volume_record_link.volume_record_id

JOIN sierra_view.bib_view
ON
  bib_record_volume_record_link.bib_record_id = bib_view.id
  AND bib_view.bcode2 NOT IN ( 's', 'n' )
  AND bib_view.cataloging_date_gmt IS NOT NULL

JOIN sierra_view.patron_record
ON
  hold.patron_record_id = patron_record.id
  AND patron_record.ptype_code IN ( 0 , 1 , 2 , 5 , 6 , 10 , 11 , 12 , 15 , 22 , 30 , 31 , 32 , 40 , 41, 196 )

WHERE 
(
	(
		hold.is_frozen = 'f' 
		AND (
			(hold.delay_days = 0) 
			OR (EXTRACT(epoch FROM (SELECT (NOW() - placed_gmt)))/86400::int > delay_days)
		) 
	)
	OR ( patron_record.ptype_code = 196 )
)  -- # Changed criteria to allow frozen/delay_days holds for Admin cards 20160504 LMK
AND (hold.placed_gmt < current_date - interval '90 days' ) 

GROUP BY 
hold.record_id,
bib_view.id,
bib_view.record_num,
bib_view.title,
bib_view.cataloging_date_gmt,
bib_view.bcode2,
sierra_view.volume_record.id

ORDER BY 
bcode2,
callnum,
title
```



## Find holds that are greater than a certain number of days (365 in the example below)
```sql
SELECT
-- number of days the hold has been active (86400 seconds per day)
(EXTRACT(EPOCH FROM current_timestamp - placed_gmt)/86400)::int as days_active,
r.record_type_code || r.record_num || 'a' as record_num,
pr.record_type_code || pr.record_num || 'a' as patron_record_num,
h.placed_gmt,
h.expires_gmt,
h.is_frozen,
h.delay_days,
h.is_ill

FROM
sierra_view.hold AS h

JOIN
sierra_view.record_metadata as r
ON
  r.id = h.record_id

JOIN
sierra_view.record_metadata as pr
ON
  pr.id = h.patron_record_id

WHERE
(EXTRACT(EPOCH FROM current_timestamp - placed_gmt)/86400)::int > 365
AND h.is_frozen = false

ORDER BY
days_active DESC
```

## Get bib information from holds that are INN-Reach or ILL
```sql
-----

-- this query will get hold, bib, and item information from holds that are 
-- INN-Reach or ILL 

-----

DROP TABLE IF EXISTS temp_holds_data;
CREATE TEMP TABLE temp_holds_data AS
SELECT
p.ptype_code,
p.home_library_code as patron_home_library_code,
n.last_name || ', ' ||n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') AS "patron_name",
r.record_type_code || r.record_num as record_num,

CASE
	WHEN r.record_type_code = 'i' THEN (
		SELECT
		-- i.item_status_code
		CASE
			WHEN i.item_status_code = '-' THEN 'AVAILABLE'
			WHEN i.item_status_code = 'm' THEN 'MISSING'
			WHEN i.item_status_code = 'z' THEN 'CL RETURNED'
			WHEN i.item_status_code = 'o' THEN 'LIB USE ONLY'
			WHEN i.item_status_code = 'n' THEN 'BILLED NOTPAID'
			WHEN i.item_status_code = '$' THEN 'BILLED PAID'
			WHEN i.item_status_code = 't' THEN 'IN TRANSIT'
			WHEN i.item_status_code = '!' THEN 'ON HOLDSHELF'
			WHEN i.item_status_code = 'l' THEN 'LOST'
			-- At INN-Reach sites, the following additional codes and definitions are standard:
			WHEN i.item_status_code = '@' THEN 'OFF SITE'
			WHEN i.item_status_code = '#' THEN 'RECEIVED'
			WHEN i.item_status_code = '%' THEN 'RETURNED'
			WHEN i.item_status_code = '&' THEN 'REQUEST'
			WHEN i.item_status_code = '_' THEN 'REREQUEST'
			WHEN i.item_status_code = '(' THEN 'PAGED'
			WHEN i.item_status_code = ')' THEN 'CANCELLED'
			WHEN i.item_status_code = '1' THEN 'LOAN REQUESTED'
			ELSE i.item_status_code
		END
		FROM
		sierra_view.item_record as i

		WHERE
		i.record_id = r.id

		LIMIT 1
	)
	ELSE NULL
END as item_record_status,

-- get the bib record id from holds (which can be item-level, volume-level, or bib-level)
CASE
	WHEN r.record_type_code = 'i' THEN (
		SELECT
		l.bib_record_id

		FROM
		sierra_view.bib_record_item_record_link as l

		WHERE
		l.item_record_id = h.record_id

		LIMIT 1
	)

	WHEN r.record_type_code = 'j' THEN (
		SELECT
		l.bib_record_id

		FROM
		sierra_view.bib_record_volume_record_link as l

		WHERE
		l.volume_record_id = h.record_id

		LIMIT 1
	)

	WHEN r.record_type_code = 'b' THEN (
		h.record_id
	)

	ELSE NULL
END as bib_record_id,

CASE
	WHEN h.status = '0' THEN 'On hold'
	WHEN h.status = 'b' THEN 'Bib hold ready for pickup.'
	WHEN h.status = 'j' THEN 'Volume hold ready for pickup.'
	WHEN h.status = 'i' THEN 'Item hold ready for pickup.'
	WHEN h.status = 't' THEN 'Bib, item, or volume in transit to pickup location.'
END as hold_status,

h.*

FROM
sierra_view.hold as h

LEFT OUTER JOIN
sierra_view.record_metadata as r
ON
  r.id = h.record_id

LEFT OUTER JOIN
sierra_view.patron_record as p
ON
  p.record_id = h.patron_record_id

LEFT OUTER JOIN
sierra_view.patron_record_fullname as n
ON
  n.patron_record_id = h.patron_record_id

WHERE
-- uncomment / comment out here to limit to INN-Reach / ILL holds 
(	is_ir IS true
	OR is_ill IS true
)
;
-----

-----
SELECT 
p.best_title,
p.publish_year,
t.*

FROM 
temp_holds_data as t

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = t.bib_record_id
```

