# Sanity Checks

## Verify that all bibs with multiple location's have a location of 'multi'
```sql
-- verify that all bibs with more than two locations have
DROP TABLE IF EXISTS temp_bibs_multi_loc;
CREATE TEMP TABLE temp_bibs_multi_loc AS
SELECT
r.id,
r.record_num,
count(l.bib_record_id) as count_locations

FROM
sierra_view.record_metadata as r

LEFT OUTER JOIN
sierra_view.bib_record_location as l
ON
  l.bib_record_id = r.id

WHERE
r.record_type_code = 'b'

GROUP BY
r.id,
r.record_num,
l.bib_record_id

HAVING
count(l.bib_record_id) > 2;
---


SELECT
*

FROM
temp_bibs_multi_loc as b

LEFT OUTER JOIN
sierra_view.bib_record_location as l
ON
  (
	l.bib_record_id = b.id
	AND l.location_code = 'multi'
  )

WHERE
l.id IS null;
---


-- just to verify further that each bib is going to have at least 3 entries in the bib_location table if it has 'multi' as a location
-- SELECT
-- min(count_locations),
-- max(count_locations)
-- 
-- FROM
-- temp_bibs_multi_loc

```