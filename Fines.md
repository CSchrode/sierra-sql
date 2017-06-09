# Fines

## Find patron's with fines previous to statute of limitations and find what that amount is
```sql
SELECT
'p' || r.record_num || 'a' as patron_record_num,
p.owed_amt as current_owed_amt,

(
	SELECT
	SUM(f.item_charge_amt - f.paid_amt)

	FROM
	sierra_view.fine as f

	WHERE
	f.assessed_gmt < '2012-09-01'
	AND f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
) AS sum_prev_2012_09_01,

-- compute the amount after the purge
(p.owed_amt - (
	SELECT
	SUM(f.item_charge_amt - f.paid_amt)

	FROM
	sierra_view.fine as f

	WHERE
	f.assessed_gmt < '2012-09-01'
	AND f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
)) as post_purge_owed_amt,

date_trunc('day',p.expiration_date_gmt) as expiration_date_gmt,
date_trunc('day',p.activity_gmt) as activity_gmt,

(
	SELECT
	date_trunc('day', f.assessed_gmt)

	FROM
	sierra_view.fine as f

	WHERE
	f.patron_record_id = p.record_id

	ORDER BY
	f.assessed_gmt ASC

	LIMIT
	1
) as first_assessed_fine_date_gmt

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.patron_record as p
ON
  r.id = p.record_id

WHERE
r.deletion_date_gmt is null
AND r.campus_code = ''
-- AND p.expiration_date_gmt >= '2017-06-09'
AND r.id IN (
	SELECT
	f.patron_record_id

	FROM
	sierra_view.fine as f

	WHERE
	-- the date of the statute of limitations fines we're looking for
	f.assessed_gmt < '2012-09-01'

	GROUP BY
	f.patron_record_id
)

ORDER BY
p.owed_amt,
first_assessed_fine_date_gmt
```