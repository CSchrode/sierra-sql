# Holds for a title over a time interval
```sql
-- grab the transaction data for the title we want to look at
DROP TABLE IF EXISTS temp_holds_data;
CREATE TEMP TABLE temp_holds_data AS
SELECT
c.*

FROM
sierra_view.record_metadata as r

JOIN
sierra_view.bib_record as b
ON
  b.record_id = r.id

JOIN
sierra_view.circ_trans as c
ON
  c.bib_record_id = r.id

WHERE
r.record_num = 3325533
AND c.op_code = 'nb';
--


-- create the table for the intervals we'll look at data for
DROP TABLE IF EXISTS temp_date_ranges;
CREATE TEMP TABLE temp_date_ranges AS
SELECT
*
FROM
generate_series((
	-- start series
	SELECT
	date_trunc('hour', MIN(t.transaction_gmt)::timestamp)
	FROM
	temp_holds_data as t
	),
	-- end series
	(
	SELECT
	date_trunc('hour', MAX(t.transaction_gmt)::timestamp)
	FROM
	temp_holds_data as t
	),
	-- set a reasonable interval here ... maybe think about using a computed value if automating this
	'2 hours'  
) as date_target;
--


-- select our interval date target, and count the holds placed until that interval
SELECT
t.date_target,
(
	SELECT
	COUNT(*)

	FROM
	temp_holds_data as t1

	WHERE
	t1.transaction_gmt::timestamp <= t.date_target::timestamp
)

FROM
temp_date_ranges as t

ORDER BY
t.date_target ASC;
```


# Circulations from Main Grouped by Call Number (in 100 Dewey groups)

```sql
DROP TABLE IF EXISTS temp_call_groups;
CREATE TEMP TABLE temp_call_groups (
id SERIAL NOT NULL,
group_value VARCHAR(12),
group_name VARCHAR(512)
);
INSERT INTO temp_call_groups (group_value, group_name) VALUES 
-- these are the ten main classes
-- ('100', 'Philosophy & psychology'),
-- ('200', 'Religion'),
-- ('300', 'Social sciences'),
-- ('400', 'Language'),
-- ('500', 'Science'),
-- ('600', 'Technology'),
-- ('700', 'Arts & recreation'),
-- ('800', 'Literature'),
-- ('900', 'History & geography'),
--
-- these are the hundred divisions
('000', 'Computer science, knowledge & systems'),
('010', 'Bibliographies'),
('020', 'Library & information sciences'),
('030', 'Encyclopedias & books of facts'),
('040', '[Unassigned]'),
('050', 'Magazines, journals & serials'),
('060', 'Associations, organizations & museums'),
('070', 'News media, journalism & publishing'),
('080', 'Quotations'),
('090', 'Manuscripts & rare books'),
('100', 'Philosophy'),
('110', 'Metaphysics'),
('120', 'Epistemology'),
('130', 'Parapsychology & occultism'),
('140', 'Philosophical schools of thought'),
('150', 'Psychology'),
('160', 'Logic'),
('170', 'Ethics'),
('180', 'Ancient, medieval & eastern philosophy'),
('190', 'Modern western philosophy'),
('200', 'Religion'),
('210', 'Philosophy & theory of religion'),
('220', 'The Bible'),
('230', 'Christianity & Christian theology'),
('240', 'Christian practice & observance'),
('250', 'Christian pastoral practice & religious orders'),
('260', 'Christian organization, social work & worship'),
('270', 'History of Christianity'),
('280', 'Christian denominations'),
('290', 'Other religions'),
('300', 'Social sciences, sociology & anthropology'),
('310', 'Statistics'),
('320', 'Political science'),
('330', 'Economics'),
('340', 'Law'),
('350', 'Public administration & military science'),
('360', 'Social problems & social services'),
('370', 'Education'),
('380', 'Commerce, communications & transportation'),
('390', 'Customs, etiquette & folklore'),
('400', 'Language'),
('410', 'Linguistics'),
('420', 'English & Old English languages'),
('430', 'German & related languages'),
('440', 'French & related languages'),
('450', 'Italian, Romanian & related languages'),
('460', 'Spanish & Portuguese languages'),
('470', 'Latin & Italic languages'),
('480', 'Classical & modern Greek languages'),
('490', 'Other languages'),
('500', 'Science'),
('510', 'Mathematics'),
('520', 'Astronomy'),
('530', 'Physics'),
('540', 'Chemistry'),
('550', 'Earth sciences & geology'),
('560', 'Fossils & prehistoric life'),
('570', 'Life sciences; biology'),
('580', 'Plants (Botany)'),
('590', 'Animals (Zoology)'),
('600', 'Technology'),
('610', 'Medicine & health'),
('620', 'Engineering'),
('630', 'Agriculture'),
('640', 'Home & family management'),
('650', 'Management & public relations'),
('660', 'Chemical engineering'),
('670', 'Manufacturing'),
('680', 'Manufacture for specific uses'),
('690', 'Building & construction'),
('700', 'Arts'),
('710', 'Landscaping & area planning'),
('720', 'Architecture'),
('730', 'Sculpture, ceramics & metalwork'),
('740', 'Drawing & decorative arts'),
('750', 'Painting'),
('760', 'Graphic arts'),
('770', 'Photography & computer art'),
('780', 'Music'),
('790', 'Sports, games & entertainment'),
('800', 'Literature, rhetoric & criticism'),
('810', 'American literature in English'),
('820', 'English & Old English literatures'),
('830', 'German & related literatures'),
('840', 'French & related literatures'),
('850', 'Italian, Romanian & related literatures'),
('860', 'Spanish & Portuguese literatures'),
('870', 'Latin & Italic literatures'),
('880', 'Classical & modern Greek literatures'),
('890', 'Other literatures'),
('900', 'History'),
('910', 'Geography & travel'),
('920', 'Biography & genealogy'),
('930', 'History of ancient world (to ca. 499)'),
('940', 'History of Europe'),
('950', 'History of Asia'),
('960', 'History of Africa'),
('970', 'History of North America'),
('980', 'History of South America'),
('990', 'History of other areas'),
--
-- these are the special PLCH defined call numbers
('fiction', 'PLCH fiction'),
('m','PLCH music score'),
('easy', 'PLCH easy'),
('other', 'PLCH other'),
('', 'PLCH none')
;
-- select * from temp_call_groups;
DROP TABLE IF EXISTS temp_circs_from_main;
CREATE TEMP TABLE temp_circs_from_main AS
SELECT
extract (year from t.transaction_gmt) || 'W' || extract(week from t.transaction_gmt) as transaction_week,
(
	SELECT
-- 	v.field_content as callnumber
	regexp_matches(
		regexp_replace(trim(v.field_content), '(\|[a-z]{1})', '', 'ig'), -- get the call number strip the subfield indicators
		-- looking for :
		-- groups where 3 or more numbers in a row (dewey)
		-- words "fiction", "easy", or "other" appear anywhere in the string
		-- two or more letters appear in succession
		'((^[0-9]{3})|(^m.*)|(fiction.*)|(easy.*)|(other.*)|([a-z]{2,}))', 
		'gi'
	) AS call_number
	
	FROM
	sierra_view.varfield as v
	WHERE
	-- get the call number from the bib record
	v.record_id = l.bib_record_id
	AND v.varfield_type_code = 'c'
	ORDER BY 
	v.occ_num
	LIMIT 1
)[1] AS call_class,
count(t.item_record_id)
FROM
sierra_view.circ_trans AS t
LEFT OUTER JOIN
sierra_view.item_record AS i
ON
  i.record_id = t.item_record_id
LEFT OUTER JOIN
sierra_view.bib_record_item_record_link AS l
ON
  l.item_record_id = i.record_id
WHERE
-- transactions from last week
extract(week from t.transaction_gmt) = extract (week from (NOW() - interval '1 week'))
-- type of transaction is checkout 'o'
AND t.op_code = 'o'
-- item type is book (0) or music score (157)
AND i.itype_code_num IN (0,157)
-- item type is Juvenile Books (2, 22, 159)
-- AND i.itype_code_num IN (2, 22, 159)
-- item type is Teen Books (4, 24)
-- AND i.itype_code_num IN (4, 24)
-- from all Main locations
-- to get an updated list check here ...
-- https://github.com/plch/sierra-sql/wiki/Info#find-stat-group-aka-terminal-number-statistical-group-code-number-and-name
AND t.stat_group_code_num IN (
0,
1,
9,
10,
11,
12,
13,
14,
21,
31,
41,
42,
43,
44,
51,
61,
62
)
GROUP BY
call_class,
transaction_week;
-- FULL OUTER JOIN two temp tables to get all values to our final counts
SELECT 
(
	SELECT
	extract(year from NOW()) || 'W' || extract(week from (NOW() - interval '1 week'))
) as transaction_week,
* 
FROM 
(
	SELECT 
	g.group_value,
	g.group_name
	FROM
	temp_call_groups as g
) as t1
FULL OUTER JOIN
(
	SELECT 
	CASE 	
		WHEN m.call_class ~ '^[0-9]{3}' THEN substring(m.call_class from 1 for 2) || '0'
		WHEN m.call_class ~* '^m[0-9]{1}.*' THEN lower(substring(m.call_class from 1 for 1))
		ELSE lower(m.call_class)
	END as class,
	SUM(m.count)
	FROM 
	temp_circs_from_main AS m
	GROUP BY
	class
) AS t2
ON t2.class = t1.group_value
ORDER BY
t1.group_value,
t2.class;
```