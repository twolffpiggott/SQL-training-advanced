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
### Coding Practice
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

## Partitions

Types of partitions:
* Partition by range
* Partition by list
* Partition by hash
* Partition by keys

Partition by range:
```sql
CREATE TABLE "invoice" (
  "id" int(11) unsigned NOT NULL AUTO_INCREMENT,
  "employee_id" int(11) unsigned NOT NULL DEFAULT '0',
  "invoiced_date" date NOT NULL,
  "paid_flag" tinyint(4) NOT NULL DEFAULT '0',
  "insert_dt" datetime NOT NULL,
  "insert_user_id" int(11) NOT NULL DEFAULT '-1',
  "insert_process_code" varchar(255) DEFAULT NULL,
  "update_dt" timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  "update_user_id" int(11) NOT NULL DEFAULT '-1',
  "update_process_code" varchar(255) DEFAULT NULL,
  "deleted_flag" tinyint(4) NOT NULL DEFAULT '0',
  PRIMARY KEY ("id", "invoiced_date")
) ENGINE=InnoDB
	DEFAULT CHARSET=utf8
    PARTITION BY RANGE( YEAR(invoiced_date) )
    SUBPARTITION BY HASH( MONTH(invoiced_date) )
    SUBPARTITIONS 12 (
        PARTITION p1984 VALUES LESS THAN (1985),
        PARTITION p1985 VALUES LESS THAN (1986),
        PARTITION p1986 VALUES LESS THAN (1987),
        PARTITION p1987 VALUES LESS THAN (1988),
        PARTITION p1988 VALUES LESS THAN (1989),
        PARTITION p1989 VALUES LESS THAN (1990),
        PARTITION p1990 VALUES LESS THAN (1991),
        PARTITION p1991 VALUES LESS THAN (1992),
        PARTITION p1992 VALUES LESS THAN (1993),
        PARTITION p1993 VALUES LESS THAN (1994),
        PARTITION pOTHER VALUES LESS THAN MAXVALUE
    )
;
```
Partition by list:
```sql
CREATE TABLE employee (
    id INT NOT NULL,
    store_id INT,
		...
)
PARTITION BY LIST(store_id) (
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);
```

### Coding Practice

Create a new table `sample_staff`.`invoice_partitioned` based on `sample_staff`.`invoice`, but change the following:
* add one more column: `department_code`
* remove the current partitions & sub-partitions

Then, copy data from `invoice` to the new table and also fill in the new column based on the department which the user was a part at the time of `invoiced_date`.

Add new `LIST` partitioning to `invoice` based on the `department_code` (see `sample_staff`.`department`.`code`).

```sql
CREATE TABLE "sample_staff"."invoice_partitioned"
SELECT
  "invoice"."id",
  "invoice"."employee_id",
  "invoice"."invoiced_date",
  "invoice"."paid_flag",
  "invoice"."insert_dt",
  "invoice"."insert_user_id",
  "invoice"."insert_process_code",
  "invoice"."update_dt",
  "invoice"."update_user_id",
  "invoice"."update_process_code",
  "invoice"."deleted_flag",
  "department"."code" as department_code
FROM "sample_staff"."invoice"
INNER JOIN "sample_staff"."department_employee_rel" ON 1=1
  AND "invoice"."employee_id" = "department_employee_rel"."employee_id"
  AND "invoice"."invoiced_date" BETWEEN "department_employee_rel"."from_date" AND IFNULL("department_employee_rel"."to_date", '2002-08-01')
  -- the IFNULL function allows an alternative value to be returned if an expression evaluates as NULL
  AND "department_employee_rel"."deleted_flag" = 0
INNER JOIN "sample_staff"."department" ON 1=1
  AND "department"."id" = "department_employee_rel"."department_id";

-- check whether table is partitioned (shows info for all tables)
SHOW TABLE STATUS;

SELECT DISTINCT("department"."code")
FROM "sample_staff"."department"
LIMIT 50;

SHOW COLUMNS
FROM "sample_staff"."invoice_partitioned";

ALTER TABLE "sample_staff"."invoice_partitioned"
PARTITION BY LIST COLUMNS (department_code) (
  PARTITION pDepts1 VALUES IN ('MKT', 'HR', 'PROD'),
  PARTITION pDepts2 VALUES IN ('FIN', 'RES', 'QA'),
  PARTITION pDepts3 VALUES IN ('SAL', 'DEV', 'CS')
);
```

## Variables

Types of variables:

* **Global** - server system variables `@@version`
* **Session** - user-defined variables `@var`
* **Local** - declared in a function or procedure `DECLARE var2 INT`;

### Session variables
Scoped within a session, defined by user.
```sql
SELECT @var3 := IFNULL(@var3, 0) + 2;

SET @var4 = (SELECT COUNT(*) FROM "sample_staff"."employee");
SELECT @var4;
```

Using Session variables enables flexible queries:
```sql
SELECT
	"date"."date",
	@day_of_week := DAYOFWEEK(date.date) AS day_of_week,
	CASE
		WHEN @day_of_week = 2 /* Monday */ THEN 'Thanks God Its Monday!'
		ELSE 'Good morning'
	END AS welcome_message,
	CASE
		WHEN @day_of_week = 6 /* Friday */ THEN 'Have a great weekend!'
		ELSE 'Good bye'
	END AS good_bye_message
FROM "sample_staff"."date"
WHERE 1=1
	AND "date"."date" BETWEEN '2016-01-01' AND '2016-01-14';

SET @employee_id = '10001';

SELECT
  "salary"."id",
  "salary"."employee_id",
  "salary"."salary_amount"
FROM "sample_staff"."salary"
WHERE 1=1
  AND "salary"."employee_id" = @employee_id;

-- Define the variables
SET @employee_id = 10001;
SET @columns = 'employee_id, salary_amount';
SET @table_name = 'salary';

-- Compose the query
SET @select_query = CONCAT('SELECT id, ', @columns, ' FROM ',
			@table_name, ' WHERE employee_id = ', @employee_id);

-- Prepare & execute
PREPARE stmt FROM @select_query;
EXECUTE stmt;
```

### Coding Practice

```sql
SET @company_average = (SELECT AVG("salary_amount") FROM "sample_staff"."salary");
SET @year_month =  '2000-01-01';

SELECT
  @year_month AS "year_month",
  "department"."id",
  "department"."name",
  AVG("salary_subset"."salary_amount") AS "department_average_salary",
  @company_average AS "company_average_salary"
FROM
  (
  SELECT
    "salary"."salary_amount",
    "salary"."employee_id"
  FROM "sample_staff"."salary"
  WHERE 1=1
    AND "salary"."from_date" <=  STR_TO_DATE('2001-01-01', '%Y-%m-%d')
    AND "salary"."to_date" >=  STR_TO_DATE('2001-01-01', '%Y-%m-%d')
  ) AS "salary_subset"
INNER JOIN
  (
  SELECT
    "department_employee_rel"."department_id",
    "department_employee_rel"."employee_id"
  FROM "sample_staff"."department_employee_rel"
  WHERE 1=1
    AND "department_employee_rel"."from_date" <=  STR_TO_DATE('2001-01-01', '%Y-%m-%d')
    AND "department_employee_rel"."to_date" >=  STR_TO_DATE('2001-01-01', '%Y-%m-%d')
  ) AS "dept_empl_subset"
ON 1=1
  AND "salary_subset"."employee_id" = "dept_empl_subset"."employee_id"
INNER JOIN
  "sample_staff"."department"
ON 1=1
  AND "dept_empl_subset"."department_id" = "department"."id"
GROUP BY "department"."id";
```

## Analytic (window) functions

Not supported in MYSQL 5.7. Like window functions such as `ROW_NUMBER` in Postgres. Window/analytic functions are available in MYSQL as of v8.0.2. See [here](https://mysqlserverteam.com/mysql-8-0-2-introducing-window-functions/) for a good discussion: 'a window function can be thought of as just another SQL function, except that its value is based on the value of other rows in addition to the values of the for which it is called, i.e. they function as a window into other rows.'

Window functions can be emulated using variables:
```sql
SET @dummy_row_number = 0;

SELECT
  @dummy_row_number := @dummy_row_number + 1 as "dummy_row_number",
  "department".*
FROM "sample_staff"."department";
```
Above the row number assignment implicitly depends on the ordering and values assigned to previous rows.

More excellent insight into groupwise selection in MYSQL in [this blog post](https://www.xaprb.com/blog/2006/12/07/how-to-select-the-firstleastmax-row-per-group-in-sql/). Following from which, an efficient way to select the top N rows from each group:

```sql
set @num := 0, @type := '';

select type, variety, price
from (
   select type, variety, price,
      @num := if(@type = type, @num + 1, 1) as row_number,
      @type := type as dummy
  from fruits
  order by type, price
) as x where x.row_number <= 2;
```

### Coding practice
Draft until `sample_staff.user-stat` table is populated.

## Functions
Cases like the below are effective, but can be cumbersome when translating across tables and into different contexts.

```sql
SELECT /* Is multinight? */
	@checkin_date := '2016-05-09' AS checkin_date,
	@checkout_date := '2016-05-10' AS checkout_date,
	CASE
		WHEN DATEDIFF(@checkout_date, @checkin_date) = 1 THEN 0
		ELSE 1
	END AS is_multinight
;
```
This is where functions come in- to wrap this into a more flexible form.
```sql
DROP FUNCTION IF EXISTS "FC_IS_MULTINIGHT";

-- the delimiter is temporarily changed so the function can be properly defined using the old delimeter without prematurely signalling to the engine that the statement is over.

DELIMITER //

CREATE FUNCTION "FC_IS_MULTINIGHT"(
	checkin_date DATE,
	checkout_date DATE
) RETURNS TINYINT(1)
BEGIN
	RETURN CASE
	  WHEN DATEDIFF(checkout_date, checkin_date) = 1 THEN FALSE
	  ELSE TRUE
	END;
END;
//

DELIMITER ;
```
The function can now be used in the general case:
```sql
SELECT
	@checkin_date := '2016-05-09' AS checkin_date,
	@checkout_date := '2016-05-10' AS checkout_date,
  FC_IS_MULTINIGHT(@checkin_date, @checkout_date) AS is_multinight
;
```

### Coding standards for functions and Procedures

* Always use `DELIMITER`, even if it's not necessary
* Split parameters on a new line
* Uppercase keywords, lowercase parameters, variables and table/column names
* Uppercase function and procedure names
* Comment a lot
* Add prefixes
  * Functions: `FC_`
  * Procedures: `INS_` / `DEL_` / `UPD_` / `SEL_`
* When saving code to repository, add delimiter & `DELETE ... IF EXISTS`

### Coding Practice

Distance between two coordinates:
```sql
DROP FUNCTION IF EXISTS "FC_HAVERSINE_DISTANCE";
DELIMITER //
CREATE FUNCTION "FC_HAVERSINE_DISTANCE"(
  lat1 FLOAT(23,19),
  lon1 FLOAT(23,19),
  lat2 FLOAT(23,19),
  lon2 FLOAT(23,19)
) RETURNS FLOAT(23,19)
BEGIN
  DECLARE r INT;
  DECLARE dlat FLOAT(23,19);
  DECLARE dlon FLOAT(23,19);
  DECLARE a FLOAT(23,19);
  DECLARE c FLOAT(23,19);
  DECLARE d FLOAT(23,19);

  SET r = 6371;
  SET dlat = RADIANS(lat2-lat1);
  SET dlon = RADIANS(lon2-lon1);
  SET a = SIN(dlat/2) * SIN(dlat/2) + COS(RADIANS(lat1)) * COS(RADIANS(lat2)) * SIN(dlon/2) * SIN(dlon/2);
  SET c = 2 * ATAN(SQRT(a) / SQRT(1-a));
  SET d = r * c;
  RETURN  d;
END;
//
DELIMITER ;

-- distance between Cape Town and Stellenbosch
SELECT FC_HAVERSINE_DISTANCE(-33.924869, 18.424055, -33.932105, 18.860152);
```
