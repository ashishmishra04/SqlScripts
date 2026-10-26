# SQL CTE: 20 Most Asked Interview Questions (Senior Level)

Write the SQL yourself first, then check the answers at the bottom. Examples use the `employees` and `orders` tables from the earlier guides. Where a query needs `manager_id` values, assume the following: Bob and Carol report to Alice (1), David and Eva report to Bob (2), and Frank reports to Carol (3). Answers use ANSI-style SQL, and engine differences are noted where they matter.

---

# QUESTIONS

## Section A: CTE Basics and Readability

**Q1.** Write a CTE that returns the names and salaries of IT employees, then filter that CTE for salaries above 80000.

**Q2.** Rewrite this nested subquery using a CTE:
```sql
SELECT emp_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

**Q3.** Using two chained CTEs, return employees who earn more than their own department's average salary.

**Q4.** Write one query that uses the same CTE twice: show each department's total salary and its percentage of the company-wide total.

**Q5.** Use a CTE with an explicit column list to return the top 2 earners per department, breaking ties by `emp_id`.

**Q6.** Using a CTE on `orders`, return customers who have more than 2 completed orders and a completed total above 1000, along with their order count and total.

## Section B: Recursive CTEs

**Q7.** Starting from Alice (emp_id 1), return everyone who reports to her directly or indirectly, with each person's level (Alice = 0).

**Q8.** Return each employee's full reporting path as a string, such as `Alice > Bob > David`.

**Q9.** For each manager, return the total number of people who report to them directly or indirectly.

**Q10.** Generate every date from 2024-01-01 to 2024-01-31 without using a table.

**Q11.** Generate the first 10 Fibonacci numbers using a recursive CTE.

**Q12.** A `flights` table contains a cycle (A to B, B to C, C to A). Write a recursive CTE that lists routes starting at A, prevents revisiting a city, and caps the number of hops at 5.

## Section C: CTEs in Data Modification

**Q13.** Delete duplicate rows from `orders` where `(customer_id, order_date, amount)` repeats, keeping the row with the lowest `order_id`. Write the version for SQL Server and the version for PostgreSQL.

**Q14.** Give a 5% raise to employees in the bottom salary quartile, using a CTE with `NTILE(4)`. Write it for PostgreSQL.

**Q15.** Insert one summary row per department (headcount and total salary) into a table `dept_summary`, using a CTE.

**Q16.** In PostgreSQL, move cancelled orders from before 2024 out of `orders` and into `orders_archive` in one statement. Both tables have the same columns.

## Section D: Performance, Scope, and Concepts

**Q17.** When would you choose a CTE, a subquery, or a temp table? Give a concrete scenario for each.

**Q18.** In PostgreSQL 12+, is a CTE always computed once? Show how to force each behavior.

**Q19.** Why does this query fail, and how do you fix it?
```sql
WITH x AS (SELECT 1 AS n)
SELECT n FROM x;

SELECT n FROM x;
```

**Q20.** A recursive CTE has no stop condition and runs forever in SQL Server. How do you cap it, and what happens when the cap is reached?

---
---

# ANSWERS

## Section A: CTE Basics and Readability

**A1.**
```sql
WITH it_staff AS (
    SELECT emp_name, salary
    FROM employees
    WHERE department = 'IT'
)
SELECT emp_name, salary
FROM it_staff
WHERE salary > 80000;
```
The CTE isolates the business step, and the outer query applies the second filter. The result is Alice (90000).

**A2.**
```sql
WITH company_avg AS (
    SELECT AVG(salary) AS avg_salary
    FROM employees
)
SELECT e.emp_name, e.salary
FROM employees e
CROSS JOIN company_avg c
WHERE e.salary > c.avg_salary;
```
The company average is 67500 (540000 / 8). The CTE names that value, so the logic is easier to read and reuse. Since `company_avg` returns one row, `CROSS JOIN` is safe here.

**A3.**
```sql
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
),
above_avg AS (
    SELECT e.emp_name, e.department, e.salary, d.avg_salary
    FROM employees e
    JOIN dept_avg d ON d.department = e.department
    WHERE e.salary > d.avg_salary
)
SELECT *
FROM above_avg
ORDER BY department, salary DESC;
```
The second CTE depends on the first. This chaining pattern is the main reason CTEs improve readability over nested subqueries.

**A4.**
```sql
WITH dept_totals AS (
    SELECT department, SUM(salary) AS dept_total
    FROM employees
    GROUP BY department
)
SELECT department,
       dept_total,
       ROUND(100.0 * dept_total / (SELECT SUM(dept_total) FROM dept_totals), 2) AS pct_of_payroll
FROM dept_totals;
```
A CTE can be referenced several times in the same statement. Here, `dept_totals` is used in both the main query and the scalar subquery. The database may evaluate it once or twice, depending on the engine and its optimizer.

**A5.**
```sql
WITH ranked (emp_name, department, salary, rn) AS (
    SELECT emp_name, department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, emp_id)
    FROM employees
)
SELECT emp_name, department, salary, rn
FROM ranked
WHERE rn <= 2;
```
The column list `(emp_name, department, salary, rn)` renames the outputs. It is supported in PostgreSQL, SQL Server, MySQL 8+, and SQLite. The `emp_id` tiebreaker makes the result deterministic.

**A6.**
```sql
WITH cust_totals AS (
    SELECT customer_id,
           COUNT(*)    AS order_count,
           SUM(amount) AS total_amount
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
)
SELECT customer_id, order_count, total_amount
FROM cust_totals
WHERE order_count > 2
  AND total_amount > 1000;
```
Filtering on aggregates needs `HAVING` or an outer query. A CTE makes the aggregation step explicit and keeps the final filter readable.

## Section B: Recursive CTEs

**A7.**
```sql
WITH RECURSIVE team AS (
    -- Anchor: Alice, level 0
    SELECT emp_id, emp_name, manager_id, 0 AS level
    FROM employees
    WHERE emp_id = 1

    UNION ALL

    -- Recursive: anyone whose manager is already in the team
    SELECT e.emp_id, e.emp_name, e.manager_id, t.level + 1
    FROM employees e
    JOIN team t ON e.manager_id = t.emp_id
)
SELECT emp_id, emp_name, level
FROM team
ORDER BY level, emp_id;
```
The anchor starts the walk, and each recursive step moves one level down the hierarchy. The `RECURSIVE` keyword is required in PostgreSQL, MySQL, and SQLite. It is optional in SQL Server.

**A8.**
```sql
WITH RECURSIVE chain AS (
    SELECT emp_id, emp_name,
           CAST(emp_name AS VARCHAR(500)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.emp_id, e.emp_name,
           CAST(c.path || ' > ' || e.emp_name AS VARCHAR(500))
    FROM employees e
    JOIN chain c ON e.manager_id = c.emp_id
)
SELECT emp_name, path
FROM chain
ORDER BY path;
```
Each step appends the employee's name to the parent's path. Use `||` in PostgreSQL, Oracle, and SQLite. In SQL Server, use `+` instead. The `CAST` keeps the string length consistent across iterations.

**A9.**
```sql
WITH RECURSIVE reports AS (
    -- Anchor: each direct report, paired with their manager
    SELECT manager_id AS root_id, emp_id
    FROM employees
    WHERE manager_id IS NOT NULL

    UNION ALL

    -- Recursive: the reports of each person found so far
    SELECT r.root_id, e.emp_id
    FROM reports r
    JOIN employees e ON e.manager_id = r.emp_id
)
SELECT root_id AS manager_id,
       COUNT(*) AS total_reports
FROM reports
GROUP BY root_id;
```
The `root_id` column carries the original manager through every level. Each row in `reports` is one (manager, descendant) pair, so counting by manager gives the total number of direct and indirect reports.

**A10.**
```sql
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' AS dt
    UNION ALL
    SELECT dt + 1
    FROM dates
    WHERE dt < DATE '2024-01-31'
)
SELECT dt
FROM dates;
```
In PostgreSQL, `dt + 1` adds one day. In MySQL, use `DATE_ADD(dt, INTERVAL 1 DAY)`. In SQL Server, use `DATEADD(DAY, 1, dt)`. The `WHERE` clause is the stop condition.

**A11.**
```sql
WITH RECURSIVE fib (n, a, b) AS (
    SELECT 1, 0, 1
    UNION ALL
    SELECT n + 1, b, a + b
    FROM fib
    WHERE n < 10
)
SELECT n, a AS fib_value
FROM fib;
```
Each row holds the current pair (a, b). The next row shifts the pair forward. The output is 0, 1, 1, 2, 3, 5, 8, 13, 21, 34.

**A12.**
```sql
WITH RECURSIVE routes AS (
    -- Anchor: direct flights out of A
    SELECT from_city, to_city,
           CAST(from_city || '-' || to_city AS VARCHAR(500)) AS path,
           1 AS hops
    FROM flights
    WHERE from_city = 'A'

    UNION ALL

    SELECT r.from_city, f.to_city,
           CAST(r.path || '-' || f.to_city AS VARCHAR(500)),
           r.hops + 1
    FROM routes r
    JOIN flights f ON f.from_city = r.to_city
    WHERE r.hops < 5                                  -- hard cap on depth
      AND r.path NOT LIKE '%' || f.to_city || '%'     -- never revisit a city
)
SELECT path, hops
FROM routes;
```
There are two protections against the cycle. The `NOT LIKE` check stops a route from visiting a city twice, and the hop cap is a safety net in case the path check misses something. The `LIKE` check is simplified: it can give false matches when one city name is a substring of another, so use delimiters or an array in production.

## Section C: CTEs in Data Modification

**A13.**

SQL Server (a CTE can be the target of `DELETE`):
```sql
WITH dupes AS (
    SELECT order_id,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date, amount
               ORDER BY order_id) AS rn
    FROM orders
)
DELETE FROM dupes
WHERE rn > 1;
```

PostgreSQL (`DELETE` cannot target a CTE directly, so delete by key):
```sql
WITH dupes AS (
    SELECT order_id,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date, amount
               ORDER BY order_id) AS rn
    FROM orders
)
DELETE FROM orders
WHERE order_id IN (SELECT order_id FROM dupes WHERE rn > 1);
```
Always run the `SELECT` version first to confirm which rows will go.

**A14.**
```sql
WITH quartiles AS (
    SELECT emp_id,
           NTILE(4) OVER (ORDER BY salary) AS quartile
    FROM employees
)
UPDATE employees e
SET salary = CAST(ROUND(e.salary * 1.05) AS INT)
FROM quartiles q
WHERE q.emp_id = e.emp_id
  AND q.quartile = 1;
```
The CTE assigns each employee a quartile, and the `UPDATE ... FROM` applies the raise only to quartile 1. The `CAST` is needed because `ROUND` on a numeric value returns a numeric type, and `salary` is an `INT`. In SQL Server, use `UPDATE e SET ... FROM employees e JOIN quartiles q ...`.

**A15.**
```sql
WITH dept_stats AS (
    SELECT department,
           COUNT(*)    AS headcount,
           SUM(salary) AS total_salary
    FROM employees
    GROUP BY department
)
INSERT INTO dept_summary (department, headcount, total_salary)
SELECT department, headcount, total_salary
FROM dept_stats;
```
Putting `WITH` before `INSERT` works in PostgreSQL, SQL Server, MySQL 8+, and SQLite. Some engines also accept `INSERT INTO ... WITH ... SELECT`, but the leading form is the most portable.

**A16.**
```sql
WITH moved AS (
    DELETE FROM orders
    WHERE status = 'CANCELLED'
      AND order_date < DATE '2024-01-01'
    RETURNING *
)
INSERT INTO orders_archive
SELECT * FROM moved;
```
This is a data-modifying CTE, supported in PostgreSQL. The rows deleted by the first statement are returned with `RETURNING` and inserted into the archive in the same atomic statement. If the insert fails, the delete is rolled back as well.

## Section D: Performance, Scope, and Concepts

**A17.**

| Option | Choose it when | Example scenario |
|---|---|---|
| **CTE** | The logic is used once or a few times in one statement, and you want readable steps | Ranking employees and filtering the top 2 per department in one query |
| **Subquery** | A single, small scalar or existence check is needed | `WHERE salary > (SELECT AVG(salary) FROM employees)` |
| **Temp table** | Results are reused across several statements, need indexes, or are large enough that a statistics-driven plan matters | Staging 50 million cleaned order rows once, then running five reports against them |

The main trade-off is scope. CTEs last only for one statement, while temp tables last for the session and can be indexed.

**A18.**
```sql
-- Force a single evaluation, even if referenced several times (PostgreSQL 12+)
WITH big AS MATERIALIZED (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
)
SELECT a.customer_id, a.total, b.total AS total_again
FROM big a
JOIN big b ON b.customer_id = a.customer_id;

-- Allow the optimizer to inline it into the main query
WITH big AS NOT MATERIALIZED (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
)
SELECT customer_id, total
FROM big
WHERE customer_id = 42;
```
No. Since PostgreSQL 12, a CTE referenced once is inlined by default, which lets filters push down into it. A CTE referenced more than once is materialized by default. `MATERIALIZED` and `NOT MATERIALIZED` override this. Other engines have different rules, so check the execution plan.

**A19.**
The query fails because a CTE exists only for the statement that defines it. The second `SELECT` is a new statement, so `x` no longer exists.

Fix with a temp table, which persists for the session:
```sql
CREATE TEMPORARY TABLE x AS
SELECT 1 AS n;

SELECT n FROM x;
```
Use a temp table when the result is needed across statements. Use a view when the definition should be permanent.

**A20.**

SQL Server stops the recursion with `MAXRECURSION`:
```sql
WITH r AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM r          -- no stop condition
)
SELECT n
FROM r
OPTION (MAXRECURSION 50);
```
The default limit is 100. With `OPTION (MAXRECURSION 50)`, the query stops after 50 levels and returns an error. Setting `0` removes the limit, which is risky. The best practice is to include a stop condition in the query itself, such as `WHERE n < 50`. PostgreSQL and MySQL have no built-in cap by default, so the stop condition is the only protection. MySQL also offers `cte_max_recursion_depth`, which defaults to 1000.
