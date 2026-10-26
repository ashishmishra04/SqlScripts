# SQL Window Functions: Detailed Guide with Examples

A window function performs a calculation across a set of rows related to the current row, without collapsing them into one row the way `GROUP BY` does. Every row keeps its identity, and the result is added as a new column.

## Syntax

```sql
function_name(expression) OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3 [ASC|DESC], ...]
    [frame_clause]
)
```

- **PARTITION BY**: splits rows into groups (windows). Like `GROUP BY`, but rows are not collapsed.
- **ORDER BY**: defines the row order inside each partition.
- **Frame clause**: defines which rows around the current row are included (e.g., `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`).

---

## Sample Data

```sql
CREATE TABLE employees (
    emp_id     INT,
    emp_name   VARCHAR(50),
    department VARCHAR(30),
    salary     INT,
    hire_date  DATE
);

INSERT INTO employees VALUES
(1, 'Alice', 'IT',    90000, '2020-01-15'),
(2, 'Bob',   'IT',    75000, '2021-03-10'),
(3, 'Carol', 'IT',    75000, '2022-07-01'),
(4, 'David', 'HR',    60000, '2019-05-20'),
(5, 'Eva',   'HR',    65000, '2021-11-11'),
(6, 'Frank', 'Sales', 50000, '2020-08-30'),
(7, 'Grace', 'Sales', 70000, '2022-02-14'),
(8, 'Henry', 'Sales', 55000, '2023-01-05');
```

### Verify the Data

```sql
SELECT * FROM employees;
```

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

**What it does:** `CREATE TABLE` builds the table and `INSERT` adds 8 rows across 3 departments. Every example below uses this data.

---

## 1. Aggregate Window Functions

### 1a. AVG and SUM with PARTITION BY

```sql
SELECT emp_name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       SUM(salary) OVER ()                        AS company_total
FROM employees;
```

| emp_name | department | salary | dept_avg | company_total |
|---|---|---|---|---|
| Alice | IT | 90000 | 80000.00 | 540000 |
| Bob | IT | 75000 | 80000.00 | 540000 |
| Carol | IT | 75000 | 80000.00 | 540000 |
| David | HR | 60000 | 62500.00 | 540000 |
| Eva | HR | 65000 | 62500.00 | 540000 |
| Frank | Sales | 50000 | 58333.33 | 540000 |
| Grace | Sales | 70000 | 58333.33 | 540000 |
| Henry | Sales | 55000 | 58333.33 | 540000 |

**What's happening:**
- `PARTITION BY department` creates separate windows for IT, HR, and Sales. Every IT row gets the IT average (240000 / 3 = 80000).
- `OVER ()` with nothing inside treats the whole table as one window, so every row gets the company total (540000).
- No rows are removed, unlike `GROUP BY`. The Sales average is a repeating decimal (175000 / 3), which the database rounds for display.

### 1b. Running Total (SUM with ORDER BY)

```sql
SELECT emp_name, department, hire_date, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY hire_date) AS running_total
FROM employees;
```

| emp_name | department | hire_date | salary | running_total |
|---|---|---|---|---|
| Alice | IT | 2020-01-15 | 90000 | 90000 |
| Bob | IT | 2021-03-10 | 75000 | 165000 |
| Carol | IT | 2022-07-01 | 75000 | 240000 |
| David | HR | 2019-05-20 | 60000 | 60000 |
| Eva | HR | 2021-11-11 | 65000 | 125000 |
| Frank | Sales | 2020-08-30 | 50000 | 50000 |
| Grace | Sales | 2022-02-14 | 70000 | 120000 |
| Henry | Sales | 2023-01-05 | 55000 | 175000 |

**What's happening:**
- Adding `ORDER BY` makes the window cumulative. Each row sums its own salary plus all earlier rows in the same department.
- The last row in each department holds that department's total (IT 240000, HR 125000, Sales 175000).

---

## 2. Ranking Functions

| Function | Behavior on ties | Example sequence |
|---|---|---|
| `ROW_NUMBER()` | Unique number, ties broken arbitrarily | 1, 2, 3, 4 |
| `RANK()` | Same rank for ties, **gaps** after | 1, 2, 2, 4 |
| `DENSE_RANK()` | Same rank for ties, **no gaps** | 1, 2, 2, 3 |
| `NTILE(n)` | Splits rows into n roughly equal buckets | 1, 1, 2, 2 |

### 2a. ROW_NUMBER, RANK, DENSE_RANK, NTILE

```sql
SELECT emp_name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
       RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk,
       NTILE(2)     OVER (PARTITION BY department ORDER BY salary DESC) AS bucket
FROM employees;
```

| emp_name | department | salary | row_num | rnk | dense_rnk | bucket |
|---|---|---|---|---|---|---|
| Alice | IT | 90000 | 1 | 1 | 1 | 1 |
| Bob | IT | 75000 | 2 | 2 | 2 | 1 |
| Carol | IT | 75000 | 3 | 2 | 2 | 2 |
| Eva | HR | 65000 | 1 | 1 | 1 | 1 |
| David | HR | 60000 | 2 | 2 | 2 | 2 |
| Grace | Sales | 70000 | 1 | 1 | 1 | 1 |
| Henry | Sales | 55000 | 2 | 2 | 2 | 1 |
| Frank | Sales | 50000 | 3 | 3 | 3 | 2 |

**What's happening:**
- **ROW_NUMBER** gives every row a unique number. Bob and Carol tie at 75000, so which one gets 2 is arbitrary and may change between runs.
- **RANK** gives tied rows the same rank and then skips a number. Bob and Carol both get 2, so there is no 3 in IT.
- **DENSE_RANK** gives tied rows the same rank without skipping. The sequence is 1, 2, 2 with no gap.
- **NTILE(2)** splits each department into 2 buckets. With 3 rows, the first bucket gets the extra row, so IT is 1, 1, 2. HR has 2 rows, so each bucket gets one. Sales is 1, 1, 2.

### 2b. Top N Per Group (Top 2 Earners per Department)

```sql
SELECT *
FROM (
    SELECT emp_name, department, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk <= 2;
```

| emp_name | department | salary | rnk |
|---|---|---|---|
| Alice | IT | 90000 | 1 |
| Bob | IT | 75000 | 2 |
| Carol | IT | 75000 | 2 |
| Eva | HR | 65000 | 1 |
| David | HR | 60000 | 2 |
| Grace | Sales | 70000 | 1 |
| Henry | Sales | 55000 | 2 |

**What's happening:**
- The inner query ranks employees within each department. The outer query filters on that rank.
- Frank is removed because he ranks 3rd in Sales.
- Carol appears alongside Bob because `DENSE_RANK` gives tied salaries the same rank. With `ROW_NUMBER`, only one of them would appear, so the choice of ranking function matters.
- The filter must live in an outer query because window functions run after `WHERE`.

---

## 3. Value (Offset) Functions

| Function | Purpose |
|---|---|
| `LAG(col, n, default)` | Value from n rows **before** the current row |
| `LEAD(col, n, default)` | Value from n rows **after** the current row |
| `FIRST_VALUE(col)` | First value in the window frame |
| `LAST_VALUE(col)` | Last value in the window frame |
| `NTH_VALUE(col, n)` | The nth value in the window frame |

### 3a. LAG and LEAD

```sql
SELECT emp_name, department, hire_date, salary,
       LAG(salary, 1, 0)  OVER (PARTITION BY department ORDER BY hire_date) AS prev_salary,
       LEAD(salary, 1, 0) OVER (PARTITION BY department ORDER BY hire_date) AS next_salary,
       salary - LAG(salary) OVER (PARTITION BY department ORDER BY hire_date) AS diff_from_prev
FROM employees;
```

| emp_name | department | hire_date | salary | prev_salary | next_salary | diff_from_prev |
|---|---|---|---|---|---|---|
| Alice | IT | 2020-01-15 | 90000 | 0 | 75000 | NULL |
| Bob | IT | 2021-03-10 | 75000 | 90000 | 75000 | -15000 |
| Carol | IT | 2022-07-01 | 75000 | 75000 | 0 | 0 |
| David | HR | 2019-05-20 | 60000 | 0 | 65000 | NULL |
| Eva | HR | 2021-11-11 | 65000 | 60000 | 0 | 5000 |
| Frank | Sales | 2020-08-30 | 50000 | 0 | 70000 | NULL |
| Grace | Sales | 2022-02-14 | 70000 | 50000 | 55000 | 20000 |
| Henry | Sales | 2023-01-05 | 55000 | 70000 | 0 | -15000 |

**What's happening:**
- `LAG(salary, 1, 0)` looks one row back within the department. The first row has nothing before it, so the default `0` is used.
- `LEAD(salary, 1, 0)` looks one row forward. The last row in each department gets the default `0`.
- `diff_from_prev` uses `LAG` with no default, so the first row returns `NULL`. Any arithmetic with `NULL` is `NULL`.
- This pattern is the basis for month-over-month or year-over-year change calculations.

### 3b. FIRST_VALUE (Highest-Paid Employee per Department)

```sql
SELECT emp_name, department, salary,
       FIRST_VALUE(emp_name) OVER (
           PARTITION BY department ORDER BY salary DESC
       ) AS highest_paid
FROM employees;
```

| emp_name | department | salary | highest_paid |
|---|---|---|---|
| Alice | IT | 90000 | Alice |
| Bob | IT | 75000 | Alice |
| Carol | IT | 75000 | Alice |
| David | HR | 60000 | Eva |
| Eva | HR | 65000 | Eva |
| Frank | Sales | 50000 | Grace |
| Grace | Sales | 70000 | Grace |
| Henry | Sales | 55000 | Grace |

**What's happening:**
- Every row in a department shows the name of the first row in its window, which is the highest earner because of `ORDER BY salary DESC`.
- The default frame runs from the start of the partition through the current row, so the first row is always visible.

---

## 4. Frame Clause (Controlling the Window)

The frame specifies which rows relative to the current row are used.

```sql
ROWS BETWEEN <start> AND <end>
```

Boundaries: `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`.

### 4a. Moving Average (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

```sql
SELECT emp_name, hire_date, salary,
       AVG(salary) OVER (
           ORDER BY hire_date
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3
FROM employees;
```

| emp_name | hire_date | salary | moving_avg_3 |
|---|---|---|---|
| David | 2019-05-20 | 60000 | 60000.00 |
| Alice | 2020-01-15 | 90000 | 75000.00 |
| Frank | 2020-08-30 | 50000 | 66666.67 |
| Bob | 2021-03-10 | 75000 | 71666.67 |
| Eva | 2021-11-11 | 65000 | 63333.33 |
| Grace | 2022-02-14 | 70000 | 70000.00 |
| Carol | 2022-07-01 | 75000 | 70000.00 |
| Henry | 2023-01-05 | 55000 | 66666.67 |

**What's happening:**
- The frame covers the current row and the two rows before it, so each average uses up to 3 rows.
- The first row has only itself, and the second row averages two rows. From the third row onward, it is a full 3-row average.
- For example, Bob's value is (90000 + 50000 + 75000) / 3 = 71666.67.
- There is no `PARTITION BY`, so the window crosses department boundaries.

### 4b. LAST_VALUE Gotcha and Fix

**The gotcha:**

```sql
SELECT emp_name, department, salary,
       LAST_VALUE(emp_name) OVER (
           PARTITION BY department ORDER BY salary DESC
       ) AS lowest_paid
FROM employees;
```

| emp_name | department | salary | lowest_paid |
|---|---|---|---|
| Alice | IT | 90000 | Alice |
| Bob | IT | 75000 | Bob or Carol (tie) |
| Carol | IT | 75000 | Bob or Carol (tie) |
| Eva | HR | 65000 | Eva |
| David | HR | 60000 | David |
| Grace | Sales | 70000 | Grace |
| Henry | Sales | 55000 | Henry |
| Frank | Sales | 50000 | Frank |

**The fix:**

```sql
SELECT emp_name, department, salary,
       LAST_VALUE(emp_name) OVER (
           PARTITION BY department ORDER BY salary DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS lowest_paid
FROM employees;
```

| emp_name | department | salary | lowest_paid |
|---|---|---|---|
| Alice | IT | 90000 | Bob or Carol (tie) |
| Bob | IT | 75000 | Bob or Carol (tie) |
| Carol | IT | 75000 | Bob or Carol (tie) |
| Eva | HR | 65000 | David |
| David | HR | 60000 | David |
| Grace | Sales | 70000 | Frank |
| Henry | Sales | 55000 | Frank |
| Frank | Sales | 50000 | Frank |

**What's happening:**
- **Gotcha:** With `ORDER BY` present, the default frame ends at the current row. `LAST_VALUE` then returns the current row's own name, so the output is not useful.
- **Fix:** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` makes the frame cover the whole partition, so `LAST_VALUE` returns the true last row (the lowest earner).
- In IT, Bob and Carol tie at 75000, so either can be "last." The result depends on the engine and is not guaranteed. Add a tiebreaker column (e.g., `emp_id`) to `ORDER BY` to make it deterministic.

---

## 5. Distribution Functions

```sql
SELECT emp_name, salary,
       PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank,
       CUME_DIST()    OVER (ORDER BY salary) AS cumulative_dist
FROM employees;
```

| emp_name | salary | pct_rank | cumulative_dist |
|---|---|---|---|
| Frank | 50000 | 0.0000 | 0.1250 |
| Henry | 55000 | 0.1429 | 0.2500 |
| David | 60000 | 0.2857 | 0.3750 |
| Eva | 65000 | 0.4286 | 0.5000 |
| Grace | 70000 | 0.5714 | 0.6250 |
| Bob | 75000 | 0.7143 | 0.8750 |
| Carol | 75000 | 0.7143 | 0.8750 |
| Alice | 90000 | 1.0000 | 1.0000 |

**What's happening:**
- **PERCENT_RANK** = (rank - 1) / (total rows - 1). With 8 rows, the denominator is 7. Frank is at the bottom (0), and Alice is at the top (1).
- **CUME_DIST** = (rows with value less than or equal to the current one) / total rows. Frank has 1 of 8 rows at or below him (0.125). Bob and Carol both have 7 of 8 (0.875) because they tie.
- Use `PERCENT_RANK` for "where does this sit from bottom to top" and `CUME_DIST` for "what fraction earn this much or less."

---

## 6. WINDOW Clause (Reusable Named Window)

```sql
SELECT emp_name,
       RANK() OVER w AS rnk,
       SUM(salary) OVER w AS running_sum
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

| emp_name | rnk | running_sum |
|---|---|---|
| Alice | 1 | 90000 |
| Bob | 2 | 240000 |
| Carol | 2 | 240000 |
| David | 2 | 125000 |
| Eva | 1 | 65000 |
| Frank | 3 | 175000 |
| Grace | 1 | 70000 |
| Henry | 2 | 125000 |

**What's happening:**
- `WINDOW w AS (...)` defines the window once, and both functions reuse it. This avoids repeating the same `OVER` clause.
- David's `rnk` is 2 because, within HR, Eva (65000) ranks above David (60000).
- `running_sum` uses the default `RANGE` frame, which treats tied salaries as peers. Bob and Carol both show 240000 because the frame includes both of their salaries.
- Supported in PostgreSQL, MySQL 8+, and SQLite 3.28+. SQL Server and Oracle do not support the `WINDOW` clause, so the `OVER (...)` definition must be repeated there.

---

## Window Function vs GROUP BY

| | GROUP BY | Window Function |
|---|---|---|
| Rows returned | One per group | All original rows |
| Row-level columns | No (only grouped or aggregated) | Yes |
| Typical use | Summaries | Rankings, running totals, comparisons |

## Key Points to Remember

- Window functions are evaluated **after** `WHERE`, `GROUP BY`, and `HAVING`, but **before** the final `ORDER BY`.
- You cannot filter on them in `WHERE`. Wrap the query in a subquery or CTE.
- Supported in PostgreSQL, MySQL 8+, SQL Server, Oracle, SQLite 3.25+, BigQuery, and Snowflake.
