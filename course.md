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

Bad query:
```sql
select
    e.id, concat(e.first_name, ' ', e.last_name) AS employee_full_name, e.first_name, d.id AS department_id, d.name AS last_department_name
from employee e
inner join ( select der.employee_id, max(der.id) AS max_id
    from department_employee_rel der
    where der.deleted_flag = 0
    group by der.employee_id
) derm ON derm.employee_id = e.id
inner join department_employee_rel der ON der.id = derm.max_id
    and der.deleted_flag = 0
inner join department d ON d.id = der.department_id
    and d.deleted_flag = 0
where e.id IN (10010, 10040, 10050, 91050, 205357)
    and e.deleted_flag = 0
limit 100;
```
Good query:
```sql
SELECT /* Select last department of employees */
	"employee"."id" AS employee_id, /* NOTE: added alias */
	CONCAT("employee"."first_name", ' ', "employee"."last_name") AS employee_full_name,
	"employee"."first_name" AS first_name,
	"department"."id" AS department_id,
	"department"."name" AS last_department_name

FROM "sample_staff"."employee"

INNER JOIN (
	SELECT /* Select the last department ID per employee */
		"department_employee_rel"."employee_id",
		MAX("department_employee_rel"."id") AS max_id
	FROM "sample_staff"."department_employee_rel"
	WHERE 1=1
		AND "department_employee_rel"."deleted_flag" = 0
	GROUP BY
		"department_employee_rel"."employee_id"
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

### Best Practices V2
* Store code in a git repository
* Do not use "select *"
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


