Find holds that are greater than a certain number of days (365 in the example below)

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