# Fines

# Post fines purge report for statute of limitations fines
```sql
DROP TABLE IF EXISTS temp_patron_purge_amts;

CREATE TEMP TABLE temp_patron_purge_amts AS 

SELECT
date_trunc('day', p.paid_date_gmt::timestamp) as date_purged,
p.patron_record_metadata_id,
SUM ((p.item_charge_amt + p.processing_fee_amt + p.billing_fee_amt) - p.last_paid_amt) as total_purged 

FROM
sierra_view.fines_paid as p

WHERE
p.payment_status_code = '6'

GROUP BY
p.patron_record_metadata_id,
date_purged;

SELECT
r.record_type_code || r.record_num || 'a' as patron_record_num,
f.last_name || ', ' || f.first_name || ' ' || f.middle_name as patron_name,
(
	SELECT
	string_agg(e.index_entry , ',' order by e.occurrence)

	FROM
	sierra_view.phrase_entry as e

	WHERE
	e.record_id = r.id
	AND e.index_tag || e.varfield_type_code = 'bb'
) AS barcodes,
t.total_purged,
t.date_purged

FROM
temp_patron_purge_amts as t

LEFT OUTER JOIN
sierra_view.record_metadata as r
ON
  r.id = t.patron_record_metadata_id

LEFT OUTER JOIN
sierra_view.patron_record_fullname as f
ON
  f.patron_record_id = r.id

ORDER BY
t.date_purged DESC,
f.last_name,
f.first_name,
f.middle_name;
```


# Find patrons with fines that are within a certain range...
```sql
select
'p' || r.record_num || 'a' as patron_record_num,
p.expiration_date_gmt,
SUM( (f.item_charge_amt + f.billing_fee_amt + f.processing_fee_amt) - f.paid_amt) as fine_sum

from
sierra_view.patron_record as p

JOIN
sierra_view.fine as f
ON
  f.patron_record_id = p.record_id

JOIN
sierra_view.record_metadata as r
on
  r.id = p.record_id

WHERE
-- p.expiration_date_gmt < '2017-06-15'
-- and 
p.record_id IN
(
	select

	f.patron_record_id

	FROM
	sierra_view.fine as f

	group by f.patron_record_id

	HAVING 
	SUM( (f.item_charge_amt + f.processing_fee_amt + f.billing_fee_amt) - f.paid_amt) <= 0.01
)

group by
patron_record_num,
p.expiration_date_gmt


order by
p.expiration_date_gmt desc
```


## Find information about patrons who will have fines purged after statue of limitations

```sql
SELECT
'p' || r.record_num || 'a' as patron_record_num,
(
	SELECT
	e.index_entry as barcode
	from sierra_view.phrase_entry as e

	WHERE
	e.record_id = r.id
	AND e.index_tag || e.varfield_type_code = 'bb'

	ORDER BY
	e.occurrence asc

	LIMIT 1
) as barcode,
p.owed_amt as current_owed_amt,

(
	SELECT
	CASE 
		WHEN SUM( (f.item_charge_amt + f.processing_fee_amt + billing_fee_amt) - f.paid_amt ) = p.owed_amt
		THEN true

		ELSE false
	END

	FROM
	sierra_view.fine as f

	WHERE
	f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
) AS computed_equal_current,

(
	SELECT
	SUM( (f.item_charge_amt + f.processing_fee_amt + billing_fee_amt) - f.paid_amt )

	FROM
	sierra_view.fine as f

	WHERE

	-- the date of the statute of limitations fines we're looking for
	f.assessed_gmt <= date_trunc('day', (NOW() - INTERVAL '5 years') )
	AND f.patron_record_id = p.record_id

	GROUP BY
	f.patron_record_id
) AS sum_prev_statue_lim_date,

-- compute the amount after the purge
(	
	p.owed_amt - (
		SELECT
		SUM( (f.item_charge_amt + f.processing_fee_amt + billing_fee_amt) - f.paid_amt )

		FROM
		sierra_view.fine as f

		WHERE
		-- the date of the statute of limitations fines we're looking for (5 years prior to today's date)
		f.assessed_gmt <= date_trunc('day', (NOW() - INTERVAL '5 years') )
		AND f.patron_record_id = p.record_id

		GROUP BY
		f.patron_record_id
	)
) as post_purge_owed_amt,

date_trunc('day',p.expiration_date_gmt)::date as expiration_date,
date_trunc('day',p.activity_gmt)::date as activity,

(
	SELECT
	date_trunc('day', f.assessed_gmt)::date

	FROM
	sierra_view.fine as f

	WHERE
	f.patron_record_id = p.record_id

	ORDER BY
	f.assessed_gmt ASC

	LIMIT
	1
) as first_assessed_fine_date

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
	f.assessed_gmt <= date_trunc('day', (NOW() - INTERVAL '5 years') )

	GROUP BY
	f.patron_record_id
)

ORDER BY
computed_equal_current,
p.owed_amt,
first_assessed_fine_date
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

