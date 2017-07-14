## 90 day holds reports

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