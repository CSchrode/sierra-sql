![PLCH_Logo_Horizontal_Red.png](https://raw.githubusercontent.com/plch/brand-and-logos/master/PLCH_Logo_Horizontal_Red.png "PLCH")
# General Tips for the sierra-db

* To get record numbers--for use in the Sierra Desktop Application import record numbers feature for example, we can join the table ```sierra_view.record_metadata```, appending the columns (and extra checkdigit character, ‘a’) ```sierra_view.record_type_code || sierra_view.record_number || ‘a’```

Example:
```sql
SELECT
r.record_type_code || r.record_num || 'a' as record_number

FROM
sierra_view.record_metadata as r

WHERE
r.record_type_code || r.campus_code = 'p'
and r.deletion_date_gmt is null

limit 5
```

... would produce the following output for example:
```
p2160402a
p1117209a
p2072447a
p1152091a
p1507488a
```

* Picking the correct primary / foreign keys for joins can be tricky.
    * Make sure you carefully examine the sierra DNA documentation 
    * ```id``` generally is the primary key, as it is in ```sierra_view.record_metadata```. This means that it'll appear in other tables. For example, it might appear as 

```bib_record_id``` in table ```sierra_view.bib_record_item_record_link``` 

or

```record_id``` in table ```sierra_view.bib_record```

* Joining varfields in results can be tricky, since these  can be repeatable fields. 
    * Typically you want to avoid multiple rows of results being produced because of a unexpected (or expected) repeating varfields. 
    * An example of this could be notes fields, added authors, etc, which could appear multiple times. The following query should return only one row for the one patron, but instead we get two, because the patron has two barcodes assigned to them.
```sql
SELECT
p.record_id,
v.*

FROM
sierra_view.patron_record as p

-- get the barcode
JOIN
sierra_view.varfield as v
ON
  v.record_id = p.record_id
  AND v.varfield_type_code = 'b'

WHERE
p.record_id = 481037410624
```

To avoid this, we could either select only one value for the barcode, or select all matching values and aggregate them into one value (so that we only get one row of results back as expected) with the ```string_agg()``` function that postgresql offers as a subquery in the select statement:

```sql
SELECT
p.record_id,
(
	SELECT
	string_agg(v.field_content, ',' order by v.occ_num)

	FROM
	sierra_view.varfield as v

	WHERE
	v.record_id = p.record_id
	AND v.varfield_type_code = 'b'
)

FROM
sierra_view.patron_record as p

WHERE
p.record_id = 481037410624
```

This might produce data similar to this
```53933149,49548969 / PREV ID```

* Some of the views combine multiple tables, and can be convenient, but can be much slower than other methods of retrieving data.
```sierra_view.*_view```
Example: ```sierra_view.bib_view``` combines data from multiple tables, but query speeds can be slower than accessing other tables more directly. 

```sql
SELECT
*
FROM
sierra_view.bib_view

LIMIT 1000
```
```(1.6 seconds)```

VS

```sql
SELECT
*
FROM
sierra_view.bib_record as b

JOIN
sierra_view.bib_record_property as p
ON
  p.bib_record_id = b.record_id

LIMIT 1000
```

```(543 milliseconds)```