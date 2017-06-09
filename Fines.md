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

	-- the date of the statute of limitations fines we're looking for
	f.assessed_gmt < '2012-09-01'
	AND f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
) AS sum_prev_2012_09_01,

-- compute the amount after the purge
(p.owed_amt - 
	(
	SELECT
	SUM(f.item_charge_amt - f.paid_amt)

	FROM
	sierra_view.fine as f

	WHERE
	-- the date of the statute of limitations fines we're looking for
	f.assessed_gmt < '2012-09-01'
	AND f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
	)
) as post_purge_owed_amt,

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


## Find negative fines

```sql
select
'p' || r.record_num || 'a' as record_num,
f.assessed_gmt,
f.invoice_num,
f.item_charge_amt,
f.processing_fee_amt,
f.billing_fee_amt,
f.charge_code,
f.charge_location_code,
f.paid_gmt,
f.terminal_num,
f.paid_amt,
f.initials,
f.created_code,
f.is_print_bill,
f.description

from
sierra_view.fine as f

JOIN
sierra_view.record_metadata as r
ON
  r.id = f.patron_record_id

WHERE
( (f.item_charge_amt + f.billing_fee_amt + f.processing_fee_amt) - f.paid_amt) < 0
and (assessed_gmt >= '2017-01-01' and assessed_gmt < '2018-01-01')

