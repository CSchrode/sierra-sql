# Fines

# Fines paid by date
```sql
SELECT
f.fine_assessed_date_gmt::date as "Date Assessed",
-- n.last_name || ', ' || n.first_name || ' ' || n.middle_name as "Patron Name",
n.last_name || ', ' ||n.first_name || COALESCE(' ' || NULLIF(n.middle_name, ''), '') AS "Patron Name",
r.record_num as "Patron Record",
null as "Patron Unique ID",
f.invoice_num as "Invoice",
-- f.old_invoice_num,
cast(f.item_charge_amt as money) as "Charge Amt.",
cast(f.processing_fee_amt as money) as "Processing Fee",

CASE 
  WHEN f.charge_type_code = '1' THEN 'manual charge'
  WHEN f.charge_type_code = '2' THEN 'overdue'
  WHEN f.charge_type_code = '3' THEN 'replacement'
  WHEN f.charge_type_code = '4' THEN 'adjustment (OVERDUEX)'
  WHEN f.charge_type_code = '5' THEN 'lost book'
  WHEN f.charge_type_code = '6' THEN 'overdue renewed'
  WHEN f.charge_type_code = '7' THEN 'rental'
  WHEN f.charge_type_code = '8' THEN 'rental adjustment (RENTALX)'
  WHEN f.charge_type_code = '9' THEN 'debit'
  WHEN f.charge_type_code = 'a' THEN 'notice'
  WHEN f.charge_type_code = 'b' THEN 'credit card'
  WHEN f.charge_type_code = 'p' THEN 'program (i.e., Program Registration)'  
  ELSE null
END AS "Charge Type",

-- null as "Owning Location", -- is this just the item location code? not sure how that works with floating
f.charge_location_code as "Owning Location",

f.paid_date_gmt::date as "Date Paid",
f.tty_num as "Statistics Group",
cast(f.last_paid_amt as money) as "Last Payment",
-- null as "Login", -- not sure where this is coming from either
f.iii_user_name as "Login",
CASE
  WHEN f.fine_creation_mode_code = 'a' THEN 'automatic'
  WHEN f.fine_creation_mode_code = 'm' THEN 'manual'
  WHEN f.fine_creation_mode_code = 'x' THEN 'adjustment'
  ELSE null
END as "Creation Mode",
f.description as "Description",
cast(f.paid_now_amt as money) as "Amount Paid",

CASE
  WHEN f.payment_status_code = '0' THEN 'no payment'
  WHEN f.payment_status_code = '1' THEN 'full payment'
  WHEN f.payment_status_code = '2' THEN 'partial payment'
  WHEN f.payment_status_code = '3' THEN 'waive'
  WHEN f.payment_status_code = '4' THEN 'item busy'
  WHEN f.payment_status_code = '5' THEN 'will pay'
  WHEN f.payment_status_code = '6' THEN 'purge'
  WHEN f.payment_status_code = '7' THEN 'credit'
  WHEN f.payment_status_code = '8' THEN 'adjustment'
  ELSE null
END as "Payment Status",
f.payment_type_code as "Payment Type",
f.payment_note as "Payment Note"

FROM
sierra_view.fines_paid as f

JOIN
sierra_view.record_metadata as r
ON
  r.id = f.patron_record_metadata_id

LEFT OUTER JOIN
sierra_view.patron_record_fullname as n
ON
  n.patron_record_id = f.patron_record_metadata_id

WHERE
f.paid_date_gmt::date = '2018-03-23'::date

ORDER BY 
f.patron_record_metadata_id,
f.paid_date_gmt
```


# Post fines purge report for statute of limitations fines
```sql
DROP TABLE IF EXISTS temp_patron_purge_amts;

CREATE TEMP TABLE temp_patron_purge_amts AS 

SELECT
date_trunc('day', p.paid_date_gmt::timestamp) as date_purged,
p.patron_record_metadata_id,
SUM ( (p.item_charge_amt + p.processing_fee_amt + p.billing_fee_amt) - (p.last_paid_amt + p.paid_now_amt) ) as total_purged 

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
t.date_purged,
r.id as patron_record_id

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

