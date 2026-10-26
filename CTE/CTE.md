# Common Table Expressions (CTEs)

A CTE is a named, temporary result set defined with `WITH` at the start of a query. You can reference it like a table in the main query. It lasts only for that one statement. CTEs make complex queries easier to read, and recursive CTEs can traverse hierarchical data.

---

## Syntax

```sql
WITH cte_name [(column1, column2, ...)] AS (
    -- CTE query
    SELECT ...
)
SELECT ...
FROM cte_name;
```

Multiple CTEs are separated by commas:

```sql
WITH cte1 AS (...),
     cte2 AS (...)
SELECT ...
```

---

## Sample Data

The examples use the `employees` table:

| emp_id | emp_name | department | salary | hire_date |
|---|---|---|---|---|
| 1 | Alice | IT | 90000 | 2020-01-15 |
| 2 | Bob | IT | 75000 | 2021-03-10 |
| 3 | Carol | IT | 75000 | 2022-07-01 |
| 4 | David | HR | 60000 | 2019-05-20 |
| 5 | Eva | HR | 65000 | 2021-11-11 |
| 6 | Frank | Sales | 50000 | 2020-08-30 |
| 7 | Grace | Sales | 70000 | 2022-02-14 |
| 8 | Henry | Sales | 55000 | 2023-01-05 |

---

## 1. A Simple CTE

Suppose you want the IT employees who earn more than 80000. Without a CTE, you would repeat the filter in a subquery. With a CTE, you name the step first:

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

**Result:**

| emp_name | salary |
|---|---|
| Alice | 90000 |

**What's happening:**
- The CTE `it_staff` is built first and holds only IT rows.
- The main query filters that named result. Reading top to bottom matches the order of your logic.

---

## 2. Multiple CTEs (Chaining Steps)

You can define several CTEs, and later ones can use earlier ones. This example finds employees who earn more than their own department's average.

```sql
WITH dept_stats AS (
    SELECT department,
           AVG(salary) AS avg_salary,
           COUNT(*)    AS headcount
    FROM employees
    GROUP BY department
),
high_earners AS (
    SELECT e.emp_name, e.department, e.salary, d.avg_salary
    FROM employees e
    JOIN dept_stats d ON e.department = d.department
    WHERE e.salary > d.avg_salary
)
SELECT *
FROM high_earners
ORDER BY department, salary DESC;
```

**Result:**

| emp_name | department | salary | avg_salary |
|---|---|---|---|
| Eva | HR | 65000 | 62500.00 |
| Alice | IT | 90000 | 80000.00 |
| Grace | Sales | 70000 | 58333.33 |

**What's happening:**
- `dept_stats` computes each department's average and headcount.
- `high_earners` joins back to `employees` and keeps only rows above the average. It uses `dept_stats`, so it depends on the first CTE.
- Bob and Carol earn exactly 5000 less than the IT average, so they are excluded. The same goes for David, Henry, and Frank.

---

## 3. CTE with a Window Function

Window functions cannot be used in `WHERE`, so a CTE is a clean way to filter on their results. This is the "top 2 per department" pattern from the window function guide:

```sql
WITH ranked AS (
    SELECT emp_name, department, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT emp_name, department, salary, rnk
FROM ranked
WHERE rnk <= 2;
```

**Result:**

| emp_name | department | salary | rnk |
|---|---|---|---|
| Alice | IT | 90000 | 1 |
| Bob | IT | 75000 | 2 |
| Carol | IT | 75000 | 2 |
| Eva | HR | 65000 | 1 |
| David | HR | 60000 | 2 |
| Grace | Sales | 70000 | 1 |
| Henry | Sales | 55000 | 2 |

**What's happening:** The window function runs inside the CTE, and the outer query filters the result. This gives the same result as the subquery version, but it is easier to read.

---

## 4. Recursive CTE

A recursive CTE references itself. It has two parts joined by `UNION ALL`:

1. **Anchor member:** the starting rows.
2. **Recursive member:** a query that uses the previous iteration's rows. It repeats until it returns no new rows.

Example: an org chart. First, a table with a manager column:

```sql
CREATE TABLE staff (
    emp_id     INT,
    emp_name   VARCHAR(50),
    manager_id INT
);

INSERT INTO staff VALUES
(1, 'Alice', NULL),   -- top of the hierarchy
(2, 'Bob',   1),
(3, 'Carol', 1),
(4, 'David', 2),
(5, 'Eva',   2),
(6, 'Frank', 3);
```

The recursive query:

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: start with the top manager (no manager)
    SELECT emp_id, emp_name, manager_id, 1 AS level
    FROM staff
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find employees whose manager is already in the chart
    SELECT s.emp_id, s.emp_name, s.manager_id, o.level + 1
    FROM staff s
    JOIN org_chart o ON s.manager_id = o.emp_id
)
SELECT emp_id, emp_name, manager_id, level
FROM org_chart
ORDER BY level, emp_id;
```

**Result:**

| emp_id | emp_name | manager_id | level |
|---|---|---|---|
| 1 | Alice | NULL | 1 |
| 2 | Bob | 1 | 2 |
| 3 | Carol | 1 | 2 |
| 4 | David | 2 | 3 |
| 5 | Eva | 2 | 3 |
| 6 | Frank | 3 | 3 |

**What's happening, step by step:**
1. **Iteration 0 (anchor):** Alice is returned at level 1.
2. **Iteration 1:** Employees whose manager is Alice (Bob, Carol) are found, at level 2.
3. **Iteration 2:** Employees whose manager is Bob or Carol (David, Eva, Frank) are found, at level 3.
4. **Iteration 3:** No employees report to David, Eva, or Frank, so no rows are returned and recursion stops.

**Notes:**
- PostgreSQL, MySQL 8+, and SQLite need the `RECURSIVE` keyword. SQL Server and Oracle do not use it.
- Use `UNION ALL` unless you need duplicates removed. `UNION` adds a deduplication step.

### Simple Number Generator

A recursive CTE can also generate sequences:

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 5
)
SELECT n FROM nums;
```

**Result:** 1, 2, 3, 4, 5

The `WHERE n < 5` condition is the stop condition. **Without it, the recursion never ends.**

---

## 5. Recursion Limits

Some databases stop runaway recursion automatically:
- **SQL Server:** `OPTION (MAXRECURSION 100)` sets the limit. The default is 100, and 0 removes the limit.
- **PostgreSQL, MySQL:** There is no default row limit, so you must write a correct stop condition.

---

## CTE vs Subquery vs Temp Table vs View

| Feature | CTE | Subquery | Temp table | View |
|---|---|---|---|---|
| Scope | One statement | One statement | Session | Permanent until dropped |
| Readability | High | Lower when nested | High | High |
| Can reference itself (recursion) | Yes | No | No | No |
| Reused across many statements | No (within one query) | No | Yes | Yes |
| Can be indexed | No | No | Yes | Not directly |

---

## Key Points

- A CTE exists only for the statement that defines it.
- Use CTEs to break complex logic into named steps, avoid repeating subqueries, and filter window function results.
- Recursive CTEs handle hierarchies (org charts, categories, bill of materials) and sequences. Always include a stop condition.
- In PostgreSQL 12+, a CTE referenced once is usually inlined by the optimizer. Use `AS MATERIALIZED` to force it to be computed once when that helps performance. Check the execution plan when performance matters.
