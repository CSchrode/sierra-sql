# Find duplicate patron barcodes that start with “SR”
temporary patron cards are issued barcodes starting with SR, but are sometimes accidentally duplicated. This query will find those patron record nums.

```sql 
SELECT
-- patron record number, barcode, and create date. 
r.record_num,
p.index_entry,
r.creation_date_gmt

FROM
sierra_view.phrase_entry as p

JOIN
sierra_view.record_metadata as r
ON
  r.id = p.record_id

JOIN
sierra_view.patron_record as pr
ON
  pr.record_id = r.id

where 
p.index_tag || p.index_entry IN
(
SELECT
'b' || e.index_entry

FROM
sierra_view.phrase_entry as e

JOIN
sierra_view.record_metadata as r
ON
  r.id = e.record_id

WHERE
e.index_tag || e.index_entry SIMILAR TO 'b' || 'sr[0-9]{1,}'

GROUP BY
e.index_entry

HAVING COUNT(e.index_entry) > 1
)
```