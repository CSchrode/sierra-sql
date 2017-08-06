## Find all the unique domains (and the times used) stored as 856 links in the bib records
```sql
SELECT
substring(s.content from '.*://([^/]*)') as domain_name,
count(*) as count

FROM
sierra_view.subfield as s

WHERE
s.marc_tag || s.field_type_code || s.tag = '856yu'

GROUP BY
domain_name

ORDER BY
domain_name
```


## Find bib records with the proxy url prefix
```sql
SELECT
r.record_type_code || r.record_num || 'a' as record_num,
v.field_content

FROM
sierra_view.varfield as v

JOIN
sierra_view.record_metadata as r
ON
  r.id = v.record_id

WHERE
v.marc_tag || v.varfield_type_code = '856y'
and v.field_content ~* 'research\.cincinnatilibrary\.org'
```