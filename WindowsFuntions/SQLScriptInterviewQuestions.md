# SQL Window Functions: 50 Hands-On Interview Questions (Senior Level)

All questions use the tables below unless stated otherwise. Write the SQL yourself first, then check the answers section at the bottom. Answers use ANSI-style SQL (PostgreSQL / SQL Server / MySQL 8+ compatible unless noted).

**Sample data:** run `window_functions_schema_and_data.sql` to create all seven `tbl_*` tables and load dummy rows. The data includes edge cases on purpose: salary ties, `order_id` gaps, duplicate orders, login streaks, 30-minute session boundaries, overlapping meetings, and NULL sensor readings.

## Schema

```sql
CREATE TABLE tbl_employees (
    emp_id      INT PRIMARY KEY,
    emp_name    VARCHAR(50),
    department  VARCHAR(30),
    salary      INT,
    hire_date   DATE,
    manager_id  INT
);

CREATE TABLE tbl_orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,
    order_date   DATE,
    amount       DECIMAL(10,2),
    status       VARCHAR(20)      -- 'COMPLETED', 'CANCELLED'
);

CREATE TABLE tbl_stock_prices (
    ticker       VARCHAR(10),
    trade_date   DATE,
    close_price  DECIMAL(10,2)
);

CREATE TABLE tbl_logins (
    user_id     INT,
    login_date  DATE
);

CREATE TABLE tbl_sessions (
    user_id       INT,
    event_time    TIMESTAMP,
    event_type    VARCHAR(20)
);

CREATE TABLE tbl_meetings (
    room_id     INT,
    start_time  TIMESTAMP,
    end_time    TIMESTAMP
);

CREATE TABLE tbl_readings (       -- used by Q45
    sensor_id   INT,
    ts          TIMESTAMP,
    value       DECIMAL(10,2)
);
```

---

# QUESTIONS

## Section A: Ranking and Top-N

**Q1.** Return the highest-paid employee in each department. If there is a tie, return all tied employees.

**Q2.** Return the 3rd highest distinct salary in the company. If it does not exist, return NULL.

**Q3.** Return the top 2 salaries per department, including all ties at each of those salary levels.

**Q4.** Return exactly 2 employees per department (the 2 highest paid), breaking ties by earliest `hire_date`, then lowest `emp_id`.

**Q5.** For each customer, return their most recent completed order (all columns).

**Q6.** Remove duplicate rows from `tbl_orders` where `(customer_id, order_date, amount)` repeats, keeping the row with the lowest `order_id`. Write the `SELECT` that identifies the rows to delete.

**Q7.** Split employees into 4 salary quartiles company-wide and return each quartile's minimum salary, maximum salary, and headcount.

**Q8.** Return employees whose salary is above the median salary of their department. Do not use a median function.

**Q9.** Return the percentage of employees in each department whose salary is lower than or equal to a given employee's salary, using `CUME_DIST`.

**Q10.** Return the second most recent order for every customer, excluding customers with only one order.

## Section B: Running Totals and Aggregates

**Q11.** Return a running total of `amount` per customer ordered by `order_date`, with `order_id` as tiebreaker, counting only completed orders.

**Q12.** Return each order along with the customer's total spend and the percentage this order represents of that total.

**Q13.** Return each department's share of the total company payroll, one row per department, without using a subquery or `GROUP BY` alone.

**Q14.** Return a running count of cancelled orders per customer, reset each calendar month.

**Q15.** Return each employee with the department's highest salary and the difference between that and the employee's salary.

**Q16.** Return the cumulative percentage of total sales by customer, ordered from largest to smallest spender (a Pareto / 80-20 analysis), and flag the customers that make up the first 80%.

**Q17.** Return the running maximum of `amount` per customer over time.

**Q18.** Return the average order amount per customer excluding the current order.

## Section C: LAG, LEAD and Comparisons

**Q19.** For each order, return the number of days since the same customer's previous order.

**Q20.** Return month-over-month revenue growth percentage for completed orders.

**Q21.** Return customers whose order amount strictly increased in 3 consecutive orders.

**Q22.** For each ticker, return days where `close_price` was higher than both the previous and next day (local peaks).

**Q23.** Return the first and last order amount per customer in a single row each, without self-joins or `GROUP BY`.

**Q24.** For each employee, return the salary of the next person hired in the same department and the difference.

**Q25.** Return the year-over-year revenue change by month (compare each month to the same month in the previous year) using `LAG` with an offset.

**Q26.** Return orders where the amount differs from the same customer's previous order amount by more than 50%.

## Section D: Frames, Moving Windows

**Q27.** Return a 7-day moving average of `close_price` per ticker, using only the 7 preceding trading rows including the current one.

**Q28.** Return a true 7 calendar-day moving average of daily order totals, even when some days have no orders. Use a `RANGE` frame or a date spine.

**Q29.** Return, for each ticker and day, the highest close price in the following 5 rows (excluding the current row).

**Q30.** Explain and fix: a query using `LAST_VALUE(amount) OVER (PARTITION BY customer_id ORDER BY order_date)` returns the current row's amount. Write the corrected query.

**Q31.** Return the running total of salaries per department using `RANGE`, and show how the result differs from `ROWS` when two employees have the same `hire_date`.

**Q32.** Return each order with a count of orders by the same customer in the previous 30 days (excluding the current order).

**Q33.** Return the day on which each ticker reached its maximum rolling 5-day average.

## Section E: Gaps, Islands, and Sessionization

**Q34.** Find the longest streak of consecutive login days for each user.

**Q35.** Return the start and end date of each consecutive login streak per user.

**Q36.** Find gaps in `order_id` values (missing IDs), returning the start and end of each missing range.

**Q37.** Group `tbl_sessions` events into sessions where a new session starts after 30 minutes of inactivity. Return `user_id`, `session_id`, `session_start`, `session_end`, and event count.

**Q38.** Return users who logged in on at least 3 consecutive days at any point.

**Q39.** Return all rows from `tbl_meetings` that overlap with at least one other meeting in the same room.

**Q40.** Merge overlapping meetings in each room into continuous busy blocks.

**Q41.** For each ticker, return runs of consecutive days where the price kept rising, with the run length.

## Section F: Advanced and Scenario Problems

**Q42.** Return the median salary per department using window functions only (handle even and odd counts).

**Q43.** Return the rows where the customer's order was the first order of the month for that customer.

**Q44.** Calculate customer retention: for each cohort month (first order month), return the number of customers active in month 0, 1, 2, and 3 after their first order.

**Q45.** Fill forward missing values: given the table `tbl_readings(sensor_id, ts, value)` with NULLs in `value`, return each row with the last non-NULL value up to that point (portable, without `IGNORE NULLS`).

**Q46.** Return the employee hierarchy depth for each employee and the rank of their salary among peers at the same depth.

**Q47.** Return, for each department, the employee with the longest tenure and the one with the shortest, in a single row per department.

**Q48.** Return the top 3 customers per month by revenue, but only for months in which total revenue exceeded 10,000.

**Q49.** Compute each customer's rolling lifetime value: the sum of completed order amounts in the 90 days up to and including each order, with proper handling of date-based (not row-based) windows.

**Q50.** Explain how you would diagnose and tune a slow window function query on a table with 500 million rows. Name specific techniques and show an example index.

---
---

# ANSWERS

## Section A: Ranking and Top-N

**A1.**
```sql
SELECT emp_id, emp_name, department, salary
FROM (
    SELECT *, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM tbl_employees
) t
WHERE rnk = 1;
```
`RANK` (or `DENSE_RANK`) keeps all ties at rank 1. `ROW_NUMBER` would return only one. With the sample data, Finance returns both Eva Green and Olivia Jackson (125000 each).

**A2.**
```sql
SELECT MAX(salary) AS third_highest
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM tbl_employees
) t
WHERE rnk = 3;
```
`MAX` over an empty set returns NULL, which handles the "does not exist" case without extra logic.

**A3.**
```sql
SELECT *
FROM (
    SELECT *, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM tbl_employees
) t
WHERE rnk <= 2;
```
`DENSE_RANK` counts distinct salary levels, so rank 2 is the second distinct salary.

**A4.**
```sql
SELECT *
FROM (
    SELECT *, ROW_NUMBER() OVER (
               PARTITION BY department
               ORDER BY salary DESC, hire_date ASC, emp_id ASC) AS rn
    FROM tbl_employees
) t
WHERE rn <= 2;
```
`ROW_NUMBER` with a fully deterministic `ORDER BY` always returns exactly 2 rows per department.

**A5.**
```sql
SELECT *
FROM (
    SELECT o.*, ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC) AS rn
    FROM tbl_orders o
    WHERE status = 'COMPLETED'
) t
WHERE rn = 1;
```
Filter in the inner `WHERE` before ranking, so cancelled orders never take rank 1. `order_id` breaks same-day ties.

**A6.**
```sql
SELECT order_id
FROM (
    SELECT order_id,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date, amount
               ORDER BY order_id) AS rn
    FROM tbl_orders
) t
WHERE rn > 1;
```
Rows with `rn > 1` are the duplicates. Use this result in a `DELETE ... WHERE order_id IN (...)`.

**A7.**
```sql
SELECT quartile,
       MIN(salary) AS min_salary,
       MAX(salary) AS max_salary,
       COUNT(*)    AS headcount
FROM (
    SELECT salary, NTILE(4) OVER (ORDER BY salary) AS quartile
    FROM tbl_employees
) t
GROUP BY quartile
ORDER BY quartile;
```
`NTILE` assigns buckets by row position, so equal salaries can fall in different quartiles. Mention this caveat in an interview.

**A8.**
```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*)     OVER (PARTITION BY department)                 AS cnt
    FROM tbl_employees
),
medians AS (
    SELECT department, AVG(salary * 1.0) AS median_salary
    FROM ranked
    WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)
    GROUP BY department
)
SELECT e.*
FROM tbl_employees e
JOIN medians m ON m.department = e.department
WHERE e.salary > m.median_salary;
```
For odd counts, both expressions give the middle row. For even counts, they give the two middle rows, which are averaged. Integer division is assumed (SQL Server, PostgreSQL integers).

**A9.**
```sql
SELECT emp_id, emp_name, department, salary,
       ROUND((100 * CUME_DIST() OVER (PARTITION BY department ORDER BY salary))::numeric, 2) AS pct_at_or_below
FROM tbl_employees;
```
`CUME_DIST` returns the fraction of rows with value less than or equal to the current row's value. In PostgreSQL it returns `double precision`, and `ROUND(x, 2)` only exists for `numeric`, so the cast is required. In SQL Server and MySQL, drop the `::numeric` cast.

**A10.**
```sql
SELECT *
FROM (
    SELECT o.*, ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC) AS rn
    FROM tbl_orders o
) t
WHERE rn = 2;
```
Customers with a single order never produce `rn = 2`, so they are excluded automatically. In the sample data, customer 7 has one order and is excluded.

## Section B: Running Totals and Aggregates

**A11.**
```sql
SELECT order_id, customer_id, order_date, amount,
       SUM(amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM tbl_orders
WHERE status = 'COMPLETED';
```
Adding `order_id` makes the order deterministic. An explicit `ROWS` frame avoids the `RANGE` default, which treats same-date rows as peers and gives them equal totals.

**A12.**
```sql
SELECT order_id, customer_id, amount,
       SUM(amount) OVER (PARTITION BY customer_id) AS customer_total,
       ROUND(100.0 * amount / SUM(amount) OVER (PARTITION BY customer_id), 2) AS pct_of_total
FROM tbl_orders
WHERE status = 'COMPLETED';
```

**A13.**
```sql
SELECT DISTINCT department,
       SUM(salary) OVER (PARTITION BY department) AS dept_payroll,
       ROUND(100.0 * SUM(salary) OVER (PARTITION BY department)
             / SUM(salary) OVER (), 2) AS pct_of_payroll
FROM tbl_employees;
```
`DISTINCT` collapses the repeated per-employee rows to one per department. An alternative is `GROUP BY department` with `SUM(SUM(salary)) OVER ()`.

**A14.**
```sql
SELECT order_id, customer_id, order_date,
       SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) OVER (
           PARTITION BY customer_id, DATE_TRUNC('month', order_date)
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cancelled_running
FROM tbl_orders;
```
Adding the month to `PARTITION BY` resets the count. Use `DATE_TRUNC` (PostgreSQL), `DATETRUNC` (SQL Server 2022+), or `DATE_FORMAT(order_date, '%Y-%m')` (MySQL).

**A15.**
```sql
SELECT emp_id, emp_name, department, salary,
       MAX(salary) OVER (PARTITION BY department)          AS dept_max,
       MAX(salary) OVER (PARTITION BY department) - salary AS gap_to_top
FROM tbl_employees;
```

**A16.**
```sql
WITH spend AS (
    SELECT customer_id, SUM(amount) AS total_spend
    FROM tbl_orders
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
),
cum AS (
    SELECT customer_id, total_spend,
           SUM(total_spend) OVER (ORDER BY total_spend DESC, customer_id
                                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running,
           SUM(total_spend) OVER () AS grand_total
    FROM spend
)
SELECT customer_id, total_spend,
       ROUND(100.0 * running / grand_total, 2) AS cum_pct,
       CASE WHEN running - total_spend < 0.8 * grand_total THEN 'Y' ELSE 'N' END AS in_top_80
FROM cum;
```
The flag tests the cumulative total before the current customer, so the customer who crosses the 80% line is still included.

**A17.**
```sql
SELECT order_id, customer_id, order_date, amount,
       MAX(amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_max
FROM tbl_orders;
```

**A18.**
```sql
SELECT order_id, customer_id, amount,
       AVG(amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS avg_prior_orders
FROM tbl_orders;
```
This is the average of all prior orders. To exclude the current row from the whole partition instead, use `(SUM(amount) OVER (PARTITION BY customer_id) - amount) / NULLIF(COUNT(*) OVER (PARTITION BY customer_id) - 1, 0)`.

## Section C: LAG, LEAD and Comparisons

**A19.**
```sql
SELECT order_id, customer_id, order_date,
       order_date - LAG(order_date) OVER (
           PARTITION BY customer_id ORDER BY order_date, order_id) AS days_since_prev
FROM tbl_orders;
```
This is PostgreSQL date subtraction. In SQL Server use `DATEDIFF(day, LAG(order_date) OVER (...), order_date)`. In MySQL use `DATEDIFF(order_date, LAG(order_date) OVER (...))`.

**A20.**
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS revenue
    FROM tbl_orders
    WHERE status = 'COMPLETED'
    GROUP BY 1
)
SELECT month, revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) AS mom_growth_pct
FROM monthly;
```
`NULLIF` protects against division by zero. The first month returns NULL because there is no previous month.

**A21.**
```sql
WITH x AS (
    SELECT customer_id, order_id, order_date, amount,
           LAG(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS p1,
           LAG(amount, 2) OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS p2
    FROM tbl_orders
)
SELECT DISTINCT customer_id
FROM x
WHERE amount > p1 AND p1 > p2;
```
Rows where `p2` is NULL (fewer than 3 orders) fail the comparison, so they drop out.

**A22.**
```sql
SELECT ticker, trade_date, close_price
FROM (
    SELECT *,
           LAG(close_price)  OVER (PARTITION BY ticker ORDER BY trade_date) AS prev_p,
           LEAD(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) AS next_p
    FROM tbl_stock_prices
) t
WHERE close_price > prev_p AND close_price > next_p;
```
The first and last day of each ticker have a NULL neighbor, so they are excluded.

**A23.**
```sql
SELECT DISTINCT customer_id,
       FIRST_VALUE(amount) OVER (
           PARTITION BY customer_id ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS first_amount,
       LAST_VALUE(amount) OVER (
           PARTITION BY customer_id ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_amount
FROM tbl_orders;
```
Without the full frame, `LAST_VALUE` returns the current row.

**A24.**
```sql
SELECT emp_id, emp_name, department, hire_date, salary,
       LEAD(salary) OVER (PARTITION BY department ORDER BY hire_date, emp_id) AS next_hire_salary,
       LEAD(salary) OVER (PARTITION BY department ORDER BY hire_date, emp_id) - salary AS diff
FROM tbl_employees;
```

**A25.**
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS revenue
    FROM tbl_orders
    WHERE status = 'COMPLETED'
    GROUP BY 1
)
SELECT month, revenue,
       LAG(revenue, 12) OVER (ORDER BY month) AS revenue_last_year,
       ROUND(100.0 * (revenue - LAG(revenue, 12) OVER (ORDER BY month))
             / NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0), 2) AS yoy_pct
FROM monthly;
```
Offset 12 is correct only if every month exists with no gaps. With missing months, join to a calendar table first or self-join on `month - INTERVAL '1 year'`. The sample data covers 18 consecutive months, so Jan to Jun 2024 have a prior-year value.

**A26.**
```sql
SELECT *
FROM (
    SELECT o.*,
           LAG(amount) OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS prev_amount
    FROM tbl_orders o
) t
WHERE prev_amount IS NOT NULL
  AND ABS(amount - prev_amount) > 0.5 * prev_amount;
```

## Section D: Frames, Moving Windows

**A27.**
```sql
SELECT ticker, trade_date, close_price,
       AVG(close_price) OVER (
           PARTITION BY ticker ORDER BY trade_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM tbl_stock_prices;
```
`6 PRECEDING` plus the current row makes 7 rows. The first 6 rows average over fewer rows. Add a `CASE WHEN COUNT(*) OVER (...) = 7` guard if you want NULL until the window is full.

**A28.**
```sql
WITH daily AS (
    SELECT order_date, SUM(amount) AS day_total
    FROM tbl_orders
    WHERE status = 'COMPLETED'
    GROUP BY order_date
)
SELECT order_date, day_total,
       AVG(day_total) OVER (
           ORDER BY order_date
           RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS ma7_calendar
FROM daily;
```
`RANGE` with an interval works in PostgreSQL 11+ and Oracle. Note that it averages only the days that have orders. For a true calendar average that counts days with no orders as zero, left join `daily` to a generated date spine and use `COALESCE(day_total, 0)` with `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`. SQL Server does not support interval `RANGE` frames, so use the date spine approach there.

**A29.**
```sql
SELECT ticker, trade_date, close_price,
       MAX(close_price) OVER (
           PARTITION BY ticker ORDER BY trade_date
           ROWS BETWEEN 1 FOLLOWING AND 5 FOLLOWING) AS max_next_5
FROM tbl_stock_prices;
```
Frames that start at `1 FOLLOWING` exclude the current row.

**A30.**
```sql
SELECT order_id, customer_id, order_date, amount,
       LAST_VALUE(amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_amount
FROM tbl_orders;
```
With `ORDER BY` and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. The frame stops at the current row, so the "last" value is the current row's own value. Extend the frame to `UNBOUNDED FOLLOWING`, or use `FIRST_VALUE` with a reversed `ORDER BY`.

**A31.**
```sql
SELECT emp_id, department, hire_date, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY hire_date
                         RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_range,
       SUM(salary) OVER (PARTITION BY department ORDER BY hire_date
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_rows
FROM tbl_employees;
```
If two employees share a `hire_date`, `RANGE` gives both the same total, because peers are included together. `ROWS` increments row by row in an arbitrary order unless you add a tiebreaker. In the sample data, Frank Miller and Grace Lee (Engineering, 2018-01-10) show the difference.

**A32.**
```sql
SELECT order_id, customer_id, order_date,
       COUNT(*) OVER (
           PARTITION BY customer_id
           ORDER BY order_date
           RANGE BETWEEN INTERVAL '30 days' PRECEDING AND INTERVAL '1 day' PRECEDING) AS orders_prev_30d
FROM tbl_orders;
```
This works in PostgreSQL 11+ because `order_date` is a date column. Same-day orders are excluded by the `1 day PRECEDING` end boundary. On engines without interval `RANGE`, use a correlated subquery or a self-join on date ranges.

**A33.**
```sql
WITH ma AS (
    SELECT ticker, trade_date,
           AVG(close_price) OVER (
               PARTITION BY ticker ORDER BY trade_date
               ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS ma5,
           COUNT(*) OVER (
               PARTITION BY ticker ORDER BY trade_date
               ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS n
    FROM tbl_stock_prices
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY ticker ORDER BY ma5 DESC) AS rnk
    FROM ma
    WHERE n = 5
)
SELECT ticker, trade_date, ma5
FROM ranked
WHERE rnk = 1;
```
Filtering on `n = 5` excludes the early partial windows.

## Section E: Gaps, Islands, and Sessionization

**A34.**
```sql
WITH d AS (
    SELECT DISTINCT user_id, login_date FROM tbl_logins
),
grp AS (
    SELECT user_id, login_date,
           login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)) * INTERVAL '1 day' AS island
    FROM d
)
SELECT user_id, MAX(streak) AS longest_streak
FROM (
    SELECT user_id, island, COUNT(*) AS streak
    FROM grp
    GROUP BY user_id, island
) s
GROUP BY user_id;
```
The classic islands trick: subtracting the row number from a consecutive date gives a constant value for each unbroken streak. De-duplicating first is essential (user 1 logs in twice on 3 Jan in the sample data).

**A35.**
```sql
WITH d AS (
    SELECT DISTINCT user_id, login_date FROM tbl_logins
),
grp AS (
    SELECT user_id, login_date,
           login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)) * INTERVAL '1 day' AS island
    FROM d
)
SELECT user_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS streak_days
FROM grp
GROUP BY user_id, island
ORDER BY user_id, streak_start;
```
SQL Server version: `DATEADD(day, -ROW_NUMBER() OVER (...), login_date)` as the island key.

**A36.**
```sql
SELECT order_id + 1 AS gap_start,
       next_id - 1  AS gap_end
FROM (
    SELECT order_id,
           LEAD(order_id) OVER (ORDER BY order_id) AS next_id
    FROM tbl_orders
) t
WHERE next_id - order_id > 1;
```
Each row with a jump greater than 1 to the next ID defines one missing range.

**A37.**
```sql
WITH flagged AS (
    SELECT user_id, event_time,
           CASE WHEN event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time)
                     > INTERVAL '30 minutes'
                  OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
                THEN 1 ELSE 0 END AS is_new_session
    FROM tbl_sessions
),
numbered AS (
    SELECT *, SUM(is_new_session) OVER (
                  PARTITION BY user_id ORDER BY event_time
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_id
    FROM flagged
)
SELECT user_id, session_id,
       MIN(event_time) AS session_start,
       MAX(event_time) AS session_end,
       COUNT(*)        AS event_count
FROM numbered
GROUP BY user_id, session_id;
```
The pattern is flag boundaries with `LAG`, then a cumulative `SUM` of the flags to build a session number. It is one of the most common senior-level window patterns. A gap of exactly 30 minutes stays in the same session because the test is strictly greater than.

**A38.**
```sql
WITH d AS (
    SELECT DISTINCT user_id, login_date FROM tbl_logins
),
x AS (
    SELECT user_id, login_date,
           LEAD(login_date, 2) OVER (PARTITION BY user_id ORDER BY login_date) AS d2
    FROM d
)
SELECT DISTINCT user_id
FROM x
WHERE d2 = login_date + 2;
```
Because dates are distinct and sorted, if the date two rows ahead is exactly 2 days later, the three dates must be consecutive.

**A39.**
```sql
SELECT room_id, start_time, end_time
FROM (
    SELECT m.*,
           MAX(end_time) OVER (
               PARTITION BY room_id ORDER BY start_time, end_time
               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prev_max_end,
           LEAD(start_time) OVER (PARTITION BY room_id ORDER BY start_time, end_time) AS next_start
    FROM tbl_meetings m
) t
WHERE start_time < prev_max_end
   OR end_time > next_start;
```
A meeting overlaps an earlier one if it starts before the running maximum end time of earlier meetings. It overlaps a later one if it ends after the next meeting's start. Together these two tests flag both sides of each overlap. Back-to-back meetings (one ends at 10:00, the next starts at 10:00) do not count as overlapping.

**A40.**
```sql
WITH x AS (
    SELECT room_id, start_time, end_time,
           MAX(end_time) OVER (
               PARTITION BY room_id ORDER BY start_time, end_time
               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prev_max_end
    FROM tbl_meetings
),
flagged AS (
    SELECT *,
           CASE WHEN prev_max_end IS NULL OR start_time > prev_max_end THEN 1 ELSE 0 END AS new_block
    FROM x
),
blocks AS (
    SELECT *, SUM(new_block) OVER (
                  PARTITION BY room_id ORDER BY start_time, end_time
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS block_id
    FROM flagged
)
SELECT room_id, block_id,
       MIN(start_time) AS busy_start,
       MAX(end_time)   AS busy_end
FROM blocks
GROUP BY room_id, block_id;
```
A new block starts only when the current meeting starts after everything before it has ended. Use `>=` instead of `>` if back-to-back meetings should merge. As written, back-to-back meetings merge, because a start equal to the previous end is not greater than it.

**A41.**
```sql
WITH x AS (
    SELECT ticker, trade_date, close_price,
           CASE WHEN close_price > LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date)
                THEN 0 ELSE 1 END AS reset
    FROM tbl_stock_prices
),
g AS (
    SELECT *, SUM(reset) OVER (
                  PARTITION BY ticker ORDER BY trade_date
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
    FROM x
)
SELECT ticker, grp,
       MIN(trade_date) AS run_start,
       MAX(trade_date) AS run_end,
       COUNT(*)        AS run_length
FROM g
GROUP BY ticker, grp
HAVING COUNT(*) > 1;
```
Each non-rising day starts a new group. The first day of every group is a non-rising day, so the run includes it as its base. Subtract 1 from `run_length` if you want the number of rising days only.

## Section F: Advanced and Scenario Problems

**A42.**
```sql
WITH r AS (
    SELECT department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*)     OVER (PARTITION BY department)                 AS cnt
    FROM tbl_employees
)
SELECT department, AVG(salary * 1.0) AS median_salary
FROM r
WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)
GROUP BY department;
```
Odd count: both expressions equal the middle row. Even count: they equal the two middle rows. PostgreSQL also offers `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)`, which is simpler when allowed.

**A43.**
```sql
SELECT *
FROM (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, DATE_TRUNC('month', order_date)
               ORDER BY order_date, order_id) AS rn
    FROM tbl_orders o
) t
WHERE rn = 1;
```

**A44.**
```sql
WITH first_order AS (
    SELECT customer_id,
           DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM tbl_orders
    GROUP BY customer_id
),
activity AS (
    SELECT DISTINCT o.customer_id,
           f.cohort_month,
           (EXTRACT(YEAR FROM DATE_TRUNC('month', o.order_date)) * 12
            + EXTRACT(MONTH FROM DATE_TRUNC('month', o.order_date)))
         - (EXTRACT(YEAR FROM f.cohort_month) * 12
            + EXTRACT(MONTH FROM f.cohort_month)) AS month_offset
    FROM tbl_orders o
    JOIN first_order f ON f.customer_id = o.customer_id
)
SELECT cohort_month,
       COUNT(DISTINCT CASE WHEN month_offset = 0 THEN customer_id END) AS m0,
       COUNT(DISTINCT CASE WHEN month_offset = 1 THEN customer_id END) AS m1,
       COUNT(DISTINCT CASE WHEN month_offset = 2 THEN customer_id END) AS m2,
       COUNT(DISTINCT CASE WHEN month_offset = 3 THEN customer_id END) AS m3
FROM activity
GROUP BY cohort_month
ORDER BY cohort_month;
```
The window-function version of the cohort step replaces `MIN(order_date)` in a CTE with `MIN(order_date) OVER (PARTITION BY customer_id)`, avoiding a separate aggregation and join.

**A45.**
```sql
WITH g AS (
    SELECT sensor_id, ts, value,
           COUNT(value) OVER (
               PARTITION BY sensor_id ORDER BY ts
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
    FROM tbl_readings
)
SELECT sensor_id, ts, value,
       MAX(value) OVER (PARTITION BY sensor_id, grp) AS filled_value
FROM g;
```
`COUNT(value)` ignores NULLs, so it increments only on non-NULL rows. Each non-NULL value and the NULLs that follow it share one group number, and `MAX` over the group returns that non-NULL value. Leading NULLs (sensor 2's first row in the sample data) stay NULL because there is nothing earlier to fill from. Oracle, Snowflake, and others offer `LAST_VALUE(value IGNORE NULLS)` as the shorter form.

**A46.**
```sql
WITH RECURSIVE h AS (
    SELECT emp_id, emp_name, department, salary, manager_id, 1 AS depth
    FROM tbl_employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.emp_name, e.department, e.salary, e.manager_id, h.depth + 1
    FROM tbl_employees e
    JOIN h ON e.manager_id = h.emp_id
)
SELECT emp_id, emp_name, depth, salary,
       DENSE_RANK() OVER (PARTITION BY depth ORDER BY salary DESC) AS salary_rank_at_depth
FROM h;
```
This combines a recursive CTE with a ranking window function. SQL Server omits the `RECURSIVE` keyword.

**A47.**
```sql
SELECT DISTINCT department,
       FIRST_VALUE(emp_name) OVER (
           PARTITION BY department ORDER BY hire_date ASC, emp_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS longest_tenure,
       FIRST_VALUE(emp_name) OVER (
           PARTITION BY department ORDER BY hire_date DESC, emp_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS shortest_tenure
FROM tbl_employees;
```
Two `FIRST_VALUE` calls with opposite sort orders avoid the `LAST_VALUE` frame pitfall.

**A48.**
```sql
WITH monthly_customer AS (
    SELECT DATE_TRUNC('month', order_date) AS month,
           customer_id,
           SUM(amount) AS revenue
    FROM tbl_orders
    WHERE status = 'COMPLETED'
    GROUP BY 1, 2
),
ranked AS (
    SELECT *,
           SUM(revenue) OVER (PARTITION BY month) AS month_total,
           ROW_NUMBER() OVER (PARTITION BY month ORDER BY revenue DESC, customer_id) AS rn
    FROM monthly_customer
)
SELECT month, customer_id, revenue, month_total
FROM ranked
WHERE rn <= 3
  AND month_total > 10000;
```
Two window functions with the same `PARTITION BY` run in one pass. The filters on `rn` and `month_total` both sit in the outer `WHERE`. In the sample data, four months exceed 10,000: March 2023, November 2023, February 2024, and May 2024.

**A49.**
```sql
SELECT order_id, customer_id, order_date, amount,
       SUM(amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date
           RANGE BETWEEN INTERVAL '89 days' PRECEDING AND CURRENT ROW) AS rolling_90d
FROM tbl_orders
WHERE status = 'COMPLETED';
```
`RANGE` with an interval windows by date, not by row count. It requires PostgreSQL 11+ or Oracle. In SQL Server or MySQL, use a correlated subquery or a `CROSS APPLY` / lateral join:

```sql
SELECT o.order_id, o.customer_id, o.order_date, o.amount,
       (SELECT SUM(i.amount)
        FROM tbl_orders i
        WHERE i.customer_id = o.customer_id
          AND i.status = 'COMPLETED'
          AND i.order_date BETWEEN o.order_date - INTERVAL '89 days' AND o.order_date) AS rolling_90d
FROM tbl_orders o
WHERE o.status = 'COMPLETED';
```
Note that `RANGE` includes all rows sharing the current `order_date`, so same-day orders see each other's amounts.

**A50.**

Diagnosis steps:
1. **Read the execution plan** (`EXPLAIN (ANALYZE, BUFFERS)` in PostgreSQL, actual plan in SQL Server). Look for a `Sort` or `WindowAgg` node spilling to disk, and for a large `Sort` below the window operator.
2. **Check the number of distinct windows.** Each different `PARTITION BY` / `ORDER BY` combination triggers its own sort. Group functions that share the same window spec, or use a named `WINDOW` clause so the optimizer can reuse one sort.
3. **Filter early.** Push `WHERE` conditions into the inner query or CTE so fewer rows enter the window. A window function can only run after filtering, so reduce rows first.
4. **Index to avoid the sort.** Create an index matching `PARTITION BY` columns followed by `ORDER BY` columns, with the output columns included:

```sql
-- PostgreSQL
CREATE INDEX ix_tbl_orders_cust_date
    ON tbl_orders (customer_id, order_date, order_id)
    INCLUDE (amount);

-- SQL Server
CREATE NONCLUSTERED INDEX ix_tbl_orders_cust_date
    ON tbl_orders (customer_id, order_date, order_id)
    INCLUDE (amount);
```
   With this index the engine reads rows already ordered by partition and order key, which removes the sort. In SQL Server, a columnstore index with batch mode on rowstore (compat level 150+) can also speed up window aggregates significantly.
5. **Prefer `ROWS` over `RANGE`.** `RANGE` (and the default frame) can use a slower on-disk spool in SQL Server. An explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is much faster for running totals.
6. **Reduce width.** Select only needed columns before the window step. Wide rows make sorts and spills much more expensive.
7. **Memory settings.** Raise `work_mem` (PostgreSQL) for the session, or check memory grants and spills in SQL Server, so sorts stay in memory.
8. **Partition-level parallelism.** Partition the table by the window's `PARTITION BY` key (for example, hash or range partitioning) so each partition can be processed in parallel and partitions can be pruned.
9. **Pre-aggregate or materialize.** If the same running totals are queried often, compute them incrementally into a summary table or materialized view instead of recalculating over 500 million rows each time.
10. **Avoid repeating the window in many CTE layers.** Each layer can force a re-sort. Compute everything you need in as few passes as possible.
