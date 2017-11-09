# More granular "Claimed returned" search / report
```sql
DROP TABLE IF EXISTS temp_claims_returned;
--


CREATE TEMP TABLE temp_claims_returned AS

SELECT
r.record_type_code || r.record_num || 'a' as item_record_num,
(
	SELECT
	string_agg(v.field_content, ',' order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = r.id
	AND v.varfield_type_code = 'b'
) AS item_barcodes,
substring(v.field_content, '^.{15}')::date as claimed_date,
v.record_id,
v.field_content

FROM
sierra_view.varfield as v

JOIN
sierra_view.record_metadata as r
ON
  (r.id = v.record_id)

WHERE
v.varfield_type_code = 'x'
AND v.marc_tag is null
AND v.field_content ~ '^.{17}Claimed returned'
AND r.record_type_code = 'i'
;
---


ANALYSE temp_claims_returned;
---


-- produce our results
SELECT
*

FROM
temp_claims_returned as c

WHERE
c.claimed_date >= ( (DATE_TRUNC('month', NOW()) - INTERVAL '5 years 1 month') )::date -- first day of 5 years and 1 month ago
AND c.claimed_date < ( (DATE_TRUNC('month', NOW())) - INTERVAL '5 years' )::date -- last day of 5 years and 1 month ago
;
---


-- -- get the count of claims returned from each year
-- SELECT
-- extract(year from c.claimed_date) AS claimed_year,
-- count(*)
-- 
-- FROM
-- temp_claims_returned as c
-- 
-- GROUP BY
-- claimed_year
-- 
-- ORDER BY
-- claimed_year

```


# Find cases where an item record is linked to more than one bib record (improved)
```sql
--- find same item record linked to multiple bib records 
SELECT
r.record_type_code || r.record_num || 'a' as bib_record_num,
ir.record_type_code || ir.record_num || 'a' as item_record_num

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.bib_record_item_record_link as l
ON
  l.bib_record_id = r.id

JOIN
sierra_view.record_metadata as ir
ON
  ir.id = l.item_record_id

WHERE
l.item_record_id IN (
	SELECT
	l.item_record_id
	-- ,count(item_record_id)

	FROM
	sierra_view.bib_record_item_record_link as l

	GROUP BY
	l.item_record_id

	HAVING
	count(item_record_id) > 1
);
```

# Find cases where an item record is linked to more than one bib record
```sql
-- find item records that are linked to more than one bib record
SELECT
l.item_record_id,
count(bib_record_id)

FROM
sierra_view.bib_record_item_record_link as l

GROUP BY
l.item_record_id

HAVING
count(bib_record_id) > 1
```


# Count number of items by criteria
```sql
SELECT
-- i.agency_code_num,
p.name as item_agency,
count(*)

FROM
sierra_view.item_record as i

JOIN
sierra_view.agency_property_myuser as p
ON
  p.code = i.agency_code_num

WHERE
i.item_status_code IN (
'-',
'o',
'r',
't',
'!'
)
AND substring(i.location_code,1,2) NOT IN (
'vi',
'yt',
'ym',
'yc',
'yh'
)

GROUP BY
p.name,
p.display_order

ORDER BY
p.display_order
```