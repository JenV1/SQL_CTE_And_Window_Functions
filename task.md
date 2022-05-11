# Task - Employee Records

The major corporation BusinessCorp&#8482; wants to do some analysis of varioius metrics around its employees and have contracted the job out to you. They have provided you with two SQL files, you will need to write the queries which will extract the data they are looking for.

## Setup

- Create a database to run the files against
- Use the `psql -d database -f file` command to run the SQL files
- Before starting to write the queries take some time to look at the data and figure out the relationships between the tables - maybe even draw an ERD to help.

## Tasks

### CTEs

1) Calculate the average salary of all employees

```sql

SELECT AVG(salary) FROM employees;

```

2) Calculate the average salary of the employees in each team (hint: you'll need to `JOIN` and `GROUP BY` here)

```sql

SELECT 
	departments.name,
	ROUND(AVG(salary),2) AS average_team_salary
FROM employees
JOIN departments
ON employees.department_id = departments.id
GROUP BY departments.id;

```

3) Using a CTE find the ratio of each employees salary to their team average, eg. an employee earning £55000 in a team where the average is £50000 has a ratio of 1.1

```sql

WITH avg_team_salary_table (id, avg_salary) AS(
	SELECT 
		departments.id,
		ROUND(AVG(salary),2) AS average_team_salary 
	FROM employees
	JOIN departments
	ON employees.department_id = departments.id
	GROUP BY departments.id
)

SELECT 
	employees.first_name,
	employees.salary,
	avg_team_salary_table.avg_salary AS avg_department_salary,
	ROUND(employees.salary / avg_team_salary_table.avg_salary::NUMERIC, 3) AS ratio_to_avg
FROM employees
INNER JOIN avg_team_salary_table
ON employees.department_id = avg_team_salary_table.id;

```

4) Find the employee with the highest ratio in Argentina

```sql

WITH avg_team_salary_table (id, avg_salary) AS(
	SELECT 
		departments.id,
		ROUND(AVG(salary),2) AS average_team_salary 
	FROM employees
	JOIN departments
	ON employees.department_id = departments.id
	GROUP BY departments.id
)

SELECT 
	employees.first_name,
	employees.salary,
	employees.country,
	avg_team_salary_table.avg_salary AS avg_department_salary,
	ROUND(employees.salary / avg_team_salary_table.avg_salary::NUMERIC, 3) AS ratio_to_avg
FROM employees
INNER JOIN avg_team_salary_table
ON employees.department_id = avg_team_salary_table.id
WHERE country = 'Argentina'
ORDER BY ratio_to_avg DESC
LIMIT 1;

```

5) **Extension:** Add a second CTE calculating the average salary for each country and add a column showing the difference between each employee's salary and their country average

```sql

WITH avg_team_salary_table (id, avg_salary) AS(
	SELECT 
		departments.id,
		ROUND(AVG(salary),2) AS average_team_salary 
	FROM employees
	JOIN departments
	ON employees.department_id = departments.id
	GROUP BY departments.id
),

avg_country_salary_table (country, avg_country_salary) AS (
	SELECT
		employees.country,
		AVG(employees.salary)
	FROM employees
	GROUP BY employees.country
)

SELECT 
	employees.first_name,
	employees.salary,
	employees.country,
	avg_team_salary_table.avg_salary AS avg_department_salary,
	ROUND(employees.salary / avg_team_salary_table.avg_salary::NUMERIC, 3) AS ratio_to_avg_department_salary,
	ROUND(avg_country_salary, 2) AS avg_country_salary,
	ROUND(ABS(salary - avg_country_salary), 2) AS difference_between_country_avg_salary
FROM employees
INNER JOIN avg_team_salary_table
ON employees.department_id = avg_team_salary_table.id
INNER JOIN avg_country_salary_table
ON avg_country_salary_table.country = employees.country;

```

---

### Window Functions

1) Find the running total of salary costs as the business has grown and hired more people

```sql

SELECT 
	first_name,
	salary,
	start_date,
	SUM(salary) OVER (ORDER BY start_date) AS running_total_salary_costs
FROM employees;

```

2) Determine if any employees started on the same day (hint: some sort of ranking may be useful here)

```sql
WITH ranking_by_start_day (name, salary, start_date, rank) AS (	
	SELECT 
		first_name,
		salary,
		start_date,
		RANK() OVER (ORDER BY start_date) AS starting_rank
	FROM employees
)
	
SELECT
	rank, 
	COUNT(*) AS number_in_rank
FROM ranking_by_start_day 
GROUP BY rank
ORDER BY number_in_rank DESC
LIMIT 1;
```

3) Find how many employees there are from each country

```sql
SELECT 
	country,
	COUNT(*) AS number_of_employees
FROM employees
GROUP BY country;
```

4) Show how the average salary cost for each department has changed as the number of employees has increased

```sql
SELECT
	first_name,
	salary,
	department_id,
	ROUND(AVG(salary) OVER(PARTITION BY department_id ORDER BY start_date) ,2) AS running_department_avg_salary 
FROM employees
ORDER BY department_id, start_date;
```

5) **Extension:** Research the `EXTRACT` function and use it in conjunction with `PARTITION` and `COUNT` to show how many employees started working for BusinessCorp&#8482; each year. If you're feeling adventurous you could further partition by month...

```sql
SELECT 
	COUNT(*) AS number_of_employees_joined,
	EXTRACT(YEAR FROM start_date) AS year
FROM employees
GROUP BY year;

```

```sql

WITH employees_by_year_month (no_employees_year_month) AS (
	SELECT 
		COUNT(*) AS number_of_employees_joined,
		EXTRACT(YEAR FROM start_date) AS year,
		EXTRACT(MONTH FROM start_date) AS month
	FROM employees
	GROUP BY year, month
)

SELECT * FROM employees_by_year_month ORDER BY year;

```

---

### Combining the two

1) Find the maximum and minimum salaries

```sql
SELECT 
	MAX(salary) as max_salary,
	MIN(salary) as min_salary
FROM employees
```

2) Find the difference between the maximum and minimum salaries and each employee's own salary

```sql
WITH mins_and_maxs (name, salary, max_salary, min_salary) AS (	
	SELECT 
		first_name,
		salary,
		MAX(salary) OVER() AS max_salary,
		MIN(salary) OVER() AS min_salary
	FROM employees
)

SELECT 
	name,
	salary,
	max_salary,
	min_salary,
	max_salary - salary AS difference_between_max_salary,
	salary - min_salary AS difference_between_min_salary
FROM mins_and_maxs
ORDER BY difference_between_max_salary DESC;
```

3) Order the employees by start date. Research how to calculate the **median** salary value and the **standard deviation** in salary values and show how these change as more employees join the company

```sql
create function median_sfunc (
    state integer[], data integer
) returns integer[] as
$$
begin
    if state is null then
        return array[data]; -- if the array is null, return a singleton
    else
        return state || data; -- otherwise append to the existing array
    end if;
end;
$$ language plpgsql;

create function median_ffunc (
    state integer[]
) returns double precision as
$$
begin
    return (state[(array_length(state, 1) + 1)/ 2] 
        + state[(array_length(state, 1) + 2) / 2]) / 2.;
end;
$$ language plpgsql;

create aggregate median (integer) (
    sfunc     = median_sfunc,
    stype     = integer[],
    finalfunc = median_ffunc
    );


SELECT
	employees.first_name,
	employees.start_date,
	employees.salary,
	STDDEV(employees.salary) OVER (ORDER BY employees.start_date) AS rolling_salary_standard_deviation,
	median(employees.salary) OVER (ORDER BY employees.start_date) AS rolling_salary_median
FROM employees;

```

4) Limit this query to only Research & Development team members and show a rolling value for only the 5 most recent employees.

```sql

SELECT
	employees.first_name,
	employees.start_date,
	employees.salary,
	STDDEV(employees.salary) OVER (ORDER BY employees.start_date DESC) AS rolling_salary_standard_deviation,
	median(employees.salary) OVER (ORDER BY employees.start_date DESC) AS rolling_salary_median
FROM employees 
JOIN departments
ON departments.id = employees.department_id
WHERE departments.name = 'Research and Development'
ORDER BY start_date DESC 
LIMIT 5;
```

