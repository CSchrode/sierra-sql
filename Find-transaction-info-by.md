# Find transaction info by ...

## Patron barcode:
```sql
-- put the patron barcode in at the WHERE clause below
SELECT
c.transaction_gmt,
e.index_entry as patron_barcode,
c.application_name,
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
id2reckey(c.item_record_id) as item_num,
id2reckey(c.volume_record_id) as vol_num,
id2reckey(c.bib_record_id) as bib_num

FROM
sierra_view.circ_trans as c

JOIN
sierra_view.patron_record as p
ON
  p.record_id = c.patron_record_id

-- 
LEFT OUTER JOIN
sierra_view.phrase_entry as e
ON
  e.record_id = p.record_id

LEFT OUTER JOIN
sierra_view.phrase_entry as ie
ON
  ie.record_id = c.item_record_id

WHERE
-- put the patron barcode in at the end ...
e.index_tag || e.index_entry = 'b' || '63344998'

ORDER BY 
c.transaction_gmt DESC
;
```

## item barcode
```sql
-- put the item barcode in at the WHERE clause below
SELECT
c.transaction_gmt,
e.index_entry as item_barcode,
c.application_name,
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
id2reckey(c.patron_record_id) as patron_num,
id2reckey(c.item_record_id) as item_num,
id2reckey(c.volume_record_id) as vol_num,
id2reckey(c.bib_record_id) as bib_num

FROM
sierra_view.circ_trans as c

LEFT JOIN
sierra_view.phrase_entry as e
ON
  e.record_id = c.item_record_id

WHERE
-- put the item barcode in at the end ...
e.index_tag || e.index_entry = 'b' || '0989053860015'

ORDER BY 
c.transaction_gmt DESC
;
```