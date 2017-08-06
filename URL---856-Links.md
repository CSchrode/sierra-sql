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