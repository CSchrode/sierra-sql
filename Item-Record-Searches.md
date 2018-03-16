# Item Record Searches

## Search for bib / item records. In this example, items have a specific itype code number, and start with a specific call number
```sql
---
-- first, create a temp table containing some item / bib information for items
-- having the itype we set below in the WHERE clause
DROP TABLE IF EXISTS temp_item_data;
CREATE TEMP TABLE temp_item_data AS
SELECT
i.itype_code_num,
i.location_code,
i.item_status_code,
i.is_suppressed,
rb.record_type_code || rb.record_num || 'a' as bib_record_num,
ri.record_type_code || ri.record_num || 'a' as item_record_num,
(
-- 	SELECT 
-- 	v.field_content as callnumber

	SELECT
	-- get the call number strip the subfield indicators
	regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig')

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = bib_record_id
	AND v.varfield_type_code = 'c'

	ORDER BY
	v.occ_num

	LIMIT 1
) as callnumber

FROM
sierra_view.item_record as i

-- link the item to the bib
JOIN
sierra_view.bib_record_item_record_link as l
ON
  (l.item_record_id = i.record_id)

-- get the item record number from here
JOIN
sierra_view.record_metadata as ri
ON
  (ri.id = i.record_id)

-- get the bib record number from here
JOIN
sierra_view.record_metadata as rb
ON
  (rb.id = l.bib_record_id) AND (rb.campus_code = '')

-- PUT YOUR 
WHERE
i.itype_code_num = '199'
;
-- end creating our temp table
---


--- 
-- next, query our temp table for call numbers in our range
SELECT
*

FROM
temp_item_data as t

-- in this example, i'm looking for call numbers that start with SL (using a regular expression)
WHERE
t.callnumber ~* '^sl.*'
```