## Find location codes for all items, and the count of items in each of those locations
```sql
SELECT
i.location_code,
count(*)

FROM
sierra_view.item_record as i

GROUP BY
i.location_code

ORDER BY
count(*) DESC
```