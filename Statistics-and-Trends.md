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