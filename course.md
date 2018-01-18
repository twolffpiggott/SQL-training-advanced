## Coding Style
### Best Practices V1
* Use inline comments
* Use tabs instead of spaces
* Use 1=1
* Split column names to new lines
* Add empty lines for complex queries
* Do not shorten table name aliases
* Use the backtick (`) before and after table & column names

**Note that backticks are not ANSI-SQL compliant, which implies that they do
not work in SQL Server.**
 Setting:
```sql
SET GLOBAL sql_mode='ANSI';
SET SESSION sql_mode='ANSI';
```
allows double quotes to be used in place of backticks for cross-database compatibility.

### Best Practices V2
* Store code in a git repository
* Do not use "select <asterisk>"
* Separate attributes (columns) to rows
* Set naming convention and don’t allow exceptions
* Be descriptive, don’t use acronyms
* Use audit columns
* Batch delete & updates (example)
* Reference the owner of an object
* Table names always singular
* Use "WHERE 1=1"
* Old vs. new JOIN style
* Prefix database objects
* Don’t use column rows in ORDER BY
* Use LIMIT 1 as much as possible (example)
* Use correct data type, it makes a difference

### Coding Practice
Bad query:
```sql
select e.id AS employee_id, concat(e.first_name, ' ', e.last_name) AS employee_full_name, d.id AS department_id, d.name AS last_department_name from employee e inner join ( select der.employee_id, max(der.id) AS max_id from department_employee_rel der where der.deleted_flag = 0 group by der.employee_id ) derm ON derm.employee_id = e.id inner join department_employee_rel der ON der.id = derm.max_id and der.deleted_flag = 0 inner join department d ON d.id = der.department_id and d.deleted_flag = 0 where e.id IN (10010, 10040, 10050, 91050, 205357) and e.deleted_flag = 0 limit 100;
```
Good query:
```sql
SELECT  
  "employee"."id" AS employee_id,
  CONCAT("employee"."first_name", ' ', "employee"."last_name") AS employee_full_name,
  "department"."id" as department_id,
  "department"."name" as department_name

FROM "sample_staff"."employee"

INNER JOIN(
  SELECT
    "department_employee_rel"."employee_id",
    max("department_employee_rel"."id") AS max_id
  FROM "sample_staff"."department_employee_rel"
  WHERE 1=1
    AND "department_employee_rel"."deleted_flag" = 0
  GROUP BY "department_employee_rel"."employee_id"
) "department_employee_rel_max" ON 1=1
  AND "department_employee_rel_max"."employee_id" = "employee"."id"

INNER JOIN "sample_staff"."department_employee_rel" ON 1=1
	AND "department_employee_rel"."id" = "department_employee_rel_max"."max_id"
	AND "department_employee_rel"."deleted_flag" = 0 /* Make sure to exclude deleted entities */

INNER JOIN "sample_staff"."department" ON 1=1
	AND "department"."id" = "department_employee_rel"."department_id"
	AND "department"."deleted_flag" = 0 /* Make sure to exclude deleted entities */

WHERE 1=1
	AND "employee"."id" IN (10010, 10040, 10050, 91050, 205357) /* A list of employee_id's */
	AND "employee"."deleted_flag" = 0 /* Make sure to exclude deleted entities */

LIMIT 100
;
```
Create a new view v_user_login which will select user's recent logins. Attributes to select:

* user_login.id
* user_login.user_id
* user.name
* user_login.ip_address
* user_login.ip_address (show in a standard notation xxx.xxx.xxx.xxx)
* user_login.login_dt
```sql
CREATE OR REPLACE VIEW "sample_staff"."v_user_login" AS
  SELECT
    "user_login"."id" AS "user_login_id",
    "user_login"."user_id",
    "user"."name" AS "user_name",
    "user_login"."ip_address" AS "ip_address_integer",
    INET_NTOA("user_login"."ip_address") AS "ip_address",
    "user_login"."login_dt"
  FROM "sample_staff"."user_login"
  INNER JOIN "sample_staff"."user" ON 1=1
    AND "user"."id" = "user_login"."user_id"
  WHERE 1=1
    AND "user_login"."deleted_flag" = 0
  ORDER BY
    "user_login"."id" DESC;
  ```
  Bad query:
  ```sql
  insert into bi_data.valid_offers (offer_id, hotel_id, price_usd, original_price, original_currency_code,
        checkin_date, checkout_date, breakfast_included_flag, valid_from_date, valid_to_date)
  select of.id, of.hotel_id, of.sellings_price as price_usd, of.sellings_price as original_price,
      lc.code AS original_currency_code, of.checkin_date, of.checkout_date, of.breakfast_included_flag,
      of.offer_valid_from, of.offer_valid_to
  from  enterprise_data.offer_cleanse_date_fix of, primary_data.lst_currency lc
  where of.currency_id=1 and lc.id=1;
  ```
Good query:
```sql
INSERT INTO "bi_data"."valid_offers" ("offer_id", "hotel_id", "price_usd", "original_price", "original_currency_code",
      "checkin_date", "checkout_date", "breakfast_included_flag", "valid_from_date", "valid_to_date")
SELECT
  "offer_cleanse_date_fix"."id",
  "offer_cleanse_date_fix"."hotel_id",
  "offer_cleanse_date_fix"."sellings_price" AS "price_usd",
  "offer_cleanse_date_fix"."sellings_price" AS "original_price",
  "lst_currency"."code" AS "original_currency_code",
  "offer_cleanse_date_fix"."checkin_date",
  "offer_cleanse_date_fix"."checkout_date",
  "offer_cleanse_date_fix"."breakfast_included_flag",
  "offer_cleanse_date_fix"."offer_valid_from",
  "offer_cleanse_date_fix"."offer_valid_to"
FROM "enterprise_data"."offer_cleanse_date_fix"
INNER JOIN "primary_data"."lst_currency" ON 1=1
  AND "offer_cleanse_date_fix"."currency_id" = "lst_currency"."id";
```

## Indices
### Unique Indices
Add a unique key:
```sql
ALTER TABLE "department" ADD UNIQUE INDEX "ak_department" ("name")
```
Handling a merge resulting in duplicates for a unique key:
```sql
SELECT	*	/*	Check	the	salary	of	employee	with	ID	=	499998	*/
FROM	"salary"
WHERE	1=1
  AND	"employee_id"	=	499998
  AND	"from_date"	=	'1993-12-27'
  AND	"to_date"	=	'1994-12-27'
;

INSERT	INTO	"salary"	("employee_id",	"from_date",	"to_date",	"insert_dt",
"insert_process_code")
VALUES
(499998,	'1993-12-27',	'1994-12-27',	NOW(),	'merge-insert')
ON	DUPLICATE	KEY	UPDATE
		"salary"."salary_amount"	=	"salary"."salary_amount"	*	10,
		"salary"."update_process_code"	=	'merge-update'
;
```

### Composite Indices
A composite index indexes data in a tree-like structure- splitting first on one variable, then another, and so on.
```sql
SELECT
  "department_employee_rel".*
FROM	"department_employee_rel"
WHERE	1=1
  AND	"department_employee_rel"."department_id"	=	3
  AND	"department_employee_rel"."employee_id"	IN	(10005,	10006,	10007)
  AND	"department_employee_rel"."from_date"	=	'1989-09-12'
  AND	"department_employee_rel"."to_date"	IS	NULL
;
```
Check	salary table	-	it	doesn't	have	a	single	index	on employee_id .
But	there's	a	composite	index	on	multiple	attributes:
* employee_id
* from_date
* to_date
```sql
SELECT	*	/*	Ignore	index	-	2-3	seconds	*/
FROM	"salary" IGNORE	INDEX	("ak_salary")
WHERE	1=1
	AND	"salary"."employee_id"	=	499998
;
SELECT	*	/*	Use	index	-	very	fast	*/
FROM	"salary"	USE	INDEX	("ak_salary")
WHERE	1=1
	AND	"salary"."employee_id"	=	499998
;
```
If you have a three-column index on `(col1, col2, col3)`, you have indexed search capabilities on

* `(col1)`
* `(col1, col2)`
* and `(col1, col2, col3)`

The order of the columns in an index matters if you'd like to query partially. For instance, searching on `(col2, col3)` would not use the index.

### Partial Index
Instead of indexing on an entire last name, you could index on the first 4 bytes:
```sql
ALTER TABLE phone_book ADD INDEX (last_name(4))
```
For
```sql
USE sample_ip;

SHOW INDEX FROM ip_address_varchar20;

-- Delete indexes if there are any
ALTER TABLE "ip_address_varchar20" DROP INDEX "idx_ip_address_3chars";
ALTER TABLE "ip_address_varchar20" DROP INDEX "idx_ip_address_7chars";
ALTER TABLE "ip_address_varchar20" DROP INDEX "idx_ip_address_all_chars";

```
Create indexes in `sample_ip` database, table `ip_address_varchar20`.

```sql
-- First 3 characters
CREATE INDEX idx_ip_address_3chars ON sample_ip.ip_address_varchar20 (ip_address(3));

-- First 7 characters (xxx.xxx)
CREATE INDEX idx_ip_address_7chars ON sample_ip.ip_address_varchar20 (ip_address(7));

-- All characters (xxx.xxx.xxx.xxx)
CREATE INDEX idx_ip_address_all_chars ON sample_ip.ip_address_varchar20 (ip_address);
```

Now see the difference in performance - run the queries below.

```sql
-- Execution time: 0.65 seconds
SELECT *
FROM "sample_ip"."ip_address_varchar20" IGNORE INDEX ("idx_ip_address_3chars") IGNORE INDEX ("idx_ip_address_7chars") IGNORE INDEX ("idx_ip_address_all_chars")
WHERE 1=1
	AND "ip_address_varchar20"."ip_address" = '123.194.160.219'
;

-- Execution time: 0.16 seconds
SELECT *
FROM "sample_ip"."ip_address_varchar20" USE INDEX ("idx_ip_address_3chars")
WHERE 1=1
	AND "ip_address_varchar20"."ip_address" = '123.194.160.219'
;

-- Execution time: 20ms
SELECT *
FROM "sample_ip"."ip_address_varchar20" USE INDEX ("idx_ip_address_7chars")
WHERE 1=1
	AND "ip_address_varchar20"."ip_address" = '123.194.160.219'
;

-- Execution time: 1ms
SELECT *
FROM "sample_ip"."ip_address_varchar20" USE INDEX ("idx_ip_address_all_chars")
WHERE 1=1
	AND "ip_address_varchar20"."ip_address" = '123.194.160.219'
;
```

### Index hints
Main index hints for the SQL engine:
* `USE INDEX`
* `FORCE INDEX`
* `IGNORE INDEX`
`USE INDEX` and `FORCE INDEX` are in practice equivalent- the SQL engine seldom disregards the `USE INDEX` hint.
```sql
EXPLAIN SELECT * /* Check the salary of employee with ID = 499998 */
FROM "sample_staff"."salary" IGNORE INDEX ("ak_salary") IGNORE INDEX ("idx_employee_id")
WHERE 1=1
	AND "salary"."employee_id" = 499997
	AND "salary"."from_date" = '1993-12-27'
	AND "salary"."to_date" = '1994-12-27'
;

-- Add a new index
ALTER TABLE "salary" ADD INDEX "idx_employee_id" ("employee_id");

EXPLAIN SELECT * /* Check the salary of employee with ID = 499998 */
FROM "sample_staff"."salary" USE INDEX ("ak_salary")
WHERE 1=1
	AND "salary"."employee_id" = 499998
	AND "salary"."from_date" = '1993-12-27'
	AND "salary"."to_date" = '1994-12-27'
;

EXPLAIN SELECT * /* Check the salary of employee with ID = 499998 */
FROM "sample_staff"."salary" USE INDEX ("idx_employee_id")
WHERE 1=1
	AND "salary"."employee_id" = 499997
	AND "salary"."from_date" = '1993-12-27'
	AND "salary"."to_date" = '1994-12-27'
;
```
** When functions are used to transform the data, the SQL engine doesn't use the index. i.e. don't transform data on the fly if you want to use a given index. **

### Using two indexes

```sql
/* "EXPLAIN SELECT" shows that the engine uses the single key "ak_employee". */
SELECT
	employee.id,
	employee.personal_code
FROM employee
WHERE (
		employee.personal_code = '7C-91159'
--		OR
--    id BETWEEN 12340 AND 12400
	)
;
/* "EXPLAIN SELECT" shows that the engine chose "UNION". */
SELECT
	employee.id,
	employee.personal_code
FROM employee
WHERE (
		employee.personal_code = '7C-91159'
		OR
    id BETWEEN 12340 AND 12400
	)
;
```
`EXPLAIN SELECT` shows that the engine chose `UNION`. This is equivalent to the following:
```sql
SELECT
	"employee"."id",
	"employee"."personal_code"
FROM "employee"
WHERE "employee"."personal_code" = '7C-91159'

UNION ALL

SELECT
	"employee"."id",
	"employee"."personal_code"
FROM "employee"
WHERE "id" BETWEEN 12340 AND 12400;
```
## Coding Practice
```sql
CREATE INDEX idx_personal_code_2chars ON "sample_staff"."employee" (personal_code(2));

EXPLAIN SELECT "employee"."personal_code"
FROM "employee" USE INDEX ("ak_employee")
WHERE "employee"."personal_code" = 'AA-751492';
-- 1 miliseconds execution

EXPLAIN SELECT "employee"."personal_code"
FROM "employee" USE INDEX ("idx_personal_code_2chars")
WHERE "employee"."personal_code" = 'AA-751492';
-- 10 miliseconds execution

-- number of pages in the index
SELECT	/*	Select	all	indexes	from	table	'employee'	and	their	size	*/
  sum("stat_value")	AS	pages,
  "index_name"	AS	index_name,
	sum("stat_value")	*	@@innodb_page_size	/	1024	/	1024 AS	size_mb
FROM	"mysql"."innodb_index_stats"
WHERE	1=1
  AND	"table_name"	=	'employee'
	AND	"database_name"	=	'sample_staff'
	AND	"stat_description"	=	'Number	of	pages	in	the	index'
GROUP	BY
  "index_name"
;
```
Optimise the following queries:
```sql
-- Original query 1
EXPLAIN SELECT "contract"."archive_code"
FROM "contract"
WHERE 1=1
 AND "contract"."archive_code" = 'DA970'
 AND "contract"."deleted_flag" = 0
 AND "contract"."sign_date" >= '1990-01-01'
;
-- Modifications
SHOW INDEX FROM "contract";
ALTER TABLE "contract" ADD INDEX "idx_archive_code" ("archive_code");
ALTER TABLE "contract" ADD INDEX "idx_sign_date" ("sign_date");
ALTER TABLE "contract" ADD INDEX "idx_deleted_flag" ("deleted_flag");
-- alternatively use composite index
ALTER	TABLE	contract ADD INDEX	idx_archive_code_sign_date (archive_code,sign_date);
-- New query 1
EXPLAIN SELECT "contract"."archive_code"
FROM "contract" USE INDEX ("idx_archive_code")
WHERE 1=1
 AND "contract"."archive_code" = 'DA970'
 AND "contract"."deleted_flag" = 0
 AND "contract"."sign_date" >= '1990-01-01'
;
-- Original query 2
EXPLAIN SELECT "contract"."archive_code"
FROM "contract"
WHERE 1=1
 AND "contract"."archive_code" = 'DA970'
 AND "contract"."deleted_flag" = 0
;
```
