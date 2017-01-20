# Work like a DB :)

This is site based on test DB, to run some example SQL queries.



## How to use

Site is divided into *two different parts* as tables, queries.

### Tables

Here is DB scheme:

![scheme](readme/scheme.png)

You are able to view content of particular table, insert new data into it, update
existing data or delete it.

Here is how `departments` table looks like:

![departments table view](readme/dept_table.png)

You can **delete** some rows from table by clicking on 'X' icon in 'trash' column.

To **insert** or **update** rows you should go to tab called 'Modify' on the right side.
You will be asked to enter some information neccessary to process queries.

![dept\_manager table modification](readme/dept_manager_mod.png)

**NOTE!** You need to fill all the fields with values to make modification.

### Queries

There are 6 simple queries and 3 set comparison queries provided as examples.
To run query you need to go to tab called 'Queries' and select query you want to run.

Let's have a look at example query 5.

![example query 5](readme/example_query_5.png)

As you can see queries are parametrized. In this particular case we need to provide
title, start and end date to get number of coworkers with given title per department.

Fortunatelly, **default values are good enough** to run this query, so let's try it.

![example query 5 result](readme/example_query_5_res.png)

To get back to 'Queries' page you can always use top navigation bar.

That's it for now. Hope this example is useful :)

## Example queries

**NOTE!** All the queries as parametrized with date window `[from_date, to_date]`.

1. Select employees of `{$gender}` and their salaries.

	```sql
	SELECT first_name, last_name, gender, average_salary
	FROM employees
		JOIN (
			SELECT emp_no, AVG(salary) AS average_salary
			FROM salaries
			WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			GROUP BY emp_no
		) AS emp_salary
			ON employees.emp_no = emp_salary.emp_no
	WHERE gender = '{$gender}'
	ORDER BY average_salary DESC
	LIMIT 20;
	```

2. Select managers by department.

	```sql
	SELECT dept_name, first_name, last_name, from_date, to_date
	FROM departments
		JOIN dept_manager
			ON departments.dept_no = dept_manager.dept_no
		JOIN employees
			ON dept_manager.emp_no = employees.emp_no
	WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
	ORDER BY dept_name
	LIMIT 20;
	```

3. Get average salary per title.

	```sql
	SELECT title, AVG(avg_salary) AS average_salary
	FROM titles
		JOIN (
			SELECT emp_no, AVG(salary) AS avg_salary
			FROM salaries
			WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			GROUP BY emp_no
		) AS emp_avg
			ON titles.emp_no = emp_avg.emp_no
	GROUP BY title
	ORDER BY average_salary DESC
	LIMIT 20;
	```

4. Minimal salary of employee with `{$emp_no}`.

	```sql
	SELECT first_name, last_name, min_salary
	FROM employees
		JOIN (
			SELECT emp_no, MIN(salary) AS min_salary
				FROM salaries
				WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
				GROUP BY emp_no
		) AS emp_min_salary
			ON employees.emp_no = emp_min_salary.emp_no
	ORDER BY min_salary
	LIMIT 20;
	```

5. Number of employees with `{$title}` per department.

	```sql
	SELECT dept_name, COUNT(dept_emp.emp_no) AS emp_number
	FROM departments
		JOIN dept_emp
			ON departments.dept_no = dept_emp.dept_no
		JOIN titles
			ON titles.emp_no = dept_emp.emp_no
	WHERE dept_emp.from_date >= '{$from_date}' AND dept_emp.to_date <= '{$to_date}'
		AND titles.from_date >= '{$from_date}' AND titles.to_date <= '{$to_date}'
		AND title = '{$title}'
	GROUP BY dept_name
	ORDER BY emp_number DESC
	LIMIT 20;
	```
6. List titles of particular employee with `{$emp_no}`.

	```sql
	SELECT title, from_date, to_date
	FROM titles
	WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
		AND emp_no = '{$emp_no}'
	ORDER BY from_date DESC
	LIMIT 20;
	```

## Set comparisons

1. Employees with some titles of `{$emp_no}`.

	```sql
	SELECT emp_no, first_name, last_name
	FROM employees AS e
	WHERE NOT EXISTS (
		SELECT *
		FROM titles
		WHERE emp_no = e.emp_no AND from_date >= '{$from_date}' AND to_date <= '{$to_date}' AND title NOT IN (
			SELECT title
			FROM titles
			WHERE emp_no = {$emp_no} AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
		)
	)
	ORDER BY emp_no
	LIMIT 20;
	```

2. Employees working at the same department as `{$emp_no}`.

	```sql
	SELECT e.emp_no, first_name, last_name
	FROM dept_emp as e
		JOIN employees
			ON e.emp_no = employees.emp_no
	WHERE from_date >= '{$from_date}' AND to_date <= '{$to_date}'
		AND NOT EXISTS (
			SELECT *
			FROM dept_emp
			WHERE emp_no = {$emp_no} AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
				AND dept_no NOT IN (
					SELECT dept_no
					FROM dept_emp
					WHERE emp_no = e.emp_no AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
				)
		) AND NOT EXISTS (
			SELECT *
			FROM dept_emp
			WHERE emp_no = e.emp_no AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			AND dept_no NOT IN (
				SELECT dept_no
				FROM dept_emp
				WHERE emp_no = {$emp_no} AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			)
		)
	ORDER BY e.emp_no
	LIMIT 20;
	```

3. Find employees managed by some of managers of `{$emp_no}`.

	```sql
	SELECT e.emp_no, first_name, last_name, dept_no
	FROM dept_manager AS e
		JOIN employees
			ON e.emp_no = employees.emp_no
	WHERE NOT EXISTS (
		SELECT *
		FROM dept_manager
		WHERE emp_no = '{$emp_no}' AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
		AND dept_no NOT IN (
			SELECT dept_no
			FROM dept_manager
			WHERE emp_no = e.emp_no AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
		)
	)
	ORDER BY emp_no
	LIMIT 20;
	```

4. Find employees whose salaries are superset of `{$emp_no}`.

	```sql
	SELECT emp_no, first_name, last_name
	FROM employees AS e
	WHERE NOT EXISTS (
		SELECT *
		FROM salaries
		WHERE emp_no = '{$emp_no}' AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			AND salary NOT IN (
				SELECT salary
				FROM salaries
				WHERE emp_no = e.emp_no AND from_date >= '{$from_date}' AND to_date <= '{$to_date}'
			)
	)
	ORDER BY emp_no
	LIMIT 20;
	```



## DISCLAIMER

To the best of my knowledge, this data is fabricated, and
it does not correspond to real people.
Any similarity to existing people is purely coincidental.
