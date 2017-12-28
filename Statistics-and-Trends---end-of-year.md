# Checkout totals by top bib records having items with specific item types
```sql
SELECT 
SUM ( item_view.last_year_to_date_checkout_total ) as sum_checkouts_ytd,
bib_view.record_num,
bib_view.title,
--varfield.field_content as callnum
(
  SELECT 
  sierra_view.varfield.field_content 
  
  FROM
  sierra_view.varfield

  WHERE 
  sierra_view.varfield.record_id = sierra_view.bib_view.id
  AND sierra_view.varfield.varfield_type_code = 'c' 
  
  LIMIT 1 
) AS callnum

FROM  
sierra_view.bib_record_item_record_link, 
sierra_view.item_view, 
sierra_view.bib_view--, 
--sierra_view.varfield

WHERE 
item_view.id = bib_record_item_record_link.item_record_id
AND bib_record_item_record_link.bib_record_id = bib_view.id

--Call Number:
--AND sierra_view.varfield.record_id = sierra_view.bib_view.id 
--AND sierra_view.varfield.record_type_code = 'b' 
--AND sierra_view.varfield.varfield_type_code = 'c' 
--AND field_content ~* '.*fiction.*'  --Fiction
--AND field_content ~ '^\|a(?:\d\d|M)\d'  --Non-fiction
 
--Item Type: 
-- un-comment for specific output
--AND item_view.itype_code_num NOT IN ( 30, 31, 32, 33, 34, 35, 37, 136 )  --Overall (no magazines)
AND item_view.itype_code_num IN ( 0, 1, 20, 21, 46 )  --Adult Books
--AND item_view.itype_code_num IN ( 4, 5 )  --Teen Books
--AND item_view.itype_code_num IN ( 2, 3 )  --Juvenile Books  
--AND item_view.itype_code_num IN ( 70, 60, 77, 90, 93, 100, 101, 82, 65, 130, 134, 163 )  --Adult AV
--AND item_view.itype_code_num IN ( 72, 92 ) --Teen AV
--AND item_view.itype_code_num IN ( 71, 61, 66, 78, 91 ) --Juvenile AV
  
--AND item_view.itype_code_num IN ( 30 ) --Adult Magazine
--AND item_view.itype_code_num IN ( 32 ) --Teen Magazine
--AND item_view.itype_code_num IN ( 31 ) --Juvenile Magazine

--AND item_view.itype_code_num IN ( 70, 71, 72, 73 ) --Audiobooks on CD
--AND item_view.itype_code_num IN ( 71 ) --Juvenile Audiobooks on CD
--AND item_view.itype_code_num IN ( 72 ) --Teen Audiobooks on CD

--AND item_view.itype_code_num IN ( 77, 78, 79 ) --Music on CD
--AND item_view.itype_code_num IN ( 78 ) --Juvenile Music on CD

--AND item_view.itype_code_num IN ( 20, 21, 22, 23, 24, 26, 27 ) --Large Print Books

--AND item_view.itype_code_num IN ( 90, 91, 92, 93, 94 ) --Playaways
--AND item_view.itype_code_num IN ( 91 ) --Juvenile Playaways
--AND item_view.itype_code_num IN ( 92 ) --Teen Playaways

--AND item_view.itype_code_num IN ( 100, 101 ) --Video
  
GROUP BY 
bib_record_item_record_link.bib_record_id, 
bib_view.record_num,
bib_view.title, 
bib_view.bcode2, 
callnum
  
ORDER BY sum_checkouts_ytd DESC
LIMIT 200
```